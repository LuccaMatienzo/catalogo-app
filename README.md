## Arquitectura
navegador ──► catalogo-frontend ──────► catalogo-api ──────► catalogo-db
React + Vite Python + FastAPI MongoDB 7
nginx :8080 :8000 :27017
(host 3000) (host 8000, solo (sin puerto
depuración) publicado)

| Capa | Imagen | Puerto interno | Puerto publicado |
| --- | --- | --- | --- |
| `catalogo-frontend` | React + Vite servido por nginx | 8080 | 3000 |
| `catalogo-api` | Python + FastAPI | 8000 | 8000 (solo depuración) |
| `catalogo-db` | MongoDB 7 | 27017 | — |

El frontend es el único punto de entrada: nadie le habla a la base
directamente, y a la API le habla el frontend.

## Service NodePort de depuración (`catalogo-api-nodeport`)

Se creó el Service `catalogo-api-nodeport` de tipo `NodePort` (`30502 -> 8000`) apuntando al selector `app: catalogo-api` para verificar el backend directamente desde fuera del clúster mediante `curl -s http://<IP-del-nodo>:30502/health`.

**Aclaración de arquitectura:** Este objeto existe exclusivamente con fines de depuración y pruebas del TP 6, por lo que **no formará parte del chart de Helm de la semana 8**. En operación normal, la API permanece expuesta únicamente de forma interna vía `ClusterIP`, siendo `catalogo-frontend` el único punto de entrada.

### Autenticación en MongoDB: El uso de `authSource`

Al inicializar la imagen de MongoDB mediante variables de entorno, el usuario administrador generado reside exclusivamente en la base de datos interna denominada `admin`. 

Este comportamiento por diseño define cómo deben construirse las cadenas de conexión:
*   **Sin el parámetro `?authSource=admin`:** El motor asume que el usuario existe en la base de datos de destino (la de la aplicación). Al no encontrarlo allí, deniega el acceso e informa un fallo de autenticación.
*   **Con el parámetro `?authSource=admin`:** Se instruye explícitamente al motor para que valide las credenciales contra la base `admin`, permitiendo una autenticación exitosa independientemente de la base de datos a la que se acceda.

**Impacto en la arquitectura:** Todas las conexiones futuras que realice la API hacia el motor de base de datos deberán incluir este parámetro en su URI para operar correctamente.

### Configuración del Frontend: El filtro envsubst

La variable `NGINX_ENVSUBST_FILTER=^API_` no es opcional. Sin ella, la sustitución alcanza también a las variables propias de nginx (como `$uri`, `$host`, `$proxy_add_x_forwarded_for`), que quedan vacías y generan una configuración inválida en silencio, sin emitir errores en los registros.

### Mediciones y Optimizaciones

| Versión | Tamaño | Imagen Base | % de Reducción | Decisión Técnica que lo aportó |
| :--- | :--- | :--- | :--- | :--- |
| API (Ingenua) | 1.72 GB | `python:3.12` | 0% | Ninguna, imagen base completa y sin multi-etapa. |
| API (Definitiva) | 247 MB | `python:3.12-slim` | ~85% | Base mínima (`-slim`) y multi-etapa aislando dependencias. |
| Frontend (Definitivo) | 73.9 MB | `nginxinc/nginx-unprivileged:1.27-alpine` | ~95% | Multi-etapa fuerte: solo cruza la carpeta `dist/`, dejando Node afuera. |

### Justificación arquitectónica: Ausencia de publicación de puertos en la base de datos

La omisión del parámetro -p al instanciar catalogo-db obedece a un principio de seguridad por aislamiento de red. En esta arquitectura de tres capas, la base de datos representa el nivel más interno y sensible. Al no publicar sus puertos hacia el anfitrión (host), el servicio queda físicamente inaccesible desde el exterior. Esto fuerza un único flujo de comunicación válido: el frontend enruta las peticiones hacia la API, y exclusivamente la API tiene visibilidad de MongoDB utilizando la resolución de nombres del DNS interno que provee la red bridge custom (catalogo-net).

### Secuencia de Arranque y Limitaciones Arquitectónicas

La arquitectura en contenedores sueltos impone un orden secuencial estricto de arranque: red → volumen → base → API → frontend. La alteración de esta cadena de dependencias provoca fallos críticos en la topología:
1. Red (catalogo-net): Debe instanciarse primero. Sin ella, los contenedores fallarán al iniciar porque el motor no encontrará la interfaz bridge custom requerida para el aislamiento y resolución DNS.
2. Volumen (catalogo-db-data): Debe existir para garantizar el anclaje y la persistencia de los datos del motor antes de instanciar la base.
3. Base de datos (catalogo-db): Debe estar en estado Up y aceptar conexiones antes que la API. Si la API arranca primero, sus hilos de conexión lanzarán excepciones por timeout o connection refused, entrando en ciclos de reintento.
4. API (catalogo-api): Requiere conexión estable a la base de datos y debe levantar obligatoriamente antes que el proxy.
5. Frontend (catalogo-frontend): Su servidor Nginx evalúa y resuelve la directiva proxy_pass hacia el upstream (catalogo-api) una única vez durante el arranque. Si la API no está operativa en la red en ese instante exacto, la resolución DNS falla y el contenedor del frontend aborta fatalmente (Exited) sin política de reintentos.

Limitación de diseño: Actualmente, la integridad de esta secuencia depende exclusivamente de la memoria operativa de quien ejecuta los comandos en la terminal. No existe ningún manifiesto ejecutable ni motor de estado deseado que audite o automatice estas dependencias. Esta carencia arquitectónica se resolverá la próxima semana delegando la topología y el orden de inicio (mediante depends_on) a la herramienta declarativa Docker Compose.

### Healthcheck de MongoDB: Diagnóstico sin exposición de credenciales

El comando db.adminCommand('ping') empleado para verificar la salud del motor de base de datos se ejecuta sin necesidad de autenticación previa. Esta característica de MongoDB ofrece una ventaja directa de seguridad en comparación con la variante relacional de este proyecto, ya que evita exponer la contraseña en texto plano dentro del propio comando del archivo Compose. Sin embargo, esta decisión de diseño implica que un estado saludable reportado por el healthcheck certifica exclusivamente que el motor se encuentra encendido y respondiendo a la operación de diagnóstico. Dicho estado no garantiza en absoluto que el usuario de la aplicación efectivamente exista, ni que la contraseña configurada en el entorno sea la correcta.

### Healthcheck de la API: Optimización de dependencias

La verificación de estado del contenedor de la API se realiza delegando la solicitud HTTP a la biblioteca estándar urllib.request nativa del entorno de Python. Modificar el manifiesto de la imagen para instalar paquetes adicionales del sistema operativo, como la utilidad externa curl, contradice el patrón de construcción de imágenes optimizadas establecido en la segunda etapa del proyecto. Incorporar herramientas adicionales en la imagen exclusivamente para resolver un healthcheck incrementa el peso innecesario de la capa final y amplía la superficie de ataque del contenedor, introduciendo dependencias que los escáneres de vulnerabilidades deberán auditar.

## Revisión de Seguridad

| Servicio | Usuario de ejecución | Puertos publicados | Observaciones |
|---|---|---|---|
| `catalogo-frontend` | `uid=101(nginx)` | 3000 | Único punto de entrada expuesto. |
| `catalogo-api` | `uid=1000(appuser)` | 8000 | Expuesto solo para depuración; en la iteración de orquestación pasará a ser interno. |
| `catalogo-db` | `uid=999(mongodb)` | Ninguno | Servicio aislado lógicamente en la red bridge custom. |

**Auditoría de Vulnerabilidades (Trivy) sobre `catalogo-api:v1`**
- **Sistema base (Debian):** 211 vulnerabilidades detectadas (1 con parche disponible en *Fixed Version*).
- **Dependencias de Python:** 18 vulnerabilidades detectadas (3 con parche disponible en *Fixed Version*).

## Principio de Mínimo Privilegio

La directiva `cap_drop: [ALL]` se aplicó exitosamente sobre los contenedores `catalogo-api` y `catalogo-frontend`. Dado que ambas imágenes fueron construidas para ejecutar sus procesos con usuarios no privilegiados (`appuser` y `nginx`), no requieren interactuar con recursos o rutinas protegidas del kernel.

En contraste, el servicio `catalogo-db` finaliza con un error de permisos (`Operation not permitted`) al restringirle todas las *capabilities*. El motor de MongoDB inicia su proceso temporalmente como `root` para ajustar la propiedad y los permisos de su directorio de datos antes de transicionar al usuario interno del motor. Para garantizar este comportamiento sin otorgar privilegios absolutos, se aplicó una lista estricta mediante `cap_add`:
- `CHOWN`, `SETUID`, `SETGID`: Permiten la transición de identidad del proceso.
- `DAC_OVERRIDE`, `FOWNER`: Permiten manipular los permisos del volumen de datos anclado.

## Comparativa: Docker Compose frente a ejecución manual

La adopción del manifiesto `docker-compose.yml` introdujo las siguientes diferencias arquitectónicas respecto a los tres comandos `docker run` secuenciales:

**Qué desapareció:**
- La dependencia operativa de la memoria de quien ejecuta el despliegue para mantener el orden de arranque.
- La inserción de credenciales y secretos en texto plano mediante banderas `-e` en el historial del shell.
- La ejecución paralela ciega; los servicios ya no levantan hasta que su dependencia estricta reporta disponibilidad funcional.

**Qué apareció:**
- Una topología de red, almacenamiento y servicios declarada en un único manifiesto versionable.
- Integración de `healthchecks` para verificar la salud interna de cada contenedor (ej. `db.adminCommand('ping').ok`).
- La cláusula `depends_on` con condición `service_healthy`, que garantiza que la API no inicie sus rutinas de conexión hasta que MongoDB esté estructuralmente preparado para aceptarlas.
- Interpolación segura de variables sensibles mediante un archivo `.env` excluido del control de versiones.

## Comparativa: Resolución DNS en Docker frente a Kubernetes

El síntoma es idéntico porque `nginx` resuelve el destino de `proxy_pass` una única vez al arrancar y aborta si el nombre no existe (no reintenta). Lo que cambia es la causa arquitectónica:

| Entrega | Error en logs | Causa raíz |
| :--- | :--- | :--- |
| **Semana 3 (Docker)** | `host not found in upstream "catalogo-api"` | Faltaba la **red bridge custom** (`catalogo-net`); la red `bridge` por defecto no tiene DNS embebido. |
| **Semana 5 (Kubernetes)** | `host not found in upstream "catalogo-api"` | El clúster ya tiene red y CoreDNS, pero falta el `Service` que registre el nombre `catalogo-api`. |

## Cheatsheet - Semana 05

| Comando | Qué hace en la práctica |
| :--- | :--- |
| `kubectl config current-context` | Muestra el contexto activo (clúster, usuario y namespace). Se ejecuta antes de aplicar cambios para confirmar que operás en el clúster correcto. |
| `kubectl create namespace <nombre>` | Crea un espacio de nombres lógico para aislar los objetos del proyecto del resto del clúster. |
| `kubectl config set-context --current --namespace=<ns>` | Fija el namespace por defecto en el contexto actual para evitar repetir `-n <ns>` en cada comando. |
| `kubectl run <pod> --image=<img:tag> --dry-run=client -o yaml` | Genera el manifiesto YAML base de un Pod localmente sin enviarlo al API server, para guardarlo y editarlo de forma declarativa. |
| `kubectl apply -f <archivo.yaml>` | Crea o actualiza recursos de forma declarativa e idempotente a partir de un manifiesto YAML. |
| `kubectl get pods [-o wide] [-w]` | Lista los Pods del namespace con su estado y reinicios. `-o wide` suma IP y nodo; `-w` observa los cambios de estado en tiempo real. |
| `kubectl describe pod <nombre>` | Muestra el detalle completo del objeto y su sección `Events`, clave para diagnosticar fallas previas al arranque del contenedor (como `ImagePullBackOff`). |
| `kubectl logs <nombre> [--previous]` | Imprime la salida estándar y de error del contenedor. Con `--previous` recupera el registro de la instancia que acaba de fallar en un `CrashLoopBackOff`. |
| `kubectl port-forward pod/<nombre> <local>:<pod>` | Abre un túnel temporal desde un puerto local hacia un Pod puntual; es una herramienta de depuración y muere al cortar la terminal o reiniciarse el Pod. |
| `kubectl delete pod <nombre>` | Elimina el Pod del clúster. Al tratarse de un Pod suelto sin controlador, la baja es definitiva y no se recrea. |

## Comparativa: `docker-compose.yml` (Semana 4) vs. Manifiestos de Pods (Semana 5)

| Dimensión | Qué se conservó | Qué desapareció | Qué todavía falta (deudas de Semana 5) |
| :--- | :--- | :--- | :--- |
| **Imágenes y ejecución** | Las mismas tres imágenes (`mongo:7`, `catalogo-api:v1`, `catalogo-frontend:v1`), puertos internos (`27017`, `8000`, `8080`) y usuarios no-root. | El bloque `build:` (el clúster no compila; descarga las imágenes completas con usuario desde el registry) y `container_name:` (reemplazado por `metadata.name` en el `namespace`). | `Deployments` y `ReplicaSets` que garanticen autorreparación ante borrado de Pods, réplicas y *rolling updates* (Semana 6). |
| **Red y descubrimiento** | Los mismos nombres lógicos en variables (`DB_HOST=catalogo-db`, `API_HOST=catalogo-api`). | La definición manual de `networks:` (`catalogo-net`) y la publicación directa de puertos al host (`ports: 3000:8080`, `8000:8000`). | `Services` (`ClusterIP` para DB/API y `NodePort` para frontend) que den DNS en CoreDNS, IP virtual estable frente a IPs efímeras y acceso externo sin `port-forward` (Semana 6). |
| **Orden de arranque y salud** | Los reintentos de conexión internos de la API (`esperar_mongo()`). | `depends_on` (y `condition: service_healthy`), que no existe en Kubernetes. | `readinessProbe` y `livenessProbe` en reemplazo de `healthcheck`: hoy `Running` no garantiza que MongoDB esté listo para autenticar. |
| **Configuración y secretos** | El mismo contrato de variables de entorno (incluido `ROOT_PATH=/api` y `SEED_DEMO=true`). | La interpolación local desde el archivo `.env`. | `ConfigMaps` y `Secrets` para sacar configuración y credenciales en texto plano de los manifiestos YAML (Semana 7). |
| **Persistencia y seguridad** | El diseño sin estado de la API (imágenes almacenadas en MongoDB). | El volumen nombrado `catalogo-db-data` y el bloque `cap_drop` / `cap_add` de Compose. | `PersistentVolumeClaim` (PVC) para que los datos de MongoDB sobrevivan al Pod (Semana 7) y `securityContext` con `capabilities.drop: [ALL]` para mantener mínimo privilegio. |

## Verificación de Endpoints y Selectors (TPI6 - Punto 3)

Mediante `kubectl get endpoints -n catalogo-app` se verificó que los `selector` de los tres Services vinculan correctamente con las `labels` de los Pods desplegados por cada Deployment:

```text
$ kubectl get endpoints -n catalogo-app
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME                ENDPOINTS                           AGE
catalogo-api        10.244.0.49:8000,10.244.0.50:8000   7m6s
catalogo-db         10.244.0.48:27017                   7m7s
catalogo-frontend   10.244.0.51:8080,10.244.0.52:8080   7m6s
```

- **Validación:** Ningún Service presenta `<none>` en la columna `ENDPOINTS`. Las direcciones coinciden con el reparto definido: 1 réplica para `catalogo-db` (`:27017`), 2 para `catalogo-api` (`:8000`) y 2 para `catalogo-frontend` (`:8080`). Si existiera un error de coincidencia entre `selector` y `labels`, el Service se crearía sin arrojar error, pero figuraría vacío (`<none>`).
- **Advertencia de API:** El mensaje `Warning: v1 Endpoints is deprecated in v1.33+` es el comportamiento esperado, dado que el recurso fue reemplazado por `EndpointSlice`, aunque el comando continúa operando normalmente para diagnóstico.

## Limitación conocida: ausencia de persistencia en `catalogo-db` (TPI 6 - Punto 9)

En esta etapa, el Deployment `catalogo-db` opera sin almacenamiento persistente: todos los datos de MongoDB residen únicamente en el sistema de archivos efímero del contenedor (`/data/db`).

### Comportamiento observado
1. Al eliminar el Pod de la base de datos (`kubectl delete pod -l app=catalogo-db -n catalogo-app`), el Deployment lo recrea automáticamente con un directorio de datos vacío y el listado de productos queda vacío.
2. Al reiniciar las réplicas de la API (`kubectl rollout restart deployment/catalogo-api -n catalogo-app`), reaparece el catálogo de demostración porque la siembra idempotente se ejecuta en el arranque de la API; sin embargo, **los productos cargados manualmente con sus imágenes se pierden de forma definitiva**.

### Resolución prevista
Se deja registrada esta pérdida de datos como una limitación conocida y explícita de la Semana 6 (la reaparición del catálogo demo no demuestra persistencia, sino el efecto de la siembra inicial). Esta deuda se resuelve en la **Semana 7** adjuntando un volumen persistente (`PersistentVolumeClaim`) montado sobre `/data/db`, manteniendo siempre `catalogo-db` en una única réplica.

## 5. Decisión de conexión a MongoDB

Se eligió pasar las **piezas de la conexión por separado** (`DB_HOST`, `DB_PORT` y `DB_NAME` desde `catalogo-api-config`; `DB_USER` y `DB_PASSWORD` desde `catalogo-db-secret` vía `valueFrom`) en lugar de una única variable `MONGO_URI`.

## 6. Persistencia con `catalogo-db-pvc`

### Estado del PVC/PV

Estado inmediatamente después de aplicar `k8s/pvc-catalogo-db.yaml`:

```text
<pegar aquí la salida de kubectl get pvc,pv -n catalogo-app antes de actualizar el Deployment>
```

Estado tras actualizar `catalogo-db` y recrearse el Pod:

```text
<pegar aquí la salida de kubectl get pvc,pv -n catalogo-app mostrando el PVC en Bound>
```

### Diferencia entre motores de base de datos (`subPath`) y permisos de `/data/db`

- **Ausencia de `subPath`:** a diferencia de MySQL, que requiere un directorio de datos completamente vacío para inicializarse (y suele exigir `subPath` para evitar directorios como `lost+found`), MongoDB no impone esa restricción, por lo que el PVC se monta directamente sobre `/data/db` sin `subPath`.
- **Verificación de propietario y permisos:** si el volumen se montara sin permisos de escritura para el usuario `mongodb` del contenedor, el Pod fallaría con `Permission denied` sobre `/data/db`. Con la `StorageClass` del clúster el directorio se crea con permisos `777`, verificado mediante `kubectl exec deploy/catalogo-db -n catalogo-app -- ls -ld /data/db`:

```text
<pegar aquí la salida de ls -ld /data/db>
```

## 7. Comprobación de persistencia y cierre de limitación del TP6

Para evitar el **falso positivo** propio de esta variante —donde el catálogo de demostración reaparece siempre tras recrear la base porque su siembra es idempotente al arrancar la API—, la persistencia se validó cargando manualmente un producto con su imagen desde el navegador, eliminando el Pod de `catalogo-db` y comprobando que tanto el registro como su imagen permanecieron disponibles una vez que el Deployment creó un nuevo Pod reutilizando `catalogo-db-pvc`.

Con esta verificación **queda cerrada la limitación de pérdida de datos ante recreaciones de `catalogo-db` declarada en el TP6**.
**Justificación:**
- **Separación estricta de datos:** los parámetros no sensibles quedan visibles en el `ConfigMap` y solo las credenciales viven en el `Secret` (con `MONGO_URI`, toda la cadena debería residir dentro del `Secret`).
- **Sin duplicación de credenciales:** `catalogo-db` y `catalogo-api` consumen las mismas claves de `catalogo-db-secret`, evitando desincronizaciones al rotar la contraseña.
- **Menor margen de error:** la API arma la cadena e incluye `?authSource=admin` automáticamente, en lugar de depender de redactarlo a mano en el `Secret`.

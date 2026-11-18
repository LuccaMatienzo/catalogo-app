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

# Arquitectura — `catalogo-app` (Variante C)

## 1. Preguntas obligatorias del curso

### a) Configuración del frontend con `API_HOST` en tiempo de ejecución frente a `VITE_API_URL` en tiempo de construcción

Vite es un empaquetador estático: cualquier variable `VITE_*` se reemplaza literalmente dentro de los archivos JavaScript generados durante `npm run build` (etapa de construcción de la imagen). Si la dirección de la API se fijara con `VITE_API_URL`, la URL quedaría horneada de forma inmutable en el artefacto.

Para evitarlo, la SPA consume siempre la ruta relativa `/api`, y el contenedor Nginx de `catalogo-frontend` inyecta `API_HOST` y `API_PORT` en tiempo de arranque mediante `envsubst` sobre la plantilla de configuración (`/etc/nginx/templates/default.conf.template`). Esto desacopla el binario del entorno y respeta el principio de construir una única imagen inmutable que se promueve sin cambios entre entornos.

**Qué habría que rehacer si fuera al revés:** para pasar de desarrollo local (Docker Compose) a producción (Kubernetes o distintos namespaces/clústeres) habría que recompilar el bundle de Vite, reconstruir la imagen Docker con otro valor de `VITE_API_URL` y publicar una imagen diferente por cada entorno, rompiendo la trazabilidad de lo probado en staging frente a lo desplegado en producción.

### b) Por qué esta arquitectura no necesita CORS y qué cambiaría con comunicación directa

CORS (*Cross-Origin Resource Sharing*) es una política de seguridad aplicada exclusivamente por el navegador cuando código JavaScript intenta hacer peticiones a un origen distinto (diferente esquema, host o puerto) del que sirvió la página.

En `catalogo-app`, el navegador solo se comunica con un único origen (por ejemplo `http://catalogo.local` a través del Ingress o `http://<IP-nodo>:30302` vía NodePort). Las peticiones a `/api/*` llegan al mismo servidor Nginx del frontend, que actúa como proxy inverso y reenvía el tráfico internamente por red del clúster hacia `catalogo-api:8000`. Como esa segunda comunicación es servidor-a-servidor, no interviene el navegador y CORS no aplica.

**Qué habría que agregar si la SPA hablara directamente con la API:**
1. Exponer públicamente el servicio `catalogo-api` bajo otro origen (otro subdominio o puerto).
2. Configurar `CORSMiddleware` en FastAPI declarando explícitamente los orígenes permitidos (`allow_origins`), métodos (`GET`, `POST`, `PUT`, `DELETE`, `OPTIONS`) y cabeceras (`allow_headers`, como `Content-Type` o futuras cabeceras de autorización) para responder correctamente a las peticiones *preflight* (`OPTIONS`).
3. Parametrizar la lista de orígenes permitidos por entorno mediante configuración externa (ConfigMap) para evitar configuraciones inseguras (`*`) en producción.

### c) Por qué existe un único `HorizontalPodAutoscaler` entre los doce objetos

El HPA (`catalogo-api-hpa`) se aplica únicamente a `catalogo-api` porque es la capa sin estado (*stateless*) que concentra el trabajo variable de CPU: serialización/deserialización de peticiones REST y lectura/entrega de los bytes de imágenes desde MongoDB. El HPA calcula el porcentaje de utilización contra `resources.requests.cpu` (`100m` por Pod), no contra `limits` ni contra la capacidad del nodo.

**Qué ocurriría si se asignara un HPA a `catalogo-db`:**
- Con el diseño actual (`Deployment` + un único `PersistentVolumeClaim` en modo `ReadWriteOnce`), intentar escalar `catalogo-db` a más de 1 réplica provocaría fallos de montaje (*Multi-Attach*) si el scheduler ubica los Pods en nodos distintos, o conflictos de bloqueo y corrupción de datos sobre el mismo `/data/db` (`mongod.lock`) si coincidieran en el mismo nodo.
- Si cada réplica tuviera un volumen independiente sin una topología de replicación coordinada, se generaría *split-brain* (bases divergentes donde el tráfico balanceado por el Service leería y escribiría conjuntos de datos distintos). Escalar una base de datos requiere mecanismos propios del motor (Replica Set / sharding) sobre un `StatefulSet`, no un HPA sobre un Deployment.

**Por qué tampoco lo tiene `catalogo-frontend`:**
`catalogo-frontend` ejecuta Nginx sirviendo archivos estáticos livianos y haciendo proxy de `/api`. Es una carga orientada a E/S con consumo de CPU despreciable (medido en `0m` bajo operación normal con `kubectl top pods`). En esta arquitectura el cuello de botella aparece antes en la API y en la base de datos que en Nginx, por lo que mantener 2 réplicas fijas cubre redundancia básica sin agregar complejidad de autoescalado innecesaria.

---

## 2. Pregunta propia de la Variante C: qué deja de estar garantizado al pasar de 1 a 3 réplicas de la API

Al escalar `catalogo-api` de una a tres réplicas detrás del Service `catalogo-api`, **deja de estar garantizada la consistencia y permanencia de cualquier estado almacenado localmente en el Pod** (tanto en su sistema de archivos como en su memoria RAM):

1. **Aislamiento entre réplicas:** cada Pod posee su propia capa de escritura efímera. Un archivo escrito en el filesystem local de la réplica 1 (por ejemplo `/tmp/foto.bin`) no existe en las réplicas 2 y 3. Como `kube-proxy` distribuye las peticiones entrantes entre las réplicas activas, un cliente que sube una imagen en una réplica recibiría `404` cuando una petición posterior cayera en cualquiera de las otras dos.
2. **Efimeridad ante recreación:** cuando un Pod se elimina, falla o se actualiza en un rollout, su filesystem se destruye por completo; el Pod reemplazo nace limpio a partir de la imagen base.

Por este motivo, en la Variante C los bytes de las imágenes del catálogo se almacenan dentro del documento en MongoDB y no en el disco del contenedor. Al externalizar el 100% del estado persistente hacia la base de datos, `catalogo-api` es completamente *stateless*: cualquier réplica es intercambiable y puede crearse, escalarse o destruirse sin pérdida de datos.

**Corolario sobre sesiones en memoria:** si la aplicación incorporara un carrito de compras guardado en la sesión en memoria del servidor (RAM del proceso Python/FastAPI), tampoco sobreviviría al escalado a 2 o 3 réplicas ni a un reinicio del Pod: cada petición enrutada a otra réplica perdería el carrito. Resolver eso exigiría un almacén de sesiones externo y compartido (como Redis o una colección en base de datos) o estado autocontenido en el cliente/token.

La verificación empírica de aislamiento y pérdida del archivo en `/tmp/foto.bin` entre réplicas de `catalogo-api` se encuentra documentada en la **Sección 5 de `INFORME.md`**.

---

## 3. Dificultades propias de MongoDB y del despliegue en el curso

### Dificultad 1: Siembra idempotente ejecutada al arranque de la API (colección vacía `[]` y falso positivo de persistencia)
- **Cómo se manifestó:** ocurrió en dos instancias concretas. En la sexta entrega, al eliminar el Pod de `catalogo-db` (sin PVC), el listado quedó vacío y al reiniciar `catalogo-api` reaparecieron los productos demo pero desapareció el producto cargado manualmente. En la séptima entrega, al montar `catalogo-db-pvc` nuevo sobre `/data/db`, el Ingress devolvió HTTP `200 OK` con `[]` en `http://catalogo.local/api/productos` porque los Pods de `catalogo-api` ya estaban corriendo desde antes de inicializar el volumen vacío.
- **Cómo se diagnosticó:** con `curl -si http://catalogo.local/api/productos` se confirmó `200 OK` y `Content-Type: application/json` con respuesta `[]` (descartando fallos de Ingress, proxy o autenticación), y se contrastó con la creación reciente del volumen en `kubectl get pvc -n catalogo-app` y el comportamiento de `SEED_DEMO` al iniciar FastAPI.
- **Cómo se resolvió:** se ejecutó `kubectl rollout restart deploy/catalogo-api -n catalogo-app` para disparar la siembra idempotente sobre la base recién montada, y se estableció como única prueba válida de persistencia la supervivencia de un producto cargado manualmente con su imagen (`Samsung Galaxy S26`, SKU `7777`) tras eliminar el Pod de `catalogo-db`, evitando el falso positivo de la reaparición del catálogo demo.

### Dificultad 2: Parámetro de autenticación (`?authSource=admin`) y valores por defecto silenciosos de credenciales en la API
- **Cómo se manifestó:** la imagen oficial `mongo:7` crea el usuario administrador definido por `MONGO_INITDB_ROOT_USERNAME` y `MONGO_INITDB_ROOT_PASSWORD` dentro de la base lógica `admin`. Si la API se conecta mediante una `MONGO_URI` sin `?authSource=admin`, MongoDB rechaza la conexión con `Authentication failed` aunque las credenciales coincidan. Adicionalmente, si `DB_USER` o `DB_PASSWORD` no llegan por un error en el manifiesto, la API aplica valores por defecto idénticos a los de Compose y responde normalmente, generando un falso positivo de configuración.
- **Cómo se diagnosticó:** en lugar de dar por válida la configuración solo porque la aplicación respondía, se inspeccionaron directamente las variables de entorno inyectadas dentro del contenedor mediante `kubectl exec deploy/catalogo-api -n catalogo-app -- env` (verificando `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER` y `ROOT_PATH`) y comprobando la presencia de `DB_PASSWORD` con `test -n "$DB_PASSWORD"` sin imprimirla en claro.
- **Cómo se resolvió:** se optó por inyectar las piezas de conexión por separado (`DB_HOST`, `DB_PORT`, `DB_NAME` desde `catalogo-api-config` vía `envFrom`; `DB_USER` y `DB_PASSWORD` desde `catalogo-db-secret` vía `valueFrom.secretKeyRef`). Así se mantiene una única fuente de verdad para las credenciales sin duplicarlas en el Secret y la API compone la cadena agregando `?authSource=admin` de forma automática.

### Dificultad 3: Prefijo de ruta (`ROOT_PATH=/api`) para la documentación interactiva detrás del proxy e Ingress
- **Cómo se manifestó:** detrás de Nginx (`/api` -> `catalogo-api:8000`) y del Ingress (`catalogo.local/`), si FastAPI desconoce el prefijo `/api`, genera las referencias internas apuntando a `/openapi.json` en la raíz. Al abrir `http://catalogo.local/api/docs`, el navegador solicita `/openapi.json`, Nginx aplica la directiva `try_files` de la SPA devolviendo HTTP `200` con `index.html`, y Swagger UI falla silenciosamente dejando la página en blanco sin errores en los logs de la API.
- **Cómo se diagnosticó:** inspeccionando `ROOT_PATH` en el entorno del Pod y validando por CLI la URL embebida en la vista de documentación (`curl -s http://catalogo.local/api/docs | grep -i openapi`) junto con el código de respuesta de `http://catalogo.local/api/openapi.json`.
- **Cómo se resolvió:** externalizando `ROOT_PATH: "/api"` en `catalogo-api-config` (y parametrizándolo en `values.yaml` como `api.rootPath`) e inyectándolo por `envFrom`, confirmando luego `url: '/api/openapi.json'` y código `200` junto al renderizado completo de Swagger UI.

---

## 4. Qué quedaría pendiente para un entorno de producción real

1. **Observabilidad integral:** incorporar recolección de métricas de negocio e infraestructura (Prometheus + Grafana con alertas sobre latencia, tasa de errores 5xx, saturación de CPU/memoria y espacio del PVC), agregación centralizada de logs estructurados (Loki/ELK) y trazabilidad distribuida (OpenTelemetry), además de `readinessProbe` y `livenessProbe` en todos los Deployments.
2. **Integración y despliegue continuos (CI/CD):** automatizar pruebas unitarias/integración, construcción y escaneo de vulnerabilidades de imágenes (ej. Trivy), publicación con tags inmutables basados en SHA de commit o SemVer y despliegue declarativo/GitOps (Argo CD o Flux) en lugar de comandos manuales `helm upgrade`.
3. **Gestión externa de secretos:** los manifiestos de este repositorio contienen credenciales de laboratorio en `Secret` y `values.yaml` (Base64 es codificación, no cifrado). En producción se requiere cifrado en reposo en `etcd` (*EncryptionConfiguration* respaldado por KMS), políticas RBAC de mínimo privilegio y un gestor externo (HashiCorp Vault o un Secret Manager cloud integrado con External Secrets Operator o CSI Secrets Store Driver), sin almacenar secretos en Git.
4. **Copias de respaldo (backups) y recuperación ante desastres:** programar respaldos automáticos consistentes (`mongodump` o snapshots del volumen a nivel almacenamiento) enviados fuera del clúster (almacenamiento de objetos con versionado y cifrado), con política de retención y pruebas periódicas de restauración (RPO/RTO definidos).
5. **Replica Set de MongoDB (por qué no sirve un `Deployment` con un PVC en producción):**
   Desplegar MongoDB como un `Deployment` con un único PVC `ReadWriteOnce` y `strategy: Recreate` es aceptable solo en laboratorio, pero inviable en un entorno productivo por cuatro razones técnicas:
   - **Punto único de fallo (SPOF):** existe una sola réplica y un solo volumen; la caída del nodo o del disco deja sin servicio a toda la aplicación.
   - **Downtime obligatorio en cada actualización o reinicio:** al usar `Recreate` para evitar conflictos de montaje sobre `ReadWriteOnce`, la base debe detenerse por completo antes de iniciar el Pod reemplazo.
   - **Falta de identidad y almacenamiento dedicado por miembro:** un `Deployment` trata a sus Pods como idénticos e intercambiables compartiendo el mismo template de volumen. Un Replica Set de MongoDB (mínimo 3 nodos: primario + secundarios para quórum y elección automática) exige identidad de red DNS estable por miembro (`db-0`, `db-1`, `db-2`), orden de arranque/apagado controlado y un PVC independiente por cada miembro (`volumeClaimTemplates`).
   - **Operación de día 2:** en producción corresponde usar un servicio gestionado (MongoDB Atlas / equivalente cloud) o un `StatefulSet` administrado por un Operador de Kubernetes específico de MongoDB que gestione failover automático, TLS entre nodos, rotación de certificados y backups.

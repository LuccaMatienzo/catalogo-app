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

### Configuración del Frontend: El filtro envsubst (Punto 6)

La variable `NGINX_ENVSUBST_FILTER=^API_` no es opcional. Sin ella, la sustitución alcanza también a las variables propias de nginx (como `$uri`, `$host`, `$proxy_add_x_forwarded_for`), que quedan vacías y generan una configuración inválida en silencio, sin emitir errores en los registros.

### Mediciones y Optimizaciones (Punto 8)

| Versión | Tamaño | Imagen Base | % de Reducción | Decisión Técnica que lo aportó |
| :--- | :--- | :--- | :--- | :--- |
| API (Ingenua) | 1.72 GB | `python:3.12` | 0% | Ninguna, imagen base completa y sin multi-etapa. |
| API (Definitiva) | 247 MB | `python:3.12-slim` | ~85% | Base mínima (`-slim`) y multi-etapa aislando dependencias. |
| Frontend (Definitivo) | 73.9 MB | `nginxinc/nginx-unprivileged:1.27-alpine` | ~95% | Multi-etapa fuerte: solo cruza la carpeta `dist/`, dejando Node afuera. |

### Justificación arquitectónica: Ausencia de publicación de puertos en la base de datos

La omisión del parámetro -p al instanciar catalogo-db obedece a un principio de seguridad por aislamiento de red. En esta arquitectura de tres capas, la base de datos representa el nivel más interno y sensible. Al no publicar sus puertos hacia el anfitrión (host), el servicio queda físicamente inaccesible desde el exterior. Esto fuerza un único flujo de comunicación válido: el frontend enruta las peticiones hacia la API, y exclusivamente la API tiene visibilidad de MongoDB utilizando la resolución de nombres del DNS interno que provee la red bridge custom (catalogo-net).

### Secuencia de Arranque y Limitaciones Arquitectónicas (Punto 9)

La arquitectura en contenedores sueltos impone un orden secuencial estricto de arranque: red → volumen → base → API → frontend. La alteración de esta cadena de dependencias provoca fallos críticos en la topología:
1. Red (catalogo-net): Debe instanciarse primero. Sin ella, los contenedores fallarán al iniciar porque el motor no encontrará la interfaz bridge custom requerida para el aislamiento y resolución DNS.
2. Volumen (catalogo-db-data): Debe existir para garantizar el anclaje y la persistencia de los datos del motor antes de instanciar la base.
3. Base de datos (catalogo-db): Debe estar en estado Up y aceptar conexiones antes que la API. Si la API arranca primero, sus hilos de conexión lanzarán excepciones por timeout o connection refused, entrando en ciclos de reintento.
4. API (catalogo-api): Requiere conexión estable a la base de datos y debe levantar obligatoriamente antes que el proxy.
5. Frontend (catalogo-frontend): Su servidor Nginx evalúa y resuelve la directiva proxy_pass hacia el upstream (catalogo-api) una única vez durante el arranque. Si la API no está operativa en la red en ese instante exacto, la resolución DNS falla y el contenedor del frontend aborta fatalmente (Exited) sin política de reintentos.

Limitación de diseño: Actualmente, la integridad de esta secuencia depende exclusivamente de la memoria operativa de quien ejecuta los comandos en la terminal. No existe ningún manifiesto ejecutable ni motor de estado deseado que audite o automatice estas dependencias. Esta carencia arquitectónica se resolverá la próxima semana delegando la topología y el orden de inicio (mediante depends_on) a la herramienta declarativa Docker Compose.

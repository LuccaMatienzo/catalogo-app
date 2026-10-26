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

Al inicializar la imagen de MongoDB mediante variables de entorno, el usuario administrador generado reside exclusivamente en la base de datos interna denominada `admin`[cite: 3]. 

Este comportamiento por diseño define cómo deben construirse las cadenas de conexión:
*   **Sin el parámetro `?authSource=admin`:** El motor asume que el usuario existe en la base de datos de destino (la de la aplicación). Al no encontrarlo allí, deniega el acceso e informa un fallo de autenticación[cite: 3].
*   **Con el parámetro `?authSource=admin`:** Se instruye explícitamente al motor para que valide las credenciales contra la base `admin`, permitiendo una autenticación exitosa independientemente de la base de datos a la que se acceda[cite: 3].

**Impacto en la arquitectura:** Todas las conexiones futuras que realice la API hacia el motor de base de datos deberán incluir este parámetro en su URI para operar correctamente[cite: 3].

### Configuración del Frontend: El filtro envsubst (Punto 6)

La variable `NGINX_ENVSUBST_FILTER=^API_` no es opcional[cite: 1]. Sin ella, la sustitución alcanza también a las variables propias de nginx (como `$uri`, `$host`, `$proxy_add_x_forwarded_for`), que quedan vacías y generan una configuración inválida en silencio, sin emitir errores en los registros[cite: 1, 171].

### Mediciones y Optimizaciones (Punto 8)

| Versión | Tamaño | Imagen Base | % de Reducción | Decisión Técnica que lo aportó |
| :--- | :--- | :--- | :--- | :--- |
| API (Ingenua) | 1.72 GB | `python:3.12` | 0% | Ninguna, imagen base completa y sin multi-etapa[cite: 4]. |
| API (Definitiva) | 247 MB | `python:3.12-slim` | ~85% | Base mínima (`-slim`) y multi-etapa aislando dependencias[cite: 4]. |
| Frontend (Definitivo) | 73.9 MB | `nginxinc/nginx-unprivileged:1.27-alpine` | ~95% | Multi-etapa fuerte: solo cruza la carpeta `dist/`, dejando Node afuera[cite: 4]. |

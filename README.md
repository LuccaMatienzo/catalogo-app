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

Al inicializar la imagen de MongoDB mediante variables de entorno, el usuario administrador generado reside exclusivamente en la base de datos interna denominada `admin`[cite: 13]. 

Este comportamiento por diseño define cómo deben construirse las cadenas de conexión:
*   **Sin el parámetro `?authSource=admin`:** El motor asume que el usuario existe en la base de datos de destino (la de la aplicación). Al no encontrarlo allí, deniega el acceso e informa un fallo de autenticación[cite: 13].
*   **Con el parámetro `?authSource=admin`:** Se instruye explícitamente al motor para que valide las credenciales contra la base `admin`, permitiendo una autenticación exitosa independientemente de la base de datos a la que se acceda[cite: 13].

**Impacto en la arquitectura:** Todas las conexiones futuras que realice la API hacia el motor de base de datos deberán incluir este parámetro en su URI para operar correctamente[cite: 13].

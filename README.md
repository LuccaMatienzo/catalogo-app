# `catalogo-app` — Variante C (Proyecto Integrador)

Catálogo de productos con imágenes compuesto por tres capas: **Frontend** (React + Vite servido por Nginx como proxy inverso), **API** (Python + FastAPI) y **Base de datos** (MongoDB 7).

```text
navegador ──► catalogo-frontend ──────► catalogo-api ──────► catalogo-db
              React + Vite / nginx      Python + FastAPI     MongoDB 7
              :8080 (Compose: 3000)     :8000                :27017 (solo interna)
```

| Capa | Imagen | Puerto interno | Exposición |
| :--- | :--- | :---: | :--- |
| `catalogo-frontend` | React + Vite servido por Nginx unprivileged | 8080 | Compose: `3000` / K8s: Ingress `:80` y NodePort `30302` |
| `catalogo-api` | Python 3.12 + FastAPI | 8000 | Compose: `8000` (depuración) / K8s: `ClusterIP` (interna) |
| `catalogo-db` | MongoDB 7 | 27017 | Sin publicación externa (`ClusterIP` / red interna) |

## Estructura del repositorio

- `backend/` — Código fuente y `Dockerfile` multietapa de `catalogo-api`.
- `frontend/` — Código fuente y `Dockerfile` multietapa de `catalogo-frontend` con configuración de Nginx en tiempo de ejecución.
- `docker-compose.yml` y `.env.example` — Despliegue completo en una sola máquina.
- `k8s/` — Los 12 manifiestos individuales de Kubernetes (`3 Deployments`, `3 Services`, `2 ConfigMaps`, `1 Secret`, `1 PVC`, `1 Ingress`, `1 HPA`).
- `catalogo-app/` — Chart de Helm parametrizado (`Chart.yaml`, `values.yaml` y los 12 objetos en `templates/`).
- `ARQUITECTURA.md` — Respuestas técnicas y decisiones de arquitectura del proyecto.
- `INFORME.md` e `informe/` — Evidencia verificable del despliegue y funcionamiento.

> **Aviso de seguridad sobre credenciales:** `k8s/secret-catalogo-db.yaml` y `catalogo-app/values.yaml` contienen únicamente credenciales de laboratorio (`catalogo_user` / `catalogo_pass`) para reproducir el entorno académico. En `ARQUITECTURA.md` (Sección 4) se detalla su gestión en producción mediante cifrado en reposo en `etcd` o gestores externos de secretos.

---

## Camino 1: Despliegue en una sola máquina con Docker Compose

### Prerrequisitos
- Docker Engine y plugin Docker Compose v2.

### Pasos
1. Clonar el repositorio y preparar las variables de entorno a partir de la plantilla:
   ```bash
   git clone [https://github.com/LuccaMatienzo/catalogo-app.git](https://github.com/LuccaMatienzo/catalogo-app.git)
   cd catalogo-app
   cp .env.example .env
   ```
2. Construir las imágenes y levantar las tres capas (con red privada `catalogo-net`, volumen persistente `catalogo-db-data` y `healthchecks`):
   ```bash
   docker compose up --build -d
   ```
3. Verificar estado y salud de los contenedores:
   ```bash
   docker compose ps
   ```
4. Acceder a la aplicación en `http://localhost:3000` y a la documentación interactiva en `http://localhost:3000/api/docs`.
5. Para detener el entorno (agregar `-v` solo si se desea eliminar también el volumen de datos):
   ```bash
   docker compose down
   ```

---

## Camino 2: Despliegue en Kubernetes con Helm

### Prerrequisitos
- Clúster Kubernetes operativo con `StorageClass` por defecto, `nginx-ingress-controller` (en namespace `nginx-ingress`) y `metrics-server` en ejecución.
- `kubectl` y `helm` configurados contra el clúster.

### Pasos
1. Validar sintaxis del chart y comprobar que genera exactamente los 12 objetos:
   ```bash
   helm lint ./catalogo-app
   helm template catalogo-app ./catalogo-app -n catalogo-app | grep -c "^kind:"
   ```
2. Instalar la release creando el namespace `catalogo-app`:
   ```bash
   helm install catalogo-app ./catalogo-app -n catalogo-app --create-namespace
   ```
3. Verificar que los recursos estén operativos (`Running`, PVC en `Bound`, Ingress con `ADDRESS` y HPA con métrica numérica):
   ```bash
   kubectl get all,cm,secret,pvc,ingress,hpa -n catalogo-app
   ```
4. Configurar la resolución local del dominio `catalogo.local` apuntando a la IP externa del Ingress Controller:
   ```bash
   INGRESS_IP=$(kubectl get svc -n nginx-ingress nginx-ingress-controller \
     -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   echo "$INGRESS_IP catalogo.local" | sudo tee -a /etc/hosts
   ```
5. Abrir la aplicación en `http://catalogo.local` y la documentación interactiva en `http://catalogo.local/api/docs`.

### Actualización y reversión de versión de la API
```bash
# Actualizar únicamente la API a v2 (sin alterar el frontend)
helm upgrade catalogo-app ./catalogo-app -n catalogo-app \
  --reuse-values --set api.image.tag=v2

# Consultar historial y revertir a la revisión 1
helm history catalogo-app -n catalogo-app
helm rollback catalogo-app 1 -n catalogo-app
```

### Desinstalación
```bash
helm uninstall catalogo-app -n catalogo-app
kubectl delete namespace catalogo-app
```

---

## Inventario de los 12 objetos de Kubernetes (`catalogo-app`)

| # | Objeto | Tipo | Función |
| :---: | :--- | :--- | :--- |
| 1 | `catalogo-db` | Deployment | Gestiona la réplica de MongoDB 7 (`strategy: Recreate`) con PVC y credenciales desde `Secret`. |
| 2 | `catalogo-api` | Deployment | Gestiona las réplicas *stateless* de FastAPI con `resources.requests/limits`, `ConfigMap` y `Secret`. |
| 3 | `catalogo-frontend` | Deployment | Gestiona las réplicas de Nginx que sirven la SPA y actúan como proxy inverso de `/api`. |
| 4 | `catalogo-db` | Service (ClusterIP) | Expone internamente la base de datos por DNS en el puerto `27017`. |
| 5 | `catalogo-api` | Service (ClusterIP) | Expone y balancea internamente las réplicas de la API en el puerto `8000`. |
| 6 | `catalogo-frontend` | Service (NodePort) | Expone el frontend en el puerto interno `8080` y en el `30302` del nodo. |
| 7 | `catalogo-api-config` | ConfigMap | Externaliza `DB_HOST`, `DB_PORT`, `DB_NAME`, `ROOT_PATH` y `SEED_DEMO`. |
| 8 | `catalogo-frontend-config` | ConfigMap | Externaliza `API_HOST` y `API_PORT` consumidos por `envsubst` en Nginx. |
| 9 | `catalogo-db-secret` | Secret | Almacena usuario y contraseña root de MongoDB compartidos por `catalogo-db` y `catalogo-api`. |
| 10 | `catalogo-db-pvc` | PersistentVolumeClaim | Solicita `1Gi` `ReadWriteOnce` para persistir `/data/db` ante recreaciones del Pod. |
| 11 | `catalogo-app-ingress` | Ingress | Enruta tráfico HTTP de `catalogo.local/` hacia `catalogo-frontend:8080` por puerto `80`. |
| 12 | `catalogo-api-hpa` | HorizontalPodAutoscaler | Escala `catalogo-api` entre 2 y 6 réplicas según utilización de CPU (objetivo `50%`). |

---

## Notas técnicas y decisiones acumuladas durante el curso

### Optimización de imágenes y seguridad (Entregas 2–4)

| Versión | Tamaño | Imagen Base | % de Reducción | Decisión técnica |
| :--- | :--- | :--- | :--- | :--- |
| API (Ingenua) | 1.72 GB | `python:3.12` | 0% | Imagen base completa sin construcción multietapa. |
| API (Definitiva) | 247 MB | `python:3.12-slim` | ~85% | Base `-slim`, multietapa aislando dependencias y usuario no-root (`appuser`, UID 1000). |
| Frontend (Definitivo) | 73.9 MB | `nginxinc/nginx-unprivileged:1.27-alpine` | ~95% | Multietapa copiando solo `dist/` (sin Node) y ejecución no-root (`nginx`, UID 101). |

- **Filtro `NGINX_ENVSUBST_FILTER=^API_`:** evita que `envsubst` vacíe variables nativas de Nginx (`$uri`, `$host`, `$proxy_add_x_forwarded_for`) al arrancar.
- **Principio de mínimo privilegio en Compose:** `cap_drop: [ALL]` en los tres servicios; en `catalogo-db` se agregan únicamente `CHOWN`, `SETUID`, `SETGID`, `DAC_OVERRIDE` y `FOWNER` para la transición inicial hacia `uid=999(mongodb)`.
- **Healthchecks:** `db.adminCommand('ping').ok` en MongoDB (evita exponer credenciales en Compose, sabiendo que solo certifica proceso activo y no autenticación) y `urllib.request` nativo en Python (evita instalar `curl` en la imagen de la API).
- **Auditoría Trivy sobre `catalogo-api:v1`:** 211 vulnerabilidades en paquetes Debian (1 con parche disponible) y 18 en dependencias Python (3 con parche disponible).

### Decisiones en Kubernetes y Helm (Entregas 5–8)
- **Conexión API -> MongoDB (piezas separadas):** se inyectan `DB_HOST`, `DB_PORT` y `DB_NAME` desde `catalogo-api-config` (`envFrom`) y `DB_USER`/`DB_PASSWORD` desde `catalogo-db-secret` (`valueFrom.secretKeyRef`). Evita duplicar credenciales en el Secret, separa configuración no sensible de secretos y delega en la API el agregado automático de `?authSource=admin`.
- **Persistencia (`catalogo-db-pvc`):** con la `StorageClass` `local-path` (`WaitForFirstConsumer`), el PVC inicia en `Pending` hasta planificarse el Pod de MongoDB y pasa a `Bound` (`1Gi`, `RWO`). A diferencia de MySQL, MongoDB no exige directorio vacío, por lo que no se usa `subPath`; se verificó `drwxrwxrwx 5 mongodb root /data/db`. La persistencia se demostró con un producto manual + imagen sobreviviendo al borrado del Pod (cerrando la limitación declarada en el TP6).
- **Ingress con regla única `/`:** no se duplicó la regla `/api` porque Nginx en `catalogo-frontend` ya ejecuta el proxy inverso, manteniendo paridad con Compose. Se comprobó que `/api/productos` devuelve JSON `200`, mientras `/productos` es un falso positivo SPA (`200` con `index.html` por `try_files`).
- **Ciclos de vida independientes en Helm:** `helm upgrade --reuse-values --set api.image.tag=v2` actualiza únicamente `catalogo-api` manteniendo `catalogo-frontend` en `v1`.

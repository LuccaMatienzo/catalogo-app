# ----------------------------------------
# Etapa 1: Builder (Fábrica)
# ----------------------------------------
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ----------------------------------------
# Etapa 2: Final (Servidor Nginx)
# ----------------------------------------
FROM nginxinc/nginx-unprivileged:1.27-alpine
WORKDIR /usr/share/nginx/html

# (Punto 6) Configuración dinámica al arrancar
ENV API_HOST=catalogo-api \
    API_PORT=8000 \
    NGINX_ENVSUBST_FILTER=^API_

# Copiamos SOLO la carpeta dist/ generada en la etapa anterior
COPY --from=builder /app/dist /usr/share/nginx/html
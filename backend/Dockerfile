# ----------------------------------------
# Etapa 1: Builder
# ----------------------------------------
FROM python:3.12-slim AS builder

WORKDIR /app

# (b) Copiar manifiesto antes que el código para aislar la caché de instalación
COPY requirements.txt .

# (a) Instalar dependencias en /root/.local sin guardar caché del gestor
RUN pip install --user --no-cache-dir -r requirements.txt

# ----------------------------------------
# Etapa 2: Final
# ----------------------------------------
FROM python:3.12-slim

# (e) Evitar que Python almacene la salida estándar en el búfer
ENV PYTHONUNBUFFERED=1

# (f) Hornear versión observable
ARG APP_VERSION=v1
ENV APP_VERSION=${APP_VERSION}

# (d) Crear usuario no privilegiado explícitamente
RUN useradd --create-home appuser

# (c) Variables de entorno para ubicar los ejecutables instalados (como uvicorn)
ENV PATH=/home/appuser/.local/bin:$PATH

WORKDIR /app

# (c) Copiar solo el resultado del builder (librerías y binarios)
COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local

# (b) Copiar el código fuente
COPY --chown=appuser:appuser . .

# (d) Cambiar al usuario no privilegiado
USER appuser

# (g) Documentar el puerto y arrancar el servidor
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
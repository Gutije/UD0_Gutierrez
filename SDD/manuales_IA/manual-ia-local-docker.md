---
title: "Manual de instalación, configuración y operación: pila de IA local en Docker sobre Ubuntu Server"
author: "Gutije"
date: "2026-10-06"
category: "Despliegue IA"
tags: [markdown, ia, docker, nvidia, ollama, openwebui, hermes, opencode, comfyui, yolo, searxng, rag]
version: "1.0"
---

# Manual de instalación, configuración y operación: pila de IA local en Docker

Este manual se genera a partir de la especificación SDD `proyecto_ia.md` (v1.0). Está escrito paso a paso para un administrador de sistemas novato. Todos los comandos se ejecutan en el servidor Ubuntu, con un usuario normal que tenga `sudo`, salvo que se indique otra cosa.

---

## 0. Notas de diseño y desviaciones respecto a la especificación

Al convertir la especificación en ficheros ejecutables aparecieron incoherencias. Estas son las decisiones tomadas; léelas antes de empezar.

| # | Punto de la especificación | Problema | Decisión adoptada |
| :-- | :--- | :--- | :--- |
| 1 | **RAG** usa puerto 11434 (igual que Ollama) | Incumple el criterio 3 (sin colisiones de puertos): dos contenedores no pueden publicar el mismo puerto del host | El contenedor `rag` es **Qdrant** (base de datos vectorial) en el puerto **6333**. Open WebUI lo usa como almacén de vectores y Ollama genera los embeddings |
| 2 | URL de OpenCode `http://opencode:8443` | 8443 es el puerto **externo** (host). Dentro de `red-ia` el contenedor escucha en 8080 | URL interna correcta: `http://opencode:8080`. Desde fuera: `https://IP:8443` (ver nota de HTTP/HTTPS en 3.7) |
| 3 | Hermes Agent: contenedor `hermes-agent` pero URL `http://hermesagent:8000` | El nombre DNS de la red es el del contenedor | Se usa `http://hermes-agent:8000` y se añade el alias `hermesagent` para que ambas funcionen |
| 4 | Ficheros `docker-<servicio>.yml` (sección 5) frente a `docker_<servicio>.yml` (criterio 1) | Guion frente a guion bajo | Se usa **guion**, como dice la sección 5 |
| 5 | Criterio 2: "todos los contenedores usan GPU" | SearXNG, Qdrant y el gateway de Hermes no ejecutan cálculo CUDA; reservarles GPU no aporta nada | Usan GPU: **Ollama, Open WebUI (imagen `:cuda`), ComfyUI y YOLO**. Los otros no la necesitan |
| 6 | SearXNG "depende de GPU driver" | No la usa | Se elimina esa dependencia |
| 7 | GPU "RTX 3060 de 12 GB" | 12 GB de VRAM se **comparten** entre Ollama, ComfyUI, YOLO y Open WebUI | Ver sección 0.1 |
| 8 | Hermes en puerto 8000 | Su puerto por defecto de API es 8642 | Se fuerza 8000 con variables de entorno |

> **Verificar al desplegar:** Hermes Agent, OpenCode y los servidores MCP evolucionan rápido. Las imágenes y comandos de este manual se comprobaron contra su documentación en la fecha indicada en la cabecera, pero si algún comando falla, consulta la documentación oficial del proyecto correspondiente (enlaces en la sección 7).

### 0.1. Reparto de la VRAM (12 GB)

| Servicio | VRAM aproximada | Comentario |
| :--- | :--- | :--- |
| Ollama (modelo 7-8B, cuantización Q4) | 5-7 GB (más con contextos largos) | Se libera tras `OLLAMA_KEEP_ALIVE` |
| ComfyUI (SDXL) | 6-9 GB | Se queda ocupada mientras el proceso vive |
| YOLO (modelo `n` o `s`) | 0,5-1 GB | Ligero |
| Open WebUI `:cuda` | ~0,5 GB | Solo si usa embeddings/voz locales; aquí los embeddings van por Ollama |

**Regla práctica:** no uses un LLM grande y ComfyUI a la vez. Ollama está configurado para descargar el modelo de la VRAM a los 2 minutos de inactividad (`OLLAMA_KEEP_ALIVE=2m`), lo que deja sitio a ComfyUI.

---

## 1. Prerrequisitos e instalación base

### 1.1. Actualizar el sistema

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release git jq ubuntu-drivers-common
```

### 1.2. Driver NVIDIA (repositorios apt de Ubuntu)

Con Ubuntu Server basta el **driver**. El toolkit CUDA completo vive dentro de las imágenes de los contenedores, no hace falta en el host.

```bash
# Ver qué driver recomienda Ubuntu para tu tarjeta
ubuntu-drivers devices

# Instalar el recomendado para servidores/cómputo
sudo ubuntu-drivers install --gpgpu

# Reiniciar para cargar el módulo del kernel
sudo reboot
```

Tras el reinicio, comprueba:

```bash
nvidia-smi
```

Debe mostrar tu tarjeta (p. ej. RTX 3060, 12 GB), la versión del driver y la versión CUDA soportada. Si dice `command not found` o `NVIDIA-SMI has failed`, el driver no se cargó: revisa `sudo dmesg | grep -i nvidia` y que Secure Boot esté desactivado (o firma el módulo).

*(Opcional)* Si quieres `nvcc` en el host para compilar cosas fuera de Docker: `sudo apt install -y nvidia-cuda-toolkit`. No es necesario para esta pila.

### 1.3. Docker Engine y Docker Compose (repositorio oficial apt)

```bash
# Clave y repositorio de Docker
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Usar docker sin sudo (cierra sesión y vuelve a entrar después)
sudo usermod -aG docker "$USER"
```

> En Ubuntu 26.04, si `apt update` da error 404 en el repositorio de Docker (puede tardar en publicarse para una versión nueva), sustituye temporalmente `$(...)` por `noble` (24.04) en la línea del repositorio.

Verifica (ya sin `sudo`, tras volver a entrar en la sesión):

```bash
docker --version
docker compose version
docker run --rm hello-world
```

### 1.4. NVIDIA Container Toolkit (permite que Docker use la GPU)

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# Registrar el runtime en Docker y reiniciarlo
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

**Prueba de fuego** (si esto funciona, la GPU ya es visible desde contenedores):

```bash
docker run --rm --gpus all nvidia/cuda:12.6.3-base-ubuntu24.04 nvidia-smi
```

Debe imprimir la misma tabla que `nvidia-smi` del host.

---

## 2. Estructura del proyecto

Los **ficheros de configuración y construcción** viven en `$HOME/proyecto`. Los **datos persistentes** viven en `$HOME/<servicio>` (volúmenes de la especificación).

```
$HOME/
├── proyecto/                         # configuración del stack (versionable con git)
│   ├── .env                          # variables de todos los servicios
│   ├── ia.sh                         # arrancar/parar/actualizar todo
│   ├── backup.sh                     # copias de seguridad
│   ├── docker-ollama.yml
│   ├── docker-rag.yml
│   ├── docker-searxng.yml
│   ├── docker-comfyui.yml
│   ├── docker-yolo.yml
│   ├── docker-hermes-agent.yml
│   ├── docker-openwebui.yml
│   ├── docker-opencode.yml
│   ├── build/
│   │   ├── comfyui/Dockerfile
│   │   ├── opencode/Dockerfile
│   │   └── yolo/{Dockerfile, app.py}
│   └── backups/                      # copias generadas por backup.sh
│
├── ollama/                           # volumen ollama_data  → /root/.ollama
├── openwebui/                        # volumen openwebui_data → /app/backend/data
├── hermes/                           # configuración de agentes → /opt/data
├── opencode/
│   ├── config/opencode.json          # configuración de OpenCode
│   ├── data/                         # sesiones y credenciales de OpenCode
│   └── projects/                     # código de los proyectos
├── comfyui/
│   ├── models/{checkpoints,vae,loras,controlnet,clip,unet,upscale_models,embeddings}
│   ├── input/  output/  user/  custom_nodes/
├── yolo/                             # datasets, modelos y resultados
│   ├── models/  datasets/  results/
├── searxng/settings.yml              # configuración de SearXNG
└── rag/
    ├── docs/                         # documentos privados para ampliar el conocimiento
    └── qdrant/                       # almacenamiento de la base vectorial
```

Crea toda la estructura de una vez:

```bash
mkdir -p "$HOME"/proyecto/{build/comfyui,build/opencode,build/yolo,backups}
mkdir -p "$HOME"/{ollama,openwebui,hermes,searxng}
mkdir -p "$HOME"/opencode/{config,data,projects}
mkdir -p "$HOME"/comfyui/models/{checkpoints,vae,loras,controlnet,clip,unet,upscale_models,embeddings}
mkdir -p "$HOME"/comfyui/{input,output,user,custom_nodes}
mkdir -p "$HOME"/yolo/{models,datasets,results}
mkdir -p "$HOME"/rag/{docs,qdrant}
```

Crear las carpetas **tú mismo** es importante: si no existen, Docker las crea como `root` y habrá problemas de permisos (ver sección 6.4).

### 2.1. Red Docker `red-ia`

Todos los contenedores se unen a una red `bridge` propia. En una red bridge definida por el usuario, Docker proporciona **DNS interno**: cada contenedor es accesible por su nombre (`http://ollama:11434`).

```bash
docker network create --driver bridge red-ia
docker network ls | grep red-ia
```

Tabla DNS resultante (puerto **interno**, el que usan los contenedores entre sí):

| Servicio | Contenedor | URL interna (dentro de `red-ia`) | URL desde el host/LAN |
| :--- | :--- | :--- | :--- |
| Ollama | `ollama` | `http://ollama:11434` | `http://IP:11434` |
| Open WebUI | `openwebui` | `http://openwebui:8080` | `http://IP:3000` |
| Hermes Agent | `hermes-agent` (alias `hermesagent`) | `http://hermes-agent:8000` | `http://IP:8000` |
| OpenCode | `opencode` | `http://opencode:8080` | `http://IP:8443` |
| ComfyUI | `comfyui` | `http://comfyui:8188` | `http://IP:8188` |
| YOLO | `yolo` | `http://yolo:5000` | `http://IP:5000` |
| SearXNG | `searxng` | `http://searxng:8080` | `http://IP:8080` |
| RAG (Qdrant) | `rag` | `http://rag:6333` | `http://IP:6333` |

### 2.2. Mapa de puertos del host (sin colisiones)

| Puerto host | Servicio |
| :--- | :--- |
| 3000 | Open WebUI |
| 5000 | YOLO |
| 6333 | RAG (Qdrant) |
| 8000 | Hermes Agent |
| 8080 | SearXNG |
| 8188 | ComfyUI |
| 8443 | OpenCode |
| 11434 | Ollama |

Todos distintos. Comprueba antes de desplegar que el host no tiene ya nada en esos puertos:

```bash
ss -tulpn | grep -E ':(3000|5000|6333|8000|8080|8188|8443|11434)\b' || echo "Puertos libres"
```

---

## 3. Ficheros de configuración

Todos los ficheros se crean dentro de `$HOME/proyecto`:

```bash
cd "$HOME/proyecto"
```

Cada fichero `docker-<servicio>.yml` lleva `name: ia-<servicio>` para que Docker Compose los trate como proyectos independientes (así puedes parar o actualizar uno sin tocar los demás). Todos usan la red externa `red-ia`, creada en 2.1, y leen las variables del `.env` (sección 4).

### 3.1. `docker-ollama.yml`

```bash
cat > docker-ollama.yml <<'EOF'
name: ia-ollama

services:
  ollama:
    image: ${OLLAMA_IMAGE:-ollama/ollama:latest}
    container_name: ollama
    restart: unless-stopped
    ports:
      - "${OLLAMA_PORT:-11434}:11434"
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_KEEP_ALIVE=${OLLAMA_KEEP_ALIVE:-2m}
      - OLLAMA_MAX_LOADED_MODELS=${OLLAMA_MAX_LOADED_MODELS:-1}
      - OLLAMA_NUM_PARALLEL=${OLLAMA_NUM_PARALLEL:-1}
      - OLLAMA_CONTEXT_LENGTH=${OLLAMA_CONTEXT_LENGTH:-16384}
      - OLLAMA_FLASH_ATTENTION=1
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
      - TZ=${TZ:-Europe/Madrid}
    volumes:
      - ${DATA_ROOT:?Define DATA_ROOT en .env}/ollama:/root/.ollama
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "ollama", "list"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 20s

networks:
  red-ia:
    external: true
EOF
```

### 3.2. `docker-rag.yml` (Qdrant)

```bash
cat > docker-rag.yml <<'EOF'
name: ia-rag

services:
  rag:
    image: ${RAG_IMAGE:-qdrant/qdrant:latest}
    container_name: rag
    restart: unless-stopped
    ports:
      - "${RAG_PORT:-6333}:6333"
    environment:
      - TZ=${TZ:-Europe/Madrid}
    volumes:
      - ${DATA_ROOT:?Define DATA_ROOT en .env}/rag/qdrant:/qdrant/storage
    networks:
      - red-ia

networks:
  red-ia:
    external: true
EOF
```

### 3.3. `docker-searxng.yml`

Primero el fichero de configuración de SearXNG (en el volumen). Las opciones clave: activar el formato `json` (lo necesitan Open WebUI y el servidor MCP) y desactivar el *limiter* (requiere Valkey/Redis y no es útil en una red privada).

```bash
cat > "$HOME/searxng/settings.yml" <<'EOF'
use_default_settings: true

general:
  instance_name: "Buscador privado IA"

server:
  secret_key: "CAMBIAR_SECRETO"
  limiter: false
  image_proxy: true

search:
  safe_search: 0
  formats:
    - html
    - json
EOF

# Generar una clave secreta aleatoria y ajustar permisos de lectura
sed -i "s|CAMBIAR_SECRETO|$(openssl rand -hex 32)|" "$HOME/searxng/settings.yml"
chmod 644 "$HOME/searxng/settings.yml"
```

Ahora el compose:

```bash
cat > docker-searxng.yml <<'EOF'
name: ia-searxng

services:
  searxng:
    image: ${SEARXNG_IMAGE:-searxng/searxng:latest}
    container_name: searxng
    restart: unless-stopped
    ports:
      - "${SEARXNG_PORT:-8080}:8080"
    environment:
      - SEARXNG_BASE_URL=http://${HOST_IP:-localhost}:${SEARXNG_PORT:-8080}/
      - TZ=${TZ:-Europe/Madrid}
    volumes:
      - ${DATA_ROOT:?Define DATA_ROOT en .env}/searxng:/etc/searxng
    networks:
      - red-ia
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    logging:
      driver: json-file
      options:
        max-size: "1m"
        max-file: "1"

networks:
  red-ia:
    external: true
EOF
```

### 3.4. `docker-comfyui.yml` y su Dockerfile

ComfyUI no tiene imagen oficial mantenida, por lo que se construye una local sobre una imagen base CUDA de NVIDIA.

```bash
cat > build/comfyui/Dockerfile <<'EOF'
FROM nvidia/cuda:12.6.3-cudnn-runtime-ubuntu24.04

ENV DEBIAN_FRONTEND=noninteractive \
    PIP_NO_CACHE_DIR=1 \
    PYTHONUNBUFFERED=1

RUN apt-get update && apt-get install -y --no-install-recommends \
      python3 python3-venv python3-pip git ca-certificates libgl1 libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /opt
RUN git clone --depth 1 https://github.com/comfyanonymous/ComfyUI.git

WORKDIR /opt/ComfyUI
RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# PyTorch con CUDA 12.6 (compatible con RTX 30xx) + dependencias de ComfyUI
RUN pip install --upgrade pip \
 && pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu126 \
 && pip install -r requirements.txt

EXPOSE 8188
CMD ["python", "main.py", "--listen", "0.0.0.0", "--port", "8188"]
EOF
```

```bash
cat > docker-comfyui.yml <<'EOF'
name: ia-comfyui

services:
  comfyui:
    build:
      context: ./build/comfyui
    image: ia/comfyui:local
    container_name: comfyui
    restart: unless-stopped
    ports:
      - "${COMFYUI_PORT:-8188}:8188"
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
      - TZ=${TZ:-Europe/Madrid}
    volumes:
      - ${DATA_ROOT:?Define DATA_ROOT en .env}/comfyui/models:/opt/ComfyUI/models
      - ${DATA_ROOT}/comfyui/input:/opt/ComfyUI/input
      - ${DATA_ROOT}/comfyui/output:/opt/ComfyUI/output
      - ${DATA_ROOT}/comfyui/user:/opt/ComfyUI/user
      - ${DATA_ROOT}/comfyui/custom_nodes:/opt/ComfyUI/custom_nodes
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

networks:
  red-ia:
    external: true
EOF
```

> ComfyUI **arranca sin modelos**. Necesitas al menos un *checkpoint* (por ejemplo, SDXL) en `$HOME/comfyui/models/checkpoints/`. Descárgalo de un repositorio de modelos de tu elección respetando su licencia.

### 3.5. `docker-yolo.yml`, su Dockerfile y la API

Se usa la imagen oficial de Ultralytics (ya incluye PyTorch + CUDA) y se añade una API mínima con FastAPI en el puerto 5000.

```bash
cat > build/yolo/Dockerfile <<'EOF'
FROM ultralytics/ultralytics:latest

ENV PIP_BREAK_SYSTEM_PACKAGES=1 \
    PIP_NO_CACHE_DIR=1 \
    PYTHONUNBUFFERED=1

RUN pip install fastapi "uvicorn[standard]" python-multipart

WORKDIR /app
COPY app.py /app/app.py

EXPOSE 5000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "5000"]
EOF
```

```bash
cat > build/yolo/app.py <<'EOF'
import io
import os

import torch
from fastapi import FastAPI, File, UploadFile
from PIL import Image
from ultralytics import YOLO

# Los pesos se descargan la primera vez en /data/models (volumen persistente)
os.makedirs("/data/models", exist_ok=True)
os.chdir("/data/models")

MODEL_NAME = os.getenv("YOLO_MODEL", "yolo11n.pt")
CONF = float(os.getenv("YOLO_CONF", "0.25"))
DEVICE = 0 if torch.cuda.is_available() else "cpu"

model = YOLO(MODEL_NAME)
app = FastAPI(title="YOLO API")


@app.get("/health")
def health():
    return {
        "status": "ok",
        "model": MODEL_NAME,
        "cuda": torch.cuda.is_available(),
        "gpu": torch.cuda.get_device_name(0) if torch.cuda.is_available() else None,
    }


@app.post("/detect")
async def detect(file: UploadFile = File(...), conf: float = CONF):
    img = Image.open(io.BytesIO(await file.read())).convert("RGB")
    result = model.predict(img, conf=conf, device=DEVICE, verbose=False)[0]
    detections = []
    for box in result.boxes:
        detections.append({
            "class": result.names[int(box.cls)],
            "confidence": round(float(box.conf), 4),
            "bbox_xyxy": [round(float(v), 1) for v in box.xyxy[0].tolist()],
        })
    return {"count": len(detections), "detections": detections}
EOF
```

```bash
cat > docker-yolo.yml <<'EOF'
name: ia-yolo

services:
  yolo:
    build:
      context: ./build/yolo
    image: ia/yolo:local
    container_name: yolo
    restart: unless-stopped
    ports:
      - "${YOLO_PORT:-5000}:5000"
    environment:
      - YOLO_MODEL=${YOLO_MODEL:-yolo11n.pt}
      - YOLO_CONF=${YOLO_CONF:-0.25}
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
      - TZ=${TZ:-Europe/Madrid}
    volumes:
      - ${DATA_ROOT:?Define DATA_ROOT en .env}/yolo:/data
    networks:
      - red-ia
    ipc: host
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

networks:
  red-ia:
    external: true
EOF
```

### 3.6. `docker-hermes-agent.yml`

Hermes Agent (Nous Research) se publica como imagen oficial `nousresearch/hermes-agent`. El modo `gateway run` expone una API compatible con OpenAI; se fuerza al puerto 8000 de la especificación.

```bash
cat > docker-hermes-agent.yml <<'EOF'
name: ia-hermes-agent

services:
  hermes-agent:
    image: ${HERMES_IMAGE:-nousresearch/hermes-agent:latest}
    container_name: hermes-agent
    command: gateway run
    restart: unless-stopped
    shm_size: "1gb"
    ports:
      - "${HERMES_PORT:-8000}:8000"
    environment:
      - API_SERVER_ENABLED=true
      - API_SERVER_HOST=0.0.0.0
      - API_SERVER_PORT=8000
      - API_SERVER_KEY=${HERMES_API_KEY:?Define HERMES_API_KEY en .env}
      - TZ=${TZ:-Europe/Madrid}
    volumes:
      - ${DATA_ROOT:?Define DATA_ROOT en .env}/hermes:/opt/data
    networks:
      red-ia:
        aliases:
          - hermesagent

networks:
  red-ia:
    external: true
EOF
```

> **Aviso importante de la documentación de Hermes:** nunca ejecutes dos contenedores de gateway sobre el mismo directorio de datos (`$HOME/hermes`). Además, la imagen gestiona sus procesos con s6 y lee su configuración de `/opt/data/.env`, por eso en la sección 5.1 se escribe también ese fichero.

### 3.7. `docker-opencode.yml` y su Dockerfile

No hay una imagen oficial de OpenCode adecuada para servidor, así que se construye una mínima con Node.js e instalando el paquete npm `opencode-ai`, y se lanza en modo web.

```bash
cat > build/opencode/Dockerfile <<'EOF'
FROM node:22-bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
      git curl ca-certificates ripgrep openssh-client \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g opencode-ai

WORKDIR /workspace
EXPOSE 8080
CMD ["opencode", "web", "--hostname", "0.0.0.0", "--port", "8080"]
EOF
```

```bash
cat > docker-opencode.yml <<'EOF'
name: ia-opencode

services:
  opencode:
    build:
      context: ./build/opencode
    image: ia/opencode:local
    container_name: opencode
    restart: unless-stopped
    ports:
      - "${OPENCODE_PORT:-8443}:8080"
    environment:
      - OPENCODE_SERVER_USERNAME=${OPENCODE_USER:-opencode}
      - OPENCODE_SERVER_PASSWORD=${OPENCODE_PASSWORD:?Define OPENCODE_PASSWORD en .env}
      - TZ=${TZ:-Europe/Madrid}
    volumes:
      - ${DATA_ROOT:?Define DATA_ROOT en .env}/opencode/projects:/workspace
      - ${DATA_ROOT}/opencode/config:/root/.config/opencode
      - ${DATA_ROOT}/opencode/data:/root/.local/share/opencode
    networks:
      - red-ia

networks:
  red-ia:
    external: true
EOF
```

> **Sobre el puerto 8443:** por convención se asocia a HTTPS, pero OpenCode web sirve **HTTP** plano. Entra con `http://IP:8443`. Si lo vas a exponer fuera de tu LAN, ponle delante un proxy inverso con TLS (Caddy, Nginx o Traefik); queda fuera del alcance de este manual.

### 3.8. `docker-openwebui.yml`

La imagen `:cuda` aprovecha la GPU para tareas locales (voz, reranking). Las variables de integración (Ollama, SearXNG, ComfyUI, Qdrant, Hermes) vienen ya preconfiguradas.

```bash
cat > docker-openwebui.yml <<'EOF'
name: ia-openwebui

services:
  openwebui:
    image: ${OPENWEBUI_IMAGE:-ghcr.io/open-webui/open-webui:cuda}
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "${OPENWEBUI_PORT:-3000}:8080"
    environment:
      # --- Ollama ---
      - OLLAMA_BASE_URL=http://ollama:11434
      # --- Seguridad ---
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY:?Define WEBUI_SECRET_KEY en .env}
      - ENABLE_SIGNUP=${ENABLE_SIGNUP:-true}
      # --- RAG: embeddings por Ollama y vectores en Qdrant (contenedor rag) ---
      - VECTOR_DB=qdrant
      - QDRANT_URI=http://rag:6333
      - RAG_EMBEDDING_ENGINE=ollama
      - RAG_EMBEDDING_MODEL=${EMBEDDING_MODEL:-nomic-embed-text}
      - RAG_OLLAMA_BASE_URL=http://ollama:11434
      # --- Búsqueda web con SearXNG ---
      - ENABLE_WEB_SEARCH=true
      - WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>
      # --- Generación de imágenes con ComfyUI ---
      - ENABLE_IMAGE_GENERATION=true
      - IMAGE_GENERATION_ENGINE=comfyui
      - COMFYUI_BASE_URL=http://comfyui:8188
      # --- Hermes Agent como conexión compatible con OpenAI ---
      - ENABLE_OPENAI_API=true
      - OPENAI_API_BASE_URLS=http://hermes-agent:8000/v1
      - OPENAI_API_KEYS=${HERMES_API_KEY}
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
      - TZ=${TZ:-Europe/Madrid}
    volumes:
      - ${DATA_ROOT:?Define DATA_ROOT en .env}/openwebui:/app/backend/data
    networks:
      - red-ia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

networks:
  red-ia:
    external: true
EOF
```

> **Muy importante (novatos):** en Open WebUI casi todas esas variables son *PersistentConfig*: solo se leen **la primera vez** que arranca con una base de datos vacía. Después, los cambios se hacen desde la interfaz (**Panel de administración → Ajustes**) o borrando el volumen. Por eso conviene tener el `.env` correcto *antes* del primer arranque. Además, elige el almacén vectorial (`VECTOR_DB=qdrant`) antes de subir documentos: cambiarlo después obliga a reindexar.

### 3.9. Validar la sintaxis de todos los ficheros

Antes de arrancar nada (se ejecuta tras crear también el `.env` de la sección 4):

```bash
for f in docker-*.yml; do
  echo "== $f"; docker compose -f "$f" config -q && echo "OK"
done
```

Cada fichero debe devolver `OK`. Si aparece un error de YAML, revisa la indentación (siempre espacios, nunca tabuladores).

---

## 4. Fichero de entorno `.env`

Contiene las variables de **todos** los servicios. Compose lo lee automáticamente porque está en el mismo directorio que los `docker-*.yml`.

```bash
cd "$HOME/proyecto"
cat > .env <<'EOF'
# ============================================================
# Variables globales
# ============================================================
TZ=Europe/Madrid
# Carpeta base de los volúmenes de datos (se ajusta automáticamente más abajo)
DATA_ROOT=/home/usuario
# IP del servidor en tu LAN (usada por SearXNG para su URL base)
HOST_IP=localhost

# ============================================================
# Puertos publicados en el host
# ============================================================
OLLAMA_PORT=11434
OPENWEBUI_PORT=3000
HERMES_PORT=8000
OPENCODE_PORT=8443
COMFYUI_PORT=8188
YOLO_PORT=5000
SEARXNG_PORT=8080
RAG_PORT=6333

# ============================================================
# Imágenes (fija versiones concretas en producción)
# ============================================================
OLLAMA_IMAGE=ollama/ollama:latest
OPENWEBUI_IMAGE=ghcr.io/open-webui/open-webui:cuda
HERMES_IMAGE=nousresearch/hermes-agent:latest
SEARXNG_IMAGE=searxng/searxng:latest
RAG_IMAGE=qdrant/qdrant:latest

# ============================================================
# Ollama
# ============================================================
OLLAMA_KEEP_ALIVE=2m
OLLAMA_MAX_LOADED_MODELS=1
OLLAMA_NUM_PARALLEL=1
OLLAMA_CONTEXT_LENGTH=16384

# ============================================================
# Open WebUI
# ============================================================
WEBUI_SECRET_KEY=CAMBIAR
ENABLE_SIGNUP=true
EMBEDDING_MODEL=nomic-embed-text

# ============================================================
# Hermes Agent
# ============================================================
HERMES_API_KEY=CAMBIAR

# ============================================================
# OpenCode
# ============================================================
OPENCODE_USER=opencode
OPENCODE_PASSWORD=CAMBIAR

# ============================================================
# YOLO
# ============================================================
YOLO_MODEL=yolo11n.pt
YOLO_CONF=0.25
EOF

# Ajustes automáticos: ruta base y secretos aleatorios
sed -i "s|^DATA_ROOT=.*|DATA_ROOT=$HOME|" .env
sed -i "s|^WEBUI_SECRET_KEY=.*|WEBUI_SECRET_KEY=$(openssl rand -hex 32)|" .env
sed -i "s|^HERMES_API_KEY=.*|HERMES_API_KEY=$(openssl rand -hex 24)|" .env
sed -i "s|^OPENCODE_PASSWORD=.*|OPENCODE_PASSWORD=$(openssl rand -base64 18 | tr -d '/+=')|" .env
sed -i "s|^HOST_IP=.*|HOST_IP=$(hostname -I | awk '{print $1}')|" .env

# El .env contiene secretos: solo tu usuario debe leerlo
chmod 600 .env
```

Comprueba el resultado (y apunta la contraseña de OpenCode, la necesitarás para entrar):

```bash
grep -E '^(DATA_ROOT|HOST_IP|OPENCODE_PASSWORD)=' .env
```

> **Nunca subas `.env` a un repositorio git.** Si versionas `$HOME/proyecto`, añade `.env` y `backups/` a `.gitignore`.

---

## 5. Despliegue y verificación

### 5.1. Preparación previa de Hermes y OpenCode

**Hermes Agent** lee su configuración de `/opt/data/.env` (es decir, `$HOME/hermes/.env`). Escribimos ahí las variables de la API:

```bash
cd "$HOME/proyecto"
set -a; . ./.env; set +a
cat > "$DATA_ROOT/hermes/.env" <<EOF
API_SERVER_ENABLED=true
API_SERVER_HOST=0.0.0.0
API_SERVER_PORT=8000
API_SERVER_KEY=${HERMES_API_KEY}
EOF
chmod 600 "$DATA_ROOT/hermes/.env"
```

Después ejecuta **una sola vez** el asistente de configuración de Hermes (interactivo). Cuando pregunte por el proveedor de modelo, elige un endpoint **personalizado compatible con OpenAI** y usa:

- URL base: `http://ollama:11434/v1`
- Clave API: cualquier texto (Ollama no la comprueba), p. ej. `ollama`
- Modelo: el que descargues en 5.3

```bash
docker compose -f docker-hermes-agent.yml run --rm hermes-agent setup
```

> El contenedor del asistente se une a `red-ia` igualmente, pero Ollama debe estar arrancado para validar el modelo: haz el paso 5.2 y 5.3 primero si el asistente intenta conectar.

**OpenCode** necesita su fichero de configuración con el proveedor Ollama y el servidor MCP de SearXNG:

```bash
cat > "$HOME/opencode/config/opencode.json" <<'EOF'
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": { "baseURL": "http://ollama:11434/v1" },
      "models": {
        "qwen2.5-coder:7b": { "name": "Qwen2.5 Coder 7B" }
      }
    }
  },
  "model": "ollama/qwen2.5-coder:7b",
  "mcp": {
    "searxng": {
      "type": "local",
      "command": ["npx", "-y", "mcp-searxng"],
      "environment": { "SEARXNG_URL": "http://searxng:8080" },
      "enabled": true
    }
  }
}
EOF
```

### 5.2. Arranque de los servicios

#### Script de operación `ia.sh`

```bash
cat > ia.sh <<'EOF'
#!/usr/bin/env bash
# Uso: ./ia.sh {up|down|restart|ps|logs <svc>|pull|build}
set -euo pipefail
cd "$(dirname "$0")"

# Orden de arranque: primero lo que usan los demás
ORDEN=(ollama rag searxng comfyui yolo hermes-agent openwebui opencode)

case "${1:-}" in
  up)
    docker network inspect red-ia >/dev/null 2>&1 || docker network create --driver bridge red-ia
    for s in "${ORDEN[@]}"; do echo ">> up $s"; docker compose -f "docker-$s.yml" up -d --build; done ;;
  down)
    for ((i=${#ORDEN[@]}-1; i>=0; i--)); do s="${ORDEN[$i]}"; echo ">> down $s"; docker compose -f "docker-$s.yml" down; done ;;
  restart)
    "$0" down; "$0" up ;;
  ps)
    docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' ;;
  logs)
    docker compose -f "docker-${2:?indica el servicio}.yml" logs -f --tail=100 ;;
  pull)
    for s in "${ORDEN[@]}"; do echo ">> pull $s"; docker compose -f "docker-$s.yml" pull --ignore-buildable; done ;;
  build)
    for s in comfyui yolo opencode; do echo ">> build $s"; docker compose -f "docker-$s.yml" build --pull; done ;;
  *)
    echo "Uso: $0 {up|down|restart|ps|logs <servicio>|pull|build}"; exit 1 ;;
esac
EOF
chmod +x ia.sh
```

#### Primer arranque

La primera vez tarda bastante: se descargan imágenes grandes (Hermes ~5 GB, Open WebUI `:cuda` varios GB) y se construyen ComfyUI, YOLO y OpenCode. Es normal.

```bash
cd "$HOME/proyecto"
./ia.sh up
```

También puedes arrancarlos uno a uno, que es útil para diagnosticar:

```bash
docker compose -f docker-ollama.yml up -d
docker compose -f docker-rag.yml up -d
docker compose -f docker-searxng.yml up -d
docker compose -f docker-comfyui.yml up -d --build
docker compose -f docker-yolo.yml up -d --build
docker compose -f docker-hermes-agent.yml up -d
docker compose -f docker-openwebui.yml up -d
docker compose -f docker-opencode.yml up -d --build
```

### 5.3. Descargar modelos en Ollama

```bash
# LLM de propósito general (cabe en 12 GB)
docker exec -it ollama ollama pull llama3.1:8b

# Modelo de embeddings para RAG (imprescindible para el contenedor rag)
docker exec -it ollama ollama pull nomic-embed-text

# Modelo orientado a código para OpenCode
docker exec -it ollama ollama pull qwen2.5-coder:7b

docker exec -it ollama ollama list
```

### 5.4. Comprobar estado y logs

```bash
./ia.sh ps                      # estado de todos los contenedores

./ia.sh logs ollama             # logs de un servicio (Ctrl+C para salir)
docker logs --tail 50 searxng   # alternativa directa
```

Los ocho contenedores deben aparecer en estado `Up`. Si alguno está en `Restarting`, mira sus logs (sección 6.4).

### 5.5. Acceso a las URLs (sustituye `IP` por la IP del servidor)

| Servicio | URL | Qué deberías ver |
| :--- | :--- | :--- |
| Ollama | `http://IP:11434` | Texto "Ollama is running" |
| Open WebUI | `http://IP:3000` | Pantalla de registro. **El primer usuario que se registra es el administrador** |
| Hermes Agent | `http://IP:8000/health` | Respuesta JSON de estado (con la API activa) |
| OpenCode | `http://IP:8443` | Pide usuario `opencode` y la contraseña del `.env` |
| ComfyUI | `http://IP:8188` | Editor de nodos |
| YOLO | `http://IP:5000/health` | JSON con `"cuda": true` |
| SearXNG | `http://IP:8080` | Buscador |
| RAG (Qdrant) | `http://IP:6333/dashboard` | Panel de Qdrant |

Pruebas rápidas por línea de comandos:

```bash
curl -s http://localhost:11434/api/tags | jq '.models[].name'
curl -s "http://localhost:8080/search?q=ubuntu&format=json" | jq '.results[0].title'
curl -s http://localhost:5000/health | jq
curl -s http://localhost:6333/collections | jq
curl -s http://localhost:8000/health
curl -s -o /dev/null -w "ComfyUI: %{http_code}\n" http://localhost:8188
```

> Tras crear tu usuario administrador en Open WebUI, cierra los registros editando `.env` con `ENABLE_SIGNUP=false` y, como esa variable es persistente, desactívalo también en **Panel de administración → Ajustes → General → Permitir nuevos registros**.

### 5.6. Verificar el uso de la GPU

```bash
# 1) Visión global desde el host (refresco cada segundo)
watch -n 1 nvidia-smi

# 2) La GPU es visible dentro de cada contenedor
docker exec ollama nvidia-smi
docker exec comfyui nvidia-smi

# 3) PyTorch ve CUDA (ComfyUI y YOLO)
docker exec comfyui python -c "import torch; print('CUDA:', torch.cuda.is_available(), torch.cuda.get_device_name(0))"
curl -s http://localhost:5000/health | jq .gpu

# 4) Ollama realmente está usando GPU: ejecuta una consulta y mira el procesador
docker exec ollama ollama run llama3.1:8b "Responde solo: hola" >/dev/null
docker exec ollama ollama ps
```

En la columna `PROCESSOR` de `ollama ps` debe poner `100% GPU`. Si pone `CPU` o un reparto (p. ej. `40%/60% CPU/GPU`), el modelo no cabe entero en la VRAM: usa un modelo más pequeño o reduce `OLLAMA_CONTEXT_LENGTH`.

Prueba de YOLO con una imagen cualquiera:

```bash
curl -s -X POST http://localhost:5000/detect -F "file=@/ruta/a/foto.jpg" | jq
```

---

## 6. Mantenimiento y actualización

### 6.1. Copias de seguridad

Script `backup.sh` (copia configuración y datos, **excluyendo** los modelos de Ollama y de ComfyUI, que son enormes y se pueden volver a descargar; guarda la lista de modelos Ollama):

```bash
cd "$HOME/proyecto"
cat > backup.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
DEST="${BACKUP_DIR:-$HOME/proyecto/backups}"
FECHA="$(date +%F_%H%M)"
mkdir -p "$DEST"

# Lista de modelos Ollama para poder reinstalarlos
docker exec ollama ollama list > "$HOME/proyecto/modelos-ollama.txt" 2>/dev/null || true

# Para consistencia total de Qdrant y Open WebUI, lo ideal es parar antes: ./ia.sh down
tar -czf "$DEST/ia-$FECHA.tar.gz" -C "$HOME" \
  --exclude='proyecto/backups' \
  --exclude='comfyui/models' \
  proyecto openwebui hermes opencode comfyui yolo searxng rag

# Rotación: borrar copias de más de 14 días
find "$DEST" -name 'ia-*.tar.gz' -mtime +14 -delete
echo "Copia creada: $DEST/ia-$FECHA.tar.gz"
EOF
chmod +x backup.sh
./backup.sh
```

Automatizar cada domingo a las 03:00:

```bash
( crontab -l 2>/dev/null; echo "0 3 * * 0 $HOME/proyecto/backup.sh >> $HOME/proyecto/backups/backup.log 2>&1" ) | crontab -
crontab -l
```

**Restaurar** (con los servicios parados):

```bash
cd "$HOME/proyecto" && ./ia.sh down
tar -xzf backups/ia-FECHA.tar.gz -C "$HOME"
./ia.sh up
```

### 6.2. Actualización de servicios

```bash
cd "$HOME/proyecto"
./backup.sh                                  # siempre copia antes de actualizar

# Servicios con imagen publicada (Ollama, Qdrant, SearXNG, Hermes, Open WebUI)
docker compose -f docker-openwebui.yml pull
docker compose -f docker-openwebui.yml up -d  # recrea solo si hay imagen nueva

# Servicios construidos localmente (ComfyUI, YOLO, OpenCode)
docker compose -f docker-comfyui.yml build --pull --no-cache
docker compose -f docker-comfyui.yml up -d

# Todo a la vez
./ia.sh pull && ./ia.sh build && ./ia.sh up

# Limpiar imágenes antiguas sin uso
docker image prune -f
```

Si algo se rompe tras actualizar, fija la versión anterior en `.env` (por ejemplo `OPENWEBUI_IMAGE=ghcr.io/open-webui/open-webui:<etiqueta-anterior>`) y vuelve a hacer `up -d`. Por eso no se recomienda usar `latest` en producción.

Actualizar el **driver NVIDIA** requiere parar los contenedores, instalarlo y reiniciar:

```bash
./ia.sh down
sudo apt update && sudo apt upgrade -y
sudo reboot
# tras el reinicio:
nvidia-smi && cd ~/proyecto && ./ia.sh up
```

### 6.3. Uso de disco

```bash
docker system df          # espacio usado por Docker
du -sh ~/ollama ~/comfyui ~/openwebui ~/rag ~/yolo
df -h /
```

### 6.4. Resolución de errores comunes

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| `could not select device driver "nvidia"` | Falta el NVIDIA Container Toolkit o Docker no se reinició | Repetir 1.4: `sudo nvidia-ctk runtime configure --runtime=docker && sudo systemctl restart docker` |
| `network red-ia declared as external, but could not be found` | No se creó la red | `docker network create --driver bridge red-ia` |
| `port is already allocated` / `address already in use` | Otro proceso usa ese puerto | `ss -tulpn \| grep :PUERTO`, parar el proceso o cambiar el puerto en `.env` |
| `variable is not set` / `Define X en .env` | Falta variable | Editar `.env` y relanzar |
| `Permission denied` al escribir en el volumen | La carpeta pertenece a otro usuario (p. ej. Docker la creó como `root`) | Ver "Receta de permisos" abajo |
| SearXNG: `Permission denied` sobre `settings.yml` o error 403 en `format=json` | Permisos o formato `json` no habilitado | `sudo chown -R 977:977 ~/searxng` y comprobar `formats: [html, json]` en `settings.yml`; `docker restart searxng` |
| Open WebUI no lista modelos | No conecta con Ollama o variable ya persistida | En la UI: **Admin → Ajustes → Conexiones**, URL `http://ollama:11434`. Comprobar `docker exec openwebui curl -s http://ollama:11434` |
| Open WebUI: cambié una variable del compose y no hace efecto | Son PersistentConfig | Cambiar el ajuste en la interfaz (sección 3.8) |
| Ollama usa CPU, va lento | Modelo/contexto demasiado grande | Modelo más pequeño o `OLLAMA_CONTEXT_LENGTH` menor; comprobar con `ollama ps` |
| `CUDA out of memory` en ComfyUI | Ollama tiene un modelo cargado | `docker exec ollama ollama stop llama3.1:8b` o esperar al `KEEP_ALIVE` |
| Hermes no responde en `:8000` | Variables de API no leídas | Revisar `~/hermes/.env` (5.1) y `docker logs hermes-agent` |
| Un contenedor se reinicia en bucle | Error de arranque | `docker logs --tail 100 <contenedor>` y `docker inspect <contenedor> --format '{{.State.ExitCode}}'` |
| Compose avisa de "orphan containers" | Ficheros de proyectos distintos en la misma carpeta | Inofensivo: cada fichero tiene su `name:`; no uses `--remove-orphans` |

**Receta de permisos** (válida para cualquier servicio): averigua con qué usuario corre el proceso dentro del contenedor y haz propietaria a esa identidad de la carpeta del volumen.

```bash
# 1. ¿Con qué UID corre el contenedor?
docker exec searxng id
docker exec hermes-agent id

# 2. ¿A quién pertenece la carpeta en el host?
ls -ld ~/searxng ~/hermes

# 3. Ajustar propietario (sustituye 977 por el UID del paso 1)
sudo chown -R 977:977 ~/searxng

# 4. Reiniciar el servicio
docker restart searxng
```

Si el contenedor corre como `root` (Ollama, Qdrant, Open WebUI, ComfyUI, YOLO, OpenCode en esta configuración), no hace falta `chown`: puede escribir en cualquier carpeta que exista.

---

## 7. Guía interna de integración entre servicios

Todas las URLs son **internas a `red-ia`** (puerto interno del contenedor, nunca el del host). Para probar cualquier conexión desde dentro de un contenedor:

```bash
docker exec openwebui curl -s http://ollama:11434          # Open WebUI → Ollama
docker exec opencode  curl -s http://searxng:8080 -o /dev/null -w "%{http_code}\n"
```

### 7.1. Matriz de conexiones

| Origen → Destino | URL interna | Cómo se configura |
| :--- | :--- | :--- |
| Open WebUI → Ollama | `http://ollama:11434` | Variable `OLLAMA_BASE_URL` (ya en el compose) |
| Open WebUI → SearXNG | `http://searxng:8080/search?q=<query>` | `ENABLE_WEB_SEARCH`, `WEB_SEARCH_ENGINE=searxng`, `SEARXNG_QUERY_URL` |
| Open WebUI → ComfyUI | `http://comfyui:8188` | `COMFYUI_BASE_URL` + workflow en la UI (7.4) |
| Open WebUI → RAG | `http://rag:6333` | `VECTOR_DB=qdrant`, `QDRANT_URI` |
| Open WebUI → Hermes | `http://hermes-agent:8000/v1` | `OPENAI_API_BASE_URLS` + `OPENAI_API_KEYS` |
| Hermes → Ollama | `http://ollama:11434/v1` | Asistente `hermes setup` (5.1) |
| OpenCode → Ollama | `http://ollama:11434/v1` | `opencode.json` → `provider.ollama` |
| OpenCode → SearXNG | `http://searxng:8080` | Servidor MCP `mcp-searxng` en `opencode.json` |
| Cualquiera → YOLO | `http://yolo:5000/detect` | Petición HTTP `POST` con la imagen |

### 7.2. Ollama ↔ Open WebUI

1. Entra en `http://IP:3000` y crea el usuario administrador.
2. Abre un chat: el selector de modelo debe listar `llama3.1:8b` y los demás.
3. Si no aparece: **Panel de administración → Ajustes → Conexiones → Ollama API** y escribe `http://ollama:11434`. Pulsa el botón de verificar.

### 7.3. SearXNG ↔ Open WebUI y SearXNG ↔ OpenCode

**Open WebUI:** verifica en **Admin → Ajustes → Búsqueda web** que el motor es `searxng` y la URL de consulta es `http://searxng:8080/search?q=<query>`. En un chat, activa el interruptor **Búsqueda web** antes de enviar la pregunta.

**OpenCode:** el servidor MCP definido en `opencode.json` (5.1) arranca con `npx` dentro del contenedor. Dentro de OpenCode, comprueba que la herramienta `searxng` aparece en la lista de MCP y pídele algo como "busca en la web las novedades de Docker". Requisito: SearXNG debe tener `json` en `search.formats` (3.3).

Diagnóstico si falla:

```bash
docker exec opencode curl -s "http://searxng:8080/search?q=test&format=json" | head -c 300
```

Si devuelve HTML o un 403, el formato JSON no está habilitado.

### 7.4. ComfyUI ↔ Open WebUI (imágenes)

1. En ComfyUI (`http://IP:8188`), carga un workflow de texto a imagen que funcione con tu checkpoint.
2. Menú **Workflow → Export (API)** y guarda el JSON.
3. En Open WebUI: **Admin → Ajustes → Imágenes**. Motor `ComfyUI`, URL `http://comfyui:8188`.
4. Pega el JSON del workflow y asigna los *nodos* (prompt, modelo, ancho, alto, pasos) en los campos que indica la interfaz.
5. En un chat, usa el botón de imagen sobre un mensaje para generar.

### 7.5. RAG (Qdrant + embeddings de Ollama) con Open WebUI

Flujo: el documento se trocea → Ollama (`nomic-embed-text`) genera los vectores → se almacenan en Qdrant (`rag`) → en la consulta se recuperan los fragmentos relevantes y se pasan al LLM.

1. Asegúrate de haber descargado `nomic-embed-text` (5.3).
2. En Open WebUI: **Workspace → Knowledge → +**, crea una base de conocimiento.
3. Sube los documentos de `$HOME/rag/docs` (PDF, Markdown, texto...).
4. En un chat escribe `#` y selecciona la base de conocimiento, o asígnala a un modelo.
5. Verifica que los vectores están en Qdrant:

```bash
curl -s http://localhost:6333/collections | jq
```

*(Opcional)* Cargar en bloque los ficheros de `$HOME/rag/docs` mediante la API de Open WebUI. Genera una clave en **Ajustes → Cuenta → Claves API** y crea antes la base de conocimiento para obtener su identificador:

```bash
KB_ID="id-de-la-base-de-conocimiento"
TOKEN="sk-tu-clave-api"
for f in "$HOME"/rag/docs/*; do
  FID=$(curl -s -X POST http://localhost:3000/api/v1/files/ \
        -H "Authorization: Bearer $TOKEN" -F "file=@$f" | jq -r .id)
  curl -s -X POST "http://localhost:3000/api/v1/knowledge/$KB_ID/file/add" \
       -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
       -d "{\"file_id\":\"$FID\"}" > /dev/null
  echo "Subido: $f"
done
```

### 7.6. Hermes Agent

**Hermes → Ollama:** hecho en el asistente (5.1). Para que Hermes funcione bien como agente necesita un modelo con **contexto grande** y buen uso de herramientas; con 12 GB de VRAM esto es limitado, así que empieza con tareas sencillas y, si hace falta, sube `OLLAMA_CONTEXT_LENGTH` vigilando `ollama ps`. Consulta los requisitos actuales en la documentación de Hermes.

**Open WebUI → Hermes:** ya está preconfigurado (conexión OpenAI-compatible a `http://hermes-agent:8000/v1` con la clave `HERMES_API_KEY`). Verifícalo en **Admin → Ajustes → Conexiones → OpenAI API**; Hermes aparecerá como un modelo más.

Prueba directa:

```bash
source ~/proyecto/.env
curl -s http://localhost:8000/v1/models -H "Authorization: Bearer $HERMES_API_KEY" | jq
```

**Hermes → SearXNG / ComfyUI:** Hermes admite herramientas externas vía MCP. Añade en su configuración (`~/hermes/config.yaml`, consulta el formato exacto en su documentación) un servidor MCP equivalente al de OpenCode (`npx -y mcp-searxng` con `SEARXNG_URL=http://searxng:8080`). Para ComfyUI, apunta a `http://comfyui:8188` mediante un servidor MCP o skill de ComfyUI.

### 7.7. YOLO

API HTTP interna para cualquier servicio o script:

```bash
# Desde otro contenedor de red-ia
docker exec openwebui curl -s http://yolo:5000/health
```

Para que un LLM la use como herramienta, en Open WebUI crea una *Tool* (**Workspace → Tools**) que haga `POST http://yolo:5000/detect` con la imagen; en Hermes u OpenCode, un MCP/skill con la misma llamada.

### 7.8. Diagrama de flujo resumido

```
                    ┌─────────────┐
   navegador ─────► │  openwebui  │────────────┐
                    └──┬──┬──┬──┬─┘            │
        embeddings/LLM │  │  │  └─ imágenes    │ OpenAI API
                       ▼  │  │       ▼         ▼
                  ┌────────┐│  │  ┌─────────┐ ┌──────────────┐
                  │ ollama │◄──┼──│ comfyui │ │ hermes-agent │
                  └───▲────┘   │  └─────────┘ └──────┬───────┘
                      │        └─ búsqueda ──► searxng ◄─────┘ (MCP)
   opencode ──────────┘                           ▲
        └─────────── MCP ─────────────────────────┘
                  openwebui ──► rag (Qdrant)       yolo (API independiente)
```

---

## 8. Checklist de aceptación (criterios del proyecto)

| Criterio | Cómo comprobarlo |
| :--- | :--- |
| 1. YAML funcionales y sin errores | `for f in docker-*.yml; do docker compose -f "$f" config -q && echo OK; done` |
| 2. Contenedores con CUDA/GPU | `nvidia-smi` (5.6), `ollama ps` → `100% GPU`, `/health` de YOLO con `cuda: true`. SearXNG, Qdrant y Hermes no usan GPU (ver sección 0) |
| 3. Sin colisiones de puertos | Tabla 2.2 y `ss -tulpn` (2.2) |
| 4. Explicación paso a paso para un novato | Secciones 1 a 7 |

### Referencias oficiales

- Docker Engine en Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- NVIDIA Container Toolkit: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
- Ollama: https://docs.ollama.com
- Open WebUI: https://docs.openwebui.com
- Hermes Agent (Docker): https://hermes-agent.nousresearch.com/docs/user-guide/docker
- OpenCode: https://opencode.ai/docs
- ComfyUI: https://github.com/comfyanonymous/ComfyUI
- Ultralytics YOLO: https://docs.ultralytics.com
- SearXNG: https://docs.searxng.org
- Qdrant: https://qdrant.tech/documentation/

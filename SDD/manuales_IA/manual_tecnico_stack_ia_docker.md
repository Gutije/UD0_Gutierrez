---
title: "Manual técnico: Despliegue de una pila de IA local en Docker sobre Ubuntu"
author: "Gutije"
version: "1.0"
date: "2026-10-06"
category: "Despliegue IA"
tags: [docker, docker-compose, ubuntu, nvidia, cuda, ollama, open-webui, hermes-agent, opencode, comfyui, yolo, searxng, rag, local-ai]
---

# Manual técnico: Despliegue de una pila de IA local en Docker sobre Ubuntu

## 0. Introducción

Este documento define la instalación, configuración, despliegue, integración, verificación y mantenimiento de una infraestructura de Inteligencia Artificial local basada en Docker sobre Ubuntu Server, con aceleración NVIDIA.

La arquitectura parte de la especificación del proyecto original: todos los servicios se ejecutan en contenedores, comparten una red Docker llamada `red-ia` y mantienen sus datos en directorios persistentes del host bajo `$HOME/proyecto`. La especificación original identifica como servicios principales Ollama, Open WebUI, Hermes Agent, OpenCode, ComfyUI, YOLO, SearXNG y RAG. [Especificación del proyecto, secciones 1 y 2]

> **Importante:** este manual distingue entre lo que define la especificación y lo que se ha adaptado para que el despliegue sea coherente con las interfaces actuales de los proyectos. No se deben interpretar los puertos o imágenes de la especificación original como una garantía de que coincidan exactamente con las versiones actuales de cada software.

### Fuentes oficiales consultadas

- Docker Engine para Ubuntu: <https://docs.docker.com/engine/install/ubuntu/>
- NVIDIA Container Toolkit: <https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html>
- Ollama: <https://docs.ollama.com/>
- Open WebUI: <https://docs.openwebui.com/>
- Hermes Agent: <https://github.com/NousResearch/hermes-agent>
- OpenCode: <https://opencode.ai/docs/>
- SearXNG: <https://docs.searxng.org/admin/installation-docker>
- Ultralytics YOLO: <https://docs.ultralytics.com/guides/docker-quickstart/>
- ComfyUI: <https://github.com/comfyanonymous/ComfyUI>

---

# 1. Alcance y arquitectura

## 1.1. Objetivo

El objetivo es disponer de una plataforma local en la que:

1. **Ollama** ejecute los LLM y exponga una API local.
2. **Open WebUI** proporcione una interfaz tipo ChatGPT y consuma Ollama.
3. **Hermes Agent** actúe como agente autónomo y utilice Ollama mediante una API compatible con OpenAI.
4. **OpenCode** proporcione el servidor HTTP de OpenCode para desarrollo asistido por IA.
5. **ComfyUI** gestione flujos de generación y procesamiento de imágenes/vídeo sobre GPU.
6. **YOLO** proporcione una API de visión/detección que pueda recibir imágenes y devolver detecciones.
7. **SearXNG** proporcione búsqueda web privada/autohospedada.
8. **RAG** mantenga un índice vectorial de documentación privada y consulte Ollama para embeddings.

## 1.2. Arquitectura lógica

```text
                             INTERNET
                                 |
                                 |
                         +----------------+
                         |    SearXNG     |
                         |     :8080      |
                         +-------+--------+
                                 |
                  búsquedas      |      búsquedas
                                 |
              +------------------+------------------+
              |                                     |
        +-----v------+                        +-----v------+
        | Open WebUI |                        |   Hermes    |
        |   :3000    |                        |   :8000     |
        +-----+------+                        +-----+-------+
              |                                     |
              |                                     |
              | Ollama API                          | OpenAI-compatible API
              |                                     |
              +------------------+------------------+
                                 |
                         +-------v--------+
                         |     Ollama     |
                         |    :11434      |
                         | NVIDIA / CUDA  |
                         +-------+--------+
                                 |
                     +-----------+-----------+
                     |                       |
               +-----v------+          +-----v------+
               |    RAG     |          |  OpenCode   |
               |   :11435   |          |   :8443     |
               +------------+          +-------------+

      Procesamiento GPU independiente:

               +-------------+          +-------------+
               |   ComfyUI   |          |     YOLO    |
               |    :8188    |          |    :5000    |
               | NVIDIA CUDA |          | NVIDIA CUDA |
               +-------------+          +-------------+
```

## 1.3. Red Docker

La red común se denomina `red-ia`, es de tipo `bridge` y se crea explícitamente como una red externa para que todos los ficheros Compose puedan compartirla.

Dentro de Docker, los contenedores se localizan por DNS usando su nombre:

| Servicio | Contenedor | Puerto interno | Puerto host | URL interna |
|---|---|---:|---:|---|
| Ollama | `ollama` | 11434 | 11434 | `http://ollama:11434` |
| Open WebUI | `openwebui` | 8080 | 3000 | `http://openwebui:8080` |
| Hermes Agent | `hermes-agent` | 8642 | 8000 | `http://hermes-agent:8642` |
| OpenCode | `opencode` | 4096 | 8443 | `http://opencode:4096` |
| ComfyUI | `comfyui` | 8188 | 8188 | `http://comfyui:8188` |
| YOLO | `yolo` | 5000 | 5000 | `http://yolo:5000` |
| SearXNG | `searxng` | 8080 | 8080 | `http://searxng:8080` |
| RAG | `rag` | 8001 | 11435 | `http://rag:8001` |

### 1.3.1. Correcciones de puertos respecto a la especificación

La especificación original asignaba a Hermes el puerto interno `8000`, a OpenCode `8080` y a RAG `11434`. Estas asignaciones generan problemas o no coinciden con las interfaces actuales:

- Hermes Agent en modo gateway utiliza actualmente el puerto **8642**. El host puede seguir usando `8000` para respetar la intención del proyecto.
- OpenCode Server utiliza actualmente **4096** por defecto. El host puede exponerlo como `8443`.
- `11434` ya está ocupado por Ollama, por lo que RAG se expone en `11435` en el host y escucha en `8001` dentro de su contenedor.

Estas correcciones permiten cumplir el criterio de aceptación de evitar colisiones de puertos.

---

# 2. Requisitos previos

## 2.1. Sistema operativo

Se contempla Ubuntu Server 24.04 LTS y Ubuntu Server 26.04 LTS. La documentación actual de Docker Engine incluye explícitamente Ubuntu 24.04 (Noble) y Ubuntu 26.04 (Resolute) entre las versiones soportadas. [Docker Docs]

Comprobar versión:

```bash
cat /etc/os-release
uname -a
uname -m
```

Se espera una arquitectura `x86_64`/`amd64` para la plataforma de referencia.

## 2.2. Hardware

La especificación de proyecto contempla máquinas con NVIDIA, incluyendo equipos con RTX 3060 de 12 GB.

Comprobar GPU:

```bash
lspci | grep -i nvidia
```

Después de instalar los controladores:

```bash
nvidia-smi
```

Para una RTX 3060 de 12 GB, se recomienda priorizar modelos locales de tamaño moderado y cuantizaciones compatibles con la memoria disponible. El hecho de disponer de 12 GB de VRAM no significa que cualquier modelo pueda cargarse íntegramente en GPU.

## 2.3. Red y almacenamiento

Recomendaciones mínimas prácticas para un laboratorio local:

- CPU: 8 hilos o más.
- RAM: 32 GB recomendados.
- GPU NVIDIA: 12 GB VRAM como base para el escenario del proyecto.
- SSD: 150 GB libres como punto de partida; se recomienda disponer de bastante más si se almacenarán varios LLM, checkpoints de difusión, datasets y documentos RAG.
- Conectividad a Internet durante la instalación y para descargar imágenes/modelos.

Los tamaños reales de los modelos de IA varían y pueden ser mucho mayores que los ejemplos indicados aquí.

---

# 3. Preparación del sistema Ubuntu

## 3.1. Actualización del sistema

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y ca-certificates curl gnupg git jq unzip htop vim nano tree pciutils
```

Reiniciar si se ha actualizado el kernel:

```bash
sudo reboot
```

## 3.2. Comprobar zona horaria

```bash
timedatectl
```

Configurar la zona horaria de Madrid, si corresponde al servidor:

```bash
sudo timedatectl set-timezone Europe/Madrid
```

---

# 4. Instalación de NVIDIA y soporte CUDA

## 4.1. Principio de funcionamiento

En este escenario, el **driver NVIDIA se instala en el host**. El contenedor recibe acceso a la GPU mediante NVIDIA Container Toolkit. No es necesario instalar el driver NVIDIA completo dentro de cada contenedor.

NVIDIA documenta la instalación del Container Toolkit mediante APT y la posterior integración con Docker con `nvidia-ctk runtime configure --runtime=docker`. [NVIDIA Container Toolkit]

## 4.2. Detectar el driver recomendado por Ubuntu

En Ubuntu puede consultarse el driver recomendado mediante:

```bash
ubuntu-drivers devices
```

Instalar automáticamente el recomendado:

```bash
sudo ubuntu-drivers autoinstall
```

Reiniciar:

```bash
sudo reboot
```

Comprobar:

```bash
nvidia-smi
```

> **Nota:** la instalación concreta del driver puede variar según la generación de GPU y la versión de Ubuntu. Para equipos de producción se recomienda seleccionar una rama de driver soportada por la GPU y por la versión de Ubuntu instalada, en lugar de asumir una versión exacta en este documento.

## 4.3. Instalar NVIDIA Container Toolkit

```bash
sudo apt-get update
sudo apt-get install -y --no-install-recommends ca-certificates curl gnupg2
```

Añadir la clave y repositorio oficial de NVIDIA:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

Configurar Docker:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Comprobar el runtime:

```bash
docker info | grep -i -E 'Runtimes|nvidia'
```

## 4.4. Test de GPU dentro de un contenedor

Antes de desplegar la pila, comprobar que un contenedor puede ver la GPU:

```bash
sudo docker run --rm --gpus all nvidia/cuda:12.3.2-base-ubuntu22.04 nvidia-smi
```

La salida debe mostrar la misma GPU que muestra `nvidia-smi` en el host.

> La versión de la imagen CUDA usada en este test es solamente un ejemplo de validación. No es necesario que coincida exactamente con la versión de CUDA de los contenedores de aplicaciones.

---

# 5. Instalación de Docker Engine y Docker Compose

## 5.1. Eliminar paquetes conflictivos

```bash
sudo apt remove -y \
  docker.io docker-compose docker-compose-v2 docker-doc docker-buildx \
  podman-docker containerd runc
```

Docker indica que estos paquetes pueden entrar en conflicto con los paquetes oficiales de Docker Engine. [Docker Docs]

## 5.2. Configurar el repositorio oficial de Docker

```bash
sudo apt update
sudo apt install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Crear el repositorio:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
# remove accidental empty append marker if any
true
EOF
```

Actualizar índices:

```bash
sudo apt update
```

## 5.3. Instalar Docker Engine y Compose V2

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Arranque automático:

```bash
sudo systemctl enable --now docker
```

Comprobar:

```bash
docker --version
docker compose version
sudo systemctl status docker --no-pager
```

Prueba oficial:

```bash
sudo docker run hello-world
```

## 5.4. Permitir usar Docker sin sudo

```bash
sudo usermod -aG docker "$USER"
```

Aplicar el grupo en la sesión actual:

```bash
newgrp docker
```

Comprobar:

```bash
docker run hello-world
```

> Añadir un usuario al grupo `docker` otorga capacidades equivalentes a un alto nivel de privilegios sobre el host. En un servidor multiusuario debe evaluarse el modelo de seguridad antes de hacerlo.

---

# 6. Estructura del proyecto

Crear el proyecto:

```bash
mkdir -p "$HOME/proyecto"
cd "$HOME/proyecto"
```

Estructura recomendada:

```text
$HOME/proyecto/
├── .env
├── .gitignore
├── README.md
├── docker-ollama.yml
├── docker-openwebui.yml
├── docker-hermes-agent.yml
├── docker-opencode.yml
├── docker-comfyui.yml
├── docker-yolo.yml
├── docker-searxng.yml
├── docker-rag.yml
├── scripts/
│   ├── create-network.sh
│   ├── deploy.sh
│   ├── down.sh
│   ├── logs.sh
│   ├── status.sh
│   ├── gpu-test.sh
│   └── backup.sh
├── searxng/
│   └── settings.yml
├── comfyui/
│   ├── models/
│   ├── input/
│   ├── output/
│   ├── custom_nodes/
│   └── user/
├── yolo/
│   ├── app/
│   │   ├── app.py
│   │   └── requirements.txt
│   ├── models/
│   └── data/
├── rag/
│   ├── app/
│   │   ├── app.py
│   │   └── requirements.txt
│   ├── documents/
│   └── chroma/
├── ollama/
├── openwebui/
├── hermes/
├── opencode/
└── backups/
```

Crear directorios:

```bash
mkdir -p \
  "$HOME/proyecto/ollama" \
  "$HOME/proyecto/openwebui" \
  "$HOME/proyecto/hermes" \
  "$HOME/proyecto/opencode" \
  "$HOME/proyecto/comfyui/models" \
  "$HOME/proyecto/comfyui/input" \
  "$HOME/proyecto/comfyui/output" \
  "$HOME/proyecto/comfyui/custom_nodes" \
  "$HOME/proyecto/comfyui/user" \
  "$HOME/proyecto/yolo/app" \
  "$HOME/proyecto/yolo/models" \
  "$HOME/proyecto/yolo/data" \
  "$HOME/proyecto/rag/app" \
  "$HOME/proyecto/rag/documents" \
  "$HOME/proyecto/rag/chroma" \
  "$HOME/proyecto/searxng" \
  "$HOME/proyecto/backups"
```

---

# 7. Fichero `.env`

Crear `$HOME/proyecto/.env`:

```dotenv
PROJECT_DIR=/home/USUARIO/proyecto
HOST_BIND_IP=0.0.0.0
TZ=Europe/Madrid

OLLAMA_PORT=11434
OPENWEBUI_PORT=3000
HERMES_PORT=8000
OPENCODE_PORT=8443
COMFYUI_PORT=8188
YOLO_PORT=5000
SEARXNG_PORT=8080
RAG_PORT=11435

OLLAMA_URL=http://ollama:11434
OPENWEBUI_URL=http://openwebui:8080
HERMES_URL=http://hermes-agent:8642
OPENCODE_URL=http://opencode:4096
COMFYUI_URL=http://comfyui:8188
YOLO_URL=http://yolo:5000
SEARXNG_URL=http://searxng:8080
RAG_URL=http://rag:8001

OLLAMA_HOST=0.0.0.0:11434
OLLAMA_KEEP_ALIVE=5m
OLLAMA_NUM_PARALLEL=1

RAG_EMBED_MODEL=nomic-embed-text
DEFAULT_LLM_MODEL=gemma4:31b

WEBUI_SECRET_KEY=CAMBIAR_POR_UNA_CLAVE_ALEATORIA_LARGA
WEBUI_NAME=IA Local
ENABLE_WEB_SEARCH=true
WEB_SEARCH_ENGINE=searxng
WEB_SEARCH_RESULT_COUNT=5
WEB_SEARCH_CONCURRENT_REQUESTS=5
SEARXNG_QUERY_URL=http://searxng:8080/search
SEARXNG_LANGUAGE=all

OPENAI_API_KEY=
OPENROUTER_API_KEY=
NOUS_API_KEY=
OLLAMA_API_KEY=
HERMES_OLLAMA_URL=http://ollama:11434/v1

OPENCODE_SERVER_PASSWORD=CAMBIAR_POR_UNA_CLAVE_LARGA
OPENCODE_HOSTNAME=0.0.0.0
OPENCODE_SERVER_PORT=4096

YOLO_MODEL=yolo11n.pt
YOLO_CONFIDENCE=0.25

RAG_HOST=0.0.0.0
RAG_PORT_INTERNAL=8001
RAG_OLLAMA_URL=http://ollama:11434
RAG_CHROMA_PATH=/data/chroma
RAG_DOCUMENTS_PATH=/data/documents
RAG_CHUNK_SIZE=1000
RAG_CHUNK_OVERLAP=150
RAG_TOP_K=5
```

Generar secretos:

```bash
openssl rand -hex 32
```

Proteger el fichero:

```bash
chmod 600 .env
```

---

# 8. Configuración de SearXNG

Crear `$HOME/proyecto/searxng/settings.yml`:

```yaml
use_default_settings: true

server:
  bind_address: 0.0.0.0
  port: 8080
  secret_key: "CAMBIAR_SECRET_KEY"
  public_instance: false
  limiter: false

search:
  formats:
    - html
    - json
```

Generar el secreto y sustituirlo:

```bash
openssl rand -hex 32
```

> Los motores concretos de SearXNG evolucionan y pueden depender de la configuración de la instancia. Activar únicamente los engines que se necesiten.

---

# 9. `docker-ollama.yml`

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    environment:
      OLLAMA_HOST: ${OLLAMA_HOST}
      OLLAMA_KEEP_ALIVE: ${OLLAMA_KEEP_ALIVE}
      OLLAMA_NUM_PARALLEL: ${OLLAMA_NUM_PARALLEL}
    ports:
      - "${HOST_BIND_IP}:${OLLAMA_PORT}:11434"
    volumes:
      - "${PROJECT_DIR}/ollama:/root/.ollama"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      red-ia:
        aliases:
          - ollama

networks:
  red-ia:
    external: true
```

Inicializar:

```bash
docker exec -it ollama ollama pull ${DEFAULT_LLM_MODEL}
docker exec -it ollama ollama pull ${RAG_EMBED_MODEL}
docker exec -it ollama ollama list
```

Probar API:

```bash
curl http://127.0.0.1:11434/api/tags
```

---

# 10. `docker-openwebui.yml`

```yaml
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:cuda
    container_name: openwebui
    restart: unless-stopped
    environment:
      OLLAMA_BASE_URL: ${OLLAMA_URL}
      WEBUI_SECRET_KEY: ${WEBUI_SECRET_KEY}
      WEBUI_NAME: ${WEBUI_NAME}
      ENABLE_WEB_SEARCH: ${ENABLE_WEB_SEARCH}
      WEB_SEARCH_ENGINE: ${WEB_SEARCH_ENGINE}
      WEB_SEARCH_RESULT_COUNT: ${WEB_SEARCH_RESULT_COUNT}
      WEB_SEARCH_CONCURRENT_REQUESTS: ${WEB_SEARCH_CONCURRENT_REQUESTS}
      SEARXNG_QUERY_URL: ${SEARXNG_QUERY_URL}
      SEARXNG_LANGUAGE: ${SEARXNG_LANGUAGE}
    ports:
      - "${HOST_BIND_IP}:${OPENWEBUI_PORT}:8080"
    volumes:
      - "${PROJECT_DIR}/openwebui:/app/backend/data"
    networks:
      red-ia:
        aliases:
          - openwebui

networks:
  red-ia:
    external: true
```

Abrir:

```text
http://IP_DEL_SERVIDOR:3000
```

En la administración, verificar:

```text
Ollama URL: http://ollama:11434
Web Search: activado
Engine: searxng
SearXNG Query URL: http://searxng:8080/search
```

---

# 11. `docker-hermes-agent.yml`

```yaml
services:
  hermes-agent:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-agent
    restart: unless-stopped
    command: ["gateway", "run"]
    environment:
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      OPENROUTER_API_KEY: ${OPENROUTER_API_KEY}
      NOUS_API_KEY: ${NOUS_API_KEY}
      OLLAMA_API_KEY: ${OLLAMA_API_KEY}
      TZ: ${TZ}
    ports:
      - "${HOST_BIND_IP}:${HERMES_PORT}:8642"
    volumes:
      - "${PROJECT_DIR}/hermes:/opt/data"
    networks:
      red-ia:
        aliases:
          - hermes-agent
          - hermesagent

networks:
  red-ia:
    external: true
```

Primera configuración:

```bash
docker compose -f docker-hermes-agent.yml run --rm --no-deps hermes-agent hermes setup
```

Conectar con Ollama:

```bash
docker exec -it hermes-agent hermes model
```

Seleccionar `Custom endpoint (self-hosted / VLLM / etc.)` y utilizar:

```text
Base URL: http://ollama:11434/v1
API Key: vacía o no-key
Model: <modelo instalado en Ollama>
```

Alternativamente, el `config.yaml` persistente puede contener:

```yaml
model:
  default: "<modelo>"
  provider: "custom"
  base_url: "http://ollama:11434/v1"
```

---

# 12. `docker-opencode.yml`

```yaml
services:
  opencode:
    image: ghcr.io/anomalyco/opencode:latest
    container_name: opencode
    restart: unless-stopped
    command:
      - serve
      - --hostname
      - ${OPENCODE_HOSTNAME}
      - --port
      - "${OPENCODE_SERVER_PORT}"
    environment:
      OPENCODE_SERVER_PASSWORD: ${OPENCODE_SERVER_PASSWORD}
    ports:
      - "${HOST_BIND_IP}:${OPENCODE_PORT}:4096"
    volumes:
      - "${PROJECT_DIR}/opencode:/workspace"
    networks:
      red-ia:
        aliases:
          - opencode

networks:
  red-ia:
    external: true
```

Comprobar:

```bash
docker logs --tail 200 opencode
curl -I http://127.0.0.1:8443
```

> OpenCode Server es un servidor HTTP/OpenAPI. No debe confundirse la API HTTP con una interfaz web completa tipo ChatGPT.

---

# 13. `docker-comfyui.yml`

```yaml
services:
  comfyui:
    image: ghcr.io/ai-dock/comfyui:latest
    container_name: comfyui
    restart: unless-stopped
    ports:
      - "${HOST_BIND_IP}:${COMFYUI_PORT}:8188"
    volumes:
      - "${PROJECT_DIR}/comfyui/models:/workspace/models"
      - "${PROJECT_DIR}/comfyui/input:/workspace/input"
      - "${PROJECT_DIR}/comfyui/output:/workspace/output"
      - "${PROJECT_DIR}/comfyui/custom_nodes:/workspace/custom_nodes"
      - "${PROJECT_DIR}/comfyui/user:/workspace/user"
    environment:
      TZ: ${TZ}
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      red-ia:
        aliases:
          - comfyui

networks:
  red-ia:
    external: true
```

> La imagen se utiliza como imagen de ejemplo mantenida por terceros, no como imagen oficial de ComfyUI. Validar el tag antes de producción. Para una instalación plenamente controlada, construir una imagen propia a partir del repositorio oficial de ComfyUI.

Comprobar:

```bash
docker exec -it comfyui nvidia-smi
docker logs --tail 200 comfyui
```

Abrir:

```text
http://IP_DEL_SERVIDOR:8188
```

---

# 14. YOLO como API HTTP

La imagen de Ultralytics se utiliza como base GPU. La API de puerto 5000 es una capa FastAPI del proyecto.

## 14.1. `yolo/app/requirements.txt`

```text
fastapi>=0.115,<1
uvicorn[standard]>=0.30,<1
python-multipart>=0.0.9,<1
```

## 14.2. `yolo/app/app.py`

```python
from __future__ import annotations

import os
import tempfile
from pathlib import Path
from typing import Any

from fastapi import FastAPI, File, HTTPException, UploadFile
from ultralytics import YOLO

MODEL_NAME = os.getenv("YOLO_MODEL", "yolo11n.pt")
CONFIDENCE = float(os.getenv("YOLO_CONFIDENCE", "0.25"))

app = FastAPI(title="YOLO Local API", version="1.0.0")
model = YOLO(MODEL_NAME)


@app.get("/healthz")
def healthz() -> dict[str, Any]:
    return {"status": "ok", "model": MODEL_NAME, "confidence": CONFIDENCE}


@app.post("/predict")
async def predict(file: UploadFile = File(...)) -> dict[str, Any]:
    if not file.content_type or not file.content_type.startswith("image/"):
        raise HTTPException(status_code=400, detail="El fichero debe ser una imagen")

    suffix = Path(file.filename or "image.jpg").suffix or ".jpg"

    with tempfile.NamedTemporaryFile(suffix=suffix, delete=False) as tmp:
        tmp_path = Path(tmp.name)
        tmp.write(await file.read())

    try:
        results = model.predict(source=str(tmp_path), conf=CONFIDENCE, verbose=False)
        result = results[0]
        detections = []
        names = result.names

        if result.boxes is not None:
            for box in result.boxes:
                cls_id = int(box.cls.item())
                detections.append({
                    "class_id": cls_id,
                    "class_name": names.get(cls_id, str(cls_id)),
                    "confidence": float(box.conf.item()),
                    "xyxy": [round(float(v), 2) for v in box.xyxy[0].tolist()],
                })

        return {
            "filename": file.filename,
            "model": MODEL_NAME,
            "detections": detections,
        }
    finally:
        tmp_path.unlink(missing_ok=True)
```

## 14.3. `yolo/Dockerfile`

```dockerfile
FROM ultralytics/ultralytics:latest

WORKDIR /app

COPY app/requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt

COPY app/app.py /app/app.py

EXPOSE 5000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "5000"]
```

## 14.4. `docker-yolo.yml`

```yaml
services:
  yolo:
    build:
      context: ./yolo
      dockerfile: Dockerfile
    image: proyecto-ia/yolo:latest
    container_name: yolo
    restart: unless-stopped
    environment:
      YOLO_MODEL: ${YOLO_MODEL}
      YOLO_CONFIDENCE: ${YOLO_CONFIDENCE}
    ports:
      - "${HOST_BIND_IP}:${YOLO_PORT}:5000"
    volumes:
      - "${PROJECT_DIR}/yolo/models:/root/.config/Ultralytics"
      - "${PROJECT_DIR}/yolo/data:/data"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      red-ia:
        aliases:
          - yolo

networks:
  red-ia:
    external: true
```

Pruebas:

```bash
docker compose -f docker-yolo.yml build
docker compose -f docker-yolo.yml up -d
curl http://127.0.0.1:5000/healthz

docker exec -it yolo nvidia-smi
```

Predicción:

```bash
curl -X POST \
  -F "file=@/ruta/a/imagen.jpg" \
  http://127.0.0.1:5000/predict | jq
```

---

# 15. `docker-searxng.yml`

```yaml
services:
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    restart: unless-stopped
    ports:
      - "${HOST_BIND_IP}:${SEARXNG_PORT}:8080"
    volumes:
      - "${PROJECT_DIR}/searxng:/etc/searxng:rw"
    cap_drop:
      - ALL
    networks:
      red-ia:
        aliases:
          - searxng

networks:
  red-ia:
    external: true
```

Verificación:

```bash
curl "http://127.0.0.1:8080/search?q=Ubuntu&format=json" | jq
```

---

# 16. Servicio RAG

## 16.1. Arquitectura

```text
Documento -> chunking -> embedding en Ollama -> ChromaDB
Consulta  -> embedding en Ollama -> ChromaDB -> fragmentos
```

El servicio RAG se mantiene separado para poder ser consumido por scripts y otros agentes, aunque Open WebUI disponga de capacidades RAG propias.

## 16.2. `rag/app/requirements.txt`

```text
fastapi>=0.115,<1
uvicorn[standard]>=0.30,<1
chromadb>=1,<2
httpx>=0.27,<1
pypdf>=5,<7
python-multipart>=0.0.9,<1
```

## 16.3. `rag/app/app.py`

```python
from __future__ import annotations

import os
import uuid
from pathlib import Path
from typing import Any

import chromadb
import httpx
from fastapi import FastAPI, File, HTTPException, UploadFile
from pypdf import PdfReader

OLLAMA_URL = os.getenv("RAG_OLLAMA_URL", "http://ollama:11434")
EMBED_MODEL = os.getenv("RAG_EMBED_MODEL", "nomic-embed-text")
CHROMA_PATH = os.getenv("RAG_CHROMA_PATH", "/data/chroma")
DOCUMENTS_PATH = os.getenv("RAG_DOCUMENTS_PATH", "/data/documents")
CHUNK_SIZE = int(os.getenv("RAG_CHUNK_SIZE", "1000"))
CHUNK_OVERLAP = int(os.getenv("RAG_CHUNK_OVERLAP", "150"))
TOP_K = int(os.getenv("RAG_TOP_K", "5"))

Path(DOCUMENTS_PATH).mkdir(parents=True, exist_ok=True)
Path(CHROMA_PATH).mkdir(parents=True, exist_ok=True)

client = chromadb.PersistentClient(path=CHROMA_PATH)
collection = client.get_or_create_collection(name="documents")

app = FastAPI(title="Local RAG API", version="1.0.0")


def chunk_text(text: str) -> list[str]:
    text = text.strip()
    if not text:
        return []

    chunks: list[str] = []
    start = 0
    step = max(1, CHUNK_SIZE - CHUNK_OVERLAP)

    while start < len(text):
        end = min(len(text), start + CHUNK_SIZE)
        chunk = text[start:end].strip()
        if chunk:
            chunks.append(chunk)
        if end >= len(text):
            break
        start += step

    return chunks


def read_file(path: Path) -> str:
    suffix = path.suffix.lower()
    if suffix == ".pdf":
        reader = PdfReader(str(path))
        return "\n".join(page.extract_text() or "" for page in reader.pages)
    if suffix in {".md", ".txt", ".csv", ".log"}:
        return path.read_text(encoding="utf-8", errors="ignore")
    raise ValueError(f"Tipo no soportado: {suffix}")


async def embed(text: str) -> list[float]:
    async with httpx.AsyncClient(timeout=120) as client_http:
        response = await client_http.post(
            f"{OLLAMA_URL}/api/embed",
            json={"model": EMBED_MODEL, "input": text},
        )
        response.raise_for_status()
        data = response.json()
        embeddings = data.get("embeddings")
        if not embeddings:
            raise RuntimeError("Ollama no devolvió embeddings")
        return embeddings[0]


@app.get("/healthz")
async def healthz() -> dict[str, Any]:
    try:
        async with httpx.AsyncClient(timeout=10) as client_http:
            response = await client_http.get(f"{OLLAMA_URL}/api/tags")
            response.raise_for_status()
        return {"status": "ok", "ollama": "ok", "model": EMBED_MODEL}
    except Exception as exc:
        raise HTTPException(status_code=503, detail=str(exc)) from exc


@app.post("/ingest/file")
async def ingest_file(file: UploadFile = File(...)) -> dict[str, Any]:
    suffix = Path(file.filename or "document.txt").suffix.lower()
    if suffix not in {".pdf", ".md", ".txt", ".csv", ".log"}:
        raise HTTPException(status_code=400, detail="Formato no soportado")

    safe_name = f"{uuid.uuid4().hex}{suffix}"
    destination = Path(DOCUMENTS_PATH) / safe_name
    destination.write_bytes(await file.read())

    try:
        text = read_file(destination)
        chunks = chunk_text(text)
        if not chunks:
            raise HTTPException(status_code=400, detail="El documento no contiene texto utilizable")

        ids = []
        embeddings = []
        documents = []
        metadatas = []

        for index, chunk in enumerate(chunks):
            ids.append(f"{safe_name}:{index}")
            embeddings.append(await embed(chunk))
            documents.append(chunk)
            metadatas.append({
                "source": file.filename or safe_name,
                "stored_file": safe_name,
                "chunk": index,
            })

        collection.add(
            ids=ids,
            embeddings=embeddings,
            documents=documents,
            metadatas=metadatas,
        )

        return {"status": "ok", "source": file.filename, "chunks": len(chunks)}
    except Exception:
        destination.unlink(missing_ok=True)
        raise


@app.post("/query")
async def query(payload: dict[str, Any]) -> dict[str, Any]:
    query_text = str(payload.get("query", "")).strip()
    top_k = int(payload.get("top_k", TOP_K))

    if not query_text:
        raise HTTPException(status_code=400, detail="query es obligatorio")

    vector = await embed(query_text)
    result = collection.query(
        query_embeddings=[vector],
        n_results=top_k,
        include=["documents", "metadatas", "distances"],
    )

    items = []
    docs = result.get("documents", [[]])[0]
    metas = result.get("metadatas", [[]])[0]
    distances = result.get("distances", [[]])[0]

    for idx, doc in enumerate(docs):
        items.append({
            "document": doc,
            "metadata": metas[idx],
            "distance": distances[idx],
        })

    return {"query": query_text, "results": items}
```

## 16.4. `rag/Dockerfile`

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY app/requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt

COPY app/app.py /app/app.py

EXPOSE 8001

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8001"]
```

## 16.5. `docker-rag.yml`

```yaml
services:
  rag:
    build:
      context: ./rag
      dockerfile: Dockerfile
    image: proyecto-ia/rag:latest
    container_name: rag
    restart: unless-stopped
    environment:
      RAG_OLLAMA_URL: ${RAG_OLLAMA_URL}
      RAG_EMBED_MODEL: ${RAG_EMBED_MODEL}
      RAG_CHROMA_PATH: ${RAG_CHROMA_PATH}
      RAG_DOCUMENTS_PATH: ${RAG_DOCUMENTS_PATH}
      RAG_CHUNK_SIZE: ${RAG_CHUNK_SIZE}
      RAG_CHUNK_OVERLAP: ${RAG_CHUNK_OVERLAP}
      RAG_TOP_K: ${RAG_TOP_K}
    ports:
      - "${HOST_BIND_IP}:${RAG_PORT}:8001"
    volumes:
      - "${PROJECT_DIR}/rag/documents:/data/documents"
      - "${PROJECT_DIR}/rag/chroma:/data/chroma"
    networks:
      red-ia:
        aliases:
          - rag

networks:
  red-ia:
    external: true
```

Pruebas:

```bash
curl http://127.0.0.1:11435/healthz | jq
curl -X POST -F "file=@README.md" http://127.0.0.1:11435/ingest/file | jq
curl -X POST \
  -H 'Content-Type: application/json' \
  -d '{"query":"¿Cuál es el objetivo del proyecto?","top_k":5}' \
  http://127.0.0.1:11435/query | jq
```

---

# 17. Crear la red Docker

```bash
docker network create --driver bridge red-ia 2>/dev/null || true
docker network inspect red-ia
```

---

# 18. Despliegue completo

## 18.1. En orden

```bash
cd "$HOME/proyecto"

docker network create --driver bridge red-ia 2>/dev/null || true

docker compose -f docker-ollama.yml up -d
docker compose -f docker-searxng.yml up -d
docker compose -f docker-comfyui.yml up -d
docker compose -f docker-yolo.yml build
docker compose -f docker-yolo.yml up -d
docker compose -f docker-openwebui.yml up -d
docker compose -f docker-hermes-agent.yml up -d
docker compose -f docker-opencode.yml up -d
docker compose -f docker-rag.yml build
docker compose -f docker-rag.yml up -d
```

## 18.2. En una sola orden

```bash
docker compose \
  -f docker-ollama.yml \
  -f docker-openwebui.yml \
  -f docker-hermes-agent.yml \
  -f docker-opencode.yml \
  -f docker-comfyui.yml \
  -f docker-yolo.yml \
  -f docker-searxng.yml \
  -f docker-rag.yml \
  up -d --build
```

---

# 19. Scripts de operación

## 19.1. `scripts/create-network.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

docker network inspect red-ia >/dev/null 2>&1 || \
  docker network create --driver bridge red-ia

echo "red-ia disponible"
```

## 19.2. `scripts/deploy.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

cd "$(dirname "$(dirname "$0")")"
./scripts/create-network.sh

docker compose \
  -f docker-ollama.yml \
  -f docker-openwebui.yml \
  -f docker-hermes-agent.yml \
  -f docker-opencode.yml \
  -f docker-comfyui.yml \
  -f docker-yolo.yml \
  -f docker-searxng.yml \
  -f docker-rag.yml \
  up -d --build

docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

## 19.3. `scripts/down.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$(dirname "$0")")"

docker compose \
  -f docker-ollama.yml \
  -f docker-openwebui.yml \
  -f docker-hermes-agent.yml \
  -f docker-opencode.yml \
  -f docker-comfyui.yml \
  -f docker-yolo.yml \
  -f docker-searxng.yml \
  -f docker-rag.yml \
  down
```

## 19.4. `scripts/status.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
```

## 19.5. Activar scripts

```bash
chmod +x scripts/*.sh
```

---

# 20. Verificación completa

## 20.1. Contenedores

```bash
docker ps
```

Debe aparecer como mínimo:

```text
ollama
openwebui
hermes-agent
opencode
comfyui
yolo
searxng
rag
```

## 20.2. Servicios HTTP

```bash
curl -fsS http://127.0.0.1:11434/api/tags >/dev/null && echo 'Ollama OK'
curl -fsS 'http://127.0.0.1:8080/search?q=test&format=json' >/dev/null && echo 'SearXNG OK'
curl -fsS http://127.0.0.1:5000/healthz >/dev/null && echo 'YOLO OK'
curl -fsS http://127.0.0.1:11435/healthz >/dev/null && echo 'RAG OK'
```

## 20.3. GPU

```bash
nvidia-smi
docker exec ollama nvidia-smi
docker exec comfyui nvidia-smi
docker exec yolo nvidia-smi
```

Open WebUI y SearXNG no requieren GPU para cumplir su función principal.

---

# 21. Integración interna exacta

## 21.1. Ollama ↔ Open WebUI

```text
Open WebUI -> http://ollama:11434
```

## 21.2. SearXNG ↔ Open WebUI

```text
Open WebUI -> http://searxng:8080/search
```

Variables:

```text
ENABLE_WEB_SEARCH=true
WEB_SEARCH_ENGINE=searxng
SEARXNG_QUERY_URL=http://searxng:8080/search
```

## 21.3. Ollama ↔ Hermes Agent

```text
Hermes -> http://ollama:11434/v1
```

Seleccionar un modelo con tool calling compatible con el uso de agente.

## 21.4. Ollama ↔ OpenCode

La URL del proveedor local debe apuntar al endpoint compatible con OpenAI de Ollama:

```text
http://ollama:11434/v1
```

La configuración concreta del proveedor depende de la versión actual de OpenCode.

## 21.5. RAG ↔ Ollama

```text
RAG -> http://ollama:11434/api/embed
```

Modelo por defecto:

```text
nomic-embed-text
```

## 21.6. YOLO ↔ otros servicios

```text
POST http://yolo:5000/predict
```

## 21.7. ComfyUI ↔ otros servicios

```text
http://comfyui:8188
```

Para automatizar workflows debe utilizarse la API correspondiente a la versión desplegada.

---

# 22. Firewall

Puertos del host:

```text
3000   Open WebUI
5000   YOLO
8000   Hermes
8080   SearXNG
8188   ComfyUI
8443   OpenCode
11434  Ollama
11435  RAG
```

Comprobar:

```bash
sudo ss -ltnp | grep -E ':3000|:5000|:8000|:8080|:8188|:8443|:11434|:11435'
```

Para una LAN de ejemplo:

```bash
sudo ufw allow from 192.168.1.0/24 to any port 3000 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 5000 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 8000 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 8080 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 8188 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 8443 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 11434 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 11435 proto tcp
```

Sustituir la subred por la real. No abrir estos puertos a Internet sin autenticación, TLS y control de acceso.

---

# 23. Volúmenes y persistencia

| Host | Servicio | Contenido |
|---|---|---|
| `$HOME/proyecto/ollama` | Ollama | modelos y datos |
| `$HOME/proyecto/openwebui` | Open WebUI | usuarios, chats y configuración |
| `$HOME/proyecto/hermes` | Hermes | configuración y sesiones |
| `$HOME/proyecto/opencode` | OpenCode | workspace |
| `$HOME/proyecto/comfyui/models` | ComfyUI | modelos generativos |
| `$HOME/proyecto/comfyui/input` | ComfyUI | entradas |
| `$HOME/proyecto/comfyui/output` | ComfyUI | resultados |
| `$HOME/proyecto/comfyui/custom_nodes` | ComfyUI | custom nodes |
| `$HOME/proyecto/yolo/models` | YOLO | cache/modelos |
| `$HOME/proyecto/yolo/data` | YOLO | datos |
| `$HOME/proyecto/searxng` | SearXNG | configuración |
| `$HOME/proyecto/rag/documents` | RAG | documentos |
| `$HOME/proyecto/rag/chroma` | RAG | índice vectorial |

---

# 24. Backups

## 24.1. Backup completo

```bash
cd "$HOME/proyecto"
DATE=$(date +%Y%m%d-%H%M%S)

tar -czf "backups/proyecto-ia-$DATE.tar.gz" \
  .env \
  ollama \
  openwebui \
  hermes \
  opencode \
  comfyui \
  yolo \
  searxng \
  rag
```

## 24.2. Backup ligero de configuración

```bash
DATE=$(date +%Y%m%d-%H%M%S)
mkdir -p "backups/$DATE"
cp .env "backups/$DATE/"
cp docker-*.yml "backups/$DATE/"
cp -a searxng "backups/$DATE/"
```

---

# 25. Mantenimiento y actualización

## 25.1. Estado

```bash
docker ps
docker stats
nvidia-smi
df -h
du -sh "$HOME/proyecto"/*
```

## 25.2. Pull y recreación

```bash
docker compose \
  -f docker-ollama.yml \
  -f docker-openwebui.yml \
  -f docker-hermes-agent.yml \
  -f docker-opencode.yml \
  -f docker-comfyui.yml \
  -f docker-yolo.yml \
  -f docker-searxng.yml \
  -f docker-rag.yml \
  pull

docker compose \
  -f docker-ollama.yml \
  -f docker-openwebui.yml \
  -f docker-hermes-agent.yml \
  -f docker-opencode.yml \
  -f docker-comfyui.yml \
  -f docker-yolo.yml \
  -f docker-searxng.yml \
  -f docker-rag.yml \
  up -d --build
```

## 25.3. Fijar versiones

Durante laboratorio se puede usar `latest`. Antes de producción, fijar tags o digests de imágenes que hayan sido probados.

---

# 26. Gestión de modelos Ollama

```bash
docker exec -it ollama ollama list
docker exec -it ollama ollama pull <modelo>
docker exec -it ollama ollama rm <modelo>
```

Ver tamaño:

```bash
du -sh "$HOME/proyecto/ollama"
```

---

# 27. Permisos

Buscar ficheros root:

```bash
find "$HOME/proyecto" -user root -ls | head -n 50
```

No ejecutar `chown -R` sobre todo el proyecto sin comprobar primero qué usuario utiliza cada imagen.

Ejemplo solo para Open WebUI cuando proceda:

```bash
sudo chown -R "$USER":"$USER" "$HOME/proyecto/openwebui"
```

---

# 28. Logs y diagnóstico

```bash
docker logs --tail 200 -f ollama
docker logs --tail 200 -f openwebui
docker logs --tail 200 -f hermes-agent
docker logs --tail 200 -f opencode
docker logs --tail 200 -f comfyui
docker logs --tail 200 -f yolo
docker logs --tail 200 -f searxng
docker logs --tail 200 -f rag
```

Diagnóstico del host Docker:

```bash
journalctl -u docker --since today --no-pager
```

---

# 29. Errores comunes

## 29.1. `permission denied`

```bash
sudo usermod -aG docker "$USER"
newgrp docker
```

## 29.2. `port is already allocated`

```bash
sudo ss -ltnp | grep ':3000'
```

Cambiar la variable correspondiente en `.env`.

## 29.3. Open WebUI no encuentra Ollama

```bash
docker exec openwebui getent hosts ollama
docker exec openwebui sh -c 'wget -qO- http://ollama:11434/api/tags'
```

## 29.4. Open WebUI no encuentra SearXNG

```bash
docker exec openwebui getent hosts searxng
docker exec openwebui sh -c 'wget -qO- "http://searxng:8080/search?q=test&format=json"'
```

## 29.5. Hermes no encuentra Ollama

La URL desde el contenedor es:

```text
http://ollama:11434/v1
```

No usar `localhost`.

## 29.6. GPU no visible

```bash
nvidia-smi
nvidia-container-cli info
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

## 29.7. RAG no genera embeddings

```bash
docker exec ollama ollama list | grep nomic-embed-text
```

Si no aparece:

```bash
docker exec ollama ollama pull nomic-embed-text
```

Prueba directa:

```bash
curl http://127.0.0.1:11434/api/embed \
  -H 'Content-Type: application/json' \
  -d '{"model":"nomic-embed-text","input":"texto de prueba"}' | jq
```

---

# 30. Seguridad

- `.env` debe tener permisos `600`.
- No publicar APIs administrativas directamente a Internet.
- No utilizar `privileged: true` por comodidad.
- Limitar puertos con UFW.
- No montar el sistema raíz del host dentro de los contenedores.
- Revisar custom nodes de ComfyUI antes de instalarlos.
- Tratar los documentos RAG como entradas no confiables.
- Hacer backups antes de actualizaciones.
- Fijar versiones de imágenes en producción.
- Usar TLS y reverse proxy para acceso externo.

Archivo `.gitignore` recomendado:

```gitignore
.env
backups/
ollama/
openwebui/
hermes/
opencode/
comfyui/models/
comfyui/output/
yolo/models/
rag/chroma/
rag/documents/
__pycache__/
*.pyc
```

---

# 31. Procedimiento de rollback

1. Restaurar el backup de configuración.
2. Volver al tag de imagen previamente validado.
3. Recrear el servicio afectado.
4. Ejecutar las pruebas de salud.

Ejemplo:

```bash
docker compose -f docker-openwebui.yml up -d
curl -fsS http://127.0.0.1:3000 >/dev/null
```

Para rollbacks fiables, guardar el `docker inspect` o el inventario de tags/digests utilizados en cada versión.

---

# 32. Verificación de Compose

Cada fichero:

```bash
for f in docker-*.yml; do
  echo "===== $f ====="
  docker compose -f "$f" config >/dev/null
  echo "OK"
done
```

Pila completa:

```bash
docker compose \
  -f docker-ollama.yml \
  -f docker-openwebui.yml \
  -f docker-hermes-agent.yml \
  -f docker-opencode.yml \
  -f docker-comfyui.yml \
  -f docker-yolo.yml \
  -f docker-searxng.yml \
  -f docker-rag.yml \
  config >/tmp/proyecto-ia-compose.yml
```

---

# 33. Validación final de aceptación

La especificación del proyecto establece como criterios que los Compose no tengan errores sintácticos, que los contenedores GPU utilicen NVIDIA/CUDA, que no haya colisiones de puertos y que el procedimiento esté explicado paso a paso. [Especificación del proyecto, sección 6]

Checklist:

```text
[ ] Ubuntu soportado
[ ] nvidia-smi funciona
[ ] NVIDIA Container Toolkit configurado
[ ] docker compose funciona
[ ] red-ia existe
[ ] Ollama responde
[ ] LLM descargado
[ ] embedding model descargado
[ ] Open WebUI responde
[ ] Open WebUI -> Ollama funciona
[ ] SearXNG devuelve JSON
[ ] Open WebUI -> SearXNG funciona
[ ] Hermes configurado
[ ] Hermes -> Ollama funciona
[ ] OpenCode Server responde
[ ] ComfyUI responde
[ ] ComfyUI ve la GPU
[ ] YOLO /healthz responde
[ ] YOLO /predict responde
[ ] RAG /healthz responde
[ ] RAG ingesta documentos
[ ] RAG devuelve fragmentos
[ ] backups configurados
[ ] firewall configurado
```

---

# 34. Consideraciones específicas para una RTX 3060 de 12 GB

El proyecto contempla equipos con RTX 3060 de 12 GB. El límite operativo más importante es la VRAM: cargar simultáneamente modelos grandes, ComfyUI y tareas de visión puede provocar falta de memoria.

Recomendaciones:

- utilizar modelos cuantizados cuando sea posible;
- mantener una concurrencia baja en Ollama;
- evitar ejecutar simultáneamente inferencias de gran consumo en Ollama y ComfyUI;
- monitorizar con `nvidia-smi`;
- reducir contexto/batch/tamaño de modelo en caso de OOM;
- mantener suficiente margen de VRAM para picos.

La elección del LLM definitivo debe validarse sobre el equipo real. El modelo del `.env` es una referencia y no una garantía de ajuste para todas las GPUs de 12 GB.

---

# 35. Observaciones sobre la especificación original

## 35.1. Puertos

La especificación original definía:

- Hermes: interno 8000 / externo 8000.
- OpenCode: interno 8080 / externo 8443.
- RAG: interno 11434 / externo 11434.

Este manual conserva los puertos externos donde es posible, pero usa los puertos internos actuales o no conflictivos:

| Servicio | Especificación | Este manual | Motivo |
|---|---:|---:|---|
| Hermes | 8000 | 8642 interno / 8000 host | gateway actual |
| OpenCode | 8080 | 4096 interno / 8443 host | servidor actual |
| RAG | 11434 | 8001 interno / 11435 host | evita colisión con Ollama |

## 35.2. GPU

La especificación asigna GPU a varios servicios. Operativamente, GPU debe reservarse a los procesos que ejecutan inferencia. Open WebUI y SearXNG pueden funcionar sin GPU.

## 35.3. RAG

RAG se ha implementado como microservicio separado porque la especificación lo trata como servicio propio, aunque conceptualmente RAG es una arquitectura/técnica y no un único producto.

---

# 36. Resumen de URLs

| Servicio | URL desde el host | URL desde `red-ia` |
|---|---|---|
| Ollama | `http://IP:11434` | `http://ollama:11434` |
| Open WebUI | `http://IP:3000` | `http://openwebui:8080` |
| Hermes | `http://IP:8000` | `http://hermes-agent:8642` |
| OpenCode | `http://IP:8443` | `http://opencode:4096` |
| ComfyUI | `http://IP:8188` | `http://comfyui:8188` |
| YOLO | `http://IP:5000` | `http://yolo:5000` |
| SearXNG | `http://IP:8080` | `http://searxng:8080` |
| RAG | `http://IP:11435` | `http://rag:8001` |

---

# 37. Fuentes oficiales

- Docker Engine para Ubuntu: <https://docs.docker.com/engine/install/ubuntu/>
- NVIDIA Container Toolkit: <https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html>
- Ollama: <https://docs.ollama.com/>
- Ollama embeddings: <https://ollama.com/library/nomic-embed-text>
- Open WebUI: <https://docs.openwebui.com/getting-started/quick-start/>
- Open WebUI + Ollama: <https://docs.openwebui.com/getting-started/quick-start/connect-a-provider/starting-with-ollama/>
- Open WebUI + SearXNG: <https://docs.openwebui.com/features/chat-conversations/web-search/providers/searxng/>
- Hermes Agent: <https://github.com/NousResearch/hermes-agent>
- Hermes Agent + Ollama local: <https://github.com/NousResearch/hermes-agent/blob/main/website/docs/guides/local-ollama-setup.md>
- Hermes Agent Docker: <https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/docker.md>
- OpenCode Server: <https://opencode.ai/docs/server/>
- OpenCode container: <https://github.com/anomalyco/opencode/pkgs/container/opencode>
- SearXNG Docker: <https://docs.searxng.org/admin/installation-docker>
- Ultralytics Docker: <https://docs.ultralytics.com/guides/docker-quickstart/>
- ComfyUI: <https://github.com/comfyanonymous/ComfyUI>

---

# 38. Comando de validación integral

Ejecutar desde `$HOME/proyecto`:

```bash
set -e

printf '\n[1] Docker\n'
docker --version
docker compose version

printf '\n[2] NVIDIA host\n'
nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv

printf '\n[3] Red\n'
docker network inspect red-ia >/dev/null
echo 'red-ia: OK'

printf '\n[4] Compose syntax\n'
for f in docker-*.yml; do
  docker compose -f "$f" config >/dev/null
  echo "$f: OK"
done

printf '\n[5] Containers\n'
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'

printf '\n[6] Ollama\n'
curl -fsS http://127.0.0.1:11434/api/tags >/dev/null
echo 'Ollama: OK'

printf '\n[7] SearXNG\n'
curl -fsS 'http://127.0.0.1:8080/search?q=test&format=json' >/dev/null
echo 'SearXNG: OK'

printf '\n[8] YOLO\n'
curl -fsS http://127.0.0.1:5000/healthz >/dev/null
echo 'YOLO: OK'

printf '\n[9] RAG\n'
curl -fsS http://127.0.0.1:11435/healthz >/dev/null
echo 'RAG: OK'

printf '\n[10] GPU containers\n'
docker exec ollama nvidia-smi >/dev/null
docker exec comfyui nvidia-smi >/dev/null
docker exec yolo nvidia-smi >/dev/null
echo 'GPU containers: OK'

printf '\nVALIDACION FINALIZADA\n'
```

---

# 39. Conclusión

La pila queda organizada por contenedores independientes, con una red Docker común, persistencia en `$HOME/proyecto`, aceleración NVIDIA donde aporta valor y endpoints internos definidos mediante DNS de Docker.

La arquitectura centra la inferencia en Ollama, utiliza Open WebUI como interfaz, Hermes Agent y OpenCode como herramientas de trabajo con modelos, SearXNG como buscador, ComfyUI y YOLO como componentes de generación/visión y un RAG independiente basado en embeddings de Ollama y almacenamiento vectorial.

Antes de producción deben validarse las versiones concretas de cada imagen, fijarse tags o digests, proteger las interfaces con autenticación y TLS cuando proceda, restringirse los puertos y automatizarse los backups y el rollback.

---

# Anexo A. Correspondencia con la especificación original

| Requisito original | Tratamiento |
|---|---|
| Ubuntu Server 24.04/26.04 | Cubierto |
| NVIDIA GPU | Cubierto |
| Docker + Compose | Cubierto |
| Red `red-ia` | Cubierto |
| Ollama | Cubierto |
| Open WebUI | Cubierto |
| Hermes Agent | Cubierto |
| OpenCode | Cubierto |
| ComfyUI | Cubierto con advertencia sobre imagen de terceros |
| YOLO | Cubierto con API propia |
| SearXNG | Cubierto |
| RAG separado | Cubierto |
| Un fichero Compose por servicio | Cubierto |
| `.env` | Cubierto |
| Arranque y logs | Cubierto |
| GPU/NVIDIA | Cubierto |
| Backups | Cubierto |
| Actualizaciones | Cubierto |
| Integración interna | Cubierto |
| Sin colisiones de puertos | Cubierto con ajustes documentados |
| Guía paso a paso | Cubierto |

---

# Anexo B. Notas de uso

- Sustituir `USUARIO` en `.env` por el usuario real del servidor.
- Cambiar inmediatamente las claves marcadas como `CAMBIAR_*`.
- Validar la imagen de ComfyUI antes de producción.
- Validar el modelo LLM elegido para la VRAM real.
- Registrar la versión de las imágenes y modelos después de cada despliegue.
- Mantener copia de seguridad del `.env` de forma segura y no versionarlo en Git.

EOF
wc -l /mnt/data/manual_tecnico_stack_ia_docker.md
wc -c /mnt/data/manual_tecnico_stack_ia_docker.md
sed -n '315,345p' /mnt/data/manual_tecnico_stack_ia_docker.md

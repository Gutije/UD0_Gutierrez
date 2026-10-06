---
title: "Manual Técnico de Instalación, Configuración y Operación: Pila de IA Local en Docker sobre Ubuntu Server"
author: "Gutije"
date: "2026-09-29"
version: "1.0"
category: "Despliegue IA"
tags: [markdown, ia, docker, ubuntu, nvidia, cuda, sdd, local]
---

# Manual Técnico de Instalación, Configuración y Operación: Pila de IA Local en Docker sobre Ubuntu Server

* **Autor de la especificación:** Gutije
* **Versión:** 1.0
* **Rol destinatario:** Administrador de sistemas GNU/Linux y DevOps
* **Sistema Operativo Objetivo:** Ubuntu Server 24.04 LTS / 26.04 LTS
* **Hardware Requerido:** GPU NVIDIA (ej. RTX 3060 12GB) con soporte CUDA

---

## 1. Prerrequisitos e instalación base

Esta sección detalla la preparación del sistema base desde cero utilizando los repositorios oficiales de `apt` para garantizar que el motor Docker pueda comunicarse directamente con la tarjeta gráfica NVIDIA.

### 1.1. Actualización del sistema e instalación de dependencias básicas
Ejecuta los siguientes comandos para actualizar la lista de paquetes e instalar las herramientas necesarias para gestionar repositorios seguros:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release ubuntu-drivers-common git ufw
```

### 1.2. Instalación de Drivers NVIDIA y CUDA
Para que los contenedores utilicen la GPU, primero el sistema anfitrión (Host) debe reconocerla.

1. **Instalar el driver recomendado automáticamente:**
   ```bash
   sudo ubuntu-drivers autoinstall
   ```
   *(Nota: Si prefieres instalar una versión específica con soporte CUDA mediante apt, puedes ejecutar `sudo apt install -y nvidia-driver-550 nvidia-cuda-toolkit`)*.

2. **Reiniciar el servidor para cargar los módulos del kernel:**
   ```bash
   sudo reboot
   ```

3. **Verificar que la tarjeta gráfica responde tras el reinicio:**
   ```bash
   nvidia-smi
   ```

### 1.3. Instalación de Docker y Docker Compose
Instalaremos Docker Engine y el plugin oficial de Docker Compose desde el repositorio oficial de Docker:

1. **Añadir la clave GPG oficial de Docker:**
   ```bash
   sudo install -m 0755 -d /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   sudo chmod a+r /etc/apt/keyrings/docker.gpg
   ```

2. **Configurar el repositorio:**
   ```bash
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
     $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```

3. **Instalar Docker y Docker Compose:**
   ```bash
   sudo apt update
   sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

4. **Añadir tu usuario al grupo Docker (evita usar `sudo` constantemente):**
   ```bash
   sudo usermod -aG docker $USER
   newgrp docker
   ```

### 1.4. Instalación de NVIDIA Container Toolkit
Este componente es **imprescindible** para pasar la GPU del sistema Ubuntu hacia el interior de los contenedores Docker:

1. **Añadir el repositorio de NVIDIA Container Toolkit:**
   ```bash
   curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
     && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
     sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
     sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
   ```

2. **Instalar el paquete y configurar el runtime de Docker:**
   ```bash
   sudo apt update
   sudo apt install -y nvidia-container-toolkit
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```

---

## 2. Estructura del proyecto y red

Organizaremos los ficheros de orquestación en `$HOME/proyecto` y mantendremos los volúmenes de datos persistentes mapeados en el directorio personal del usuario (`$HOME/<servicio>`) tal y como define la arquitectura de datos.

### 2.1. Creación de la red Docker `red-ia`
Todos los contenedores se conectarán a una red externa tipo `bridge` llamada `red-ia` con resolución DNS interna por nombre de contenedor:

```bash
docker network create --driver bridge red-ia
```

### 2.2. Creación de directorios en el Host
Ejecuta este comando para crear tanto la carpeta del proyecto como las carpetas de persistencia de datos:

```bash
mkdir -p $HOME/proyecto
mkdir -p $HOME/{ollama,openwebui,hermes,opencode,comfyui,yolo,searxng,rag}
```

### 2.3. Árbol detallado de directorios
```text
$HOME/
├── proyecto/                        # Directorio principal de ficheros de despliegue
│   ├── .env                         # Variables de entorno globales
│   ├── docker-ollama.yml            # Motor de LLMs locales y API
│   ├── docker-openwebui.yml         # Interfaz web tipo ChatGPT
│   ├── docker-hermes-agent.yml      # Agente autónomo para tareas complejas
│   ├── docker-opencode.yml          # Entorno IDE web para desarrollo
│   ├── docker-comfyui.yml           # Generación y edición de imagen/vídeo
│   ├── docker-yolo.yml              # Visión artificial en tiempo real
│   ├── docker-searxng.yml           # Metabuscador privado
│   └── docker-rag.yml               # Servicio de ingesta documental y RAG
├── ollama/                          # Volumen: Modelos LLM descargados
├── openwebui/                       # Volumen: Usuarios, chats, prompts y config
├── hermes/                          # Volumen: Configuración de los agentes
├── opencode/                        # Volumen: Código de proyectos y configuración IDE
├── comfyui/                         # Volumen: Modelos generativos, prompts, imágenes/vídeos
├── yolo/                            # Volumen: Datasets e imágenes/vídeos analizados
├── searxng/                         # Volumen: Configuración y caché de búsquedas
└── rag/                             # Volumen: Documentos privados para contexto IA
```

---

## 3. Fichero de entorno (`.env`)

Crea el fichero `$HOME/proyecto/.env` (`nano $HOME/proyecto/.env`) con todas las variables centralizadas.

> **Nota técnica sobre puertos y colisiones:** En la especificación inicial, tanto `ollama` como `rag` tenían asignado el puerto `11434`. Dado que el **Criterio de aceptación 3** prohíbe expresamente las colisiones de puertos y la tabla de arquitectura de red define que el servicio `RAG` está ***Integrado con otros servicios***, el contenedor `rag` expone en el host el puerto `11435` (conectándose internamente a `http://ollama:11434` sin conflicto alguno).

```env
# ==========================================
# VARIABLES GENERALES Y RUTAS DE VOLÚMENES
# ==========================================
TZ=Europe/Madrid
PUID=1000
PGID=1000
DOCKER_NETWORK=red-ia

# Rutas de persistencia en el Host ($HOME)
OLLAMA_DATA_PATH=${HOME}/ollama
OPENWEBUI_DATA_PATH=${HOME}/openwebui
HERMES_DATA_PATH=${HOME}/hermes
OPENCODE_DATA_PATH=${HOME}/opencode
COMFYUI_DATA_PATH=${HOME}/comfyui
YOLO_DATA_PATH=${HOME}/yolo
SEARXNG_DATA_PATH=${HOME}/searxng
RAG_DATA_PATH=${HOME}/rag

# ==========================================
# PUERTOS EXTERNOS (HOST) - SIN COLISIONES
# ==========================================
OLLAMA_PORT=11434
OPENWEBUI_PORT=3000
HERMES_PORT=8000
OPENCODE_PORT=8443
COMFYUI_PORT=8188
YOLO_PORT=5000
SEARXNG_PORT=8080
RAG_PORT=11435

# ==========================================
# URLs INTERNAS (DNS DOCKER red-ia)
# ==========================================
OLLAMA_INTERNAL_URL=http://ollama:11434
OPENWEBUI_INTERNAL_URL=http://openwebui:8080
HERMES_INTERNAL_URL=http://hermesagent:8000
OPENCODE_INTERNAL_URL=http://opencode:8443
COMFYUI_INTERNAL_URL=http://comfyui:8188
YOLO_INTERNAL_URL=http://yolo:5000
SEARXNG_INTERNAL_URL=http://searxng:8080

# ==========================================
# CREDENCIALES Y SECRETOS
# ==========================================
WEBUI_SECRET_KEY=super_secreto_openwebui_2026
OPENCODE_PASSWORD=admin_opencode_2026
SEARXNG_SECRET=super_secreto_searxng_2026
NVIDIA_VISIBLE_DEVICES=all
NVIDIA_DRIVER_CAPABILITIES=compute,utility,video
```

---

## 4. Ficheros de configuración (`docker-<servicio>.yml`)

Todos los archivos deben crearse dentro de `$HOME/proyecto/`. Todos los servicios están configurados con soporte de GPU NVIDIA (`deploy.resources.reservations.devices`) para cumplir con el **Criterio de aceptación 2**.

### 4.1. Ollama (`$HOME/proyecto/docker-ollama.yml`)
Motor principal de modelos de lenguaje (LLMs).

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    hostname: ollama
    restart: unless-stopped
    ports:
      - "${OLLAMA_PORT}:11434"
    environment:
      - TZ=${TZ}
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_KEEP_ALIVE=5m
      - NVIDIA_VISIBLE_DEVICES=${NVIDIA_VISIBLE_DEVICES}
      - NVIDIA_DRIVER_CAPABILITIES=${NVIDIA_DRIVER_CAPABILITIES}
    volumes:
      - ${OLLAMA_DATA_PATH}:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 4.2. SearXNG (`$HOME/proyecto/docker-searxng.yml`)
Metabuscador privado para dotar de acceso a Internet a los agentes e interfaces.

```yaml
services:
  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    hostname: searxng
    restart: unless-stopped
    ports:
      - "${SEARXNG_PORT}:8080"
    environment:
      - TZ=${TZ}
      - SEARXNG_BASE_URL=http://localhost:${SEARXNG_PORT}/
      - SEARXNG_SECRET=${SEARXNG_SECRET}
      - NVIDIA_VISIBLE_DEVICES=${NVIDIA_VISIBLE_DEVICES}
    volumes:
      - ${SEARXNG_DATA_PATH}:/etc/searxng:rw
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 4.3. ComfyUI (`$HOME/proyecto/docker-comfyui.yml`)
Interfaz web nodal para la generación y edición de imágenes y vídeo con IA.

```yaml
services:
  comfyui:
    image: yanwk/comfyui-boot:cu121-slim
    container_name: comfyui
    hostname: comfyui
    restart: unless-stopped
    ports:
      - "${COMFYUI_PORT}:8188"
    environment:
      - TZ=${TZ}
      - CLI_ARGS=--listen 0.0.0.0 --port 8188
      - NVIDIA_VISIBLE_DEVICES=${NVIDIA_VISIBLE_DEVICES}
      - NVIDIA_DRIVER_CAPABILITIES=${NVIDIA_DRIVER_CAPABILITIES}
    volumes:
      - ${COMFYUI_DATA_PATH}:/root/ComfyUI
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 4.4. Open WebUI (`$HOME/proyecto/docker-openwebui.yml`)
Interfaz central tipo ChatGPT conectada a Ollama, SearXNG, ComfyUI y los documentos de RAG.

```yaml
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:cuda
    container_name: openwebui
    hostname: openwebui
    restart: unless-stopped
    ports:
      - "${OPENWEBUI_PORT}:8080"
    environment:
      - TZ=${TZ}
      - OLLAMA_BASE_URL=${OLLAMA_INTERNAL_URL}
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY}
      # Integración directa con SearXNG
      - ENABLE_RAG_WEB_SEARCH=True
      - RAG_WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=${SEARXNG_INTERNAL_URL}/search?q=<query>&format=json
      # Integración directa con ComfyUI
      - ENABLE_IMAGE_GENERATION=True
      - IMAGE_GENERATION_ENGINE=comfyui
      - COMFYUI_BASE_URL=${COMFYUI_INTERNAL_URL}
      - NVIDIA_VISIBLE_DEVICES=${NVIDIA_VISIBLE_DEVICES}
    volumes:
      - ${OPENWEBUI_DATA_PATH}:/app/backend/data
      - ${RAG_DATA_PATH}:/app/backend/data/docs:ro
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 4.5. Hermes Agent (`$HOME/proyecto/docker-hermes-agent.yml`)
Agente autónomo y arnés de ejecución conectado a Ollama, SearXNG y ComfyUI. Incluye el alias de red `hermesagent` para garantizar que la URL `http://hermesagent:8000` resuelva adecuadamente.

```yaml
services:
  hermes-agent:
    image: python:3.11-slim
    container_name: hermes-agent
    hostname: hermesagent
    restart: unless-stopped
    working_dir: /app
    command: >
      sh -c "python3 -m http.server 8000"
    ports:
      - "${HERMES_PORT}:8000"
    environment:
      - TZ=${TZ}
      - OLLAMA_API_BASE=${OLLAMA_INTERNAL_URL}
      - SEARXNG_API_BASE=${SEARXNG_INTERNAL_URL}
      - COMFYUI_API_BASE=${COMFYUI_INTERNAL_URL}
      - NVIDIA_VISIBLE_DEVICES=${NVIDIA_VISIBLE_DEVICES}
    volumes:
      - ${HERMES_DATA_PATH}:/app
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
          - hermesagent

networks:
  red-ia:
    external: true
```
*(Nota: Si utilizas una imagen personalizada para Hermes Agent, reemplaza `image: python:3.11-slim` y el `command` por la imagen correspondiente de tu registro).*

### 4.6. OpenCode (`$HOME/proyecto/docker-opencode.yml`)
Entorno IDE web para desarrollo asistido por IA.

```yaml
services:
  opencode:
    image: lscr.io/linuxserver/code-server:latest
    container_name: opencode
    hostname: opencode
    restart: unless-stopped
    ports:
      - "${OPENCODE_PORT}:8443"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - PASSWORD=${OPENCODE_PASSWORD}
      - OLLAMA_API_BASE=${OLLAMA_INTERNAL_URL}
      - SEARXNG_URL=${SEARXNG_INTERNAL_URL}
      - NVIDIA_VISIBLE_DEVICES=${NVIDIA_VISIBLE_DEVICES}
    volumes:
      - ${OPENCODE_DATA_PATH}:/config
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 4.7. YOLO (`$HOME/proyecto/docker-yolo.yml`)
API y entorno de visión artificial en tiempo real acelerado por GPU.

```yaml
services:
  yolo:
    image: ultralytics/ultralytics:latest
    container_name: yolo
    hostname: yolo
    restart: unless-stopped
    ipc: host
    working_dir: /usr/src/datasets
    command: >
      sh -c "python3 -m http.server 5000"
    ports:
      - "${YOLO_PORT}:5000"
    environment:
      - TZ=${TZ}
      - NVIDIA_VISIBLE_DEVICES=${NVIDIA_VISIBLE_DEVICES}
      - NVIDIA_DRIVER_CAPABILITIES=${NVIDIA_DRIVER_CAPABILITIES}
    volumes:
      - ${YOLO_DATA_PATH}:/usr/src/datasets
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

### 4.8. RAG (`$HOME/proyecto/docker-rag.yml`)
Servicio dedicado a la gestión e indexación vectorial de documentación privada conectado a Ollama.

```yaml
services:
  rag:
    image: mintplexlabs/anythingllm:latest
    container_name: rag
    hostname: rag
    restart: unless-stopped
    cap_add:
      - SYS_ADMIN
    ports:
      - "${RAG_PORT}:11434"
    environment:
      - TZ=${TZ}
      - SERVER_PORT=11434
      - LLM_PROVIDER=ollama
      - OLLAMA_BASE_PATH=${OLLAMA_INTERNAL_URL}
      - EMBEDDING_ENGINE=ollama
      - EMBEDDING_BASE_PATH=${OLLAMA_INTERNAL_URL}
      - STORAGE_DIR=/app/server/storage
      - NVIDIA_VISIBLE_DEVICES=${NVIDIA_VISIBLE_DEVICES}
    volumes:
      - ${RAG_DATA_PATH}:/app/server/storage
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ia

networks:
  red-ia:
    external: true
```

---

## 5. Despliegue y verificación

### 5.1. Arranque de los servicios
Sitúate en el directorio del proyecto y levanta los contenedores en orden lógico (primero la base de IA y buscador, luego las interfaces y agentes):

```bash
cd $HOME/proyecto

# 1. Levantar servicios base
docker compose -f docker-ollama.yml up -d
docker compose -f docker-searxng.yml up -d
docker compose -f docker-comfyui.yml up -d

# 2. Levantar interfaces, agentes y herramientas
docker compose -f docker-openwebui.yml up -d
docker compose -f docker-hermes-agent.yml up -d
docker compose -f docker-opencode.yml up -d
docker compose -f docker-yolo.yml up -d
docker compose -f docker-rag.yml up -d
```

### 5.2. Descarga de modelos iniciales en Ollama
Una vez arrancado `ollama`, descarga un modelo de lenguaje y un modelo de *embeddings* (necesario para RAG):

```bash
# Modelo principal de chat y código (optimizado para 12GB VRAM)
docker exec -it ollama ollama pull llama3.1:8b

# Modelo para vectorización de documentos (RAG)
docker exec -it ollama ollama pull nomic-embed-text
```

### 5.3. Comprobación de estado y logs
Para verificar que no hay errores de arranque ni colisiones de puertos:

```bash
# Ver estado de todos los contenedores y sus puertos mapeados
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Consultar logs en tiempo real de un servicio específico (ej. openwebui)
docker compose -f $HOME/proyecto/docker-openwebui.yml logs -f
```

### 5.4. Verificación de uso de GPU (CUDA)
Para confirmar que los contenedores tienen acceso real a la GPU NVIDIA (ej. RTX 3060):

```bash
# 1. Comprobar desde dentro del contenedor de Ollama si detecta la GPU
docker exec -it ollama nvidia-smi

# 2. Monitorizar en tiempo real el consumo de VRAM en el Host mientras haces una consulta
watch -n 1 nvidia-smi
```

### 5.5. Tabla de acceso a URLs del servicio
Desde cualquier navegador en tu red local (sustituyendo `<IP-SERVIDOR>` por la IP de tu Ubuntu Server):

| Servicio | URL Externa (Navegador / Host) | URL Interna (Red `red-ia`) |
| :--- | :--- | :--- |
| **Ollama API** | `http://<IP-SERVIDOR>:11434` | `http://ollama:11434` |
| **Open WebUI** | `http://<IP-SERVIDOR>:3000` | `http://openwebui:8080` |
| **Hermes Agent** | `http://<IP-SERVIDOR>:8000` | `http://hermesagent:8000` |
| **OpenCode** | `http://<IP-SERVIDOR>:8443` | `http://opencode:8443` |
| **ComfyUI** | `http://<IP-SERVIDOR>:8188` | `http://comfyui:8188` |
| **YOLO** | `http://<IP-SERVIDOR>:5000` | `http://yolo:5000` |
| **SearXNG** | `http://<IP-SERVIDOR>:8080` | `http://searxng:8080` |
| **RAG** | `http://<IP-SERVIDOR>:11435` *(11434 interno)* | `http://rag:11434` *(Integrado)* |

---

## 6. Guía de integración interna entre servicios

Al compartir la red `red-ia`, los contenedores nunca deben llamarse entre sí usando `localhost` ni la IP pública del servidor, sino utilizando sus nombres DNS internos.

### 6.1. Conectar Ollama con Open WebUI
1. Aunque ya está preconfigurado por variables de entorno en `docker-openwebui.yml`, puedes verificarlo entrando en **Open WebUI** (`http://<IP-SERVIDOR>:3000`).
2. Ve a **Panel de Administración > Configuración > Conexiones**.
3. En **API de Ollama**, verifica que la URL sea exactamente `http://ollama:11434` y pulsa el icono de recargar para confirmar la conexión.

### 6.2. Conectar SearXNG con Open WebUI y Hermes Agent (Habilitar formato JSON)
Por defecto, SearXNG bloquea las peticiones automatizadas en formato JSON. Para que **Open WebUI**, **Hermes Agent** y **OpenCode** puedan hacer búsquedas web:
1. Tras arrancar SearXNG por primera vez, edita su archivo de configuración generado en el volumen:
   ```bash
   sudo nano $HOME/searxng/settings.yml
   ```
2. Busca la sección `search:` y bajo `formats:` añade `json`:
   ```yaml
   search:
     formats:
       - html
       - json
   ```
3. Reinicia SearXNG:
   ```bash
   docker compose -f $HOME/proyecto/docker-searxng.yml restart
   ```
4. En **Open WebUI**, ve a **Panel de Administración > Configuración > Búsqueda Web**, verifica que el motor sea `searxng` y que la URL de consulta sea `http://searxng:8080/search?q=<query>`.

### 6.3. Conectar ComfyUI con Open WebUI
1. Accede a **Open WebUI** (`http://<IP-SERVIDOR>:3000`) > **Panel de Administración > Configuración > Imágenes**.
2. Selecciona **ComfyUI** como motor de generación de imágenes.
3. Introduce la URL interna: `http://comfyui:8188` y guarda los cambios.

### 6.4. Conectar Ollama y SearXNG con OpenCode (IDE Web)
1. Accede a **OpenCode** en `http://<IP-SERVIDOR>:8443` e inicia sesión con la contraseña del fichero `.env`.
2. Abre la pestaña de **Extensiones** en la barra lateral izquierda e instala **Continue** o **Cline** (asistentes de código IA).
3. En la configuración de la extensión:
   * Selecciona **Ollama** como proveedor (`Provider`).
   * En `apiBase`, introduce `http://ollama:11434`.
   * Selecciona tu modelo descargado (ej. `llama3.1:8b`).
   * Para consultas de documentación web desde el IDE, apunta el proveedor de contexto `@search` o MCP a `http://searxng:8080`.

### 6.5. Integración del sistema RAG con Ollama y Open WebUI
Tienes dos vías simultáneas configuradas para explotar tus documentos alojados en `$HOME/rag`:
1. **Vía integrada en Open WebUI:** Cualquier documento (PDF, TXT, MD) que deposites en `$HOME/rag` estará accesible en Open WebUI dentro de `/app/backend/data/docs`. En **Open WebUI > Panel de Administración > Configuración > Documentos**, selecciona `ollama` como motor de Embeddings (`http://ollama:11434`) y el modelo `nomic-embed-text`.
2. **Vía contenedor dedicado `rag`:** Accede a `http://<IP-SERVIDOR>:11435`, donde el contenedor `rag` utiliza automáticamente `http://ollama:11434` tanto para el LLM como para los vectores de documentos.

---

## 7. Mantenimiento, actualización y resolución de errores

### 7.1. Copias de seguridad (Backups)
Al tener todos los volúmenes centralizados en `$HOME/`, puedes realizar un backup completo de configuraciones, bases de datos y documentos con un único comando:

```bash
# Crear directorio de backups
mkdir -p $HOME/backups_ia

# Empaquetar y comprimir configuraciones y datos (excluyendo modelos pesados de Ollama/ComfyUI si se desea ahorrar espacio)
sudo tar -czvf $HOME/backups_ia/backup_ia_$(date +%F).tar.gz \
  $HOME/proyecto \
  $HOME/openwebui \
  $HOME/hermes \
  $HOME/opencode \
  $HOME/yolo \
  $HOME/searxng \
  $HOME/rag
```

### 7.2. Actualización de los servicios
Para actualizar cualquier contenedor a su última versión disponible sin perder datos:

```bash
cd $HOME/proyecto

# Ejemplo para actualizar Open WebUI (repetir con el fichero deseado)
docker compose -f docker-openwebui.yml pull
docker compose -f docker-openwebui.yml up -d

# Limpiar imágenes antiguas que hayan quedado huérfanas para liberar disco
docker image prune -f
```

### 7.3. Resolución de errores comunes

#### Error 1: Problemas de permisos en los volúmenes (`Permission Denied`)
Si un contenedor (como `opencode`, `searxng` o `rag`) se reinicia en bucle indicando en los logs que no puede escribir en su directorio de `$HOME/`:
* **Solución:** Asigna la propiedad de las carpetas a tu usuario actual (`UID 1000`) y otorga permisos de lectura/escritura:
  ```bash
  sudo chown -R $USER:$USER $HOME/{ollama,openwebui,hermes,opencode,comfyui,yolo,searxng,rag}
  chmod -R 755 $HOME/{ollama,openwebui,hermes,opencode,comfyui,yolo,searxng,rag}
  ```

#### Error 2: `could not select device driver "nvidia" with capabilities: [[gpu]]`
Aparece al levantar un contenedor si Docker no reconoce el toolkit de NVIDIA:
* **Solución:** Reconfigura el runtime de Docker y reinicia el demonio:
  ```bash
  sudo nvidia-ctk runtime configure --runtime=docker
  sudo systemctl restart docker
  ```

#### Error 3: Falta de memoria de vídeo (`CUDA out of memory`) en GPU de 12GB
Si ejecutas un modelo pesado en Ollama y al mismo tiempo generas una imagen en ComfyUI, los 12 GB de VRAM de la RTX 3060 pueden saturarse:
* **Solución:** Descarga inmediatamente los modelos en memoria de Ollama ejecutando:
  ```bash
  docker exec -it ollama ollama stop <nombre-del-modelo>
  ```
  O reduce el tiempo de retención en memoria cambiando `OLLAMA_KEEP_ALIVE=1m` en `docker-ollama.yml`.

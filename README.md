# Documentación de Despliegue - Aplicación Galería

Este repositorio detalla los procedimientos y configuraciones necesarios para completar el ciclo de despliegue de la aplicación, abarcando desde las pruebas iniciales en local hasta la automatización CI/CD.

---

## 1. Validación en el Entorno Local

Como etapa previa a la contenedorización, es imprescindible verificar el correcto funcionamiento de la aplicación en el host local. Se recomienda el uso de entornos virtuales para gestionar de forma aislada las dependencias de Python.

### 1.1. Configuración del entorno
Ejecute los siguientes comandos para preparar el entorno y lanzar la aplicación:

```bash
# Creación y activación del entorno virtual
python3 -m venv venv
source venv/bin/activate

# Instalación de requisitos y ejecución
pip install -r requirements.txt
python src/app.py
```

### 1.2. Verificación del servicio
Para confirmar que el servidor responde correctamente, puede realizar las siguientes peticiones de prueba:

```bash
curl http://localhost:5000
curl http://localhost:5000/status
```
## 2. Gestión de Contenedores con Docker
### 2.1. Definición del Dockerfile
Se ha diseñado un Dockerfile optimizado utilizando una imagen base ligera de Python (3-slim) para reducir el tamaño de la imagen final y mejorar la seguridad:

```bash
FROM python:3-slim
WORKDIR /app
COPY ./requirements.txt ./
RUN pip install -r requirements.txt
COPY src .
CMD gunicorn --bind 0.0.0.0:5000 app:app
```

### 2.2. Orquestación con Docker Compose
Para simplificar la gestión y el despliegue del servicio, se emplea un archivo docker-compose.yml con la siguiente estructura:

```bash
services:
  galeria:
    container_name: galeria
    image: <usuario_dockerhub>/galeria:latest
    ports:
      - 80:5000
    restart: always
```
### 2.3. Comandos de administración
Levantar el servicio: docker compose up -d

Parar y eliminar contenedores: docker compose down

Publicar imagen manualmente: docker push <usuario_dockerhub>/galeria:latest

## 3. Integración y Despliegue Continuo (CI/CD)
Se ha implementado un flujo de trabajo automatizado mediante GitHub Actions. Este pipeline se activa automáticamente al detectar cambios en el directorio src/.

### 3.1. Configuración de Secretos (Secrets)
Para la correcta ejecución del flujo de trabajo, deben definirse los siguientes secretos en el repositorio de GitHub:

DOCKERHUB_USERNAME / DOCKERHUB_TOKEN: Credenciales de acceso al registro.

AWS_HOSTNAME / AWS_USERNAME / AWS_PRIVATEKEY: Datos de conexión SSH para la instancia EC2.

### 3.2. Definición del Workflow (.github/workflows/devops02.yml)

```bash
name: CI/CD Pipeline Galeria

on:
  push:
    branches: [ main ]
    paths: [ 'src/**' ]

jobs:
  build:
    name: Build and Push Image
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: DockerHub Login
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/galeria:latest

  aws_deploy:
    name: Deploy to AWS EC2
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Transfer Configuration (SCP)
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.AWS_HOSTNAME }}
          username: ${{ secrets.AWS_USERNAME }}
          key: ${{ secrets.AWS_PRIVATEKEY }}
          source: "docker-compose.yml"
          target: /home/admin
      - name: Execution and Deployment (SSH)
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.AWS_HOSTNAME }}
          username: ${{ secrets.AWS_USERNAME }}
          key: ${{ secrets.AWS_PRIVATEKEY }}
          script: |
            sleep 60
            docker compose down --rmi all
            docker compose up -d
```

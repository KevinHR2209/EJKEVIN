# EJKEVIN — Innovatech Chile · Evaluación Parcial N°2

Proyecto de contenedorización y despliegue automatizado en AWS para la asignatura **ISY1101 - Introducción a Herramientas DevOps**.

---

## 🏗️ Arquitectura

```
Internet
   │
   ▼
┌──────────────────┐        Subred Privada
│  EC2 Frontend    │  ──►  ┌──────────────────┐
│  (IP Pública)    │       │  EC2 Backend     │
│  Nginx:80        │       │  Despachos:8080  │
│  React + Vite    │       │  Ventas:8081     │
└──────────────────┘       │  MySQL:3306      │
                           └──────────────────┘
```

- **Frontend**: React + Vite servido con Nginx, accesible desde Internet.
- **Backend Despachos**: Spring Boot REST API en puerto 8080.
- **Backend Ventas**: Spring Boot REST API en puerto 8081.
- **Base de datos**: MySQL con volúmenes Docker para persistencia.
- Solo el Frontend es accesible desde Internet (Security Groups de AWS).

---

## 📁 Estructura del Repositorio

```
EJKEVIN/
├── back-Despachos_SpringBoot/   # Microservicio de Despachos
│   ├── Dockerfile               # Multi-stage build, usuario no root
│   └── src/
├── back-Ventas_SpringBoot/      # Microservicio de Ventas
│   ├── Dockerfile               # Multi-stage build, usuario no root
│   └── src/
├── front_despacho/              # Frontend React + Vite
│   ├── Dockerfile               # Multi-stage build con Nginx
│   └── src/
├── mysql-init/                  # Scripts de inicialización de BD
├── docker-compose.yml           # Stack completo de servicios
├── .env.example                 # Variables de entorno requeridas
└── .github/workflows/
    ├── deploy-despachos.yml     # Pipeline CI/CD Despachos
    └── deploy-ventas.yml        # Pipeline CI/CD Ventas
```

---

## 🐳 Dockerfiles

Cada servicio usa **multi-stage build** para minimizar el tamaño de la imagen final:

### Backend (Spring Boot)
- **Etapa 1 (build)**: `maven:3.9.6-eclipse-temurin-21-alpine` — compila el JAR.
- **Etapa 2 (run)**: `eclipse-temurin:21-jre-alpine` — solo el JRE, sin herramientas de compilación.
- Usuario **no root** (`appuser`) por seguridad.
- Imagen final ~180MB vs ~600MB sin multi-stage.

### Frontend (React + Vite)
- **Etapa 1 (build)**: `node:20-alpine` — instala dependencias y compila los estáticos.
- **Etapa 2 (run)**: `nginx:stable-alpine` — sirve los archivos estáticos.
- Imagen final ~25MB vs ~400MB sin multi-stage.

---

## ⚙️ Variables de Entorno

Copia `.env.example` a `.env` y completa los valores:

```bash
cp .env.example .env
```

| Variable | Descripción |
|---|---|
| `MYSQL_ROOT_PASSWORD` | Contraseña root de MySQL |
| `MYSQL_DATABASE` | Nombre de la base de datos |
| `MYSQL_USER` | Usuario de la base de datos |
| `MYSQL_PASSWORD` | Contraseña del usuario |
| `VITE_API_DESPACHO` | URL del backend Despachos |
| `VITE_API_VENTAS` | URL del backend Ventas |

---

## 🚀 Levantar el Stack Localmente

```bash
# 1. Clonar el repositorio
git clone https://github.com/KevinHR2209/EJKEVIN.git
cd EJKEVIN

# 2. Configurar variables de entorno
cp .env.example .env
# Editar .env con tus valores

# 3. Levantar todos los servicios
docker-compose up -d --build

# 4. Verificar que los contenedores están corriendo
docker-compose ps

# 5. Acceder a los servicios
# Frontend:           http://localhost:80
# Backend Despachos:  http://localhost:8080
# Backend Ventas:     http://localhost:8081
```

---

## 💾 Persistencia de Datos

Se utiliza **named volume** para la base de datos MySQL:

```yaml
volumes:
  mysql_data:    # Named volume — gestionado por Docker
```

**¿Por qué named volume y no bind mount?**
- Los named volumes son gestionados por Docker, sin dependencia de la ruta del host.
- Persisten los datos aunque el contenedor sea eliminado y recreado.
- Son portables entre entornos (local, EC2, CI/CD).
- Más seguros: Docker controla los permisos del directorio.

---

## 🔄 Pipeline CI/CD (GitHub Actions)

Cada microservicio tiene su propio workflow que se activa con **push a la rama `deploy`**:

```
push a rama deploy
       │
       ▼
  1. Checkout código
       │
       ▼
  2. Configurar credenciales AWS
       │
       ▼
  3. Login a Amazon ECR
       │
       ▼
  4. docker build → docker push → ECR
       │
       ▼
  5. SSH a EC2 → docker pull → docker run
```

### GitHub Secrets requeridos

Configurar en **Settings → Secrets and variables → Actions**:

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial AWS |
| `AWS_SECRET_ACCESS_KEY` | Credencial AWS |
| `AWS_REGION` | Región AWS (ej: `us-east-1`) |
| `ECR_REGISTRY` | URL del registro ECR |
| `ECR_REPO_DESPACHOS` | Nombre del repo ECR para Despachos |
| `ECR_REPO_VENTAS` | Nombre del repo ECR para Ventas |
| `EC2_HOST_BACKEND` | IP pública o DNS de la EC2 backend |
| `EC2_USER` | Usuario SSH (ej: `ec2-user`) |
| `EC2_SSH_KEY` | Clave privada SSH |
| `DB_URL` | URL JDBC de la base de datos |
| `DB_USER` | Usuario de la BD |
| `DB_PASSWORD` | Contraseña de la BD |

---

## 🛡️ Seguridad

- Los contenedores backend corren con **usuario no root**.
- Las credenciales se gestionan como **GitHub Secrets** (nunca en el código).
- Solo el Frontend está expuesto a Internet; el Backend está en **subred privada**.
- El `.env` está en `.gitignore` — solo se sube `.env.example`.

---

## 👥 Equipo

- **Kevin HR** — KevinHR2209

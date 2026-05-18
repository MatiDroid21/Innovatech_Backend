# Innovatech Backend — Microservicios Spring Boot

Backend de la plataforma Innovatech Chile, compuesto por dos microservicios desarrollados en **Spring Boot**:
- **back-Ventas_SpringBoot** — Gestión de ventas
- **back-Despachos_SpringBoot** — Gestión de despachos

Desplegado en una instancia **EC2 privada** en AWS, contenerizado con Docker y con pipeline CI/CD automatizado mediante **GitHub Actions**.

---

## Arquitectura
EC2 Backend (subred privada)
├── Contenedor: back-ventas → Puerto 8080
├── Contenedor: back-despachos → Puerto 8081
└── Contenedor: PostgreSQL → Puerto 5432 (volumen persistente)

El frontend (EC2 pública) se comunica con el backend a través de la subred interna de la VPC. El backend **no es accesible directamente desde Internet**.

---

## Contenedorización

### Requisitos previos
- Docker >= 24.x
- Docker Compose >= 2.x

### Variables de entorno

Copia `.env.example` a `.env` y completa los valores:

```bash
cp .env.example .env
```

| Variable | Descripción |
|---|---|
| `DB_HOST` | Host de la base de datos |
| `DB_PORT` | Puerto PostgreSQL (default: 5432) |
| `DB_NAME` | Nombre de la base de datos |
| `DB_USER` | Usuario de la base de datos |
| `DB_PASSWORD` | Contraseña de la base de datos |

### Levantar el stack completo

```bash
docker compose up -d
```

### Verificar que los contenedores están corriendo

```bash
docker compose ps
docker compose logs -f
```

### Detener servicios

```bash
docker compose down
```

### Detener y eliminar volúmenes (borra datos)

```bash
docker compose down -v
```

---

## Persistencia de datos

Se utiliza un **named volume** (`postgres_data`) para garantizar que los datos de PostgreSQL persistan entre reinicios de contenedores.

```yaml
volumes:
  postgres_data:
```

Se eligió **named volume** en lugar de bind mount porque:
- Es gestionado completamente por Docker, sin dependencia del path del host.
- Es portable entre distintas máquinas y entornos (local, EC2).
- Facilita backups y restauración.

---

## Pipeline CI/CD — GitHub Actions

El pipeline se activa automáticamente con cada `push` a la rama **`deploy`**.

### Flujo del pipeline
push → rama deploy
↓

Build de imagen Docker (multi-stage)
↓

Push de imagen a Docker Hub / ECR
↓

SSH a EC2 → docker compose pull → docker compose up -d

text

### GitHub Secrets requeridos

Configurar en **Settings → Secrets and variables → Actions**:

| Secret | Descripción |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub |
| `DOCKERHUB_TOKEN` | Token de acceso Docker Hub |
| `EC2_HOST` | IP pública o privada de la instancia EC2 |
| `EC2_USER` | Usuario SSH de la EC2 (ej: `ec2-user`) |
| `EC2_SSH_KEY` | Clave privada SSH (contenido del `.pem`) |

### Activar el pipeline

```bash
git checkout deploy
git merge main   # o la rama con tus cambios
git push origin deploy
```

---

## Estructura del repositorio
Innovatech_Backend/
├── .github/
│ └── workflows/ # Pipelines GitHub Actions
├── back-Ventas_SpringBoot/
│ ├── Dockerfile # Multi-stage build
│ └── src/
├── back-Despachos_SpringBoot/
│ ├── Dockerfile # Multi-stage build
│ └── src/
├── infra/ # Configuración de infraestructura AWS
├── docker-compose.yml # Stack completo de servicios
├── .env.example # Variables de entorno requeridas
└── README.md

---

##Integrantes

- Keiton Chaves
- Sergio Soto
- Matías Chávez

**Asignatura:** Introducción a Herramientas DevOps — ISY1101  
**Institución:** Duoc UC

# Innovatech Chile — Backend

API REST del sistema de gestión de Innovatech Chile, compuesta por
dos microservicios independientes desarrollados con **Spring Boot**,
desplegados en **Amazon EKS** y conectados a una base de datos **MySQL**.

---

## Microservicios

| Servicio | Puerto | Descripción |
|---|---|---|
| `back-Ventas_SpringBoot` | 8080 | Gestión de ventas (CRUD) |
| `back-Despachos_SpringBoot` | 8081 | Gestión de despachos (CRUD) |

---

## Tecnologías utilizadas

- Java 17 + Spring Boot 3.4
- Spring Data JPA + Hibernate
- MySQL 8
- Docker + Docker Compose
- GitHub Actions (CI/CD)
- Amazon EKS (Kubernetes)
- Amazon ECR (registro de imágenes)
- Kubernetes Secrets (gestión de credenciales)

---

## Estructura del repositorio
Innovatech_Backend/
├── back-Ventas_SpringBoot/
│ └── Springboot-API-REST/ # Código fuente Spring Boot
├── back-Despachos_SpringBoot/
│ └── Springboot-API-REST-DESPACHO/
├── infra/
│ └── infra-setup.sh # Script de infraestructura AWS
├── docker-compose.yml # Levantamiento local completo
└── .env.example # Variables de entorno requeridas

text

---

## Variables de entorno

Crear un archivo `.env` basado en `.env.example`:

```env
DB_ENDPOINT=localhost
DB_PORT=3306
DB_NAME=innovatech
DB_USERNAME=admin
DB_PASSWORD=admin1234
```

> En el clúster EKS estas variables se gestionan mediante **Kubernetes Secrets**
> para evitar exponer credenciales en el código.

---

## Levantar localmente

```bash
git clone https://github.com/MatiDroid21/Innovatech_Backend.git
cd Innovatech_Backend
cp .env.example .env
# Editar .env con las credenciales locales
docker compose up -d --build
```

Servicios disponibles:
- Ventas: `http://localhost:8080/api/ventas`
- Despachos: `http://localhost:8081/api/despachos`

---

## Pipeline CI/CD (GitHub Actions)

El pipeline se activa automáticamente con cada push a la rama `deploy`.

**Job 1 — Build & Push Ventas:**
1. Construye la imagen Docker del microservicio de ventas.
2. Se autentica en Amazon ECR con credenciales AWS.
3. Publica la imagen en ECR como `inovatech-backend:ventas-latest`.

**Job 2 — Build & Push Despachos:**
1. Construye la imagen Docker del microservicio de despachos.
2. Se autentica en Amazon ECR con credenciales AWS.
3. Publica la imagen en ECR como `inovatech-backend:despachos-latest`.

**Job 3 — Deploy en EKS (depende de Job 1 y Job 2):**
1. Configura las credenciales AWS en el runner.
2. Actualiza el contexto de `kubectl` apuntando al clúster EKS.
3. Ejecuta `rollout restart` en ambos deployments.
4. Verifica el estado del despliegue con `rollout status`.

**Secrets requeridos en GitHub:**

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial de acceso AWS |
| `AWS_SECRET_ACCESS_KEY` | Clave secreta AWS |
| `AWS_SESSION_TOKEN` | Token de sesión temporal AWS Academy |
| `AWS_REGION` | Región del clúster (us-east-1) |
| `ECR_REGISTRY` | URL del registro de imágenes ECR |
| `EKS_CLUSTER_NAME` | Nombre del clúster EKS destino |

---

## Despliegue en EKS

Los microservicios corren como `Deployments` en Amazon EKS con 1 réplica activa,
escalando hasta 4 mediante **Horizontal Pod Autoscaler (HPA)** según demanda de CPU
(umbral 50%).

Las credenciales de base de datos se inyectan mediante **Kubernetes Secrets**:

```bash
kubectl create secret generic backend-secrets \
  --from-literal=DB_ENDPOINT=mysql-service \
  --from-literal=DB_PORT=3306 \
  --from-literal=DB_NAME=innovatech \
  --from-literal=DB_USERNAME=admin \
  --from-literal=DB_PASSWORD=admin1234
```

Los servicios son de tipo `ClusterIP` (acceso interno únicamente),
comunicándose con el frontend a través del DNS interno del clúster.

```bash
# Ver estado de los pods
kubectl get pods

# Ver HPA
kubectl get hpa

# Ver logs de un microservicio
kubectl logs deployment/ventas-deployment
kubectl logs deployment/despachos-deployment
```

---

## Endpoints principales

**Ventas** (`/api/ventas`):
- `GET /api/ventas` — Listar todas las ventas
- `POST /api/ventas` — Crear nueva venta
- `PUT /api/ventas/{id}` — Actualizar venta
- `DELETE /api/ventas/{id}` — Eliminar venta

**Despachos** (`/api/despachos`):
- `GET /api/despachos` — Listar todos los despachos
- `POST /api/despachos` — Crear nuevo despacho
- `PUT /api/despachos/{id}` — Actualizar despacho
- `DELETE /api/despachos/{id}` — Eliminar despacho

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
- Kubernetes Secrets (gestión de credenciales)

---

## Estructura del repositorio
Innovatech_Backend/
├── back-Ventas_SpringBoot/ # Microservicio de ventas
│ └── Springboot-API-REST/ # Código fuente Spring Boot
├── back-Despachos_SpringBoot/ # Microservicio de despachos
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

## Despliegue en EKS

Los microservicios corren como `Deployments` en Amazon EKS con 2 réplicas cada uno.
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

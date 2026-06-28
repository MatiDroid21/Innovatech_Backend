# Innovatech Chile — Backend

API REST del sistema de gestión de Innovatech Chile, compuesta por
dos microservicios independientes desarrollados con **Spring Boot**,
desplegados en **Amazon EKS** y conectados a una base de datos **MySQL**.
Para el ramo INTRODUCCION A HERRAMIENTAS DEVOPS
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


## Arquitectura de microservicios

Ambos microservicios corren como `Deployments` independientes dentro del clúster EKS.
Se comunican con MySQL a través del service interno `mysql-service` (ClusterIP).
El frontend los alcanza mediante `ventas-service` y `despachos-service` usando el
DNS interno de Kubernetes, sin exponer ningún puerto al exterior.
[Pod Frontend]
├──► ventas-service:80 ──► [Pod Ventas :8080] ──► mysql-service:3306
└──► despachos-service:80 ──► [Pod Despachos :8081] ──► mysql-service:3306
│
[Pod MySQL 8]
---

## Problemas encontrados y soluciones

### CrashLoopBackOff — Public Key Retrieval is not allowed
**Causa:** MySQL 8 usa `caching_sha2_password` por defecto, lo que requiere
que el cliente pueda recuperar la clave pública del servidor. Sin el parámetro
correcto en la URL JDBC, Spring Boot no podía autenticarse y los pods crasheaban.

**Solución:** Se agregó la variable `SPRING_DATASOURCE_URL` al Kubernetes Secret
`backend-secrets` con la URL JDBC completa incluyendo los parámetros necesarios:
jdbc:mysql://mysql-service:3306/innovatech?allowPublicKeyRetrieval=true&useSSL=false

```bash
kubectl patch secret backend-secrets -p \
  '{"data":{"SPRING_DATASOURCE_URL":"<valor en base64>"}}'

kubectl rollout restart deployment ventas-deployment
kubectl rollout restart deployment despachos-deployment
```

### Variables de entorno no inyectadas en los Deployments
**Causa:** Los manifiestos YAML originales de los deployments no referenciaban
el secret `backend-secrets`, por lo que Spring Boot no recibía las variables
de conexión a la base de datos.

**Solución:** Se aplicó un `patch` a ambos deployments para inyectar el secret
mediante `envFrom`:

```bash
kubectl patch deployment ventas-deployment \
  --patch '{"spec":{"template":{"spec":{"containers":[{"name":"ventas","envFrom":[{"secretRef":{"name":"backend-secrets"}}]}]}}}}'

kubectl patch deployment despachos-deployment \
  --patch '{"spec":{"template":{"spec":{"containers":[{"name":"despachos","envFrom":[{"secretRef":{"name":"backend-secrets"}}]}]}}}}'
```

---

## Justificación del umbral HPA (50% CPU)

Se eligió el **50% de uso de CPU** como umbral de escalado porque representa
un punto de equilibrio entre rendimiento y costo:

- Permite absorber picos de carga antes de que afecten la experiencia del usuario.
- Evita escalar prematuramente ante uso normal.
- Con instancias `t3.medium` (2 vCPU), el 50% equivale a **1 vCPU disponible
  como margen** antes de que Kubernetes agregue una nueva réplica.

Configuración aplicada:
- Mínimo: 1 réplica
- Máximo: 4 réplicas
- Target CPU: 50%

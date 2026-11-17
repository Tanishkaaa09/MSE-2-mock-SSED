# CampusHub — Dockerized 9-Service Application

## 1. Project Overview

CampusHub is a containerized application consisting of a frontend, backend API, background worker, database, cache, message broker, object storage, email testing service, and Nginx gateway.

The application code inside `app/` is provided and is not modified. This project focuses on Dockerfiles, Docker Compose orchestration, networking, persistent storage, security, and deployment configuration.

## 2. Architecture

The application consists of nine services:

| Service | Image / Build | Port | Purpose |
|---|---|---|---|
| Gateway | nginx-unprivileged | 8080 | Single entry point and reverse proxy |
| Frontend | `app/frontend` | 8080 | Serves the frontend application |
| API | `app/api` | 3000 | Backend REST API |
| Worker | Same API image | Internal | Processes background email jobs |
| PostgreSQL | postgres:16-alpine | 5432 | Persistent relational database |
| Redis | redis:7-alpine | 6379 | Application caching |
| RabbitMQ | rabbitmq:3-management-alpine | 5672 | Message queue |
| MinIO | pgsty/silo | 9000 | Object and image storage |
| MailHog | mailhog/mailhog:v1.0.1 | 1025, 8025 | Email testing |

### Request routing

The Nginx gateway routes incoming requests as follows:

- `/` → Frontend
- `/api/` → Backend API
- `/storage/` → MinIO object storage

Only the gateway is exposed publicly on port 8080. RabbitMQ and MailHog management interfaces are bound to localhost for development access.

## 3. Docker Architecture

### Frontend Dockerfile

The frontend uses a multi-stage Docker build:

1. Build stage uses Node.js 20 Alpine.
2. Dependencies are installed and the frontend is built using `npm run build`.
3. The generated `dist/` directory is copied into an unprivileged Nginx image.
4. Node.js and build dependencies are excluded from the final runtime image.
5. A HEALTHCHECK verifies that the frontend responds.

### API Dockerfile

The API uses Node.js 20 Alpine.

- Dependency manifests are copied before application source code for better layer caching.
- Only production dependencies are installed using `npm install --omit=dev`.
- Files are owned by the non-root Node user.
- The container runs as the `node` user.
- The API healthcheck verifies `/api/health`.
- The default command runs migrations and seed operations before starting the API server.

### Worker

The worker reuses the API Docker image and executes:

`node src/worker.js`

It does not build a separate image. Its API healthcheck is disabled because it runs a background worker rather than the API server.

## 4. Networking

Three Docker bridge networks isolate application communication.

| Network | Connected services | Purpose |
|---|---|---|
| frontend | Gateway, Frontend | Frontend traffic |
| backend | Gateway, API | API communication |
| data | Gateway, API, Worker, DB, Redis, RabbitMQ, MinIO, MailHog | Backend data and service communication |

The gateway connects to all three networks because it routes requests to the frontend, API, and MinIO.

Internal services communicate using Docker Compose service names instead of hardcoded IP addresses.

## 5. Persistent Storage

Named Docker volumes preserve important data across container recreation.

| Volume | Service | Purpose |
|---|---|---|
| postgres_data | PostgreSQL | Database persistence |
| redis_data | Redis | Append-only persistence |
| rabbitmq_data | RabbitMQ | Message broker data |
| minio_data | MinIO | Uploaded object storage |

Removing containers with `docker compose down` does not delete named volumes.

**Warning:** `docker compose down -v` deletes named volumes and their stored data.

## 6. Environment Configuration

Environment variable names are documented in `.env.example`.

Create your local environment file:

```powershell
Copy-Item .env.example .env
```

Update the placeholder credentials in `.env` before deployment.

The `.env` file is excluded through `.gitignore` and both Docker build-context `.dockerignore` files.

Do not commit real credentials or secrets to Git.

## 7. Running the Application

### Prerequisites

- Docker Desktop
- Docker Compose
- Git

### Start the application

```powershell
docker compose up -d --build --wait
```

This builds the application images, starts all services, and waits for configured healthchecks.

### Check running containers

```powershell
docker compose ps
```

### View logs

```powershell
docker compose logs
```

To inspect an individual service:

```powershell
docker compose logs api
docker compose logs worker
```

### Access the application

| Component | URL |
|---|---|
| CampusHub frontend and gateway | http://localhost:8080 |
| API health endpoint | http://localhost:8080/api/health |
| RabbitMQ management UI | http://127.0.0.1:15672 |
| MailHog interface | http://127.0.0.1:8025 |

### Stop the application

```powershell
docker compose down
```

This removes containers and networks while preserving named volumes.

## 8. Healthchecks and Resource Limits

Healthchecks are configured for:

- Gateway
- Frontend
- API
- PostgreSQL
- Redis
- RabbitMQ
- MinIO
- MailHog

The worker has its healthcheck disabled.

Docker Compose resource limits are configured for all nine services to control CPU and memory consumption.

Service startup dependencies use health-based conditions where required, ensuring that the API starts after its database, cache, message broker, and object storage dependencies are healthy.

## 9. Security Measures

The following security practices are implemented:

1. Frontend and gateway use unprivileged Nginx images.
2. API and worker execute as a non-root Node.js user.
3. Internal services are not exposed directly to public host ports.
4. RabbitMQ and MailHog management interfaces are restricted to localhost.
5. Environment variables are used for credentials.
6. `.env` is excluded from Git and Docker build contexts.
7. API production dependencies are installed without development dependencies.

## 10. Key Design Decisions

### Decision 1: Multi-stage frontend build

The frontend is built in a Node.js stage and served using unprivileged Nginx.

**Reason:** This reduces the final runtime image size and avoids including Node.js and build dependencies in production.

### Decision 2: Separate frontend, backend, and data networks

Three bridge networks are used to isolate different communication paths.

**Reason:** Network segmentation reduces unnecessary service connectivity while allowing the gateway to route requests to the required services.

### Decision 3: Shared API image for the worker

The worker reuses the API image but runs a different command.

**Reason:** This avoids duplicate builds and ensures that API and worker use the same application dependencies and runtime environment.

## 11. Verification and Testing

The following functional and deployment checks were performed:

- All nine containers started successfully.
- Configured service healthchecks passed.
- Frontend returned HTTP 200.
- API health endpoint returned a successful response.
- Event seed data was available.
- Redis caching returned `X-Cache: HIT`.
- Event registration returned HTTP 202.
- RabbitMQ successfully received and processed the registration job.
- MailHog received the registration confirmation email.
- Image upload and MinIO retrieval were tested.
- PostgreSQL registrations and MinIO images persisted after container recreation.
- API and worker ran as non-root users.
- Frontend and gateway ran as unprivileged Nginx users.
- API image size was approximately 244.47 MB, below the 250 MB requirement.

### Automated grading

Run the provided `grade.sh` script after configuring the environment and ensuring Docker Desktop is running.

The grading script may recreate services and remove volumes. Back up any important data before running it.

## 12. Future Monitoring Enhancements

Monitoring can be added using Prometheus and Grafana.

- Prometheus can collect application and infrastructure metrics.
- Grafana can display dashboards for CPU, memory, request rates, API latency, and service health.
- Node Exporter can provide host-level metrics.
- cAdvisor can provide container resource metrics.
- Application-level metrics can track API requests, errors, and background job processing.

These monitoring components are possible future enhancements and are not part of the current nine-service deployment.

---

**Project:** CampusHub — Dockerized 9-Service Application  
**Focus:** Docker, Docker Compose, networking, persistent volumes, container security, and service orchestration.
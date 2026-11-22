# CDAC

A full-stack web project: a Next.js front end backed by two REST APIs, one in **Spring Boot** and one in **Express**. Both APIs use **PostgreSQL** and **MongoDB**. Everything runs locally with Docker Compose and deploys to **AWS EKS** with Terraform and Helm.

```
                 ┌──────────────────────┐
                 │  client-side :3000   │  Next.js 16 · React 19 · Tailwind v4
                 └─────┬──────────┬─────┘
                       │          │
        ┌──────────────▼───┐  ┌───▼──────────────┐
        │ spring-api :8080 │  │ express-api :5000│
        │ Spring Boot 4.2  │  │ Express 5 · Bun  │
        │ Java 21          │  │ TypeScript       │
        └───┬──────────┬───┘  └───┬──────────┬───┘
            │          │          │          │
     ┌──────▼──────────▼──┐  ┌────▼──────────▼────┐
     │ PostgreSQL 16      │  │ MongoDB 7          │
     │ (RDS on AWS)       │  │ (DocumentDB on AWS)│
     └────────────────────┘  └────────────────────┘
```

## Repositories

The project is split into separate repositories, checked out side by side in one workspace folder:

| Repo | Description |
|---|---|
| `client-side` | Next.js front end |
| `spring-api` | Spring Boot REST API (JPA + Spring Data MongoDB, Actuator) |
| `express-api` | Express REST API (`pg` + Mongoose), run by Bun |
| `helm` | Helm chart `cdac-apis` that deploys both APIs to Kubernetes |
| `infra` | Terraform for the AWS environment (VPC, EKS, RDS, DocumentDB, ECR, Secrets Manager) |
| `remote-backend` | Terraform for the S3 bucket that stores the `infra` state |
| `.github` | This repo: project overview and the full-stack `docker-compose.yml` |

```
cdac project/
├── .github/          # this repo
├── client-side/
├── spring-api/
├── express-api/
├── helm/
├── infra/
└── remote-backend/
```

## Quick start (local)

**Prerequisites:** Docker with Compose v2.

From the workspace root (the folder containing all the repos):

```bash
docker compose -f .github/docker-compose.yml --project-directory . up --build
```

| Service | URL |
|---|---|
| Front end | http://localhost:3000 |
| Spring API | http://localhost:8080/api/health |
| Express API | http://localhost:5000/api/health |
| PostgreSQL | `localhost:5432`, database `cdac_db` (user/password `postgres`) |
| MongoDB | `localhost:27017`, database `cdac_db` (user `root`, password `example`) |

You can override the database credentials with `POSTGRES_USER`, `POSTGRES_PASSWORD`, `MONGO_USER` and `MONGO_PASSWORD` in your shell or in a `.env` file. Stop the stack with `down`; add `-v` to also delete the database volumes.

### Running a single service

Start only the databases, then run one app on your machine:

```bash
docker compose -f .github/docker-compose.yml --project-directory . up -d postgres mongo
```

| Service | Prerequisites | Commands |
|---|---|---|
| `client-side` | Bun | `bun install` → `bun dev` |
| `express-api` | Bun | `cp .env.example .env` → `bun install` → `bun run dev` |
| `spring-api` | JDK 21 | `./mvnw spring-boot:run` (Windows: `mvnw.cmd`) · tests: `./mvnw test` |

The defaults in each `.env.example` and in `application.yml` point at the local databases. When the databases run in Compose with root credentials, add them to `MONGO_URI`, e.g. `mongodb://root:example@localhost:27017/cdac_db?authSource=admin`.

## API

Both APIs expose the same endpoints and response format:

| Method | Path | Response |
|---|---|---|
| `GET` | `/` | `{ "message": "Running" }` |
| `GET` | `/api/health` | App environment and the status (`up` / `down`) of each database |

```json
{
  "success": true,
  "message": "API is running successfully",
  "data": {
    "environment": "development",
    "databases": { "postgres": "up", "mongodb": "up" }
  },
  "timestamp": "2026-11-22T10:00:00.000Z"
}
```

Errors use the same envelope with `"success": false`. Spring also exposes `/actuator/health` (with liveness and readiness groups on Kubernetes).

## Configuration

All services are configured through environment variables. The names are the same locally, in Compose and in the Kubernetes secrets.

| Variable | Used by | Description |
|---|---|---|
| `PORT` | both APIs | HTTP port (`8080` Spring, `5000` Express) |
| `APP_ENV` | spring-api | Active Spring profile (`development` / `production`) |
| `NODE_ENV` | express-api | Node environment |
| `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` | spring-api | PostgreSQL JDBC connection |
| `DATABASE_URL` | express-api | PostgreSQL connection URL (`postgres://…`) |
| `MONGO_URI` | both APIs | MongoDB connection string |
| `CORS_ORIGINS` | both APIs | Allowed front-end origins, comma separated |
| `JWT_SECRET`, `API_KEY` | both APIs | Application secrets |
| `NEXT_PUBLIC_SPRING_API_URL`, `NEXT_PUBLIC_EXPRESS_API_URL` | client-side | API URLs for the browser, **set at build time** |
| `SPRING_API_URL`, `EXPRESS_API_URL` | client-side | API URLs for server-side code, set at runtime |

## Deploying to AWS

```
Internet → ALB → EKS (private subnets) → RDS PostgreSQL + DocumentDB (isolated subnets)
Secrets Manager → External Secrets Operator → Kubernetes Secrets → app pods
```

**Prerequisites:** Terraform ≥ 1.10, AWS CLI v2, kubectl, Helm, and AWS credentials.

1. **State bucket** (once per AWS account):
   ```bash
   cd remote-backend && terraform init && terraform apply
   ```
2. **Infrastructure** (about 20–25 minutes):
   ```bash
   cd ../infra
   cp terraform.tfvars.example terraform.tfvars   # adjust as needed
   terraform init -backend-config=backend.hcl
   terraform plan -out tfplan && terraform apply tfplan
   $(terraform output -raw configure_kubectl)
   ```
3. **Images:** build and push each service to the ECR repos listed by `terraform output ecr_repository_urls`.
4. **Apps:** install the Helm chart using the command Terraform prints:
   ```bash
   $(terraform output -raw helm_install_command) \
     --set apps.spring-api.image.tag=v1 --set apps.express-api.image.tag=v1
   ```
5. **Check:** `kubectl get externalsecrets,pods,ingress -n cdac`

Both APIs are served from one shared ALB, Spring on port 8080 and Express on port 5000. The defaults (region `ap-south-1`, environment `dev`) cost roughly **$250+/month**, so run `terraform destroy` when you're not using it. See the `infra` and `remote-backend` READMEs for secret rotation, DocumentDB TLS, costs and the production checklist.

## Tech stack

| Layer | Technology |
|---|---|
| Front end | Next.js 16, React 19, TypeScript, Tailwind CSS v4, Bun |
| Spring API | Spring Boot 4.2, Java 21, Spring Data JPA, Spring Data MongoDB, Actuator, Lombok, Maven |
| Express API | Express 5, TypeScript, Bun, `pg`, Mongoose, Helmet, Morgan |
| Data | PostgreSQL 16, MongoDB 7 (Amazon RDS and DocumentDB 5.0 on AWS) |
| Infrastructure | Terraform, AWS (VPC, EKS, ECR, ALB, Secrets Manager), Helm, External Secrets Operator |

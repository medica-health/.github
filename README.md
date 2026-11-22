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

## Developer setup

### Tools

| Tool | Needed for |
|---|---|
| Git and a GitHub account with access to `medica-health` | everything |
| Docker Desktop (Compose v2) | running the full stack locally |
| [Bun](https://bun.sh) 1.3+ | `client-side`, `express-api` |
| JDK 21 | `spring-api` (Maven comes with the `mvnw` wrapper) |
| Terraform ≥ 1.10, AWS CLI v2, kubectl, Helm | only for `infra`, `remote-backend`, `helm` |

Optional: [GitHub CLI](https://cli.github.com) (`gh`) makes it easier to open PRs from the terminal.

### Set up the workspace (first time)

Clone every repo into **one** folder. The local Docker Compose file expects them side by side:

```bash
mkdir cdac-project && cd cdac-project

for repo in .github client-side spring-api express-api helm infra remote-backend; do
  git clone https://github.com/medica-health/$repo.git
done
```

Check that your Git identity is set, so commits are credited to you:

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
```

On Windows, keep line endings consistent with the Linux containers:

```bash
git config --global core.autocrlf input
```

Then follow [Quick start (local)](#quick-start-local) to run the stack.

## Contributing

Every change goes through a pull request. Nobody pushes directly to `main`.

### 1. Pick up the work

- Find or create a GitHub **issue** in the repo you'll change, describing what you'll do.
- Assign it to yourself so others know it's taken.
- If one change touches several repos (e.g. a new endpoint in both APIs plus the front end), open one issue per repo and link them.

### 2. Get the latest `main`

```bash
cd spring-api            # the repo you're working in
git checkout main
git pull origin main
```

### 3. Create a branch

Branch from the updated `main`. Use a prefix and a short description, plus the issue number:

```bash
git checkout -b feature/12-user-login
```

| Prefix | Use for |
|---|---|
| `feature/` | new functionality |
| `fix/` | bug fixes |
| `chore/` | dependencies, config, tooling |
| `docs/` | documentation only |
| `refactor/` | code changes with no behaviour change |

### 4. Make your changes

- Keep a branch to one topic. Smaller PRs get reviewed faster.
- **Keep the two APIs in step.** `spring-api` and `express-api` expose the same endpoints and return the same `{ success, message, data, timestamp }` envelope. If you add or change an endpoint in one, do the same in the other (or open an issue for it).
- **Environment variables:** if you add or rename one, update `.env.example`, `.github/docker-compose.yml` and, for secrets, the Helm chart and Terraform (`infra/secrets.tf`) too.
- **Never commit secrets:** no `.env` files, `*.tfvars`, `backend.hcl` or Terraform state. These are already in `.gitignore`; don't force-add them.
- `client-side` uses Next.js 16, which has breaking changes from older versions. Check the docs bundled in `node_modules/next/dist/docs/` before relying on older patterns.

### 5. Check your work locally

Run the checks for the repo you changed before you push:

| Repo | Commands |
|---|---|
| `client-side` | `bun run lint` and `bun run build` |
| `express-api` | `bun run typecheck` |
| `spring-api` | `./mvnw test` |
| `helm` | `helm lint cdac-apis` |
| `infra` / `remote-backend` | `terraform fmt -check` and `terraform validate` |

Then run the full stack with Docker Compose and try your change end to end.

The same checks run automatically in **GitHub Actions** (`.github/workflows/ci.yml` in each repo) on every pull request and every push to `main`. A PR can't be merged until they pass.

### 6. Commit

Write commit messages in the imperative mood, with a short summary line (about 50 characters) and an optional body explaining *why*:

```bash
git add <files>
git commit -m "Add login endpoint to Spring API" -m "Refs #12"
```

Prefer several small, focused commits over one large one.

### 7. Stay up to date with `main`

If `main` has moved on while you were working, rebase your branch onto it and fix any conflicts:

```bash
git fetch origin
git rebase origin/main
# fix conflicts, then: git add <files> && git rebase --continue
```

### 8. Push your branch

```bash
git push -u origin feature/12-user-login
# after a rebase on a branch you've already pushed:
git push --force-with-lease
```

Only force-push your **own** feature branch, never `main`.

### 9. Open a pull request

On GitHub, open a PR from your branch into `main`, or use the CLI:

```bash
gh pr create --base main --fill
```

The description is pre-filled from the organization's [PR template](https://github.com/medica-health/.github/blob/main/PULL_REQUEST_TEMPLATE.md). Fill in each section, which covers:

- **What** changed and **why**
- **How to test** it (commands, URLs, sample requests)
- `Closes #12` to link and auto-close the issue
- Screenshots for UI changes
- Links to related PRs in other repos, if the change spans several

Open it as a **draft** if it isn't ready for review yet.

### 10. Review

- Request at least **one reviewer** from the team.
- Respond to every comment: push a fix or reply explaining your choice.
- Push follow-up commits to the same branch; the PR updates automatically.
- **Reviewers** check that the code works, follows the existing patterns, keeps the two APIs consistent, and doesn't include secrets.

### 11. Merge

`main` is protected in every repo. A PR can only be merged when:

- it has at least **one approval** (new commits dismiss earlier approvals),
- all **CI checks pass**,
- the branch is **up to date** with `main`, and
- every review conversation is **resolved**.

Direct pushes, force-pushes and deleting `main` are blocked.

Once the PR is approved and up to date with `main`:

- Use **Squash and merge**, so `main` gets one clean commit per PR.
- Delete the branch after merging.
- Clean up locally:

```bash
git checkout main
git pull origin main
git branch -D feature/12-user-login   # -D because squash merges aren't seen as merged by Git
```

### Changes to infrastructure

For `infra`, `remote-backend` and `helm`:

- Paste the relevant `terraform plan` output (or the `helm template` / `helm diff` output) into the PR, with any secret values removed.
- Only run `terraform apply` **after** the PR is approved and merged, from an up-to-date `main`.
- Call out anything that destroys or replaces resources, or that changes costs.

## Tech stack

| Layer | Technology |
|---|---|
| Front end | Next.js 16, React 19, TypeScript, Tailwind CSS v4, Bun |
| Spring API | Spring Boot 4.2, Java 21, Spring Data JPA, Spring Data MongoDB, Actuator, Lombok, Maven |
| Express API | Express 5, TypeScript, Bun, `pg`, Mongoose, Helmet, Morgan |
| Data | PostgreSQL 16, MongoDB 7 (Amazon RDS and DocumentDB 5.0 on AWS) |
| Infrastructure | Terraform, AWS (VPC, EKS, ECR, ALB, Secrets Manager), Helm, External Secrets Operator |

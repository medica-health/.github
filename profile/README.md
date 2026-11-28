# MediCare · medica-health

**MediCare** is an online doctor appointment system. Patients find doctors near them, book, cancel and reschedule appointments, and keep their prescriptions and medical records in one place. Patients can also book over the phone through an AI voice receptionist that speaks English and Hindi.

![Repositories](https://img.shields.io/badge/repositories-5-blue)
![Next.js](https://img.shields.io/badge/Next.js-16-black?logo=next.js)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-4.2-6DB33F?logo=springboot&logoColor=white)
![Express](https://img.shields.io/badge/Express-5-000000?logo=express)
![Python](https://img.shields.io/badge/Python-3.14-3776AB?logo=python&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)

```
                       ┌──────────────────────┐        ┌───────────────────────┐
                       │  client-side :3000   │        │  receptionist :7000   │
                       │  Next.js · React     │        │  FastAPI · LangGraph  │
                       └─────┬──────────┬─────┘        │  Twilio voice calls   │
                             │          │              └──────────┬────────────┘
              ┌──────────────▼───┐  ┌───▼──────────────┐          │
              │ spring-api :8080 │  │ express-api :5000│          │
              │ auth · JWTs      │  │ doctors · slots  │          │
              │ Spring Boot      │  │ appointments     │          │
              └────────┬─────────┘  └───┬──────────┬───┘          │
                       │                │          │              │
               ┌───────▼────────────────▼──┐  ┌────▼────────┐     │
               │ PostgreSQL 16 + PostGIS   │  │ MongoDB 7   │     │
               │ Redis 7 (holds, cache)    │◄─┼─────────────┼─────┘
               └───────────────────────────┘  └─────────────┘
```

## Repositories

The organisation has **5 public repositories**:

| Repository | Language | What it does |
|---|---|---|
| [`client-side`](https://github.com/medica-health/client-side) | TypeScript | Next.js web app for patients, doctors and admins |
| [`spring-api`](https://github.com/medica-health/spring-api) | Java | Accounts and authentication. Owns the `auth` schema and issues the JWTs |
| [`express-api`](https://github.com/medica-health/express-api) | TypeScript | Domain API: doctor search ("near me" and by city), scheduling, appointments, prescriptions, medical records, reviews, admin, email and SMS notifications |
| [`receptionist`](https://github.com/medica-health/receptionist) | Python | Phone receptionist: answers calls, verifies the caller and books or changes appointments (FastAPI, LangGraph + Claude, Twilio ConversationRelay) |
| [`.github`](https://github.com/medica-health/.github) | — | This profile, the PR template and the full-stack `docker-compose.yml` |

## Quick start

**Prerequisites:** Git and Docker with Compose v2.

Clone the repositories side by side into one folder, which the Compose file expects:

```bash
mkdir medicare && cd medicare

for repo in .github client-side spring-api express-api receptionist; do
  git clone https://github.com/medica-health/$repo.git
done
```

Then start the whole stack from that folder:

```bash
docker compose -f .github/docker-compose.yml --project-directory . up --build
```

| Service | URL |
|---|---|
| Web app | http://localhost:3000 |
| Spring API | http://localhost:8080/api/health |
| Express API | http://localhost:5000/api/health |
| Receptionist | http://localhost:7000/api/health |
| PostgreSQL + PostGIS | `localhost:5432` |
| MongoDB | `localhost:27017` |
| Redis | `localhost:6379` |

Stop it with `down`; add `-v` to also delete the database volumes.

### Running one service on its own

Start only the databases, then run the app you're working on:

```bash
docker compose -f .github/docker-compose.yml --project-directory . up -d postgres mongo redis
```

| Repository | Needs | Commands |
|---|---|---|
| `client-side` | [Bun](https://bun.sh) 1.3+ | `bun install` → `bun dev` |
| `express-api` | Bun 1.3+ | `cp .env.example .env` → `bun install` → `bun run dev` |
| `spring-api` | JDK 21 | `./mvnw spring-boot:run` (Windows: `mvnw.cmd`) |
| `receptionist` | Python 3.14, [uv](https://docs.astral.sh/uv/) | `cp .env.example .env` → `uv sync` → `uv run receptionist serve` |

Each repository's README covers its own configuration.

## API conventions

Every service answers `GET /` with `{ "message": "Running" }` and `GET /api/health` with the status of its dependencies. Responses share one envelope:

```json
{
  "success": true,
  "message": "API is running successfully",
  "data": { "...": "..." },
  "timestamp": "2026-11-22T10:00:00.000Z"
}
```

Errors use the same envelope with `"success": false`.

## Contributing

Every change goes through a pull request; nobody pushes directly to `main`.

### 1. Pick up the work

Open or pick an issue in the repository you'll change and assign it to yourself, so others know it's taken.

### 2. Pull the latest `main`

```bash
cd express-api              # the repository you're working in
git checkout main
git pull origin main
```

### 3. Create a new branch

Branch from the updated `main`. Use a prefix, the issue number and a short description:

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
- Keep the services in step: if you change a booking rule, a shared database column or a notification, check whether `express-api` and `receptionist` both need it.
- Never commit `.env` files or other secrets.

Run the checks for the repository you changed:

| Repository | Commands |
|---|---|
| `client-side` | `bun run lint` and `bun run build` |
| `express-api` | `bun run lint` and `bun run typecheck` |
| `spring-api` | `./mvnw test` |

### 5. Commit

Write a short summary line in the imperative mood, and reference the issue:

```bash
git status                  # see what changed
git add <files>
git commit -m "Add login endpoint" -m "Refs #12"
```

### 6. Stay up to date with `main`

If `main` has moved on while you were working, rebase your branch onto it:

```bash
git fetch origin
git rebase origin/main
# fix any conflicts, then:
git add <files>
git rebase --continue
```

### 7. Push your branch

The first push creates the branch on GitHub and links it to your local branch:

```bash
git push -u origin feature/12-user-login
```

After that, a plain `git push` is enough. If you rebased a branch you had already pushed:

```bash
git push --force-with-lease
```

Only force-push your **own** feature branch, never `main`.

### 8. Raise the pull request

**On GitHub:** open the repository, click **Compare & pull request** on the banner for your branch, set the base to `main`, and click **Create pull request**.

**Or from the terminal** with the [GitHub CLI](https://cli.github.com):

```bash
gh pr create --base main --fill          # add --draft if it isn't ready for review
```

Fill in the [PR template](https://github.com/medica-health/.github/blob/main/PULL_REQUEST_TEMPLATE.md):

- **What** changed and **why**
- **How to test** it
- `Closes #12` to link and auto-close the issue
- Screenshots for UI changes
- Links to related PRs in other repositories, if the change spans several

### 9. Review and merge

- Request at least **one reviewer**. Answer every comment with a fix or a reply.
- Push follow-up commits to the same branch; the PR updates automatically.
- Once it's approved and CI passes, use **Squash and merge** and delete the branch.

Then clean up locally:

```bash
git checkout main
git pull origin main
git branch -D feature/12-user-login     # -D because squash merges aren't seen as merged by Git
```

## Tech stack

| Layer | Technology |
|---|---|
| Web app | Next.js 16, React 19, TypeScript, Tailwind CSS v4, Bun |
| Auth API | Spring Boot 4.2, Java 21, Spring Data JPA, Maven |
| Domain API | Express 5, TypeScript, Bun, PostGIS, Mongoose, Redis |
| Voice receptionist | Python 3.14, FastAPI, LangGraph, Claude, Twilio ConversationRelay |
| Data | PostgreSQL 16 + PostGIS, MongoDB 7, Redis 7 |
| Notifications | Resend (email), Twilio (SMS) |

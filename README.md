# Turborepo starter

This Turborepo starter is maintained by the Turborepo core team.

## Using this example

Run the following command:

```sh
npx create-turbo@latest
```

## What's inside?

This Turborepo includes the following packages/apps:

### Apps and Packages

- `docs`: a [Next.js](https://nextjs.org/) app
# monorepo-vms-deploy — DevOps README

## Project Title
- **monorepo-vms-deploy**

## Project Description
- This repository contains a small monorepo with three services: a backend (`apps/backend`), a Next.js frontend (`apps/web`), and a websocket service (`apps/websocket`). The repository includes Dockerfiles for containerizing services and a GitHub Actions workflow for deploying the backend. This README documents how to build and run services with Docker and describes the CI/CD workflow.

## Features
- Dockerized services (Dockerfiles for backend, frontend, websocket)
- CI/CD automation via GitHub Actions (`.github/workflows/cd_backend.yml`)
- Simple container-based deployment pattern (image build + remote pull/run)

## Tech Stack
- Docker (image builds and runtime)
- GitHub Actions (CI/CD workflow located at `.github/workflows/cd_backend.yml`)

## Folder structure (key files)
- `apps/`
	- `apps/backend/` — backend source and `package.json`. Backend Dockerfile: `docker/Dockerfile.backend` (exposes `8080`).
	- `apps/web/` — Next.js frontend (build outputs found in `.next/`). Frontend Dockerfile: `docker/Dockerfile.frontend` (build uses `ARG DATABASE_URL`).
	- `apps/websocket/` — websocket service. Dockerfile: `docker/Dockerfile.ws` (exposes `8081`).
- `docker/`
	- `Dockerfile.backend` — builds and starts the backend using `oven/bun:1`.
	- `Dockerfile.frontend` — builds the frontend and runs `bun run start:web`.
	- `Dockerfile.ws` — builds websocket service and runs `bun run start:websocket`.
- `.github/workflows/cd_backend.yml` — existing GitHub Actions workflow (currently checks out code and logs into Docker Hub; see CI/CD notes below).
- `docker-compose.yml` — present in repo root but currently empty; sample compose provided in this README for local development.
- `packages/db/.env` — DB environment hint file present in `packages/db`.

## How to Build & Run with Docker

Build backend image (run from repo root):
```powershell
docker build -f docker/Dockerfile.backend -t <DOCKERHUB_USER>/monorepo-backend:local .
```

Run backend container:
```powershell
docker run --rm -p 8080:8080 -e DATABASE_URL="postgres://user:pass@host:5432/db" <DOCKERHUB_USER>/monorepo-backend:local
```

Build websocket image:
```powershell
docker build -f docker/Dockerfile.ws -t <DOCKERHUB_USER>/monorepo-ws:local .
```

Run websocket container:
```powershell
docker run --rm -p 8081:8081 -e DATABASE_URL="postgres://user:pass@host:5432/db" <DOCKERHUB_USER>/monorepo-ws:local
```

Docker Compose (example)
- The repo contains an empty `docker-compose.yml` at root. Use the snippet below as a starting point (create `docker-compose.override.yml` or replace `docker-compose.yml`):

```yaml
version: '3.8'
services:
	db:
		image: postgres:15
		environment:
			POSTGRES_USER: dev
			POSTGRES_PASSWORD: dev
			POSTGRES_DB: devdb
		volumes:
			- db_data:/var/lib/postgresql/data

	backend:
		build:
			context: .
			dockerfile: docker/Dockerfile.backend
		environment:
			DATABASE_URL: postgres://dev:dev@db:5432/devdb
		ports:
			- "8080:8080"
		depends_on:
			- db

	websocket:
		build:
			context: .
			dockerfile: docker/Dockerfile.ws
		environment:
			DATABASE_URL: postgres://dev:dev@db:5432/devdb
		ports:
			- "8081:8081"
		depends_on:
			- db

volumes:
	db_data:
```

Run compose (from repo root):
```powershell
docker-compose -f docker-compose.yml -f docker-compose.override.yml up --build
```

## CI/CD Pipeline Explanation (GitHub Actions)

Current workflow
- File: `.github/workflows/cd_backend.yml` — present in the repo. It currently checks out the code and runs Docker login using `secrets.DOCKERHUB_USERNAME` and `secrets.DOCKERHUB_TOKEN`. It does not yet include build/push or deploy steps.

Recommended pipeline flow
1. Trigger: `push` to `main` (current) — consider protecting `main` and using a release branch for production deploys.
2. Steps:
	 - Checkout code (`actions/checkout@v3`).
	 - Optionally set up QEMU and Buildx for multi-arch builds.
	 - Docker login (`docker/login-action@v2`) using `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` stored as GitHub Secrets.
	 - Build and push image (`docker/build-push-action@v4`) tagging with `latest` and `${{ github.sha }}`.
	 - Deploy: use an SSH action (`appleboy/ssh-action`) or custom deploy script to pull the image on your VM and restart the container.

What steps run
- Build: create a Docker image using the appropriate `Dockerfile`.
- Test: no tests are present now; add test steps if you introduce tests.
- Deploy: SSH to VM and `docker pull` + `docker run`/`docker-compose`.

Branch rules
- Current workflow triggers on `main`. Recommended:
	- Use PRs for feature work; run CI on PRs (lint/test/build) and restrict merges to `main` via protected branches.

## Environment Variables (inferred)
- `DATABASE_URL` — used by Dockerfile build steps for migrations/codegen and at runtime.
- `DOCKERHUB_USERNAME` & `DOCKERHUB_TOKEN` — required as GitHub Actions secrets for Docker login and push.
- `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`, `DEPLOY_SSH_PORT` — if you implement SSH-based deploy in CI.

## Common Commands
- Build backend image:
```powershell
docker build -f docker/Dockerfile.backend -t myuser/monorepo-backend:latest .
```
- Run backend locally:
```powershell
docker run --rm -p 8080:8080 -e DATABASE_URL="postgres://..." myuser/monorepo-backend:latest
```
- Build & run websocket image:
```powershell
docker build -f docker/Dockerfile.ws -t myuser/monorepo-ws:latest .
docker run --rm -p 8081:8081 -e DATABASE_URL="..." myuser/monorepo-ws:latest
```
- Compose up (example):
```powershell
docker-compose up --build
```

## Troubleshooting (quick)
- "COPY failed" during `docker build`: make sure you run `docker build` from repo root because the Dockerfiles copy `./packages` and `./apps/*`.
- Migrations failing during image build: Dockerfiles run `bun run db:generate` / `bun run db:migrate`. Either provide `DATABASE_URL` as a build arg/secret or move migrations to a separate run step after the DB is available.
- GitHub Actions: Docker login failing — check that `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` are set in repository Secrets.
- SSH deploy fails — validate the SSH key, user, and host before adding them to CI secrets. Test locally first.

## Contribution
- Open an issue or a pull request for changes. Keep PRs focused to a single change (e.g., CI workflow, Dockerfile fix, service update).
- Add CI steps to validate changes (lint/tests) before merging.

## License
- No `LICENSE` file was found in the repository. Add a `LICENSE` file (for example `MIT`) to explicitly specify licensing.

## Author
- Repository owner: `shrawanyadkr2` (repo: `monorepo-vms-deploy`)

---

If you want, I can implement the full GitHub Actions build/push/deploy workflow and a production `deploy.sh` for your VM. Tell me which to do next.

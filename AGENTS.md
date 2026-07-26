# Vortex Ops — Agent Guide

## TL;DR

Monorepo: `apps/backend` (NestJS 11) + `apps/frontend` (Next.js 16.2.10).  
Infra: Docker Compose (PostgreSQL 16 + backend + frontend).

---

## Quick Start

```bash
docker compose up --build -d       # full stack
docker compose up postgres -d       # DB only (for native backend dev)
```

| Service  | Host Port | URL                     |
|----------|-----------|-------------------------|
| postgres | 5432      |                         |
| backend  | 3001      | http://localhost:3001/api |
| frontend | 3000      | http://localhost:3000   |

---

## Backend (NestJS 11)

```
apps/backend/
├── src/
│   ├── config/          # typed env + cors config
│   ├── main.ts          # entrypoint, sets global /api prefix
│   └── app.module.ts    # ConfigModule (isGlobal) imported here
├── Dockerfile           # multi-stage build
└── package.json
```

**Commands** (run from `apps/backend`):
- `npm run start:dev` — watch mode (natively, requires postgres in Docker)
- `npm run build` — compile to `dist/`
- `npm test` — unit tests (Jest, `*.spec.ts`)
- `npm run test:e2e` — e2e tests (Supertest)
- `npm run lint` — ESLint fix

**Key deps:** `@nestjs/config` (ConfigModule), `@nestjs/platform-express`.  
**Config:** `ConfigModule.forRoot({ isGlobal: true, envFilePath: '.env' })`.  
**tsconfig quirks:** `module: "nodenext"`, `moduleResolution: "nodenext"`, decorators enabled, `noImplicitAny: false`.

**New module scaffold:**
```bash
nest g module modules/<name>
nest g service modules/<name>
nest g controller modules/<name>
```

**Testing:** Jest config in package.json (`rootDir: "src"`, `testRegex: ".*\\.spec\\.ts$"`).

---

## Frontend (Next.js 16.2.10)

> ⚠️ Next.js 16 has breaking changes. Read `node_modules/next/dist/docs/` before writing code.

```
apps/frontend/
├── app/              # App Router pages
├── next.config.ts    # output: "standalone" (required for Docker)
├── Dockerfile        # multi-stage, copies .next/standalone
└── package.json
```

**Commands** (run from `apps/frontend`):
- `npm run dev` — dev server
- `npm run build` — production build (generates `.next/standalone/`)
- `npm run lint` — ESLint

**Env:** `NEXT_PUBLIC_API_URL=http://localhost:3001/api` (browser-side).

---

## Docker

```bash
docker compose up --build -d   # rebuild + start all
docker compose build backend   # build single service
docker compose logs backend    # tail logs
docker compose down            # stop + remove containers
```

**PostgreSQL:** healthcheck via `pg_isready`. Backend waits for `service_healthy`.  
**Volumes:** `vortex-postgres-data` persists DB data across restarts.  
**.env** at repo root — used by Docker Compose. Backend gets env vars via `environment:` in compose, not from `.env` file inside container.

| Variable          | How it reaches backend container |
|-------------------|----------------------------------|
| `DATABASE_URL`    | hardcoded in compose `environment:` using `@postgres:5432` |
| `JWT_SECRET`      | `${JWT_SECRET}` from compose → `.env` |
| `PORT`            | hardcoded as `3000` in compose   |

**Dockerfiles:** Both use multi-stage build (builder → runner alpine images).

---

## Environment

Single `.env` at repo root (gitignored). Copy `.env.example` for reference.  
Backend `ConfigModule` uses `envFilePath: '.env'` — resolves relative to CWD (works natively from `apps/backend/`, in Docker env vars are injected directly).

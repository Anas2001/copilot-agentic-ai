---
name: scaffold-project
description: Generate the initial multi-app monorepo project structure based on docs/architecture/stack.yml.
allowed-tools: Read Grep Glob Write Edit Create
---

# Scaffold Project Skill

## Prerequisites

**Before running this skill**, verify:
1. `docs/architecture/stack.yml` exists and has `status: confirmed`
2. The user has explicitly approved the stack via the `orchestrator-agent`

If either condition is false, stop and call `architect-agent` first.

---

## Step 1 — Read the Stack & Detect Ecosystems

Read `docs/architecture/stack.yml` and extract:

```
packageManager   → pnpm | npm | yarn | maven | gradle | pip | poetry | go
monorepo         → true | false
frontend.*       → framework, language, styling, ui
backend.*        → framework, language
database.*       → provider, orm
mobile.*         → framework, language (optional — skip if absent)
messagebroker.*  → provider (optional — skip if absent)
auth.*           → provider
testing.*        → unit, e2e
deployment.*     → container, ci
```

Then determine the **language ecosystem** for each app — this controls which root shell, shared packages, Dockerfile, and CI steps to generate:

| Ecosystem | Detected when |
|-----------|---------------|
| **Node.js / TypeScript** | `backend.language` or `frontend.language` is `typescript` or `javascript` |
| **Java / JVM** | `backend.language` is `java`, `kotlin`, or `scala` |
| **Python** | `backend.language` is `python` |
| **Go** | `backend.language` is `go` |
| **Flutter / Dart** | `mobile.framework` is `flutter` |
| **Mixed** | Multiple apps use different ecosystems — handle each app independently |

Use these detected values to fill every conditional block in the steps below. Skip any block that does not apply.

---

## Step 2 — Root Monorepo Shell

Create the root shell based on the **detected ecosystem(s)**. Always create the universal files first, then apply the ecosystem-specific additions.

### Universal files (always create)

See the `.gitignore` and `.env.example` blocks below — these are always created.

### If ecosystem includes **Node.js / TypeScript**

#### `pnpm-workspace.yaml` (if `packageManager: pnpm`)
```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

#### Root `package.json`
```json
{
  "name": "<repo-name>",
  "private": true,
  "scripts": {
    "dev": "<packageManager> --filter './apps/*' run dev",
    "build": "<packageManager> --filter './apps/*' run build",
    "test": "<packageManager> --filter './apps/*' run test",
    "lint": "<packageManager> --filter './apps/*' run lint",
    "typecheck": "<packageManager> --filter './apps/*' run typecheck",
    "db:migrate": "<packageManager> --filter api run db:migrate",
    "db:generate": "<packageManager> --filter api run db:generate"
  },
  "devDependencies": {}
}
```
> Replace `<packageManager>` with `pnpm`, `npm run`, or `yarn` based on `stack.yml`.

#### `tsconfig.base.json` (if any app uses TypeScript)
```json
{
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "bundler",
    "target": "ES2022",
    "lib": ["ES2022"],
    "paths": {}
  }
}
```

### If ecosystem includes **Java / JVM** (Maven)

#### `pom.xml` (parent multi-module)
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId><repo-name>-parent</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>pom</packaging>
  <modules>
    <module>apps/api</module>
    <!-- add more modules as needed -->
  </modules>
  <properties>
    <java.version>21</java.version>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
  </properties>
</project>
```

#### `.mvn/wrapper/maven-wrapper.properties` (optional but recommended)
```properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.6/apache-maven-3.9.6-bin.zip
```

### If ecosystem includes **Java / JVM** (Gradle)

#### `settings.gradle.kts`
```kotlin
rootProject.name = "<repo-name>"
include(":apps:api")
// add more subprojects as needed
```

#### Root `build.gradle.kts`
```kotlin
plugins {
    // shared plugin versions
}
subprojects {
    // shared configuration
}
```

### If ecosystem includes **Python**

#### `pyproject.toml` (if `packageManager: poetry` or `uv`)
```toml
[tool.poetry]
name = "<repo-name>"
version = "0.1.0"

[tool.poetry.workspaces]
packages = ["apps/*"]
```

### If ecosystem includes **Go**

#### `go.work`
```
go 1.22

use (
    ./apps/api
    // add more modules as needed
)
```

### Mixed ecosystems

When apps use different languages (e.g. Java backend + TypeScript frontend):
- Create Node.js root shell files only if at least one app is Node.js/TypeScript
- Create the Java parent POM/Gradle root only if at least one app is Java
- Use `docker-compose.yml` as the universal top-level orchestration layer
- Each app manages its own build tool independently

### `.gitignore`

Build the `.gitignore` by combining the **base entries** (always included) with every applicable **conditional block** resolved from `stack.yml`.

```
# ── Base (always included) ────────────────────────────────────────────────────
node_modules/
dist/
build/
.turbo/
coverage/
*.log
*.tsbuildinfo

# Environment variables — never commit real secrets
.env
.env.local
.env.*.local
*.env

# Editor / OS
.DS_Store
Thumbs.db
.idea/
.vscode/*
!.vscode/extensions.json
!.vscode/settings.json
```

Append the following blocks based on `stack.yml`:

**If `backend.framework: nestjs`**
```
# NestJS
apps/api/dist/
apps/api/.nest/
```

**If `frontend.framework: reactjs` (Vite)**
```
# Vite / React
apps/web/dist/
apps/web/.vite/
```

**If `frontend.framework: nextjs`**
```
# Next.js
apps/web/.next/
apps/web/out/
```

**If `database.orm: prisma`**
```
# Prisma
apps/api/prisma/migrations/dev.db
apps/api/prisma/*.db
```

**If `mobile.framework: react-native`**
```
# React Native
apps/mobile/android/.gradle/
apps/mobile/android/app/build/
apps/mobile/android/local.properties
apps/mobile/ios/Pods/
apps/mobile/ios/build/
apps/mobile/ios/*.xcworkspace/xcuserdata/
apps/mobile/.expo/
apps/mobile/node_modules/
```

**If `mobile.framework: expo`**
```
# Expo
apps/mobile/.expo/
apps/mobile/dist/
apps/mobile/web-build/
apps/mobile/node_modules/
```

**If `mobile.framework: flutter`**
```
# Flutter
apps/mobile/.dart_tool/
apps/mobile/.flutter-plugins
apps/mobile/.flutter-plugins-dependencies
apps/mobile/build/
apps/mobile/android/.gradle/
apps/mobile/android/local.properties
apps/mobile/ios/Pods/
apps/mobile/ios/.symlinks/
```

**If `deployment.container: docker`**
```
# Docker
*.pid
docker-compose.override.yml
```

**If `testing.e2e: playwright`**
```
# Playwright
playwright-report/
test-results/
```

**If `messagebroker.provider` is present**
```
# Message broker local data
.rabbitmq/
.kafka/
```

### `.env.example`
```
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/dbname

# Auth (from stack.yml auth.provider)
# TODO: Add provider-specific keys here

# Message broker (from stack.yml messagebroker.provider, if present)
# TODO: e.g. RABBITMQ_URL=amqp://localhost:5672
```

---

## Step 3 — Shared Packages

Create shared packages only when the ecosystem supports workspaces and cross-app type/config sharing is beneficial. **Skip this step entirely for single-language backends (Java, Python, Go) with no shared library needs.**

### If ecosystem is **Node.js / TypeScript** (with multiple Node.js apps)

#### `packages/types/` — Shared types
```
packages/types/
├── package.json          # name: "@repo/types"
├── tsconfig.json         # extends ../../tsconfig.base.json
└── src/
    └── index.ts          # export * from './models'
```

#### `packages/config/` — Shared environment helpers
```
packages/config/
├── package.json          # name: "@repo/config"
├── tsconfig.json
└── src/
    └── index.ts          # export const env = { ... }
```

### If ecosystem is **Java / JVM**

Create a `packages/shared/` Maven/Gradle module for shared domain models and interfaces if multiple Java apps need them. Otherwise skip.

### If ecosystem is **Python**

Create a `packages/shared/` Python package installable as a local dependency via `pip install -e ./packages/shared`.

### If ecosystem is **Go**

Create a `packages/shared/` Go module for shared types. Reference it via `replace` directives in each app's `go.mod`.

### Mixed ecosystems

Do not share code directly across language boundaries. Define API contracts (OpenAPI, Protobuf, AsyncAPI) in `docs/contracts/` instead.

---

## Step 4 — Backend App (`apps/api`)

> ⚠️ **MANDATORY STRUCTURE — NON-NEGOTIABLE**
> This folder layout is fixed for ALL backend apps, regardless of framework, language, or tech stack.
> Framework-specific sections below only add **build config files and scripts** — they do NOT change or replace this structure.
> Never deviate from this layout. If a framework convention conflicts, follow this structure and adapt the framework to fit.

### Universal backend folder structure

```
apps/api/
├── <build-config>          # package.json | pom.xml | build.gradle | pyproject.toml
├── .env.example
├── src/  (or src/main/<lang>/ for Java/Kotlin)
│   ├── adapters/           # External system adapters (payment, email, broker)
│   │   └── messaging/      # Message queue producers/consumers
│   ├── endpoints/          # API controllers/routes/handlers
│   │   └── rest/
│   │       ├── middleware/
│   │       ├── validation/
│   │       ├── dtos/
│   │       └── mappers/
│   ├── business/           # Domain logic — framework-free
│   │   ├── <feature>/
│   │   ├── repositories/   # Interfaces only
│   │   ├── adapters/       # Interfaces only
│   │   └── services/
│   ├── persistence/
│   │   └── <db-tech>/
│   │       ├── entities/
│   │       ├── mappers/
│   │       ├── migrations/
│   │       └── repositories/
│   ├── models/
│   ├── config/
│   └── utils/
└── tests/
    ├── integration/
    └── mocks/
```

> Exception — **Go** only: use `internal/` instead of `src/`, and `handlers/` instead of `endpoints/rest/` — Go language idiom. All other subdirectory names remain identical.

> The `backend-agent` enforces the dependency rules between these layers. Refer to `backend-agent` instructions for the complete rule set.

---

### Framework-specific additions

> The sections below describe ONLY the **build config files and package scripts** to add on top of the universal structure above. They do not define a folder structure.

---

### If `backend.framework: nestjs` (TypeScript)

Add to the universal structure:
```
apps/api/
├── package.json
├── tsconfig.json
├── tsconfig.build.json
├── nest-cli.json
└── prisma/                 # only if database.orm: prisma
    └── schema.prisma
```

`package.json` scripts:
```json
{
  "scripts": {
    "dev": "nest start --watch",
    "build": "nest build",
    "test": "vitest run",
    "lint": "eslint src --ext .ts",
    "typecheck": "tsc --noEmit",
    "db:migrate": "prisma migrate dev",
    "db:generate": "prisma generate"
  }
}
```

---

### If `backend.framework: express` or `fastify` (TypeScript / JavaScript)

Add to the universal structure:
```
apps/api/
├── package.json
├── tsconfig.json           # if TypeScript
└── src/
    └── main.ts             # app bootstrap
```

---

### If `backend.framework: quarkus` (Java)

Use Maven or Gradle (from `packageManager`):
```
apps/api/
├── pom.xml  (or build.gradle.kts)
└── src/
    ├── main/
    │   ├── java/com/example/
    │   │   ├── adapters/
    │   │   │   └── messaging/      # Message queue producers/consumers
    │   │   ├── endpoints/rest/     # JAX-RS resources
    │   │   │   ├── middleware/
    │   │   │   ├── validation/
    │   │   │   ├── dtos/
    │   │   │   └── mappers/
    │   │   ├── business/
    │   │   │   ├── repositories/   # Interfaces only
    │   │   │   ├── adapters/       # Interfaces only
    │   │   │   └── services/
    │   │   ├── persistence/
    │   │   │   └── <db-tech>/
    │   │   │       ├── entities/
    │   │   │       ├── mappers/
    │   │   │       ├── migrations/
    │   │   │       └── repositories/
    │   │   ├── models/
    │   │   └── config/
    │   └── resources/
    │       ├── application.properties
    │       └── META-INF/resources/
    └── test/
        └── java/com/example/
            ├── integration/
            └── mocks/
```

`pom.xml` scripts (run via `mvn`):
```
mvn quarkus:dev          # dev mode
mvn test                 # unit tests
mvn package              # build
```

---

### If `backend.framework: spring-boot` (Java / Kotlin)

```
apps/api/
├── pom.xml  (or build.gradle.kts)
└── src/
    ├── main/
    │   ├── java/com/example/
    │   │   ├── adapters/
    │   │   │   └── messaging/      # Message queue producers/consumers
    │   │   ├── endpoints/rest/     # @RestController
    │   │   │   ├── middleware/
    │   │   │   ├── validation/
    │   │   │   ├── dtos/
    │   │   │   └── mappers/
    │   │   ├── business/
    │   │   │   ├── repositories/   # Interfaces only
    │   │   │   ├── adapters/       # Interfaces only
    │   │   │   └── services/
    │   │   ├── persistence/
    │   │   │   └── <db-tech>/
    │   │   │       ├── entities/
    │   │   │       ├── mappers/
    │   │   │       ├── migrations/
    │   │   │       └── repositories/
    │   │   ├── models/
    │   │   └── config/
    │   └── resources/
    │       └── application.yml
    └── test/
        └── java/com/example/
            ├── integration/
            └── mocks/
```

---

### If `backend.framework: fastapi` or `django` (Python)

```
apps/api/
├── pyproject.toml
├── requirements.txt        # or managed by poetry/uv
└── src/
    ├── adapters/
    │   └── messaging/      # Message queue producers/consumers
    ├── endpoints/
    │   └── rest/           # routers / views
    │       ├── middleware/
    │       ├── validation/
    │       ├── dtos/
    │       └── mappers/
    ├── business/
    │   ├── repositories/   # Interfaces only
    │   ├── adapters/       # Interfaces only
    │   └── services/
    ├── persistence/
    │   └── <db-tech>/
    │       ├── entities/
    │       ├── mappers/
    │       ├── migrations/
    │       └── repositories/
    ├── models/
    ├── config/
    └── main.py             # app bootstrap
tests/
├── integration/
└── mocks/
```

---

### If `backend.framework: gin`, `echo`, or `fiber` (Go)

> Go idiom: uses `internal/` instead of `src/`, and `handlers/` instead of `endpoints/rest/` — maps to the same logical layer.

```
apps/api/
├── go.mod
├── go.sum
├── internal/
│   ├── adapters/
│   │   └── messaging/      # Message queue producers/consumers
│   ├── handlers/           # HTTP handlers (equivalent to endpoints/rest/)
│   │   ├── middleware/
│   │   ├── validation/
│   │   ├── dtos/
│   │   └── mappers/
│   ├── business/
│   │   ├── repositories/   # Interfaces only
│   │   ├── adapters/       # Interfaces only
│   │   └── services/
│   ├── persistence/
│   │   └── <db-tech>/
│   │       ├── entities/
│   │       ├── mappers/
│   │       ├── migrations/
│   │       └── repositories/
│   ├── models/
│   └── config/
└── tests/
    ├── integration/
    └── mocks/
```

---

### ORM / Database scaffolding (all frameworks)

| `database.orm` | Files to create |
|---|---|
| `prisma` | `apps/api/prisma/schema.prisma` with datasource block |
| `typeorm` | `src/persistence/<db>/entities/` with base entity |
| `hibernate` / `panache` | JPA entity classes in `persistence/<db>/entities/` |
| `sqlalchemy` | `src/persistence/<db>/models.py` |
| `gorm` | `internal/persistence/<db>/models.go` |
| none | Empty `persistence/` folder with README |

---

## Step 5 — Frontend App (`apps/web`)

Apply the block matching `frontend.framework` in `stack.yml`. The **feature-based folder structure** is universal — only the file formats, config files, and build tools differ.

### Universal frontend folder structure

```
apps/web/
├── <build-config>          # package.json | pubspec.yaml
├── <framework-config>      # vite.config.* | next.config.* | angular.json | etc.
├── <test-config>           # vitest.config.* | jest.config.* | playwright.config.*
├── public/
└── src/
    ├── <entry>             # main.ts | main.tsx | main.js | App.vue | etc.
    ├── app/                # Route/page entry points (framework-specific name)
    ├── features/           # Feature-based modules
    │   └── <feature>/
    │       ├── components/
    │       ├── hooks/       # or composables/ (Vue) | services/ (Angular)
    │       ├── queries/     # Data fetching
    │       ├── store/       # Local state
    │       └── types        # Feature types (.ts | .d.ts | interface files)
    ├── components/         # Shared UI components
    ├── lib/
    │   ├── api/             # API client
    │   └── utils/
    ├── styles/
    ├── types/
    └── config/
        └── env              # Environment helpers
```

> The `frontend-agent` enforces the conventions and naming within this structure based on the loaded skill.

---

### If `frontend.framework: reactjs` (Vite)

```
apps/web/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── index.html
├── tailwind.config.ts      # if styling: tailwind
├── components.json         # if ui: shadcn
└── vitest.config.ts
```

`package.json` scripts:
```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "test": "vitest run",
    "test:e2e": "playwright test",
    "lint": "eslint src",
    "typecheck": "tsc --noEmit"
  }
}
```

---

### If `frontend.framework: nextjs`

```
apps/web/
├── package.json
├── tsconfig.json
├── next.config.ts
├── tailwind.config.ts      # if styling: tailwind
└── src/
    └── app/                # Next.js App Router
        ├── layout.tsx
        └── page.tsx
```

---

### If `frontend.framework: vuejs`

```
apps/web/
├── package.json
├── tsconfig.json
├── vite.config.ts          # with @vitejs/plugin-vue
├── tailwind.config.ts      # if styling: tailwind
└── src/
    ├── main.ts
    ├── App.vue
    ├── router/             # vue-router
    ├── stores/             # Pinia stores (equivalent to store/)
    └── features/
        └── <feature>/
            ├── components/
            ├── composables/ # Vue equivalent of hooks/
            ├── queries/
            └── types.ts
```

---

### If `frontend.framework: angular`

```
apps/web/
├── package.json
├── angular.json
├── tsconfig.json
└── src/
    ├── main.ts
    ├── app/
    │   ├── app.component.ts
    │   ├── app.config.ts
    │   └── app.routes.ts
    └── features/
        └── <feature>/
            ├── components/
            ├── services/    # Angular equivalent of hooks/
            ├── store/       # NgRx / Signal store
            └── models/
```

---

### If `frontend.framework: svelte` or `sveltekit`

```
apps/web/
├── package.json
├── svelte.config.js
├── vite.config.ts
└── src/
    ├── app.html
    ├── routes/             # SvelteKit file-based routing
    └── lib/
        ├── components/
        ├── stores/
        └── utils/
```

---

## Step 6 — Mobile App (`apps/mobile`) — skip if not in stack

**Skip this step** if `stack.yml` has no `mobile` section.

Apply the block matching `mobile.framework`. The `mobile-agent` will enforce framework-specific conventions after scaffolding.

### Universal mobile folder structure

```
apps/mobile/
├── <build-config>          # package.json | pubspec.yaml | build.gradle
└── src/  (or lib/ for Flutter)
    ├── <entry>             # index.js | main.dart | App.tsx
    ├── navigation/         # Route configuration
    ├── features/
    │   └── <feature>/
    │       ├── screens/    # or pages/ or views/
    │       ├── components/
    │       ├── hooks/       # or services/
    │       ├── queries/
    │       └── types
    ├── components/
    ├── lib/
    │   ├── api/
    │   └── utils/
    └── config/
```

### If `mobile.framework: react-native`

```
apps/mobile/
├── package.json
├── tsconfig.json
├── app.json
├── babel.config.js
└── metro.config.js
```

### If `mobile.framework: expo`

```
apps/mobile/
├── package.json
├── tsconfig.json
├── app.json             # or app.config.ts
└── babel.config.js
```

### If `mobile.framework: flutter`

```
apps/mobile/
├── pubspec.yaml
├── analysis_options.yaml
└── lib/
    ├── main.dart
    ├── app/
    ├── features/
    │   └── <feature>/
    │       ├── screens/
    │       ├── widgets/
    │       ├── providers/
    │       └── models/
    └── shared/
        ├── widgets/
        └── utils/
```

### If `mobile.framework: ionic`

```
apps/mobile/
├── package.json
├── tsconfig.json
├── ionic.config.json
└── src/
    ├── main.ts
    └── app/
```

---

## Step 7 — Docker & Infrastructure

### `docker-compose.yml` (development)
```yaml
services:
  # Database — from stack.yml database.provider
  db:
    image: postgres:17-alpine          # adjust for mysql / mongodb
    restart: unless-stopped
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: appdb
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data

  # Message broker — from stack.yml messagebroker.provider (omit if absent)
  rabbitmq:
    image: rabbitmq:3-management-alpine
    restart: unless-stopped
    ports:
      - '5672:5672'
      - '15672:15672'

  # API
  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
    restart: unless-stopped
    env_file: apps/api/.env
    ports:
      - '3000:3000'
    depends_on:
      - db

volumes:
  postgres_data:
```

### `apps/api/Dockerfile` — conditional per backend ecosystem

**If `backend.language: typescript` or `javascript`** (Node.js):
```dockerfile
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@latest --activate
WORKDIR /app

FROM base AS deps
COPY pnpm-workspace.yaml package.json pnpm-lock.yaml ./
COPY apps/api/package.json ./apps/api/
COPY packages/ ./packages/
RUN pnpm install --frozen-lockfile

FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/apps/api/node_modules ./apps/api/node_modules
COPY . .
RUN pnpm --filter api run build

FROM node:22-alpine AS runner
WORKDIR /app
COPY --from=builder /app/apps/api/dist ./dist
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

**If `backend.language: java` (Maven / Quarkus / Spring Boot)**:
```dockerfile
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY . .
RUN ./mvnw -pl apps/api package -DskipTests

FROM eclipse-temurin:21-jre AS runner
WORKDIR /app
COPY --from=builder /app/apps/api/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```
> For Quarkus native builds, replace with `FROM quay.io/quarkus/quarkus-micro-image:2.0` and adjust the build command.

**If `backend.language: python`**:
```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY apps/api/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY apps/api/src ./src
EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**If `backend.language: go`**:
```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY apps/api/go.mod apps/api/go.sum ./
RUN go mod download
COPY apps/api/ .
RUN go build -o server ./cmd/server

FROM alpine:3.20 AS runner
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

---

## Step 8 — CI Pipeline

Generate `.github/workflows/ci.yml` using the blocks that match the detected ecosystems.

### Universal CI header (always include)
```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
```

### Database service block (if `database.provider` is present)
```yaml
    services:
      # PostgreSQL
      postgres:                          # if database.provider: postgresql
        image: postgres:17-alpine
        env:
          POSTGRES_USER: user
          POSTGRES_PASSWORD: password
          POSTGRES_DB: appdb_test
        ports: ['5432:5432']

      # MySQL
      mysql:                             # if database.provider: mysql
        image: mysql:8-debian
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: appdb_test
        ports: ['3306:3306']

      # MongoDB
      mongodb:                           # if database.provider: mongodb
        image: mongo:7
        ports: ['27017:27017']

      # Message broker
      rabbitmq:                          # if messagebroker.provider: rabbitmq
        image: rabbitmq:3-management-alpine
        ports: ['5672:5672']
```

### Quality job — Node.js / TypeScript ecosystem
```yaml
  quality-node:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4       # swap for actions/setup-node cache: npm if using npm
        with:
          version: latest
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm test
```

### Quality job — Java / Maven ecosystem
```yaml
  quality-java:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: temurin
          cache: maven
      - run: ./mvnw verify
```

### Quality job — Java / Gradle ecosystem
```yaml
  quality-java:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: temurin
          cache: gradle
      - run: ./gradlew check
```

### Quality job — Python ecosystem
```yaml
  quality-python:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r apps/api/requirements.txt
      - run: ruff check apps/api/src
      - run: pytest apps/api/tests
```

### Quality job — Go ecosystem
```yaml
  quality-go:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true
      - run: go vet ./...
      - run: go test ./...
```

### E2E job (if `testing.e2e: playwright`)
```yaml
  e2e:
    runs-on: ubuntu-latest
    needs: [quality-node]      # adjust to the actual quality job name
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter web exec playwright install --with-deps chromium
      - run: pnpm --filter web run test:e2e
        env:
          DATABASE_URL: postgresql://user:password@localhost:5432/appdb_test
```

> Combine only the blocks relevant to your stack into one `.github/workflows/ci.yml` file.

---

## Step 9 — Linting & Formatting

Create linting and formatting config only for the ecosystems present in the stack.

### If any app uses **TypeScript / JavaScript**

#### Root `eslint.config.mjs`
```js
import tseslint from 'typescript-eslint';

export default tseslint.config(
  { ignores: ['**/dist/**', '**/node_modules/**'] },
  ...tseslint.configs.recommended,
  {
    rules: {
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    },
  },
);
```

#### Root `.prettierrc`
```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2
}
```

### If any app uses **Java / JVM**

#### `apps/api/.editorconfig` (universal formatting baseline)
```ini
root = true

[*]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.{yml,yaml,json,xml}]
indent_size = 2
```

> For Maven projects, add the `spotless-maven-plugin` or `checkstyle-maven-plugin` to `pom.xml`.
> For Gradle projects, apply the `com.diffplug.spotless` plugin in `build.gradle.kts`.

### If any app uses **Python**

#### Root `pyproject.toml` linting config
```toml
[tool.ruff]
line-length = 100
select = ["E", "F", "I"]
ignore = []

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

### If any app uses **Go**

No config file needed — use `gofmt` (built-in) and optionally add `golangci-lint`:
```yaml
# .golangci.yml
linters:
  enable:
    - errcheck
    - gosimple
    - govet
    - staticcheck
    - unused
```

---

## Step 10 — Final Checklist

After generating all files, verify:

- [ ] Root workspace config created for the detected ecosystem (pnpm-workspace.yaml | pom.xml | go.work | pyproject.toml)
- [ ] Every app has a build config appropriate to its language
- [ ] TypeScript apps have `tsconfig.json` extending `../../tsconfig.base.json`
- [ ] Java apps have correct Maven/Gradle module references in the parent build file
- [ ] `docker-compose.yml` matches every service in `stack.yml`
- [ ] `.env.example` documents every required environment variable
- [ ] `.github/workflows/ci.yml` includes only the job blocks for detected ecosystems
- [ ] No actual secrets committed — only `.env.example` placeholders
- [ ] `.gitignore` includes base entries + all conditional blocks for the resolved stack
- [ ] Linting config created for each detected language ecosystem
- [ ] `status` in `docs/architecture/stack.yml` remains `confirmed` (do not modify)

---

## Monorepo Final Structure (reference)

```
<repo-root>/
├── apps/
│   ├── api/                # NestJS backend
│   ├── web/                # React frontend
│   └── mobile/             # React Native (if in stack)
├── packages/
│   ├── types/              # @repo/types — shared TS types
│   └── config/             # @repo/config — env helpers
├── docs/
│   └── architecture/
│       ├── stack.yml
│       └── decisions/      # ADRs
├── .github/
│   ├── agents/
│   ├── workflows/
│   │   └── ci.yml
│   └── copilot-instructions.md
├── docker-compose.yml
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.base.json
├── eslint.config.mjs
├── .prettierrc
├── .gitignore
└── .env.example
```
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RuoYi-Vue v3.9.2 — a full-stack admin management system with Spring Boot 4.x backend (Java 17+) and Vue 3 + Element Plus frontend. This is the master branch using Spring Boot 4.x; separate branches exist for Spring Boot 2.x and 3.x.

The frontend (`ruoyi-ui/`) is a git submodule pointing to [RuoYi-Vue3](https://github.com/yangzongzhuan/RuoYi-Vue3), using Vue 3 + Vite + Element Plus + Pinia.

## Build & Run Commands

### Backend (Maven multi-module)
```bash
# First time: build all modules and install to local repo
mvn clean install -DskipTests

# Run in dev mode (from project root)
mvn spring-boot:run -pl ruoyi-admin

# Production: build jar and run
mvn clean package -DskipTests
java -jar ruoyi-admin/target/ruoyi-admin.jar

# Or use the shell script
./ry.sh start|stop|restart|status
```

### Frontend (ruoyi-ui/ — git submodule, Vue 3 + Vite)
```bash
cd ruoyi-ui
npm install
npm run dev     # Vite dev server on port 1024, proxies /dev-api → localhost:8080
npm run build   # Production build → dist/
```

### Dev Environment (Docker)

MySQL and Redis are provided via Docker Compose in `dockerEnvironment/`:
```bash
cd dockerEnvironment
docker compose up -d mysql redis   # Start infrastructure services only
docker compose up -d               # Start all services (full stack deployment)
docker compose down                # Stop all services
```

MySQL auto-initializes from `sql/` directory on first startup.

If Chinese characters appear garbled (乱码), the MySQL data was initialized with wrong encoding. Fix by reinitializing:
```bash
cd dockerEnvironment
docker compose stop mysql
sudo rm -rf mysql/data
docker compose up -d mysql
```

### Dependencies
- **MySQL** on localhost:3306, database `ry-vue` (config in `application-druid.yml`)
- **Redis** on localhost:6379 (config in `application.yml`)
- SQL init scripts in `sql/` directory

## Architecture

### Module Dependency Graph
```
ruoyi-admin (entry point, controllers)
 ├── ruoyi-framework (Spring Security, JWT, MyBatis config, Druid, filters)
 │    └── ruoyi-system (domain entities, mappers, services for system tables)
 │         └── ruoyi-common (utilities, annotations, base classes, core types)
 ├── ruoyi-quartz (scheduled tasks, depends on ruoyi-system)
 └── ruoyi-generator (code generation with Velocity templates, depends on ruoyi-system)
```

### Key Module Responsibilities

- **ruoyi-admin**: Application entry (`RuoYiApplication.java`). All REST controllers live under `web/controller/` grouped by `system/`, `monitor/`, `tool/`, `common/`.
- **ruoyi-framework**: Security config (`SecurityConfig.java`), JWT filter, CORS, data source routing, MyBatis config, Redis cache manager, custom interceptors.
- **ruoyi-system**: Business domain — entities (`domain/`), MyBatis mappers (`mapper/`), service layer (`service/`). Covers users, roles, menus, depts, dicts, logs, config.
- **ruoyi-common**: Shared utilities and base classes. `BaseController` (pagination via `startPage()`), `BaseEntity` (audit fields), `AjaxResult` (unified response), `TableDataInfo` (paged response).
- **ruoyi-quartz**: Quartz-based job scheduling. Jobs extend `AbstractQuartzJob`, registered via `sys_job` table.
- **ruoyi-generator**: Auto-generates CRUD code (Java controller/service/mapper, Vue pages, SQL) from database tables using Velocity templates in `resources/vm/`. Config in `generator.yml`.
- **ruoyi-ui**: Vue 3 + Vite + Element Plus + Pinia SPA. Git submodule from RuoYi-Vue3. Router in `src/router/`, Pinia store in `src/store/`, API calls in `src/api/`, views in `src/views/`.

### Backend Request Flow
```
HTTP Request → Spring Security filter chain (JWT auth)
  → Controller (@PreAuthorize permission check)
    → Service (business logic, @DataScope for row-level perms)
      → MyBatis Mapper (SQL via XML in resources/mapper/)
```

### Custom Annotations
- `@Log` — Operation audit logging (records to sys_oper_log)
- `@DataScope` — Row-level data permission filtering on SQL queries
- `@Excel` / `@Excels` — Field-level Excel import/export configuration
- `@DataSource` — Switch between master/slave data sources
- `@RepeatSubmit` — Duplicate submission prevention
- `@RateLimiter` — Rate limiting via Redis
- `@Sensitive` — Data masking on response fields

### Frontend-Backend Integration
- Frontend dev server proxies `/dev-api` to backend at `localhost:8080`
- Auth via JWT token in `Authorization` header
- API responses use `AjaxResult` format: `{ code: 200, msg: "...", data: ... }`
- Paged responses: `{ total: N, rows: [...], code: 200, msg: "..." }`

## Code Conventions

- **Package structure**: `com.ruoyi.{module}.{layer}` — controller/service/mapper/domain per module
- **MyBatis XML mappers**: Located in each module's `resources/mapper/` directory, following `*Mapper.xml` naming
- **Controller response**: Return `AjaxResult` for single operations, `TableDataInfo` for paginated lists
- **Entity naming**: Domain classes in `domain/`, corresponding MyBatis XML in `resources/mapper/`
- **Frontend API modules**: One JS file per backend controller in `src/api/`, using axios wrapper from `src/utils/request.js`

## Configuration Files

- `application.yml` — Main config (server port, Redis, MyBatis, token settings)
- `application-druid.yml` — Database connection (MySQL + Druid pool). JDBC URL must include `useSSL=false&allowPublicKeyRetrieval=true` for MySQL 8.0 compatibility.
- `generator.yml` — Code generator settings
- `mybatis/mybatis-config.xml` — MyBatis global settings
- `ruoyi-ui/vite.config.js` — Frontend Vite build and dev server proxy config (port 1024)
- `ruoyi-admin/src/main/resources/logback.xml` — Log output path (currently `./logs`)
- `dockerEnvironment/docker-compose.yml` — Docker services (MySQL, Redis, backend, frontend nginx)
- `dockerEnvironment/mysql/conf/my.cnf` — MySQL character set config (utf8mb4)
- `dockerEnvironment/nginx/default.conf` — Frontend nginx config with API reverse proxy

## Local Config Changes

The following files have been modified from the original repo for local development:

- `ruoyi-admin/src/main/resources/logback.xml` — Log path changed from `/home/ruoyi/logs` to `./logs` (avoids creating system directories)
- `ruoyi-admin/src/main/resources/application-druid.yml` — JDBC URL updated with `useSSL=false&allowPublicKeyRetrieval=true` for Docker MySQL 8.0
- `ruoyi-ui/` — Replaced original Vue 2 frontend with RuoYi-Vue3 (Vue 3) as git submodule. `vite.config.js` port changed from 80 to 1024.

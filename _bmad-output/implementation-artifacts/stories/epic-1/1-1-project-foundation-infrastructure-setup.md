# Story 1.1: Project Foundation & Infrastructure Setup

**Story ID:** 1.1
**Epic:** Epic 1 - Foundation & Authentification
**Status:** ready-for-dev
**Story Key:** `1-1-project-foundation-infrastructure-setup`

---

## 📋 User Story

**As a** developer
**I want** the foundational project structure with all core infrastructure components configured
**So that** I have a solid base to implement features following DDD architecture and ADR-007 (from scratch approach)

---

## ✅ Acceptance Criteria

### Given I am setting up a new Pensine project from scratch

**When** I initialize the mobile and backend projects
**Then** the following structure is created:

#### Mobile (React Native + Expo):
- ✅ Expo custom dev client initialized with TypeScript strict mode
- ✅ WatermelonDB configured for offline-first storage
- ✅ Project structure follows DDD bounded contexts layout
- ✅ Basic navigation structure (React Navigation)

#### Backend (NestJS):
- ✅ NestJS project initialized with TypeScript strict
- ✅ PostgreSQL database connection configured
- ✅ RabbitMQ message broker connected
- ✅ Redis cache configured (cache only, not queue)
- ✅ DDD folder structure with bounded contexts (Identity, Capture, Knowledge, Opportunity, Action, Normalization, Notification, Sync infrastructure)

**And** Docker Compose file exists for local development (PostgreSQL + RabbitMQ + Redis)
**And** Environment configuration files are set up (.env templates)
**And** README with setup instructions is created
**And** No starter/boilerplate code is used (ADR-007 compliance)

---

## 🎯 Developer Context & Implementation Guide

### 🔥 CRITICAL: ADR-007 Compliance

**From Scratch Approach** - **NO BOILERPLATE ALLOWED**

**❌ DO NOT USE:**
- NestJS Clean Architecture Boilerplate
- React Native starter templates
- Full-stack starters (Expo + NestJS combined)
- Any DDD boilerplate

**✅ START FROM:**
- **Mobile:** `npx create-expo-app@latest pensine-mobile --template blank-typescript`
- **Backend:** `npx @nestjs/cli new pensine-backend`

**Rationale (from ADR-007):**
1. Architecture DDD spécifique déjà définie (8 Bounded Contexts via Event Storming)
2. Stack atypique:
   - WatermelonDB (rare dans starters Expo)
   - Whisper on-device (~500 Mo modèle)
   - RabbitMQ (rare dans starters NestJS)
3. Structure personnalisée adaptée à Pensine (Event Sourcing prévu, Sync WatermelonDB)

---

## 🏗️ Architecture Requirements

### Domain-Driven Design (DDD) - 8 Bounded Contexts

#### 🔥 Core Domain (Différenciateurs Business)

**1. Knowledge Context**
- **Responsabilité:** Digestion IA et extraction de sens
- **Ubiquitous Language:** Thought, Digestion, Summary, Tags, Extraction
- **Aggregates:** `Thought`
- **Valeur métier:** Transformer le flux de pensée brut en insights structurés

**2. Opportunity Context**
- **Responsabilité:** Détection patterns, concordance, germination d'opportunités business
- **Ubiquitous Language:** Idea, Project, Concordance, Pattern, Germination, Maturity, BusinessCase
- **Aggregates:** `Idea`, `Project`
- **Valeur métier:** Incubateur personnel d'idées business (le cœur de Pensine)

#### 🔧 Supporting Domain (Nécessaires mais non différenciateurs)

**3. Capture Context**
- **Responsabilité:** Capture multi-modale (audio, texte, image, URL)
- **Ubiquitous Language:** Capture, Recording, Snapshot
- **Aggregates:** `Capture`

**4. Normalization Context**
- **Responsabilité:** Normalisation des captures en texte exploitable
- **Ubiquitous Language:** Transcription, OCR, TextExtraction, WebScraping
- **Aggregates:** Aucun (Domain Services stateless)

**5. Action Context**
- **Responsabilité:** Gestion cycle de vie des actions (GTD)
- **Ubiquitous Language:** Todo, Task, Action, Deadline, Priority, Completion
- **Aggregates:** `Todo`

#### ⚙️ Generic Subdomain (Génériques)

**6. Identity Context**
- **Responsabilité:** Authentification, gestion utilisateurs
- **Aggregates:** `User`

**7. Notification Context**
- **Responsabilité:** Notifications push et alertes
- **Aggregates:** `Notification`

#### 🏗️ Infrastructure (Non-métier)

**8. Sync (Infrastructure)**
- **Responsabilité:** Synchronisation offline-first
- **Implémentation:** WatermelonDB + backend sync endpoint
- **Note:** Pas de Bounded Context métier, infrastructure pure

---

## 📦 Technology Stack

### Mobile (React Native + Expo)

**Framework:** React Native avec Expo (custom dev client)
- **Rationale:** Cross-platform iOS/Android, écosystème mature, connaissance utilisateur
- **Version:** Latest stable (check Expo SDK compatibility)

**Language:** TypeScript (strict mode)
- `"strict": true` in tsconfig.json
- `"strictNullChecks": true`
- `"noImplicitAny": true`

**Local Database:** WatermelonDB
- **Rationale:** Offline-first, performance, observabilité React, sync protocol
- **Installation:** `npm install @nozbe/watermelondb`
- **Adapter:** SQLite (via `@nozbe/watermelondb/adapters/sqlite`)

**Navigation:** React Navigation (v6+)
- Bottom tabs for main navigation (Feed, Actions tabs)
- Stack navigation for detail views

**Key Dependencies to Install:**
```json
{
  "@react-navigation/native": "^6.x",
  "@react-navigation/bottom-tabs": "^6.x",
  "@react-navigation/native-stack": "^6.x",
  "@nozbe/watermelondb": "latest",
  "expo": "latest SDK",
  "react-native-reanimated": "latest",
  "react-native-gesture-handler": "latest"
}
```

---

### Backend (NestJS)

**Framework:** NestJS (TypeScript)
- **Rationale:** Architecture modulaire, connaissance utilisateur, adapté DDD
- **Version:** Latest stable (v10+)
- **Alternative rejetée:** Fastify (moins d'expérience, complexité DDD à gérer)

**Database:** PostgreSQL
- **Rationale:** Robustesse transactionnelle, JSON support, connaissance équipe
- **Version:** PostgreSQL 15+ (latest stable)

**Message Broker:** RabbitMQ
- **Rationale:**
  - Élimination Redis SPOF (isolation pannes entre queue et cache)
  - Durabilité disk-based > Redis memory-only
  - Dead-letter queues pour retry logic
- **Alternative rejetée:** BullMQ (dépendance Redis = SPOF)

**Cache:** Redis (cache UNIQUEMENT, pas de queue)
- **Usage:** Token blacklist, session cache
- **NOT for:** Job queues (use RabbitMQ)

**Scheduler:** NestJS Schedule (cron jobs)
- **Rationale:** Publie vers RabbitMQ pour traitement asynchrone

**Key Dependencies to Install:**
```json
{
  "@nestjs/core": "^10.x",
  "@nestjs/common": "^10.x",
  "@nestjs/typeorm": "^10.x",
  "@nestjs/microservices": "^10.x",
  "typeorm": "^0.3.x",
  "pg": "^8.x",
  "amqplib": "^0.10.x",
  "ioredis": "^5.x",
  "@nestjs/schedule": "^4.x",
  "bcrypt": "^5.x",
  "jsonwebtoken": "^9.x"
}
```

---

### Infrastructure (Docker Compose)

**Services to Configure:**

```yaml
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: pensine
      POSTGRES_USER: pensine_user
      POSTGRES_PASSWORD: pensine_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  rabbitmq:
    image: rabbitmq:3-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: pensine_user
      RABBITMQ_DEFAULT_PASS: pensine_password
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

volumes:
  postgres_data:
  rabbitmq_data:
  redis_data:
```

---

## 📁 Required Project Structure

### Mobile Project Structure (pensine-mobile/)

```
pensine-mobile/
├── app/                          # Expo Router or React Navigation setup
├── src/
│   ├── contexts/                 # DDD Bounded Contexts
│   │   ├── capture/             # Capture Context
│   │   │   ├── domain/
│   │   │   │   ├── models/      # WatermelonDB models
│   │   │   │   └── services/
│   │   │   ├── application/     # Use cases
│   │   │   └── presentation/    # React components
│   │   ├── knowledge/           # Knowledge Context (Thought, Digestion)
│   │   ├── opportunity/         # Opportunity Context (Idea, Project)
│   │   ├── action/              # Action Context (Todo)
│   │   ├── identity/            # Identity Context (User, Auth)
│   │   ├── notification/        # Notification Context
│   │   └── sync/                # Sync Infrastructure
│   ├── database/
│   │   ├── schema.ts            # WatermelonDB schema
│   │   ├── models/              # All WatermelonDB models
│   │   └── migrations/
│   ├── navigation/              # React Navigation config
│   ├── shared/                  # Shared utilities, types
│   └── config/                  # App configuration
├── assets/
├── app.json
├── tsconfig.json
├── package.json
└── README.md
```

**Key Notes:**
- Each Bounded Context has: `domain/`, `application/`, `presentation/`
- WatermelonDB models live in both `database/models/` (registration) and `contexts/*/domain/models/` (usage)
- TypeScript strict mode enforced

---

### Backend Project Structure (pensine-backend/)

```
pensine-backend/
├── src/
│   ├── contexts/                        # DDD Bounded Contexts
│   │   ├── identity/                    # Identity Context (User, Auth)
│   │   │   ├── domain/
│   │   │   │   ├── entities/           # User entity
│   │   │   │   ├── value-objects/
│   │   │   │   └── events/             # Domain Events
│   │   │   ├── application/
│   │   │   │   ├── commands/           # CQRS Commands
│   │   │   │   ├── queries/            # CQRS Queries
│   │   │   │   └── services/           # Application Services
│   │   │   ├── infrastructure/
│   │   │   │   ├── persistence/        # TypeORM repositories
│   │   │   │   └── messaging/          # RabbitMQ consumers/publishers
│   │   │   └── presentation/
│   │   │       └── controllers/        # REST controllers
│   │   ├── capture/                     # Capture Context
│   │   ├── knowledge/                   # Knowledge Context (Thought)
│   │   ├── opportunity/                 # Opportunity Context (Idea, Project)
│   │   ├── action/                      # Action Context (Todo)
│   │   ├── normalization/               # Normalization Context (Transcription)
│   │   └── notification/                # Notification Context
│   ├── infrastructure/
│   │   ├── sync/                        # Sync endpoint (WatermelonDB protocol)
│   │   ├── database/                    # TypeORM config
│   │   ├── messaging/                   # RabbitMQ setup
│   │   └── cache/                       # Redis setup
│   ├── shared/                          # Shared kernel (types, utils, base classes)
│   └── main.ts                          # NestJS bootstrap
├── docker-compose.yml
├── .env.example
├── .env
├── tsconfig.json
├── package.json
└── README.md
```

**Key Notes:**
- Each Bounded Context has: `domain/`, `application/`, `infrastructure/`, `presentation/`
- Domain Events for cross-context communication (asynchronous via RabbitMQ)
- TypeORM for PostgreSQL persistence
- Shared kernel for cross-cutting concerns

---

## 🔒 Security & Configuration

### Environment Variables (.env templates)

**Backend (.env.example):**
```env
# Database
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=pensine
DATABASE_USER=pensine_user
DATABASE_PASSWORD=pensine_password

# RabbitMQ
RABBITMQ_URL=amqp://pensine_user:pensine_password@localhost:5672
RABBITMQ_QUEUE_DIGESTION=digestion-jobs
RABBITMQ_QUEUE_DIGESTION_DLQ=digestion-failed

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# JWT
JWT_SECRET=your_secret_key_here_change_in_production
JWT_EXPIRATION=7d

# Server
PORT=3000
NODE_ENV=development
```

**Mobile (.env.example):**
```env
# API
API_BASE_URL=http://localhost:3000

# Environment
NODE_ENV=development
```

---

## 🧪 Testing Requirements

### Backend Testing Setup

**Install Testing Dependencies:**
```json
{
  "jest": "^29.x",
  "@nestjs/testing": "^10.x",
  "@types/jest": "^29.x",
  "ts-jest": "^29.x",
  "supertest": "^6.x"
}
```

**Test Structure:**
- Unit tests for domain logic
- Integration tests for repositories and infrastructure
- E2E tests for API endpoints

### Mobile Testing Setup

**Install Testing Dependencies:**
```json
{
  "jest": "^29.x",
  "@testing-library/react-native": "^12.x",
  "@testing-library/jest-native": "^5.x"
}
```

---

## 📖 README Requirements

**Both repositories must include:**

1. **Project Description:** What is Pensine?
2. **Prerequisites:** Node.js version, dependencies
3. **Installation Steps:**
   - Clone repository
   - Install dependencies
   - Setup environment variables
   - Run Docker Compose (backend only)
   - Database migrations
4. **Running the Project:**
   - Development mode
   - Production build
   - Running tests
5. **Project Structure:** Explain DDD layout
6. **Architecture Overview:** Link to `project-pensieve` documentation repo
7. **Contributing Guidelines:** Code style, commit conventions

---

## ⚠️ Common Pitfalls to Avoid

### ❌ DO NOT:
1. **Use any boilerplate or starter template** (ADR-007 violation)
2. **Mix queue and cache in Redis** (RabbitMQ for queues, Redis for cache only)
3. **Create ACL between backend Bounded Contexts** (ADR-008: ACL only at mobile ↔ backend boundary)
4. **Skip TypeScript strict mode** (mandatory for both projects)
5. **Forget Docker Compose** (required for local dev: PostgreSQL + RabbitMQ + Redis)
6. **Use SQLite encryption at this stage** (MVP uses device-level encryption, TDE is post-MVP)

### ✅ DO:
1. **Start from official CLIs** (create-expo-app, @nestjs/cli)
2. **Follow DDD structure strictly** (8 Bounded Contexts as defined)
3. **Setup all 3 infrastructure services** (PostgreSQL, RabbitMQ, Redis)
4. **Configure WatermelonDB from scratch** (no boilerplate, follow official docs)
5. **Create .env.example templates** (never commit actual secrets)
6. **Write comprehensive READMEs** (setup instructions are critical)

---

## 🔗 References

### Official Documentation
- [Expo Documentation](https://docs.expo.dev/)
- [NestJS Documentation](https://docs.nestjs.com/)
- [WatermelonDB Documentation](https://nozbe.github.io/WatermelonDB/)
- [React Navigation](https://reactnavigation.org/)
- [TypeORM](https://typeorm.io/)
- [RabbitMQ](https://www.rabbitmq.com/documentation.html)

### Architecture Documents
- **Main Architecture:** `project-pensieve/_bmad-output/planning-artifacts/architecture.md`
- **ADR-007:** From Scratch Approach (no boilerplate)
- **ADR-008:** ACL only at mobile ↔ backend boundary
- **ADR-003:** Sync as Infrastructure (not Bounded Context)

### Epic Context
- **Epic 1:** Foundation & Authentification
- **Next Stories:** 1.2 User Registration, 1.3 User Login, 1.4 User Logout, 1.5 Password Recovery

---

## ✅ Definition of Done

**This story is DONE when:**

1. ✅ **Mobile repository initialized:**
   - Expo + TypeScript strict
   - WatermelonDB configured
   - DDD folder structure created
   - React Navigation setup
   - .env.example created
   - README with setup instructions

2. ✅ **Backend repository initialized:**
   - NestJS + TypeScript strict
   - DDD folder structure (8 Bounded Contexts)
   - PostgreSQL connection configured
   - RabbitMQ connection configured
   - Redis connection configured
   - .env.example created
   - README with setup instructions

3. ✅ **Docker Compose configured:**
   - PostgreSQL service
   - RabbitMQ service (with management UI)
   - Redis service
   - All services start without errors

4. ✅ **Code pushed to GitHub:**
   - Mobile code in: `git@github.com:yohikofox/pensieve.git` (mobile/)
   - Backend code in: `git@github.com:yohikofox/pensieve.git` (backend/)

5. ✅ **Verification:**
   - `npm install` succeeds on both projects
   - `docker-compose up` starts all services
   - Backend connects to all infrastructure services
   - Mobile app builds and runs on iOS/Android simulator
   - TypeScript compilation succeeds with no errors

---

## 📝 Developer Notes

_This section will be filled by the developer during implementation._

**Implementation Log:**
- [ ] Mobile repository created
- [ ] Backend repository created
- [ ] Docker Compose configured
- [ ] All dependencies installed
- [ ] Services verified running

**Challenges Encountered:**
_TBD_

**Learnings for Next Stories:**
_TBD_

---

**Story Created:** 2026-01-19
**Created by:** Scrum Master (BMAD Methodology)
**Context Engine Analysis:** ✅ Complete
**Ready for Development:** ✅ Yes

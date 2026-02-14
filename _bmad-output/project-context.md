---
project_name: 'pensine'
user_name: 'yohikofox'
date: '2026-02-13'
sections_completed: ['technology_stack', 'language_specific_rules', 'framework_specific_rules', 'testing_rules', 'code_quality_style_rules', 'development_workflow_rules', 'critical_dont_miss_rules']
existing_patterns_found: 15
status: 'complete'
---

# Project Context for AI Agents

_This file contains critical rules and patterns that AI agents must follow when implementing code in this project. Focus on unobvious details that agents might otherwise miss._

---

## Technology Stack & Versions

**Core Technologies:**

**Mobile** (`pensieve/mobile/`)
- Node.js **22** (`.nvmrc`)
- Expo SDK **54.0.31**
- React Native **0.81.5**
- React **19.1.0**
- TypeScript **5.9.2** (strict mode)

**Backend** (`pensieve/backend/`)
- Node.js **22**
- NestJS **11.0.1**
- TypeScript **5.7.3** (strict mode)
- TypeORM **0.3.28**

**Web/Admin** (`pensieve/web/`, `pensieve/admin/`)
- Next.js **15**
- React **19**
- TypeScript **5**

**Key Dependencies:**

**Mobile - Infrastructure:**
- WatermelonDB (offline-first DB)
- @op-engineering/op-sqlite **15.2.3**
- @supabase/supabase-js **2.90.1**
- @tanstack/react-query **5.90.20**
- zustand **5.0.10**
- tsyringe **4.10.0** (DI container)

**Mobile - AI/ML:**
- whisper.rn **0.5.4** (speech-to-text)
- llama.rn **0.10.1** (LLM on-device)
- expo-llm-mediapipe **0.6.0**

**Mobile - Tests:**
- jest **29.7.0** + babel-jest (NOT jest-expo)
- jest-cucumber **4.5.0** (BDD)
- detox **20.28.3** (E2E)

**Backend - Infrastructure:**
- PostgreSQL via pg **8.17.1**
- RabbitMQ via amqplib **0.10.9**
- MinIO **8.0.6** (S3-compatible storage)
- Redis **5.10.0**

**Backend - AI:**
- OpenAI **6.17.0** (GPT-4o-mini)
- tiktoken **1.0.22** (token counting)

**Backend - Tests:**
- jest **30.0.0**
- jest-cucumber **4.5.0** (BDD)

**⚠️ Version Constraints:**

- **Expo SDK 54 Winter runtime incompatibility** → Jest MUST use **babel-jest**, NOT `jest-expo`
- **TypeScript strict mode** everywhere (non-negotiable)
- **Node.js 22** locked via `.nvmrc`
- **Decorators required** (mobile): `experimentalDecorators: true`, `emitDecoratorMetadata: true`

---

## Critical Implementation Rules

### Language-Specific Rules (TypeScript)

**Configuration Requirements:**

**Mobile** (`mobile/tsconfig.json`):
- `strict: true` - OBLIGATOIRE
- `experimentalDecorators: true` - REQUIS pour tsyringe DI
- `emitDecoratorMetadata: true` - REQUIS pour tsyringe DI

**Backend** (`backend/tsconfig.json`):
- TypeScript 5.7.3 strict mode
- CommonJS module style (NestJS convention)
- Path aliases via `tsconfig-paths`

**Dependency Injection Critical Pattern (Mobile):**

```typescript
// ❌ WRONG: Module-level resolution fails before bootstrap
import { container } from '@/infrastructure/di/container';
const logger = container.resolve<ILogger>(TOKENS.ILogger);

// ✅ CORRECT: Lazy resolution inside hooks/effects
const useLogger = () => {
  const getLogger = () => container.resolve<ILogger>(TOKENS.ILogger);
  return getLogger();
};
```

**Backend DI Pattern:**
```typescript
// Use string tokens for swappable implementations
@Inject('IAuthorizationService')
private readonly authService: IAuthorizationService
```

**TypeScript Anti-Patterns - NEVER:**
- ❌ Use `any` - use `unknown` or proper types
- ❌ Disable strict mode
- ❌ Resolve DI container at module level (mobile)
- ❌ Use `synchronize: true` in TypeORM (backend)

---

### Framework-Specific Rules

**React/React Native (Mobile):**

**DI Bootstrap Sequence (CRITICAL):**
```
index.ts → bootstrap() → App.tsx → MainApp.tsx
```
- DI container MUST be registered BEFORE React renders
- All services registered during bootstrap, NOT at import time

**Screen Pattern - Wrapper + Content:**
```typescript
ScreenWrapper (small) → extracts route params only
  └─ ScreenContent (large) → all business logic, testable without navigation
```

**Screen Registry Pattern:**
- Centralized navigation config in `src/screens/registry.ts`
- Each screen declares: icon, i18n keys, nav options in ONE place

**State Management:**
- **Zustand** (`src/stores/`) - client state, event-driven via EventBus (NO polling)
- **React Query** - server state (API data)
- **OP-SQLite** (`src/database/`) - local persistence, synchronous queries

**Hooks Usage:**
- Custom hooks in `contexts/[context]/hooks/`
- DI resolution MUST be lazy (inside hooks, not module-level)

**NestJS (Backend):**

**Hybrid App Architecture:**
```typescript
// main.ts MUST call both:
app.connectMicroservice(/* RabbitMQ config */);
await app.startAllMicroservices();
```

**DDD Module Structure:**
```
modules/[context]/
├── domain/entities/
├── domain/events/
├── application/services/
├── application/repositories/
├── application/controllers/
├── infrastructure/
```

**Database Rules (CRITICAL):**
- `synchronize: false` - schema changes via migrations ONLY in `src/migrations/`
- NEVER use `synchronize: true`

**Authorization Pattern:**
- Guards: `@RequirePermission()`, `@RequireOwnership()`, `@AllowSharedAccess()`
- Multi-level resolution: user override → resource share → tier → role

**RabbitMQ Config:**
- Durable queues with dead-letter exchange
- Consumer prefetch = 3, timeout = 60s
- Priority support (max 10)

**Next.js (Web/Admin):**
- App Router (Next.js 15)
- Tailwind CSS
- Standalone Docker deployment

---

### Testing Rules

**Mobile Testing Configuration (CRITICAL):**

**Jest Dual Configuration:**
```typescript
// jest.config.js (Unit Tests)
// ⚠️ CRITICAL: Uses babel-jest, NOT jest-expo
// Why? Expo SDK 54 Winter runtime incompatible with Node.js test environment
testEnvironment: 'node'
transform: { '^.+\\.(ts|tsx)$': 'babel-jest' }
testMatch: ['**/__tests__/**/*.test.ts', '**/tests/acceptance/**/*.test.ts']

// jest.config.acceptance.js (BDD/Gherkin Tests)
preset: 'ts-jest'  // Different from unit tests!
testMatch: ['<rootDir>/tests/acceptance/**/*.test.ts']
moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1' }
```

**BDD/Gherkin Test Structure (Mobile & Backend):**

```
mobile/tests/acceptance/
├── features/                # 18 Gherkin .feature files
│   └── story-*.feature
├── support/
│   └── test-context.ts      # 12 in-memory mocks (MockAudioRecorder, etc.)
└── story-*.test.ts          # Step definitions (jest-cucumber)

backend/test/acceptance/
├── features/                # Gherkin .feature files
└── story-*.test.ts          # Step definitions
```

**Mock Pattern (Mobile):**
```typescript
// tests/acceptance/support/test-context.ts
export class MockAudioRecorder {
  async startRecording(): Promise<RepositoryResult<{ uri: string }>> {
    // In-memory mock, no real file system
    return success({ uri: `mock://audio_${Date.now()}.m4a` });
  }
}

// Mock WatermelonDB decorators as no-ops
moduleNameMapper: {
  '^@nozbe/watermelondb/decorators$':
    '<rootDir>/__mocks__/@nozbe/watermelondb/decorators.js'
}
```

**Backend Testing Configuration:**

```json
// test/jest-acceptance.json
{
  "preset": "ts-jest",
  "testMatch": ["**/test/acceptance/**/*.test.ts"],
  "moduleNameMapper": {
    "^@/(.*)$": "<rootDir>/src/$1"
  }
}

// test/jest-e2e.json
{
  "preset": "ts-jest",
  "testEnvironment": "node",
  "testMatch": ["**/*.e2e-spec.ts"]
}
```

**Test Execution Patterns:**

```bash
# Mobile
npm run test:unit                # babel-jest, src/**/*.test.ts
npm run test:acceptance          # ts-jest, tests/acceptance/**/*.test.ts
npm run test:e2e                 # Detox (iOS)

# Backend
npm run test                     # Unit tests, src/**/*.spec.ts
npm run test:acceptance          # jest-cucumber, test/acceptance/**/*.test.ts
npm run test:e2e                 # E2E, test/**/*.e2e-spec.ts
```

**Single Test File Execution:**
```bash
# Mobile
npx jest src/path/to/file.test.ts
npx jest --config jest.config.acceptance.js tests/acceptance/story-2-1.test.ts

# Backend
npx jest src/path/to/file.spec.ts
npx jest --config ./test/jest-acceptance.json test/acceptance/story-4-1.test.ts
```

**Critical Testing Anti-Patterns - NEVER:**
- ❌ Use `jest-expo` preset (mobile) - breaks with Expo SDK 54 Winter runtime
- ❌ Mix babel-jest and ts-jest in same test type - keep separation strict
- ❌ Import mocks at module level - use lazy resolution like DI pattern
- ❌ Skip BDD tests for new stories - Gherkin coverage required

**Coverage Requirements:**
- Unit tests: `src/**/*.spec.ts` (backend), `src/**/*.test.ts` (mobile)
- BDD tests: Mandatory for each story (Gherkin .feature + step definitions)
- E2E tests: Critical user journeys only

---

### Code Quality & Style Rules

**ESLint Configuration:**

**Backend (ESLint 9 Flat Config):**
```javascript
// eslint.config.mjs - CRITICAL: Flat config format (ESLint 9)
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import eslintPluginPrettierRecommended from 'eslint-plugin-prettier/recommended';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  eslintPluginPrettierRecommended,
  {
    languageOptions: {
      sourceType: 'module',  // ESM (tsconfig uses "nodenext")
      parserOptions: {
        projectService: true,
      }
    },
    rules: {
      '@typescript-eslint/no-explicit-any': 'off',  // Allowed in backend
      '@typescript-eslint/no-floating-promises': 'warn',
      '@typescript-eslint/no-unsafe-argument': 'warn',
      'prettier/prettier': ['error', { endOfLine: 'auto' }]
    }
  }
);
```

**Prettier Configuration:**
```json
// .prettierrc (Backend & Mobile)
{
  "singleQuote": true,
  "trailingComma": "all"
}
```

**Commit Convention - Conventional Commits (MANDATORY):**

```bash
# Format: <type>(<scope>): <subject>

# Valid types:
feat:      # New feature
fix:       # Bug fix
refactor:  # Code change that neither fixes a bug nor adds a feature
docs:      # Documentation only changes
test:      # Adding or correcting tests
chore:     # Build process or auxiliary tool changes
perf:      # Performance improvement
style:     # Code style changes (formatting, missing semi-colons, etc.)

# Examples from project history:
git commit -m "feat: Add support mode with backend permissions for debug access"
git commit -m "docs(story-5.4): complete code review and mark story as done"
git commit -m "fix(audio): prevent trim from truncating transcriptions"
git commit -m "refactor(authorization): extract multi-level permission resolver"
```

**TypeScript Strict Mode Rules:**

```typescript
// tsconfig.json (Mobile & Backend)
{
  "strict": true,  // MANDATORY - NEVER disable
  "noImplicitAny": true,
  "strictNullChecks": true,
  "strictFunctionTypes": true,
  "strictBindCallApply": true,
  "strictPropertyInitialization": true,
  "noImplicitThis": true,
  "alwaysStrict": true
}
```

**Module System Conventions:**

- **Backend**: ESM (`import/export`) - NestJS 11 with TypeScript nodenext
- **Mobile**: ESM (`import/export`) - React Native convention
- **Web**: ESM (`import/export`) - Next.js 15 App Router
- **All packages use modern ESM** - No CommonJS in this project

**Code Organization Patterns:**

```
# Backend - DDD Layered Structure
modules/[context]/
├── domain/entities/        # Pure domain models (no dependencies)
├── domain/events/          # Domain events
├── application/services/   # Business logic
├── application/repositories/  # Repository interfaces
├── application/controllers/   # HTTP/RabbitMQ controllers
└── infrastructure/            # External adapters

# Mobile - Hexagonal Architecture
contexts/[context]/
├── domain/                 # Pure interfaces, models (no dependencies)
├── data/                   # Repository implementations
├── services/               # Application services
├── hooks/                  # React hooks
└── ui/                     # UI components
```

**Naming Conventions:**

- **Files**: kebab-case (`user-service.ts`, `audio-recorder.hook.ts`)
- **Classes**: PascalCase (`UserService`, `AudioRecorder`)
- **Interfaces**: PascalCase with `I` prefix (`ILogger`, `IAuthService`)
- **Functions**: camelCase (`getUserById`, `startRecording`)
- **Constants**: UPPER_SNAKE_CASE (`MAX_RETRIES`, `DEFAULT_TIMEOUT`)
- **DI Tokens**: String literals for swappable implementations (`'IAuthorizationService'`)

**Critical Code Quality Anti-Patterns - NEVER:**
- ❌ Disable TypeScript strict mode
- ❌ Use `any` type - prefer `unknown` or proper types
- ❌ Mix module systems (CommonJS/ESM) within same package
- ❌ Skip linting (`// eslint-disable` without justification)
- ❌ Commit without conventional commit format
- ❌ Use `console.log` in production code - use proper logger

---

### Development Workflow Rules

**Infrastructure Setup:**

```bash
# 1. Start infrastructure services (PostgreSQL, RabbitMQ, MinIO, Redis)
cd pensieve/infrastructure
docker-compose up -d

# 2. Verify services health
docker-compose ps

# Services:
# - PostgreSQL:      localhost:5432
# - RabbitMQ:        localhost:5672 (AMQP), localhost:15672 (UI)
# - MinIO:           localhost:9000 (API), localhost:9001 (Console)
# - Redis:           localhost:6379
```

**Database Migrations - TypeORM (CRITICAL):**

```bash
# Backend migrations workflow
cd pensieve/backend

# ⚠️ CRITICAL: synchronize: false in TypeORM config
# ALL schema changes MUST go through migrations in src/migrations/

# Run pending migrations
npm run migration:run

# Seed authorization data (roles, permissions, tiers)
npm run seed:authorization

# Migrate existing users (if needed)
npm run migrate:users

# Generate new migration (after entity changes)
npx typeorm migration:generate src/migrations/MigrationName -d src/data-source.ts

# Create empty migration
npx typeorm migration:create src/migrations/MigrationName
```

**6 Existing Migrations:**
1. `1738796443000-CreateThoughtAndIdeaTables.ts`
2. `1738869600000-CreateTodosTable.ts`
3. `1738869700000-CreateNotificationsTable.ts`
4. `1738869800000-AddPushTokenAndNotificationPreferencesToUsers.ts`
5. `1739450000000-CreateAuthorizationTables.ts` (11 tables)
6. `1739490000000-CreateAdminUsersTable.ts`

**Development Server Workflow:**

```bash
# Backend (NestJS)
cd pensieve/backend
npm run start:dev    # Watch mode, port 3000

# Mobile (Expo)
cd pensieve/mobile
npm run start        # Expo dev client
npm run ios          # iOS simulator (dev variant)
npm run android      # Android emulator (dev variant)

# Web (Next.js)
cd pensieve/web
npm run dev          # Dev server

# Admin (Next.js)
cd pensieve/admin
npm run dev          # Dev server
```

**Docker Build & Deployment (Makefile):**

```bash
# From pensieve/ root directory

# Build all images (backend, web, admin)
make build

# Build specific image
make build-backend
make build-web
make build-admin

# Push to registry
make push REGISTRY=192.168.1.100:5000

# Build + Push (release)
make release REGISTRY=192.168.1.100:5000 VERSION=v1.0.0

# List images in registry
make list

# Clean local images
make clean
```

**Version Tagging Strategy:**
- Git short SHA by default: `$(git rev-parse --short HEAD)`
- Manual version override: `VERSION=v1.0.0`
- Dual tags: `$(VERSION)` + `latest`

**Environment Variables - CRITICAL:**

Each package has `.env.example` template:
- **Backend**: `DATABASE_URL`, `RABBITMQ_URL`, `SUPABASE_*`, `OPENAI_API_KEY`, `MINIO_*`
- **Mobile**: `EXPO_PUBLIC_SUPABASE_*`, `EXPO_PUBLIC_API_URL`
- **Infrastructure**: `POSTGRES_PASSWORD`, `RABBITMQ_PASSWORD`, `MINIO_ROOT_*`

**Git Workflow (Monorepo):**

```bash
# Monorepo structure - NO shared workspaces
pensieve/
├── mobile/          # Independent package with own node_modules
├── backend/         # Independent package with own node_modules
├── web/             # Independent package with own node_modules
├── admin/           # Independent package with own node_modules
└── infrastructure/  # Docker Compose configs

# Each package manages dependencies independently
# No root package.json, no Yarn workspaces, no Lerna
```

**Critical Workflow Anti-Patterns - NEVER:**
- ❌ Run migrations in production without backup
- ❌ Use `synchronize: true` in TypeORM (even in dev)
- ❌ Modify migrations after they've been run in production
- ❌ Skip environment variable validation before deployment
- ❌ Deploy without running tests first
- ❌ Commit `.env` files to Git (only `.env.example`)

---

### Bug Fix Workflow - TDD Pattern

**🐛 Principe fondamental** : Chaque bug corrigé DOIT avoir un test de régression associé.

Ce workflow garantit que :
- Le bug est reproductible de manière fiable
- La correction est vérifiable automatiquement
- Le bug ne peut pas réapparaître sans être détecté
- La documentation du bug reste synchronisée avec le code

**Workflow général (RED-GREEN-REFACTOR)** :

1. **🔴 RED** - Créer un test qui reproduit le bug (le test échoue)
2. **🟢 GREEN** - Corriger le code jusqu'à ce que le test passe
3. **🔵 REFACTOR** - Améliorer le code si nécessaire (tests restent verts)
4. **✅ VERIFY** - Exécuter toute la suite de tests pour éviter les régressions

---

#### Option 1 : Bug de comportement utilisateur (BDD/Gherkin)

**Quand utiliser** : Bug affectant un comportement utilisateur, une interaction UI, ou un acceptance criteria.

**Procédure** :

**1. Ajouter un scénario Gherkin dans le fichier `.feature` existant**

```gherkin
# mobile/tests/acceptance/features/story-2-1.feature

@edge-case @bug-fix
Plan du scénario: Gérer les enregistrements très courts (Bug #123)
  Contexte: Un bug empêchait de sauvegarder les enregistrements < 1s
  Quand l'utilisateur enregistre pendant <durée> millisecondes
  Et l'utilisateur arrête l'enregistrement
  Alors la Capture est créée malgré la courte durée
  Et la Capture a un statut "CAPTURED"

  Exemples:
    | durée |
    | 100   |  # ← Bug reproduit ici
    | 500   |
    | 999   |
```

**Tags à utiliser** :
- `@bug-fix` - Scénario reproduisant un bug spécifique
- `@edge-case` - Cas limite qui n'était pas géré
- `@regression` - Test de non-régression d'un bug ancien

**2. Exécuter les tests (RED phase)**

```bash
# Mobile
cd mobile
npm run test:acceptance:story-2-1

# Backend
cd backend
npm run test:acceptance -- story-4-1

# ❌ Expected capture to exist but got 0 captures
```

**3. Corriger le code (GREEN phase)**

```typescript
// AVANT (bug) :
async stopRecording(): Promise<void> {
  const { duration } = await this.audioRecorder.stopRecording();

  // ❌ BUG: Rejette les enregistrements < 1s
  if (duration < 1000) {
    throw new Error('Recording too short');
  }

  await this.captureRepo.update(this.currentCaptureId!, {
    state: 'CAPTURED',
    duration,
  });
}

// APRÈS (corrigé) :
async stopRecording(): Promise<void> {
  const { duration } = await this.audioRecorder.stopRecording();

  // ✅ FIX: Accepter tous les enregistrements, warning si court
  if (duration < 100) {
    console.warn('Recording is very short:', duration);
  }

  // Sauvegarder dans tous les cas
  await this.captureRepo.update(this.currentCaptureId!, {
    state: 'CAPTURED',
    duration,
  });
}
```

**4. Vérifier que les tests passent (GREEN)**

```bash
npm run test:acceptance:story-2-1
# ✅ All tests pass including new edge case
```

**5. Vérifier la non-régression (VERIFY)**

```bash
# Exécuter TOUS les tests d'acceptance de la story
npm run test:acceptance

# Vérifier qu'aucun autre test n'a été cassé
```

---

#### Option 2 : Bug de fonction isolée (Test unitaire)

**Quand utiliser** : Bug dans une fonction pure, un helper, un utilitaire, ou une logique métier isolée.

**Procédure** :

**1. Créer un test unitaire qui reproduit le bug**

```typescript
// mobile/src/contexts/capture/services/__tests__/audio-trimmer.test.ts

describe('AudioTrimmer', () => {
  describe('Bug #456: Trim truncates last word', () => {
    it('should preserve full transcription when trimming silence', () => {
      // ARRANGE - Setup bug scenario
      const transcription = 'Bonjour le monde';
      const audioBuffer = createMockAudioWithSilence(5000); // 5s audio, 2s silence at end

      // ACT - Execute code that has the bug
      const result = AudioTrimmer.trimSilence(audioBuffer, transcription);

      // ASSERT - Bug: transcription was truncated to "Bonjour le"
      expect(result.transcription).toBe('Bonjour le monde'); // ❌ FAILS (RED)
      expect(result.duration).toBeLessThan(4000); // Silence trimmed
    });
  });
});
```

**2. Exécuter le test (RED phase)**

```bash
cd mobile
npx jest src/contexts/capture/services/__tests__/audio-trimmer.test.ts

# ❌ Expected: "Bonjour le monde"
# ❌ Received: "Bonjour le"
```

**3. Corriger la fonction (GREEN phase)**

```typescript
// AVANT (bug) :
export class AudioTrimmer {
  static trimSilence(
    audioBuffer: AudioBuffer,
    transcription: string
  ): { buffer: AudioBuffer; transcription: string; duration: number } {
    const trimmedBuffer = this.detectAndTrimSilence(audioBuffer);

    // ❌ BUG: Truncate transcription proportionally to audio trim
    const ratio = trimmedBuffer.duration / audioBuffer.duration;
    const charCount = Math.floor(transcription.length * ratio);
    const trimmedText = transcription.substring(0, charCount);

    return {
      buffer: trimmedBuffer,
      transcription: trimmedText,
      duration: trimmedBuffer.duration,
    };
  }
}

// APRÈS (corrigé) :
export class AudioTrimmer {
  static trimSilence(
    audioBuffer: AudioBuffer,
    transcription: string
  ): { buffer: AudioBuffer; transcription: string; duration: number } {
    const trimmedBuffer = this.detectAndTrimSilence(audioBuffer);

    // ✅ FIX: Only trim audio buffer, keep full transcription
    // Silence trimming doesn't affect speech content
    return {
      buffer: trimmedBuffer,
      transcription, // ← Keep original transcription intact
      duration: trimmedBuffer.duration,
    };
  }
}
```

**4. Vérifier que le test passe (GREEN)**

```bash
npx jest src/contexts/capture/services/__tests__/audio-trimmer.test.ts

# ✅ PASS  audio-trimmer.test.ts
```

**5. Refactoriser si nécessaire (REFACTOR)**

```typescript
// Améliorer la clarté du code
export class AudioTrimmer {
  static trimSilence(
    audioBuffer: AudioBuffer,
    transcription: string
  ): TrimResult {
    // Trim only the audio buffer, not the transcription
    // Rationale: Silence detection is audio-level, transcription is already accurate
    const trimmedBuffer = this.detectAndTrimSilence(audioBuffer);

    return {
      buffer: trimmedBuffer,
      transcription, // Preserve full transcription (silence already excluded by ASR)
      duration: trimmedBuffer.duration,
    };
  }
}
```

**6. Vérifier la non-régression (VERIFY)**

```bash
# Exécuter tous les tests unitaires du contexte
npx jest src/contexts/capture

# ✅ All tests pass
```

---

#### Commandes de test utiles

**Mobile** :

```bash
# Tests d'acceptance (BDD/Gherkin)
npm run test:acceptance                          # Tous les tests
npm run test:acceptance:story-2-1                # Story spécifique
npm run test:acceptance:watch                    # Mode watch
npm run test:acceptance -- --testNamePattern="@bug-fix"  # Filtrer par tag

# Tests unitaires
npm run test:unit                                # Tous les tests
npx jest src/path/to/file.test.ts                # Fichier spécifique
npm run test:unit:watch                          # Mode watch
npm run test:coverage                            # Rapport de couverture

# Tests E2E
npm run test:e2e                                 # Detox (iOS)
```

**Backend** :

```bash
# Tests d'acceptance (BDD/Gherkin)
npm run test:acceptance                          # Tous les tests
npm run test:acceptance -- story-4-1             # Story spécifique
npm run test:acceptance -- --testNamePattern="@bug-fix"  # Filtrer par tag

# Tests unitaires
npm run test                                     # Tous les tests
npx jest src/path/to/file.spec.ts                # Fichier spécifique
npm run test:watch                               # Mode watch
npm run test:cov                                 # Rapport de couverture

# Tests E2E
npm run test:e2e                                 # Tests E2E
```

---

#### Anti-patterns - NEVER ❌

**❌ Corriger un bug sans ajouter de test**
```typescript
// ❌ WRONG: Fix without test
async stopRecording(): Promise<void> {
  const { duration } = await this.audioRecorder.stopRecording();

  // Fixed bug but no test to prevent regression
  if (duration < 100) console.warn('Short recording');

  await this.captureRepo.update(this.currentCaptureId!, { state: 'CAPTURED' });
}
```
**Impact** : Le bug peut réapparaître lors d'un refactoring futur, aucune traçabilité.

**❌ Modifier le code AVANT d'écrire le test**
```typescript
// ❌ WRONG: Fix first, test later
// 1. Fix the code
// 2. Run app manually
// 3. "Looks good, ship it"
// 4. (Never write test)
```
**Impact** : Pas de garantie que le test reproduit vraiment le bug, test peut être faux positif.

**❌ Supprimer les tests de régression "qui passent déjà"**
```typescript
// ❌ WRONG: Clean up "useless" tests
// "This test always passes, let's remove it"
git rm tests/acceptance/features/story-2-1-edge-cases.feature
```
**Impact** : Protection contre les régressions perdue, bug peut revenir sans détection.

**❌ Ne pas tagger les tests de bug fixes**
```gherkin
# ❌ WRONG: Missing tags
Scénario: Gérer les enregistrements courts
  # No @bug-fix tag, hard to find later
```
**Impact** : Impossible de filtrer les tests de régression, perte de traçabilité.

**❌ Tester manuellement au lieu d'automatiser**
```bash
# ❌ WRONG: Manual verification only
# 1. Start app
# 2. Click record
# 3. Stop after 100ms
# 4. "Works for me ✓"
# (No automated test)
```
**Impact** : Test non reproductible, dépendant de l'opérateur, impossible à exécuter en CI/CD.

**❌ Fixer plusieurs bugs dans un seul commit/test**
```gherkin
# ❌ WRONG: Multiple bugs in one scenario
Scénario: Fix audio bugs
  Quand l'utilisateur enregistre un audio court
  Et l'utilisateur a peu de stockage
  Et le microphone perd la permission
  # Too many bugs mixed together
```
**Impact** : Difficile de debugger si le test échoue, perte de granularité.

---

**✅ Bonne pratique - Workflow complet exemple** :

```bash
# 1. Identifier le bug (via rapport utilisateur, log, etc.)
# Bug #789: L'app crash si l'enregistrement dépasse 60 minutes

# 2. Créer une branche
git checkout -b fix/bug-789-recording-timeout

# 3. Ajouter un test Gherkin (RED)
# mobile/tests/acceptance/features/story-2-1.feature
@edge-case @bug-fix @bug-789
Scénario: Gérer les enregistrements de longue durée
  Quand l'utilisateur enregistre pendant 3600 secondes
  Et l'utilisateur arrête l'enregistrement
  Alors la Capture est créée sans crash
  Et la durée est 3600000ms

# 4. Exécuter le test (doit échouer - RED)
npm run test:acceptance:story-2-1
# ❌ Timeout after 60000ms

# 5. Corriger le code (GREEN)
# src/services/recording.service.ts
- const MAX_RECORDING_DURATION = 60 * 1000; // ❌ 60s
+ const MAX_RECORDING_DURATION = 24 * 60 * 60 * 1000; // ✅ 24h

# 6. Vérifier que le test passe (GREEN)
npm run test:acceptance:story-2-1
# ✅ All tests pass

# 7. Vérifier la non-régression
npm run test:acceptance
# ✅ All acceptance tests pass

# 8. Commit avec message conventionnel
git add .
git commit -m "fix(audio): support recordings up to 24 hours (Bug #789)"

# 9. Push et PR
git push origin fix/bug-789-recording-timeout
```

---

### Critical Don't-Miss Rules

**🚨 TOP 10 RÈGLES ABSOLUES - VIOLATIONS = BUGS MAJEURS 🚨**

**1. DI Container Resolution (Mobile) - MOST CRITICAL**
```typescript
// ❌ FATAL: Module-level resolution CRASHES before bootstrap
import { container } from '@/infrastructure/di/container';
const logger = container.resolve<ILogger>(TOKENS.ILogger);

// ✅ CORRECT: Lazy resolution inside hooks/effects
const useLogger = () => {
  const getLogger = () => container.resolve<ILogger>(TOKENS.ILogger);
  return getLogger();
};
```
**Impact si violé**: App crash au démarrage, "container not registered" error.

**2. Jest Configuration (Mobile) - CRITICAL**
```javascript
// ❌ FATAL: jest-expo breaks with Expo SDK 54 Winter runtime
preset: 'jest-expo'

// ✅ CORRECT: babel-jest for unit tests
transform: { '^.+\\.(ts|tsx)$': 'babel-jest' }
testEnvironment: 'node'
```
**Impact si violé**: Tests crash avec "ReferenceError: You are trying to import a file outside of the scope".

**3. TypeORM Synchronize (Backend) - CRITICAL**
```typescript
// ❌ FATAL: NEVER use synchronize, even in dev
TypeOrmModule.forRoot({ synchronize: true })

// ✅ CORRECT: Migrations only
TypeOrmModule.forRoot({ synchronize: false })
// ALL schema changes via: npm run migration:generate
```
**Impact si violé**: Data loss, schema drift, production disasters.

**4. Legacy Libraries - ABSOLUTE PROHIBITION**
```typescript
// ❌ BANNED: NEVER import from /legacy
import * from 'expo-file-system/legacy';

// ✅ CORRECT: Modern APIs only
import * from 'expo-file-system';
```
**Impact si violé**: Tech debt, security vulnerabilities, incompatibilité future.

**5. TypeScript Strict Mode - NON-NEGOTIABLE**
```json
// ❌ FATAL: NEVER disable strict
{ "strict": false }

// ✅ CORRECT: Strict mode everywhere
{ "strict": true }
```
**Impact si violé**: Type safety gone, runtime errors, maintenance nightmare.

**6. Domain Layer Pollution (DDD) - ARCHITECTURE VIOLATION**
```typescript
// ❌ WRONG: DTOs/Mappers in domain layer
// domain/Capture.model.ts
export const toDTO = (capture: Capture) => { ... }

// ✅ CORRECT: DTOs in infrastructure layer
// infrastructure/repositories/capture.repository.ts
private toDTO(capture: Capture) { ... }
```
**Impact si violé**: Clean Architecture violation, domain coupling, impossible to swap implementations.

**7. NestJS Hybrid App Bootstrap - CRITICAL**
```typescript
// ❌ WRONG: Missing microservice setup
await app.listen(3000);

// ✅ CORRECT: HTTP + RabbitMQ
app.connectMicroservice(/* RabbitMQ config */);
await app.startAllMicroservices();
await app.listen(3000);
```
**Impact si violé**: RabbitMQ consumers never start, async jobs broken.

**8. AI Attribution - ABSOLUTE PROHIBITION**
```bash
# ❌ BANNED: NEVER sign commits with AI attribution
git commit -m "feat: add feature

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"

# ✅ CORRECT: User commits only
git commit -m "feat: add feature"
```
**Impact si violé**: Confidentiality breach, professional credibility loss.

**9. BDD Test Coverage - MANDATORY**
```bash
# ❌ WRONG: Ship story without Gherkin tests
# Story marked done, no .feature file

# ✅ CORRECT: Every story has BDD tests
tests/acceptance/features/story-X-Y.feature
tests/acceptance/story-X-Y.test.ts
```
**Impact si violé**: Regression bugs, acceptance criteria not verified, QA failures.

**10. Environment Variables - SECURITY CRITICAL**
```bash
# ❌ FATAL: Commit .env to Git
git add .env && git commit

# ✅ CORRECT: Only .env.example in Git
git add .env.example
# .env in .gitignore
```
**Impact si violé**: API keys exposed, security breach, credential leak.

**⚠️ BONUS CRITICAL RULES:**

**11. Module System Consistency**
- **ALL packages use ESM** (`import`, `export`)
- Backend tsconfig: `"module": "nodenext"`
- NEVER mix CommonJS with ESM within same package
- NEVER use `require()` in TypeScript files

**12. Expo SDK Version Lock**
- Node 22 + Expo SDK 54 + React Native 0.81.5
- NEVER upgrade Expo without checking all dependencies
- NEVER downgrade to solve compatibility

**13. Test File Organization**
- Unit tests: `babel-jest` config
- Acceptance tests: `ts-jest` config
- NEVER mix test runners in same file

**14. Migration Immutability**
- NEVER modify migration after production run
- NEVER delete migration files
- ALWAYS create new migration for changes

**15. Conventional Commits**
- ALWAYS use `feat:`, `fix:`, `refactor:`, etc.
- NEVER commit without type prefix
- Impacts changelog generation, semantic versioning

---

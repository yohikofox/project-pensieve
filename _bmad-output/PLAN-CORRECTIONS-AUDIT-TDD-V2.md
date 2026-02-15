# 📋 PLAN DE CORRECTIONS AUDIT V2 - MODE TDD
**Date:** 2026-02-15 (Version 2 - Enrichi)
**Projet:** Pensieve
**Basé sur:** AUDIT-CODE-ENRICHI-2026-02-15.md

---

## 🎯 OBJECTIF

Corriger les **54 problèmes** (vs 34 v1) identifiés dans l'audit adversarial enrichi en suivant le cycle **RED-GREEN-REFACTOR** du TDD.

**Changements vs V1:**
- +7 violations BLOCKING Definition of Done
- +5 violations BLOCKING ADR-024 Clean Code
- +20 violations totales
- +78h effort (69.5h → 147.5h)

---

## 📊 VUE D'ENSEMBLE

### Métriques Globales V2

| Catégorie | Tasks | Effort Total | Problèmes Résolus |
|-----------|-------|--------------|-------------------|
| 🔴 BLOCKING DoD | 7 | 13h | 7 BLOCKING |
| 🔴 BLOCKING ADR-024 | 5 | 30.5h | 5 BLOCKING |
| 🔴 CRITICAL Original | 4 | 33h | 15 CRITICAL |
| 🟠 HIGH | 5 | 40.5h | 14 HIGH |
| 🟡 MEDIUM | 2 | 30.5h | 13 MEDIUM |
| **TOTAL** | **23** | **147.5h** | **54 issues** |

### Distribution Effort V2

```
BLOCKING DoD (13h):      ████░░░░░░░░░░░░░░░░░░░░ 9%
BLOCKING ADR-024 (30.5h):█████████████░░░░░░░░░░░░ 21%
CRITICAL (33h):          ██████████████░░░░░░░░░░░ 22%
HIGH (40.5h):            ████████████████░░░░░░░░░ 27%
MEDIUM (30.5h):          ██████████████░░░░░░░░░░░ 21%
```

**Stratégie 3 Sprints:**
- Sprint N (3 sem): BLOCKING (43.5h = 15h/sem)
- Sprint N+1 (3 sem): CRITICAL (33h = 11h/sem)
- Sprint N+2 (2 sem): HIGH+MEDIUM (71h = 35h/sem)

---

## 🔴 PHASE 0: BLOCKING - DEFINITION OF DONE (13h)

**Objectif:** Rendre le code déployable selon DoD
**Référence:** project-context.md lines 619-731

---

### Task #14: Remplacer console.log par Logger service 🚫

**ID:** 14
**Effort:** 4h
**Priorité:** BLOCKING - DoD Console Cleanliness (line 623-630)

**Problème:**
- 18 fichiers avec console.log/error en production
- Backend: 12 fichiers (main.ts, rgpd.controller.ts, minio.service.ts, etc.)
- Mobile: 6 fichiers (LLMModelService.ts, TodoRepository.ts, etc.)

**Règle DoD violée:**
```
Console Cleanliness (BLOCKING):
- Zero errors in console
- Zero warnings in console
→ Any console pollution = STORY NOT DONE
```

**TDD Steps:**

1. 🔴 RED: Tests vérifiant zéro console.log
```bash
# Créer test
touch pensieve/backend/src/__tests__/console-clean.test.ts

# Test content
describe('Console Cleanliness', () => {
  it('should have zero console.log calls in production code', () => {
    const consoleUsage = execSync('grep -r "console\\." src/').toString();
    expect(consoleUsage).toBe('');
  });
});

# Exécuter (RED)
npx jest console-clean.test.ts
```

2. 🟢 GREEN: Implémenter Logger

**Backend (NestJS Logger):**
```typescript
// Remplacer
console.log('Backend listening on:', url);

// Par
this.logger.log('Backend listening on:', url);
```

**Mobile (Custom Logger):**
```typescript
// src/utils/logger.ts
export const logger = {
  log: (message: string, ...args: any[]) => {
    if (__DEV__) {
      console.log(message, ...args);
    }
  },
  error: (message: string, ...args: any[]) => {
    if (__DEV__) {
      console.error(message, ...args);
    }
    // En prod → Sentry
  }
};

// Remplacer dans tous les fichiers
import { logger } from '@/utils/logger';
logger.log('Debug info');
```

3. 🔵 REFACTOR: ESLint rule

**Ajouter `.eslintrc.js`:**
```javascript
rules: {
  'no-console': 'error',  // Bloquer console.log
}
```

**Commandes:**
```bash
# Lister violations
cd pensieve/backend
grep -rn "console\." src/ | wc -l  # Devrait être 18

cd ../mobile
grep -rn "console\." src/ | wc -l  # Devrait être 6

# Après correction
npm run lint  # Devrait passer sans erreurs

# Valider console clean
grep -r "console\." src/ || echo "✅ Clean"
```

**Livrables:**
- [ ] Logger service créé (backend + mobile)
- [ ] 18 fichiers migrés → Logger
- [ ] ESLint rule `no-console` activée
- [ ] Tests passent

**Dépend de:** Aucun

---

### Task #15: Remplacer types `any` par types stricts 🚫

**ID:** 15
**Effort:** 1.5h
**Priorité:** BLOCKING - TypeScript Strict Mode + DoD

**Problème:**
- 6 fichiers avec `any` type
- Violation strict mode + DoD requirements

**Fichiers:**
1. `rgpd.controller.ts:31,64` - `@Req() req: any`
2. `admin-auth.controller.ts:52,68,96` - `@Request() req: any`
3. `sync.controller.ts:46,74` - `@Request() req: any`

**TDD Steps:**

1. 🔴 RED: Test type safety
```bash
# Activer ESLint rule
# .eslintrc.js
rules: {
  '@typescript-eslint/no-explicit-any': 'error'
}

# Exécuter lint (RED)
npm run lint  # Devrait échouer sur 6 fichiers
```

2. 🟢 GREEN: Créer interface typée
```typescript
// src/types/express.types.ts
import { Request } from 'express';
import { User } from '@/modules/identity/domain/user.entity';

export interface AuthenticatedRequest extends Request {
  user: User;
}

// Remplacer dans controllers
@Post('export')
async exportUserData(@Req() req: AuthenticatedRequest) {  // ✅
  const userId = req.user.id;  // Type-safe
}
```

3. 🔵 REFACTOR: Valider avec tsc
```bash
# Compiler TypeScript
npx tsc --noEmit

# Devrait passer sans erreurs
```

**Commandes:**
```bash
# Démarrer
grep -rn ": any" pensieve/backend/src/modules | wc -l  # = 6

# Après correction
npm run lint
npx tsc --noEmit
grep -rn ": any" pensieve/backend/src | wc -l  # = 0
```

**Livrables:**
- [ ] Interface `AuthenticatedRequest` créée
- [ ] 6 fichiers migrés vers types stricts
- [ ] ESLint rule `no-explicit-any` activée
- [ ] `tsc --noEmit` passe

---

### Task #16: Fix exposition error messages 🔒

**ID:** 16
**Effort:** 30min
**Priorité:** BLOCKING - Sécurité (Information Disclosure)

**Problème:**
- `error.message` exposé au client (rgpd.controller.ts:49)
- Leak possible: stack traces, DB paths, file paths

**TDD Steps:**

1. 🔴 RED: Test vérifiant messages génériques
```typescript
// rgpd.controller.spec.ts
describe('RGPD Controller Security', () => {
  it('should NOT expose error.message to client', async () => {
    // Simuler erreur DB
    jest.spyOn(rgpdService, 'exportUserData').mockRejectedValue(
      new Error('ENOENT: /var/data/user_123.json')  // Message interne
    );

    const response = await controller.exportUserData(mockRequest);

    expect(response.message).not.toContain('ENOENT');
    expect(response.message).not.toContain('/var/data');
    expect(response.message).toBe('Export failed');  // Message générique
  });
});
```

2. 🟢 GREEN: Implémenter messages génériques
```typescript
// rgpd.controller.ts:45-50
} catch (error) {
  // ✅ Logger côté serveur (pas console)
  this.logger.error('Export failed:', error.message, error.stack);

  return {
    success: false,
    message: 'Export failed',  // ✅ Message générique
    data: null,
  };
}
```

3. 🔵 REFACTOR: Enum erreurs client
```typescript
// src/types/errors.ts
export enum ClientErrorMessage {
  EXPORT_FAILED = 'Export failed',
  DELETION_FAILED = 'Deletion failed',
  UNAUTHORIZED = 'Unauthorized access',
}

// Usage
message: ClientErrorMessage.EXPORT_FAILED,
```

**Commandes:**
```bash
# Démarrer
grep -rn "error.message" pensieve/backend/src/modules/rgpd

# Exécuter test
npx jest rgpd.controller.spec.ts

# Audit security
grep -rn "error\\.message" pensieve/backend/src | wc -l  # = 0
```

**Livrables:**
- [ ] Messages génériques implémentés
- [ ] Test sécurité créé et passant
- [ ] Logger server-side configuré
- [ ] Aucun `error.message` exposé au client

---

### Task #17: Fix secret JWT hardcodé 🔒

**ID:** 17
**Effort:** 15min
**Priorité:** BLOCKING - Sécurité Critique

**Problème:**
- Fallback `'admin-secret-key-change-in-production'` (admin-auth.module.ts:20)
- Risque production si JWT_SECRET manquant

**TDD Steps:**

1. 🔴 RED: Test vérifiant throw si JWT_SECRET manquant
```typescript
// admin-auth.module.spec.ts
describe('Admin Auth Module Config', () => {
  it('should throw if JWT_SECRET is missing', () => {
    delete process.env.JWT_SECRET;

    expect(() => {
      getJwtSecret();
    }).toThrow('JWT_SECRET environment variable is required');
  });

  it('should return secret if JWT_SECRET is set', () => {
    process.env.JWT_SECRET = 'test-secret';
    expect(getJwtSecret()).toBe('test-secret');
  });
});
```

2. 🟢 GREEN: Implémenter validation
```typescript
// admin-auth.module.ts
function getJwtSecret(): string {
  const secret = process.env.JWT_SECRET;
  if (!secret) {
    throw new Error('JWT_SECRET environment variable is required');
  }
  return secret;
}

JwtModule.register({
  secret: getJwtSecret(),  // ✅ Throw si manquant
  signOptions: { expiresIn: '7d' },
}),
```

3. 🔵 REFACTOR: Documenter .env.example
```bash
# .env.example
JWT_SECRET=your-secret-key-here-min-32-chars
```

**Commandes:**
```bash
# Démarrer
grep -n "JWT_SECRET.*||" pensieve/backend/src

# Exécuter test
npx jest admin-auth.module.spec.ts

# Valider
grep "JWT_SECRET.*||" pensieve/backend/src | wc -l  # = 0
```

**Livrables:**
- [ ] Fonction `getJwtSecret()` avec validation
- [ ] Fallback hardcodé supprimé
- [ ] Tests créés et passants
- [ ] `.env.example` documenté

---

### Task #18: Supprimer imports legacy expo-file-system ❌

**ID:** 18 (ex-Task #1 réorganisée)
**Effort:** 2.5h (augmenté - 4 fichiers au lieu de 3)
**Priorité:** BLOCKING - Interdiction absolue + DoD

**Problème:**
- 4 fichiers importent `expo-file-system/legacy` (BANNI)
- Violation project-context.md lines 1000-1009 + DoD legacy ban

**Fichiers:**
1. `CaptureDevTools.tsx:23`
2. `SettingsScreen.tsx:17`
3. `SettingsScreen.test.tsx:6`
4. **NOUVEAU:** `utils/file-helpers.ts:8`

**TDD Steps:**
1. 🔴 RED: Créer test vérifiant API moderne
2. 🟢 GREEN: Remplacer imports `/legacy` → API moderne
3. 🔵 REFACTOR: Valider aucun legacy subsiste

**Commandes:**
```bash
# Démarrer
npx jest src/components/dev/__tests__/CaptureDevTools.migration.test.ts

# Valider
grep -r "expo-file-system/legacy" pensieve/mobile/src/ || echo "✅ Clean"

# ESLint ban
# .eslintrc.js
rules: {
  'no-restricted-imports': ['error', {
    patterns: ['*/legacy']
  }]
}
```

**Livrables:**
- [ ] 4 imports legacy → API moderne
- [ ] Tests migration passants
- [ ] ESLint rule bannissant `/legacy`
- [ ] Validation iOS + Android

**Dépend de:** Aucun

---

### Task #19: Fix npm vulnerabilities 🔒

**ID:** 19
**Effort:** 2h
**Priorité:** BLOCKING - DoD Zero Vulnerabilities

**Problème:**
- Backend: 5 vulnerabilities (2 low, 2 moderate, 1 high)
- Mobile: 1 high severity
- DoD requirement: 0 vulnerabilities

**TDD Steps:**

1. 🔴 RED: CI check bloquant sur vulnerabilities
```bash
# .github/workflows/ci.yml
- name: Security Audit
  run: |
    npm audit --audit-level=moderate
    if [ $? -ne 0 ]; then
      echo "❌ npm audit failed - fix vulnerabilities"
      exit 1
    fi
```

2. 🟢 GREEN: Fixer vulnerabilities
```bash
cd pensieve/backend
npm audit fix --force

# Si unfixable
npm audit  # Identifier packages
# Chercher alternatives ou documenter waiver
```

3. 🔵 REFACTOR: Automatiser checks
```bash
# pre-commit hook
npm run audit:check

# package.json
"scripts": {
  "audit:check": "npm audit --audit-level=moderate"
}
```

**Commandes:**
```bash
# Backend
cd pensieve/backend
npm audit
npm audit fix --force
npm audit --audit-level=moderate  # Devrait retourner 0

# Mobile
cd ../mobile
npm audit
npm audit fix --force
npm audit --audit-level=moderate  # Devrait retourner 0
```

**Livrables:**
- [ ] 0 vulnerabilities backend
- [ ] 0 vulnerabilities mobile
- [ ] CI check configuré
- [ ] Pre-commit hook ajouté

---

### Task #20: Fix tests skipped 🧪

**ID:** 20
**Effort:** 3h
**Priorité:** BLOCKING - DoD 100% Tests Pass

**Problème:**
- 3 tests avec `.skip()` ou `.only()`
- DoD requirement: 100% tests passants

**Fichiers:**
1. `story-2-3.test.ts` - Test flaky sync incremental
2. `sync.service.spec.ts` - Race condition concurrent uploads
3. (À identifier le 3e)

**TDD Steps:**

1. 🔴 RED: CI fail si skip/only détecté
```bash
# .github/workflows/ci.yml
- name: Check for skipped tests
  run: |
    if grep -r "\.skip\|\.only" tests/; then
      echo "❌ Skipped or focused tests detected"
      exit 1
    fi
```

2. 🟢 GREEN: Fixer tests skippés
```typescript
// story-2-3.test.ts - Fix flakiness
test('AC2.3.3: Sync incremental', async () => {
  // Ajouter wait stable
  await waitForSyncStable();

  // Ajouter retry logic
  await retryUntilSuccess(() => {
    expect(syncResult.type).toBe(ResultType.SUCCESS);
  }, { maxRetries: 3, delay: 1000 });
});
```

3. 🔵 REFACTOR: Valider stabilité
```bash
# Exécuter 10 fois consécutivement
for i in {1..10}; do
  npm run test:acceptance story-2-3.test.ts
done
```

**Commandes:**
```bash
# Lister skipped tests
grep -rn "\.skip\|\.only" pensieve/mobile/tests/

# Activer et exécuter
npx jest story-2-3.test.ts --no-skip

# Valider stabilité
npm run test:acceptance -- --runInBand
```

**Livrables:**
- [ ] 3 tests activés et stables
- [ ] CI check configuré (fail on skip/only)
- [ ] Validation stabilité (10 runs)
- [ ] 100% tests passants

---

## 🔴 PHASE 0-BIS: BLOCKING - ADR-024 CLEAN CODE (30.5h)

**Objectif:** Conformité ADR-024 NON-NÉGOCIABLES
**Référence:** ADR-024, project-context.md lines 412-460

---

### Task #21: Formater TODOs avec ticket IDs 📝

**ID:** 21
**Effort:** 4h
**Priorité:** BLOCKING - ADR-024 Tier 1

**Problème:**
- 26 TODOs sans format `// TODO(TICKET-ID):`
- Violation ADR-024 NON-NÉGOCIABLE

**Breakdown:**
- Capture context: 8 TODOs
- Action context: 4 TODOs
- Normalization context: 3 TODOs
- Backend: 11 TODOs

**TDD Steps:**

1. 🔴 RED: Pre-commit hook bloquant TODOs mal formatés
```bash
# .git/hooks/pre-commit
if grep -rn "// TODO[^(]" src/; then
  echo "❌ Malformed TODOs detected - use // TODO(TICKET-ID):"
  exit 1
fi
```

2. 🟢 GREEN: Reformater tous les TODOs
```bash
# Script de détection
grep -rn "// TODO" pensieve/mobile/src/ | grep -v "TODO("

# Pour chaque TODO:
# Option A: Créer story/sub-task → obtenir ID
# Option B: Implémenter immédiatement si < 30min
# Option C: Reformater avec ID existant

# Exemple
// TODO ADR-023: Should return Result<T>
# → Créer sub-task STORY-6-1-SUBTASK-3
# → Remplacer par
// TODO(STORY-6-1-SUBTASK-3): Migrate to Result pattern per ADR-023
```

3. 🔵 REFACTOR: ESLint rule custom
```javascript
// .eslintrc.js
rules: {
  'no-warning-comments': ['error', {
    terms: ['todo', 'fixme'],
    location: 'anywhere',
    // Require TODO(TICKET-ID) format
  }]
}
```

**Commandes:**
```bash
# Lister TODOs mal formatés
grep -rn "// TODO" src/ | grep -v "TODO(" > malformed-todos.txt

# Après correction
grep -rn "// TODO" src/ | grep -v "TODO(" | wc -l  # = 0

# Valider format
grep -rn "// TODO(" src/ | wc -l  # = 26 (tous bien formatés)
```

**Livrables:**
- [ ] 26 TODOs reformatés avec ticket IDs
- [ ] Stories/sub-tasks créées si nécessaire
- [ ] Pre-commit hook configuré
- [ ] ESLint rule activée

**Effort réparti:**
- Review + décisions: 2h
- Création tickets: 1h
- Reformatage: 1h

---

### Task #22: Supprimer code commenté 🗑️

**ID:** 22
**Effort:** 1.5h
**Priorité:** BLOCKING - ADR-024 Tier 1

**Problème:**
- 6 blocs de code commenté
- Git est l'historique → suppression obligatoire

**Fichiers:**
1. `AudioConversionService.ts:152-158` - trimSilence logic
2. `QueueDetailsScreen.tsx:58-62` - Retry API calls
3. `MediaProcessingService.ts:234-240`
4. `migrations.ts:445-456`
5. `sync.repository.ts:178-182`
6. `knowledge.service.ts:201-208`

**TDD Steps:**

1. 🔴 RED: Pre-commit hook détectant code commenté
```bash
# .git/hooks/pre-commit
# Regex détectant blocs commentés multi-lignes
if grep -Pzo "(?s)/\*.*?\*/" src/ | grep "const\|function\|if\|await"; then
  echo "❌ Commented code detected - delete it, Git is your history"
  exit 1
fi
```

2. 🟢 GREEN: Supprimer tous les blocs
```bash
# Pour chaque bloc:
# 1. Vérifier si code important → créer story avec snippet
# 2. Supprimer le bloc
# 3. Git commit avec message clair

# Exemple AudioConversionService.ts
git diff src/contexts/Normalization/services/AudioConversionService.ts
# Supprimer lines 152-158
# Commit: "refactor: remove commented trimSilence code - see Story 8.3 if needed"
```

3. 🔵 REFACTOR: ESLint plugin
```bash
npm install --save-dev eslint-plugin-no-commented-out-code

# .eslintrc.js
plugins: ['no-commented-out-code'],
rules: {
  'no-commented-out-code/no-commented-out-code': 'error'
}
```

**Commandes:**
```bash
# Démarrer
# Script custom pour détecter blocs commentés (regex avancé)

# Après suppression
npm run lint
# ESLint devrait passer

# Valider manuellement
git diff --cached  # Vérifier suppressions
```

**Livrables:**
- [ ] 6 blocs commentés supprimés
- [ ] Stories créées si code à réimplémenter
- [ ] ESLint plugin activé
- [ ] Pre-commit hook configuré

---

### Task #23: Extraire magic numbers en constantes 🔢

**ID:** 23
**Effort:** 3h
**Priorité:** BLOCKING - ADR-024 Tier 1

**Problème:**
- 8+ magic numbers détectés
- Violation lisibilité code

**Violations principales:**
1. `RetentionPolicyService.ts:331` - `k = 1024`, `i = Math.floor(...)`
2. `LLMModelService.ts:234` - `retryCount > 3`
3. `MediaProcessingService.ts:156` - `await delay(5000)`
4. `sync.service.ts:92` - `maxBatchSize: 100`
5. `rgpd.controller.ts:28` - `setTimeout(() => {}, 3600000)`

**TDD Steps:**

1. 🔴 RED: ESLint rule no-magic-numbers
```javascript
// .eslintrc.js
rules: {
  'no-magic-numbers': ['error', {
    ignore: [0, 1, -1],  // Exceptions
    ignoreArrayIndexes: true,
    enforceConst: true
  }]
}

# Exécuter lint (RED)
npm run lint  # Devrait échouer sur 8+ magic numbers
```

2. 🟢 GREEN: Extraire constantes
```typescript
// RetentionPolicyService.ts
// ❌ BEFORE
const k = 1024;
const i = Math.floor(Math.log(bytes) / Math.log(k));

// ✅ AFTER
const BYTES_PER_UNIT = 1024;
const unitIndex = Math.floor(Math.log(bytes) / Math.log(BYTES_PER_UNIT));

// LLMModelService.ts
// ❌ BEFORE
if (retryCount > 3) { }

// ✅ AFTER
const MAX_LLM_RETRIES = 3;
if (retryCount > MAX_LLM_RETRIES) { }

// MediaProcessingService.ts
// ❌ BEFORE
await delay(5000);

// ✅ AFTER
const PROCESSING_DELAY_MS = 5000;
await delay(PROCESSING_DELAY_MS);
```

3. 🔵 REFACTOR: Grouper constantes
```typescript
// src/constants/timing.ts
export const Timing = {
  PROCESSING_DELAY_MS: 5000,
  EXPORT_TIMEOUT_MS: 3600000,
  RETRY_DELAY_MS: 1000,
} as const;

// src/constants/limits.ts
export const Limits = {
  MAX_LLM_RETRIES: 3,
  MAX_SYNC_BATCH_SIZE: 100,
  BYTES_PER_UNIT: 1024,
} as const;
```

**Commandes:**
```bash
# Démarrer
npm run lint  # Lister magic numbers

# Identifier tous magic numbers (regex)
grep -rE "\b[0-9]{3,}\b" pensieve/mobile/src/ | grep -v "test\|spec"

# Après correction
npm run lint  # Devrait passer
```

**Livrables:**
- [ ] 8+ magic numbers extraits
- [ ] Constantes groupées dans fichiers dédiés
- [ ] ESLint rule `no-magic-numbers` activée
- [ ] Nommage UPPER_SNAKE_CASE respecté

---

### Task #24: Refactor violations SRP - Phase 1 📦

**ID:** 24
**Effort:** 20h
**Priorité:** BLOCKING - ADR-024 Tier 1

**Problème:**
- 7 fichiers violent SRP (> 300 lines ou > 10 methods)
- Impact maintenabilité catastrophique

**Scope Phase 1 (Sprint N):**
1. **LLMModelService.ts** - 825 lines, 77 methods → Split en 5 services (12h)
2. **LLMSettingsScreen.tsx** - 1196 lines → Extract components (6h)
3. **migrations.ts** - 1898 lines → Split par version (2h - juste planning)

**Scope Phase 2 (Sprint N+1):**
- Autres fichiers (QueueDetailsScreen, CaptureRepository, etc.)

**TDD Steps - LLMModelService.ts:**

1. 🔴 RED: Tests vérifiant façade
```typescript
// LLMModelService.spec.ts
describe('LLMModelService (Façade)', () => {
  it('should have max 10 public methods', () => {
    const methods = Object.getOwnPropertyNames(LLMModelService.prototype)
      .filter(m => !m.startsWith('_') && m !== 'constructor');

    expect(methods.length).toBeLessThanOrEqual(10);
  });

  it('should delegate to specialized services', () => {
    // Vérifier que LLMModelService utilise:
    // - LLMApiClient
    // - LLMRetryService
    // - LLMCacheService
    // - LLMValidationService
  });
});
```

2. 🟢 GREEN: Split en services
```typescript
// services/llm/LLMApiClient.ts (150 lines)
export class LLMApiClient {
  async callOpenAI(prompt: string): Promise<Result<string>> { }
  async callAnthropic(prompt: string): Promise<Result<string>> { }
  async callOllama(prompt: string): Promise<Result<string>> { }
  // 8 autres méthodes API
}

// services/llm/LLMRetryService.ts (100 lines)
export class LLMRetryService {
  async withRetry<T>(fn: () => Promise<Result<T>>): Promise<Result<T>> { }
  // Exponential backoff logic
}

// services/llm/LLMCacheService.ts (120 lines)
export class LLMCacheService {
  async get(key: string): Promise<string | null> { }
  async set(key: string, value: string): Promise<void> { }
}

// services/llm/LLMValidationService.ts (80 lines)
export class LLMValidationService {
  validatePrompt(prompt: string): Result<void> { }
  validateResponse(response: string): Result<void> { }
}

// services/llm/LLMModelService.ts (100 lines - FAÇADE)
export class LLMModelService {
  constructor(
    private apiClient: LLMApiClient,
    private retryService: LLMRetryService,
    private cacheService: LLMCacheService,
    private validationService: LLMValidationService,
  ) {}

  // 8 méthodes publiques orchestrant les services
  async processCapture(capture: Capture): Promise<Result<DigestedCapture>> {
    const validation = this.validationService.validatePrompt(capture.rawContent);
    if (validation.type !== ResultType.SUCCESS) return validation;

    const cached = await this.cacheService.get(capture.id);
    if (cached) return success(JSON.parse(cached));

    const result = await this.retryService.withRetry(() =>
      this.apiClient.callOpenAI(capture.rawContent)
    );

    if (result.type === ResultType.SUCCESS) {
      await this.cacheService.set(capture.id, JSON.stringify(result.value));
    }

    return result;
  }
}
```

3. 🔵 REFACTOR: DI configuration
```typescript
// contexts/Normalization/di-container.ts
container.registerSingleton('LLMApiClient', LLMApiClient);
container.registerSingleton('LLMRetryService', LLMRetryService);
container.registerSingleton('LLMCacheService', LLMCacheService);
container.registerSingleton('LLMValidationService', LLMValidationService);

container.registerSingleton('LLMModelService', (c) => new LLMModelService(
  c.resolve('LLMApiClient'),
  c.resolve('LLMRetryService'),
  c.resolve('LLMCacheService'),
  c.resolve('LLMValidationService'),
));
```

**TDD Steps - LLMSettingsScreen.tsx:**

1. 🔴 RED: Tests vérifiant composants extraits
```typescript
// LLMSettingsScreen.spec.tsx
describe('LLMSettingsScreen', () => {
  it('should be max 300 lines', () => {
    const lineCount = fs.readFileSync('LLMSettingsScreen.tsx', 'utf-8').split('\n').length;
    expect(lineCount).toBeLessThanOrEqual(300);
  });

  it('should use extracted components', () => {
    const { getByTestId } = render(<LLMSettingsScreen />);
    expect(getByTestId('model-selector')).toBeTruthy();
    expect(getByTestId('api-key-input')).toBeTruthy();
  });
});
```

2. 🟢 GREEN: Extract components
```typescript
// screens/settings/llm/components/ModelSelector.tsx (80 lines)
export const ModelSelector: React.FC<Props> = ({ value, onChange }) => {
  // UI sélection modèle
};

// screens/settings/llm/components/APIKeyInput.tsx (60 lines)
export const APIKeyInput: React.FC<Props> = ({ provider, value, onChange }) => {
  // UI input API key avec validation
};

// screens/settings/llm/LLMSettingsScreen.tsx (200 lines)
export const LLMSettingsScreen: React.FC = () => {
  const { settings, updateSettings } = useLLMSettings();

  return (
    <Screen>
      <ModelSelector value={settings.model} onChange={updateSettings} />
      <APIKeyInput provider={settings.provider} value={settings.apiKey} />
      {/* ... */}
    </Screen>
  );
};
```

**Commandes:**
```bash
# Mesurer lines
wc -l pensieve/mobile/src/contexts/Normalization/services/LLMModelService.ts
# Avant: 825 lines
# Après: 100 lines (façade)

# Exécuter tests
npx jest LLMModelService.spec.ts

# Valider SRP
npm run lint
```

**Livrables Phase 1:**
- [ ] LLMModelService: 825 → 100 lines, 77 → 8 methods
- [ ] LLMSettingsScreen: 1196 → 200 lines
- [ ] migrations.ts: Plan split documenté
- [ ] Tous tests passants
- [ ] CI check files > 500 lines configuré

**Dépend de:** Aucun (parallélisable avec autres tasks)

**Effort réparti:**
- LLMModelService split: 12h
- LLMSettingsScreen refactor: 6h
- migrations.ts planning: 2h

---

### Task #25: Fix noms variables non explicites 🏷️

**ID:** 25
**Effort:** 2h
**Priorité:** BLOCKING - ADR-024 Tier 1

**Problème:**
- 4 violations nommage (variables 1-lettre, abbreviations)

**Violations:**
1. `RetentionPolicyService.ts:331-335` - `k`, `i`
2. `SyncService.ts:178` - `c`, `ch`

**TDD Steps:**

1. 🔴 RED: ESLint rule id-length
```javascript
// .eslintrc.js
rules: {
  'id-length': ['error', {
    min: 3,
    exceptions: ['i', 'j', 'k'],  // Loops only
    properties: 'never'
  }]
}

# Exécuter lint (RED)
npm run lint  # Devrait échouer sur 4 variables
```

2. 🟢 GREEN: Renommer variables
```typescript
// RetentionPolicyService.ts
// ❌ BEFORE
const k = 1024;
const i = Math.floor(Math.log(bytes) / Math.log(k));

// ✅ AFTER
const BYTES_PER_UNIT = 1024;
const unitIndex = Math.floor(Math.log(bytes) / Math.log(BYTES_PER_UNIT));

// SyncService.ts
// ❌ BEFORE
private async processChanges(c: Change[]): Promise<Result<void>> {
  for (const ch of c) { }
}

// ✅ AFTER
private async processChanges(changes: Change[]): Promise<Result<void>> {
  for (const change of changes) { }
}
```

3. 🔵 REFACTOR: Code review checklist
```markdown
# Pull Request Checklist
- [ ] Aucune variable 1-lettre (sauf i,j,k dans loops)
- [ ] Noms explicites (> 3 chars)
- [ ] Pas d'abbreviations obscures
```

**Commandes:**
```bash
# Démarrer
npm run lint

# Identifier variables courtes
grep -rE "\b[a-z]\b\s*=" pensieve/mobile/src/ | grep -v "for\|test"

# Après correction
npm run lint  # Devrait passer
```

**Livrables:**
- [ ] 4 variables renommées
- [ ] ESLint rule `id-length` activée
- [ ] Code review checklist mise à jour

---

## 🔴 PHASE 1: CRITICAL - VIOLATIONS ORIGINALES (33h)

**Objectif:** Valider fonctionnel complet

---

### Task #26: Remplacer tests placeholder Story 5.4 ⚠️

**ID:** 26 (ex-Task #2)
**Effort:** 4h
**Priorité:** CRITICAL - Faux positifs tests

**Problème:**
- 13 tests avec `expect(true).toBe(true)`
- Story 5.4 marquée "done" mais ACs non validés

**TDD Steps:**
1. 🔴 RED: Identifier 13 placeholders
2. 🟢 GREEN: Remplacer par vraies assertions
3. 🔵 REFACTOR: Exécuter tests Story 5.4

**Commandes:**
```bash
# Lister placeholders
grep -n "expect(true).toBe(true)" tests/acceptance/story-5-4.test.ts

# Exécuter après corrections
npx jest --config jest.config.acceptance.js story-5-4.test.ts
```

**Dépend de:** Aucun

---

### Task #27: Remplacer tests placeholder autres stories ⚠️

**ID:** 27 (ex-Task #3)
**Effort:** 2h
**Priorité:** CRITICAL

**Problème:**
- 10 tests placeholder dans 6 fichiers
- Stories 3.1, 1.2, 2.3-2.6

**Dépend de:** Task #26 (pour pattern cohérent)

---

### Task #28: Créer feature files BDD Stories 3.3 et 7.1 ⚠️

**ID:** 28 (ex-Task #6)
**Effort:** 3h
**Priorité:** CRITICAL

**Problème:**
- 2 stories "done" SANS tests BDD
- ACs jamais vérifiés

**Livrables:**
- `story-3-3-visual-distinction.feature`
- `story-7-1-support-mode.feature`
- Step definitions avec vraies assertions

---

### Task #29: Refactor throw → Result pattern (ADR-023) 📐

**ID:** 29 (ex-Task #7)
**Effort:** 8h
**Priorité:** CRITICAL - Architecture

**Problème:**
- 11 fichiers violent ADR-023
- `throw new Error()` au lieu de `Result<T>`

**Fichiers prioritaires:**
1. SyncQueueService.ts
2. FileStorageService.ts
3. LLMModelService.ts (après Task #24 split)
4. TodoRepository.ts
5. user-features.repository.ts
6. useUserFeatures.ts
7-11. Backend services (5 fichiers)

**TDD Steps:**
1. 🔴 RED: Écrire tests attendant Result<T>
2. 🟢 GREEN: Refactor throw → return Result
3. 🔵 REFACTOR: Adapter tous les callers

**Impact:** Conformité ADR-023, error handling monadic

**Dépend de:** Task #24 (si LLMModelService dans scope)

---

### Task #30: Créer tests module Authorization (0% → 60%) 🧪

**ID:** 30 (ex-Task #8)
**Effort:** 16h
**Priorité:** CRITICAL - Sécurité

**Problème:**
- 25+ fichiers critiques SANS tests
- Système permissions non validé

**Scope:**
- 3 services (20+ tests chacun)
- 8 repositories (6-8 tests chacun)
- 3 guards (8-10 tests chacun)

**Métriques cible:**
- ~130 tests créés
- Coverage >= 60%

---

## 🟠 PHASE 2: HIGH PRIORITY (40.5h)

### Task #31: Fix CORS configuration 🔒

**ID:** 31 (ex-Task #9)
**Effort:** 1h
**Priorité:** HIGH - Sécurité

**Problème:**
- Localhost autorisé en production
- Regex trop larges
- credentials: true avec origins larges

**Solution:** Configuration basée sur NODE_ENV

---

### Task #32: Ajouter ValidationPipe global 🔒

**ID:** 32 (ex-Task #10)
**Effort:** 30min
**Priorité:** HIGH - Validation

**Solution:**
```typescript
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
}));
```

---

### Task #33: Fix exceptions NestJS 🔒

**ID:** 33 (ex-Task #11)
**Effort:** 1h
**Priorité:** HIGH

**Problème:**
- `throw new Error()` au lieu d'exceptions NestJS
- HTTP status codes incorrects

**Fichiers:**
- admin-auth.controller.ts → ForbiddenException
- sync.controller.ts → UnauthorizedException

---

### Task #34: Validation DTOs query params 🔒

**ID:** 34 (ex-Task #12)
**Effort:** 2h
**Priorité:** HIGH

**Problème:**
- Query params non validés
- Pas de type checking, injection possible

**Solution:** Créer DTOs avec class-validator

---

### Task #35: Augmenter coverage mobile (32% → 60%) 🧪

**ID:** 35 (ex-Task #13)
**Effort:** 20h
**Priorité:** HIGH - Qualité

**Problème:**
- 66 tests / 206 fichiers = 32%
- Refactoring risqué

**Plan:**
- Phase 1: Capture context (+15 tests, 8h)
- Phase 2: Knowledge context (+10 tests, 4h)
- Phase 3: Action context (+8 tests, 3h)
- Phase 4: Identity context (+6 tests, 2h)
- Phase 5: Normalization context (+10 tests, 3h)

**Total:** +49 tests pour atteindre 60% coverage

---

## 🟡 PHASE 3: MEDIUM PRIORITY (30.5h)

### Tasks #36-#37: Issues MEDIUM variées

**Regroupées pour simplicité:**

**Task #36:** Logging, Retry, Event Bus, Validation (8h)
- Logging structuré (1h)
- Retry logic LLM (2h)
- Event Bus mocks → Redis (3h)
- Validation DTO partielle (1.5h)
- Error codes unifiés (0.5h)

**Task #37:** Tests, Docs, Observabilité, Config (22.5h)
- Tests E2E manquants (8h)
- Documentation API (4h)
- Métriques observabilité (3h)
- Config TypeORM unsafe (1h)
- Hardcoded timeouts (1h)
- Dead code removal (2h)
- Autres issues MEDIUM (3.5h)

---

## 📈 PROGRESSION & TRACKING

### Utilisation des Tasks

**Voir la liste:**
```bash
/tasks
```

**Démarrer une task:**
```bash
TaskUpdate taskId=14 status=in_progress
```

**Compléter une task:**
```bash
TaskUpdate taskId=14 status=completed
```

**Trouver la prochaine task:**
```bash
TaskList
# Prendre la première task "pending" sans blocage
```

---

## 🔄 WORKFLOW TDD STANDARD

Chaque tâche suit ce pattern:

### 1. 🔴 RED - Écrire tests qui échouent
```bash
# Créer test file
touch path/to/__tests__/file.spec.ts

# Écrire tests
# describe(), it(), expect()

# Exécuter (devrait échouer - RED)
npx jest path/to/__tests__/file.spec.ts
```

### 2. 🟢 GREEN - Corriger code
```bash
# Modifier le code source
# Corriger pour faire passer les tests

# Exécuter tests (devrait passer - GREEN)
npx jest path/to/__tests__/file.spec.ts
```

### 3. 🔵 REFACTOR - Améliorer & valider
```bash
# Refactoriser si nécessaire
# Exécuter TOUS les tests affectés
npm run test
npm run test:acceptance
npm run test:e2e

# Vérifier métriques
npm run test:coverage
```

---

## 🎯 CHECKPOINTS CRITIQUES

### Après BLOCKING DoD (13h) - Fin Week 1

**Vérifications:**
- [ ] Aucun console.log en production
- [ ] Aucun type `any`
- [ ] Error messages génériques (pas de leak)
- [ ] Secret JWT validé (throw si manquant)
- [ ] Aucun import `/legacy`
- [ ] 0 npm vulnerabilities
- [ ] 0 tests skipped

**Commandes validation:**
```bash
grep -r "console\." pensieve/mobile/src pensieve/backend/src | wc -l  # = 0
grep -rn ": any" pensieve/backend/src | wc -l  # = 0
grep "JWT_SECRET.*||" pensieve/backend/src | wc -l  # = 0
grep -r "legacy" pensieve/mobile/src/ | wc -l  # = 0
npm audit --audit-level=moderate  # = 0 vulnerabilities
grep -r "\.skip\|\.only" tests/ | wc -l  # = 0
```

**Score attendu:** 5.5/10 (remontée de 4.8)

---

### Après BLOCKING ADR-024 (30.5h) - Fin Week 2-3

**Vérifications:**
- [ ] TODOs: Format `TODO(TICKET-ID):` respecté
- [ ] Aucun code commenté
- [ ] Aucun magic number
- [ ] LLMModelService: < 10 methods, < 300 lines
- [ ] LLMSettingsScreen: < 300 lines
- [ ] Noms variables explicites (> 3 chars)

**Métriques:**
```bash
grep -rn "// TODO" src/ | grep -v "TODO(" | wc -l  # = 0
# Script détection code commenté  # = 0 blocs
npm run lint  # no-magic-numbers pass
wc -l LLMModelService.ts  # < 300
wc -l LLMSettingsScreen.tsx  # < 300
```

**Score attendu:** 6.5/10

---

### Après CRITICAL (33h) - Fin Sprint N+1

**Vérifications:**
- [ ] Aucun test placeholder
- [ ] 2 feature files BDD créés
- [ ] ADR-023: Result pattern dans fichiers prioritaires
- [ ] Authorization: >= 60% coverage

**Métriques:**
```bash
grep -r "expect(true).toBe(true)" tests/ | wc -l  # = 0
ls tests/acceptance/features/story-3-3*.feature  # exists
ls tests/acceptance/features/story-7-1*.feature  # exists
npx jest --coverage src/modules/authorization
# Statements: >= 60%
```

**Score attendu:** 7.5/10

---

### Après HIGH (40.5h) - Fin Sprint N+2 Week 1

**Vérifications:**
- [ ] CORS: environment-based
- [ ] ValidationPipe: global activé
- [ ] Exceptions: NestJS typées
- [ ] Query params: DTOs validés
- [ ] Mobile coverage: >= 60%

**Métriques:**
```bash
# Backend
grep "enableCors" pensieve/backend/src/main.ts  # Check NODE_ENV
grep "useGlobalPipes" pensieve/backend/src/main.ts  # exists
grep -rn "throw new Error" pensieve/backend/src | wc -l  # = 0

# Mobile
cd pensieve/mobile && npm run test:coverage
# Coverage: >= 60%
```

**Score attendu:** 8.0/10

---

### Après MEDIUM (30.5h) - Fin Sprint N+2

**Vérifications:**
- [ ] Logging structuré
- [ ] Tests E2E créés
- [ ] Documentation API complète
- [ ] Métriques observabilité
- [ ] Dead code supprimé

**Score final cible:** **8.5/10** ✅

---

## 🚀 DÉMARRAGE RAPIDE

### 1. Voir toutes les tâches
```bash
/tasks
```

### 2. Démarrer la première tâche BLOCKING DoD
```bash
# Task #14: Console.log removal
TaskUpdate taskId=14 status=in_progress

cd pensieve/backend
# Suivre les steps TDD dans la task description
```

### 3. Compléter et passer à la suivante
```bash
# Marquer terminée
TaskUpdate taskId=14 status=completed

# Voir prochaine task
TaskList
```

---

## 📝 NOTES IMPORTANTES

### Dépendances Tasks

**Tasks dépendantes:**
- Task #27 (placeholders autres) dépend de Task #26 (pattern cohérent)
- Task #29 (Result pattern) optionnellement de Task #24 (si LLMModelService dans scope)

**Toutes les autres tasks sont indépendantes** et peuvent être faites en parallèle.

### Pause & Reprise

**Le système de tasks permet:**
- ✅ De mettre une task en pause (status reste in_progress)
- ✅ De voir où on en était (description complète)
- ✅ De reprendre exactement où on s'est arrêté
- ✅ De tracker la progression globale

**Exemple pause:**
```bash
# Vous êtes sur Task #24 (SRP refactor)
# Vous avez fait LLMModelService split (12h sur 20h)

# Pas besoin de faire quoi que ce soit
# La task reste in_progress

# À la reprise:
TaskList
# Vous voyez Task #24 in_progress
# Relire description pour voir où vous en étiez
```

### Commits Git Recommandés

**Pattern:**
```bash
git commit -m "fix(mobile): remove all console.log statements

- Replace 6 console.log with Logger service
- Add ESLint rule no-console
- Configure logger for DEV/PROD environments
- Closes Task #14

Refs: AUDIT-CODE-ENRICHI-2026-02-15.md (BLOCKING-1)
Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

**Convention:**
- Préfixe: `fix`, `feat`, `refactor`, `test`, `docs`, `chore`
- Scope: `mobile`, `backend`, `web`, `admin`, `shared`
- Référence: Task ID + Section audit

---

## 📚 RESSOURCES

### Documents Référence

- **Audit enrichi:** `_bmad-output/AUDIT-CODE-ENRICHI-2026-02-15.md`
- **Audit v1:** `_bmad-output/AUDIT-CODE-COMPLET-2026-02-15.md`
- **Plan v1:** `_bmad-output/PLAN-CORRECTIONS-AUDIT-TDD.md`
- **Project context:** `_bmad-output/project-context.md`
- **ADR-023:** `planning-artifacts/adrs/ADR-023-error-handling-strategy.md`
- **ADR-024:** `planning-artifacts/adrs/ADR-024-clean-code-standards.md`

### Helpers Existants

- `tests/acceptance/support/test-context.ts` (mocks)
- `src/types/result.types.ts` (Result pattern)
- `.env.example` (variables)

### Commandes Utiles

**Tests:**
```bash
# Mobile
npm run test:unit
npm run test:acceptance
npm run test:e2e
npm run test:coverage

# Backend
npm run test
npm run test:acceptance
npm run test:e2e
npm run test:cov
```

**Recherche:**
```bash
# Trouver tous les throw
grep -rn "throw new Error" src/

# Trouver placeholders
grep -rn "expect(true).toBe(true)" tests/

# Trouver console.log
grep -rn "console\." src/

# Trouver types any
grep -rn ": any" src/

# Trouver TODOs mal formatés
grep -rn "// TODO" src/ | grep -v "TODO("

# Trouver magic numbers
grep -rE "\b[0-9]{3,}\b" src/ | grep -v "test\|spec"
```

**Validation:**
```bash
# Lint
npm run lint

# TypeScript compile
npx tsc --noEmit

# Security audit
npm audit --audit-level=moderate

# Coverage
npm run test:coverage
```

---

## 📊 COMPARAISON V1 vs V2

| Métrique | V1 | V2 | Delta |
|----------|----|----|-------|
| **Total Tasks** | 13 | 23 | +10 |
| **Total Effort** | 69.5h | 147.5h | +78h (+112%) |
| **BLOCKING Issues** | 0 | 12 | +12 |
| **CRITICAL Issues** | 15 | 15 | 0 |
| **HIGH Issues** | 8 | 14 | +6 |
| **MEDIUM Issues** | 11 | 13 | +2 |
| **Score Qualité** | 6.2/10 | 4.8/10 | -1.4 |

**Raison augmentation effort:**
- Ajout standards ADR-024 (Clean Code NON-NÉGOCIABLES)
- Ajout Definition of Done enrichie (Console, npm audit, etc.)
- SRP violations nécessitent refactoring lourd (20h pour Task #24)

**Stratégie:**
- V1 était optimiste, V2 est réaliste
- V2 intègre debt technique révélée par nouveaux standards
- 3 sprints nécessaires vs 2 sprints estimés v1

---

**Créé:** 2026-02-15
**Version:** 2.0 (Enrichi après passe 2 audit)
**Auteur:** Senior Developer (Mode Adversarial)
**Total Tasks:** 23 (vs 13 v1)
**Total Effort:** 147.5 heures (vs 69.5h v1)
**Issues Résolues:** 54 (vs 34 v1)
**Score Cible Final:** 8.5/10

---

_Bon courage pour les corrections ! Suivez le TDD, prenez des pauses, et validez à chaque étape._ ✨

_La qualité n'est pas négociable. Chaque violation corrigée rend le code plus maintenable, sécurisé, et professionnel._ 🚀

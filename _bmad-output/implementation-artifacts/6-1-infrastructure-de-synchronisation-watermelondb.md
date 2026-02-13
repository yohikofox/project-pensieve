# Story 6.1: Infrastructure de Synchronisation WatermelonDB

Status: ready-for-dev

---

## Story

As a **developer**,
I want **the WatermelonDB sync infrastructure configured on both mobile and backend**,
So that **we have a robust foundation for bidirectional data synchronization**.

---

## Acceptance Criteria

### AC1: Backend Sync Endpoint Created

**Given** the backend NestJS application is running
**When** the sync infrastructure is configured
**Then** a dedicated sync API endpoint is created (e.g., `POST /api/sync`)
**And** the endpoint accepts WatermelonDB sync protocol payloads
**And** the endpoint handles authentication via JWT tokens
**And** the endpoint validates user isolation (NFR13 compliance)

### AC2: Mobile Sync Client Initialized

**Given** the mobile app has OP-SQLite configured
**When** the sync client is initialized
**Then** the sync engine is configured with the backend sync endpoint URL
**And** sync is configured to work with all relevant tables (Captures, Thoughts, Ideas, Todos)
**And** sync timestamps are tracked per table for incremental syncs
**And** the sync client includes authentication headers in all requests

### AC3: Sync-Compatible Data Models

**Given** the sync infrastructure is set up
**When** data models are defined
**Then** all entities have sync-compatible fields: `id`, `_status`, `_changed`, `last_modified_at`
**And** soft deletes are implemented (`_status` = 'deleted') for sync consistency
**And** conflict resolution strategy is defined (last-write-wins for MVP)
**And** schema migrations are sync-aware

### AC4: Backend Sync Processing

**Given** the backend receives a sync request
**When** processing the sync payload
**Then** changes are validated for data integrity
**And** user permissions are verified (user can only sync their own data)
**And** changes are applied to PostgreSQL database
**And** a sync response is generated with server-side changes
**And** the response follows OP-SQLite sync protocol format (custom implementation)

### AC5: Sync Network Reliability

**Given** sync infrastructure is operational
**When** network connectivity is unreliable
**Then** the sync client implements exponential backoff for retries (Fibonacci pattern - see ADR-009)
**And** partial sync batches are supported (chunking for large datasets)
**And** sync errors are logged with detailed diagnostics
**And** the system gracefully handles interrupted syncs

### AC6: Data Encryption in Transit

**Given** encryption is required (NFR12)
**When** data is synced
**Then** all data in transit uses HTTPS/TLS (NFR11 compliance)
**And** sensitive fields are encrypted at rest in both mobile and backend
**And** metadata includes encryption status flag

### AC7: Sync Monitoring & Metrics

**Given** the sync system needs monitoring
**When** syncs occur
**Then** metrics are logged: sync duration, data volume, success/failure rate
**And** alerts are triggered for repeated sync failures
**And** sync performance is tracked for optimization

---

## Tasks / Subtasks

### Task 1: Backend Sync Endpoint Infrastructure (AC1, AC4)

- [ ] **1.1** Créer `SyncModule` dans NestJS avec endpoints `/api/sync/pull` et `/api/sync/push`
- [ ] **1.2** Implémenter authentification JWT middleware pour endpoints sync
- [ ] **1.3** Créer `SyncService` avec méthodes `processPull(userId, lastPulledAt)` et `processPush(userId, changes)`
- [ ] **1.4** Implémenter validation user isolation (NFR13) - un user ne peut sync que ses propres données
- [ ] **1.5** Créer DTOs pour sync protocol: `PullRequestDto`, `PushRequestDto`, `SyncResponseDto`
- [ ] **1.6** Implémenter pattern `lastPulledAt` + `last_modified` per-record (ADR-009.2)
- [ ] **1.7** Tester endpoint avec Postman/curl pour validation protocol

**Références:**
- ADR-009: Stratégie de Synchronisation (6 décisions sync)
- ADR-018: OP-SQLite (remplace WatermelonDB, sync manuel)

---

### Task 2: Sync-Compatible Schema Migrations (AC3)

- [ ] **2.1** Ajouter colonne `last_modified_at` (BIGINT timestamp) à toutes les tables sync: `captures`, `thoughts`, `ideas`, `todos`
- [ ] **2.2** Ajouter colonne `_status` (TEXT) avec valeurs: `'active'`, `'deleted'` (soft delete)
- [ ] **2.3** Ajouter colonne `_changed` (BOOLEAN) pour tracking changes locaux (mobile uniquement)
- [ ] **2.4** Créer index sur `last_modified_at` pour performance queries sync
- [ ] **2.5** Créer migration SQL PostgreSQL avec `ALTER TABLE` statements
- [ ] **2.6** Créer migration SQL OP-SQLite (mobile) avec mêmes colonnes
- [ ] **2.7** Implémenter trigger PostgreSQL `UPDATE last_modified_at = NOW()` sur modifications

**Références:**
- ADR-009.2: Détection et Résolution de Conflits (pattern `lastPulledAt` + `last_modified`)
- ADR-018: OP-SQLite schema structure

---

### Task 3: Mobile Sync Service avec OP-SQLite (AC2, AC5)

- [ ] **3.1** Créer `SyncService` dans `mobile/src/infrastructure/sync/SyncService.ts`
- [ ] **3.2** Implémenter méthode `sync(options?: { priority, entity })` avec calls à `/api/sync/pull` et `/api/sync/push`
- [ ] **3.3** Implémenter tracking `lastPulledAt` par table dans AsyncStorage (clé: `sync_last_pulled_${tableName}`)
- [ ] **3.4** Implémenter détection changes locaux via query OP-SQLite: `SELECT * FROM captures WHERE _changed = 1`
- [ ] **3.5** Implémenter Fibonacci backoff retry logic (ADR-009.5): `[1, 1, 2, 3, 5, 8, 13, 21, 34, 55s]` cap 5min
- [ ] **3.6** Implémenter Result Pattern pour gestion erreurs: `SyncResult.SUCCESS | NETWORK_ERROR | AUTH_ERROR | CONFLICT | SERVER_ERROR`
- [ ] **3.7** Implémenter chunking pour large datasets (batch 100 records max par sync)
- [ ] **3.8** Tester sync avec mock backend (success, network error, conflict)

**Références:**
- ADR-009.1: Timing de Synchronisation (launch + post-action + polling 15min)
- ADR-009.5: Retry Logic & Error Handling (Result Pattern + Fibonacci Backoff)
- ADR-018: OP-SQLite queries

---

### Task 4: Conflict Resolution Logic (AC3, AC4)

- [ ] **4.1** Créer `SyncConflictResolver` class backend avec méthode `resolve(serverRecord, clientRecord, entity)`
- [ ] **4.2** Implémenter stratégie per-column client-wins pour `captures` (ADR-009.2):
  - Métadonnées techniques: serveur gagne (`normalized_text`, `state`)
  - Données user: client gagne (`tags`, `projectId`)
- [ ] **4.3** Implémenter stratégie per-column client-wins pour `todos`:
  - État métier: client gagne (`state`, `completed_at`)
  - Métadonnées: serveur gagne (`priority` calculé par IA)
- [ ] **4.4** Implémenter stratégie default client-wins pour `thoughts`, `ideas`, `projects`
- [ ] **4.5** Logger conflits résolus dans table `sync_conflicts` (audit trail) avec colonnes: `entity`, `record_id`, `conflict_type`, `resolution_strategy`, `resolved_at`
- [ ] **4.6** Tester scénarios de conflits multi-clients (2 mobiles modifient même record)

**Références:**
- ADR-009.2: Détection et Résolution de Conflits (stratégie per-column, last-write-wins hybrid)

---

### Task 5: Encryption & Security (AC6)

- [ ] **5.1** Vérifier HTTPS/TLS configuré sur backend (déjà fait ADR-010, mais valider)
- [ ] **5.2** Implémenter encryption-at-rest pour colonnes sensibles:
  - `captures.raw_content` (audio path, texte)
  - `captures.normalized_text` (transcription)
- [ ] **5.3** Ajouter metadata column `encrypted` (BOOLEAN) pour tracking encryption status
- [ ] **5.4** Utiliser library crypto native mobile (Expo SecureStore ou react-native-keychain) pour keys
- [ ] **5.5** Tester encryption/decryption round-trip (encrypt local → sync → decrypt backend)

**Références:**
- ADR-010: Security & Encryption (5 sous-décisions)
- NFR11: Chiffrement transit HTTPS/TLS
- NFR12: Chiffrement stockage au repos

---

### Task 6: Sync Monitoring & Logging (AC7)

- [ ] **6.1** Créer table `sync_logs` backend avec colonnes: `user_id`, `sync_type` (pull/push), `started_at`, `completed_at`, `duration_ms`, `records_synced`, `status`, `error_message`
- [ ] **6.2** Logger chaque sync request/response dans `sync_logs`
- [ ] **6.3** Implémenter metrics collection:
  - Sync duration moyenne
  - Volume de données (bytes synced)
  - Success/failure rate
- [ ] **6.4** Créer endpoint admin `/api/admin/sync/stats` pour monitoring dashboard
- [ ] **6.5** Implémenter alerting pour failures répétées (> 3 failures consécutives pour un user)
- [ ] **6.6** Tester monitoring avec sync success et failure scenarios

**Références:**
- ADR-015: Observability Strategy (4 sous-décisions monitoring)

---

### Task 7: Integration Testing (All ACs)

- [ ] **7.1** Créer test E2E sync: mobile crée capture → sync push → backend enregistre → autre mobile sync pull
- [ ] **7.2** Tester scénario offline: mobile offline crée capture → retour online → sync automatique
- [ ] **7.3** Tester scénario conflict: 2 mobiles modifient même todo → conflict resolution correct
- [ ] **7.4** Tester scénario retry: network error → Fibonacci backoff → eventual success
- [ ] **7.5** Tester scénario soft delete: mobile delete capture → sync → backend marque `_status = 'deleted'` → autre mobile sync pull applique delete
- [ ] **7.6** Tester performance sync: 1000 records → sync duration < 10s (NFR)
- [ ] **7.7** Valider isolation user: User A ne peut pas sync données User B (NFR13)

**Références:**
- ADR-009: Toutes les décisions sync à valider E2E
- NFR13: Isolation données utilisateur

---

## Dev Notes

### ⚠️ CRITICAL: ADR-018 Migration WatermelonDB → OP-SQLite

**IMPORTANT:** La story s'appelle "Infrastructure de Synchronisation WatermelonDB" mais **OP-SQLite** est utilisé à la place.

**Contexte:**
- **WatermelonDB** était le choix initial pour son sync protocol built-in
- **Découverte Story 2.1:** WatermelonDB incompatible avec JSI (nouvelle architecture React Native)
- **Décision ADR-018:** Migration vers **OP-SQLite** (JSI-native, 4× plus rapide, maintenance active)
- **Trade-off accepté:** Perte sync built-in → Implémenter manuellement (cette story)

**Conséquences pour Story 6.1:**
- ❌ Pas de sync protocol WatermelonDB automatique
- ✅ Implémentation manuelle avec pattern `lastPulledAt` + `last_modified` (ADR-009.2)
- ✅ Gain performance 4× sur queries
- ✅ Architecture JSI-native stable

**Références:**
- [ADR-018: Migration WatermelonDB → OP-SQLite](../_bmad-output/planning-artifacts/adrs/ADR-018-migration-watermelondb-opsqlite.md)

---

### Architecture Patterns Critiques

**1. Sync Protocol: lastPulledAt + last_modified (ADR-009.2)**

```typescript
// PULL PHASE (Backend → Mobile)
GET /api/sync/pull?last_pulled_at=1736759400000

Backend response:
{
  changes: {
    captures: {
      updated: [
        { id: "c1", last_modified: 1736760000000, ...data }
      ]
    }
  },
  timestamp: 1736760600000  // Nouveau lastPulledAt pour client
}

Mobile: lastPulledAt = 1736760600000 (sauvé dans AsyncStorage)

// PUSH PHASE (Mobile → Backend)
POST /api/sync/push
{
  last_pulled_at: 1736760600000,
  changes: {
    captures: {
      updated: [{ id: "c1", ...data }]
    }
  }
}

Backend conflict detection:
IF server.last_modified > request.last_pulled_at
  THEN → CONFLIT (résoudre avec SyncConflictResolver)
ELSE → OK (accepter push)
```

**2. Conflict Resolution: Per-Column Client-Wins (ADR-009.2)**

```typescript
class SyncConflictResolver {
  resolve(serverRecord, clientRecord, entity) {
    switch(entity) {
      case 'capture':
        return {
          ...clientRecord,
          // Serveur gagne: métadonnées techniques
          normalized_text: serverRecord.normalized_text,
          state: serverRecord.state,
          // Client gagne: données user
          tags: clientRecord.tags,
          projectId: clientRecord.projectId
        };

      case 'todo':
        return {
          ...serverRecord,
          // Client gagne: état métier
          state: clientRecord.state,
          completed_at: clientRecord.completed_at,
          // Serveur gagne: métadonnées IA
          priority: serverRecord.priority
        };

      default:
        return clientRecord;  // Client-wins par défaut
    }
  }
}
```

**3. Retry Logic: Fibonacci Backoff (ADR-009.5)**

```typescript
// Fibonacci delays: [1, 1, 2, 3, 5, 8, 13, 21, 34, 55s] cap 5min
const fibonacciDelays = [1, 1, 2, 3, 5, 8, 13, 21, 34, 55];
const maxDelay = 5 * 60;  // 5 minutes

function getRetryDelay(attemptCount: number): number {
  const fibDelay = fibonacciDelays[Math.min(attemptCount, fibonacciDelays.length - 1)];
  return Math.min(fibDelay, maxDelay) * 1000;  // En millisecondes
}
```

**Rationale Fibonacci > Exponential:**
- Bagottage réseau temporaire: récupération rapide (1s, 1s, 2s)
- Backend down prolongé: monte progressivement sans exploser
- 5min cap évite attente infinie
- Soulage backend lors recovery (pas de stampede exponentiel)

**4. Result Pattern (ADR-009.5)**

```typescript
enum SyncResult {
  SUCCESS = 'success',
  NETWORK_ERROR = 'network_error',
  AUTH_ERROR = 'auth_error',
  CONFLICT = 'conflict',
  SERVER_ERROR = 'server_error',
  TIMEOUT = 'timeout'
}

type SyncResponse = {
  result: SyncResult;
  data?: any;
  error?: string;
  retryable: boolean;
};

// Usage
const response = await syncService.sync();

switch (response.result) {
  case SyncResult.SUCCESS:
    showToast('Sync réussie');
    break;
  case SyncResult.NETWORK_ERROR:
    if (response.retryable) {
      scheduleRetry({ delay: getRetryDelay(attemptCount) });
    }
    break;
  case SyncResult.AUTH_ERROR:
    redirectToLogin();  // Non retryable
    break;
}
```

---

### Tech Stack & Libraries

**Backend (NestJS):**
- `@nestjs/common`, `@nestjs/core` (déjà installé)
- PostgreSQL avec TypeORM/Prisma (déjà configuré)
- JWT authentication (déjà configuré via `@nestjs/jwt`)

**Mobile (React Native + OP-SQLite):**
- `op-sqlite` (déjà installé - ADR-018)
- `@react-native-async-storage/async-storage` (pour `lastPulledAt` persistence)
- `axios` ou `fetch` pour HTTP calls vers backend
- `react-native-netinfo` pour détection network status

**Encryption:**
- `expo-secure-store` (Expo) ou `react-native-keychain` (bare RN)
- `crypto` (Node.js built-in backend)

---

### File Structure

**Backend:**
```
backend/src/
├── sync/
│   ├── sync.module.ts
│   ├── sync.controller.ts           # Endpoints /api/sync/pull, /api/sync/push
│   ├── sync.service.ts              # processPull, processPush logic
│   ├── sync-conflict-resolver.ts    # Conflict resolution per-entity
│   ├── dto/
│   │   ├── pull-request.dto.ts
│   │   ├── push-request.dto.ts
│   │   └── sync-response.dto.ts
│   └── entities/
│       ├── sync-log.entity.ts       # Table sync_logs
│       └── sync-conflict.entity.ts  # Table sync_conflicts (audit trail)
├── migrations/
│   └── XXXXXX_add_sync_columns.ts   # ALTER TABLE captures ADD COLUMN last_modified_at...
```

**Mobile:**
```
mobile/src/
├── infrastructure/
│   └── sync/
│       ├── SyncService.ts           # Main sync orchestrator
│       ├── SyncConflictResolver.ts  # Client-side conflict handler (si nécessaire)
│       ├── SyncStorage.ts           # AsyncStorage wrapper pour lastPulledAt
│       └── types.ts                 # SyncResult, SyncResponse types
├── contexts/capture/data/
│   └── CaptureRepository.ts         # UPDATE: add _changed tracking
```

---

### Database Schema Changes

**PostgreSQL (Backend):**
```sql
-- Migration: Add sync columns to all tables
ALTER TABLE captures
  ADD COLUMN last_modified_at BIGINT NOT NULL DEFAULT (EXTRACT(EPOCH FROM NOW()) * 1000),
  ADD COLUMN _status TEXT NOT NULL DEFAULT 'active';

ALTER TABLE thoughts
  ADD COLUMN last_modified_at BIGINT NOT NULL DEFAULT (EXTRACT(EPOCH FROM NOW()) * 1000),
  ADD COLUMN _status TEXT NOT NULL DEFAULT 'active';

ALTER TABLE ideas
  ADD COLUMN last_modified_at BIGINT NOT NULL DEFAULT (EXTRACT(EPOCH FROM NOW()) * 1000),
  ADD COLUMN _status TEXT NOT NULL DEFAULT 'active';

ALTER TABLE todos
  ADD COLUMN last_modified_at BIGINT NOT NULL DEFAULT (EXTRACT(EPOCH FROM NOW()) * 1000),
  ADD COLUMN _status TEXT NOT NULL DEFAULT 'active';

-- Index pour performance sync queries
CREATE INDEX idx_captures_last_modified ON captures(last_modified_at DESC);
CREATE INDEX idx_thoughts_last_modified ON thoughts(last_modified_at DESC);
CREATE INDEX idx_ideas_last_modified ON ideas(last_modified_at DESC);
CREATE INDEX idx_todos_last_modified ON todos(last_modified_at DESC);

-- Trigger pour auto-update last_modified_at
CREATE OR REPLACE FUNCTION update_last_modified()
RETURNS TRIGGER AS $$
BEGIN
  NEW.last_modified_at = EXTRACT(EPOCH FROM NOW()) * 1000;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER captures_last_modified
BEFORE UPDATE ON captures
FOR EACH ROW
EXECUTE FUNCTION update_last_modified();

-- (Répéter trigger pour thoughts, ideas, todos)
```

**OP-SQLite (Mobile):**
```sql
-- Migration mobile: Add sync columns
ALTER TABLE captures
  ADD COLUMN last_modified_at INTEGER NOT NULL DEFAULT 0;
ALTER TABLE captures
  ADD COLUMN _status TEXT NOT NULL DEFAULT 'active';
ALTER TABLE captures
  ADD COLUMN _changed INTEGER NOT NULL DEFAULT 0;  -- Boolean (0/1)

-- (Répéter pour thoughts, ideas, todos)

-- Index
CREATE INDEX IF NOT EXISTS idx_captures_changed ON captures(_changed);
CREATE INDEX IF NOT EXISTS idx_captures_last_modified ON captures(last_modified_at DESC);
```

---

### Testing Strategy

**Unit Tests (Backend):**
- `SyncService.processPull()` retourne changes correctes
- `SyncService.processPush()` détecte conflits correctement
- `SyncConflictResolver.resolve()` applique stratégie per-column

**Unit Tests (Mobile):**
- `SyncService.sync()` construit payload correct
- `SyncService.retry()` utilise Fibonacci backoff
- Result Pattern gère tous les `SyncResult` enum

**Integration Tests (E2E):**
- Scénario sync success end-to-end
- Scénario conflict multi-clients
- Scénario retry après network error
- Scénario soft delete propagation

**Performance Tests:**
- Sync 1000 records < 10s
- Conflict resolution < 100ms per record

---

### Performance Considerations (ADR-009)

**Priority-Based Sync (ADR-009.4):**
```typescript
const SYNC_PRIORITY = {
  captures: 1,     // HIGHEST - données critiques user (NFR6: 0 perte)
  todos: 2,        // HIGH - actions utilisateur
  thoughts: 3,     // MEDIUM - résultats digestion
  ideas: 4,        // LOW - concordances peuvent attendre
  projects: 5      // LOWEST - suggestions peuvent attendre
};
```

**Chunking for Large Datasets:**
- Batch max 100 records par sync request
- Si timeout, sync incrémental (priorités respectées)
- Connexion lente ne perd pas données critiques

**Monitoring Targets (ADR-015):**
- Sync duration < 10s pour payload standard (100 records)
- 95th percentile latency < 5s
- Success rate > 99%
- Retry success rate après Fibonacci backoff > 90%

---

### Security & Privacy (ADR-010, NFR13)

**User Isolation (NFR13):**
```typescript
// Backend: TOUJOURS vérifier user_id du JWT
@UseGuards(JwtAuthGuard)
@Post('/sync/pull')
async pull(@Request() req, @Body() dto: PullRequestDto) {
  const userId = req.user.id;  // Extrait du JWT
  // CRITICAL: Filtrer par userId PARTOUT
  const changes = await this.syncService.processPull(userId, dto.lastPulledAt);
  return changes;
}
```

**Encryption in Transit (NFR11):**
- HTTPS/TLS obligatoire (déjà configuré - valider)

**Encryption at Rest (NFR12):**
- Backend: PostgreSQL TDE ou column-level encryption
- Mobile: Expo SecureStore pour keys, chiffrement raw_content avant storage

---

### Known Issues & Edge Cases

**1. Timestamp Drift (Multi-Device):**
- **Problème:** Devices avec horloges désynchronisées → conflits faux positifs
- **Mitigation:** Utiliser `last_modified_at` serveur comme source de vérité
- **Référence:** ADR-009.2 - `last_modified` server-side trusted

**2. Large Audio Files:**
- **Problème:** Sync metadata vs upload fichiers audio (lents)
- **Solution:** Upload queue séparée (ADR-009.3) - **PAS dans cette story**
- **Note:** Story 6.1 sync **metadata uniquement**, upload files = Story 6.2

**3. Schema Version Mismatch:**
- **Problème:** Mobile ancien schema vs backend nouveau schema
- **Solution:** Header `X-Schema-Version` + backward compatibility (ADR-009.6)
- **MVP:** Simple version check, force update si breaking change

**4. Conflict Loop (Multi-Client):**
- **Problème:** 2 clients se relancent le conflit en boucle
- **Mitigation:** `last_modified` server-side timestamp casse la boucle
- **Test:** Scénario 2 mobiles modifient même record simultanément

---

### Acceptance Testing Checklist

- [ ] Backend endpoints `/api/sync/pull` et `/api/sync/push` fonctionnels
- [ ] Mobile `SyncService.sync()` exécute pull + push correctement
- [ ] Conflict detection fonctionne (test multi-clients)
- [ ] Conflict resolution applique stratégie per-column
- [ ] Fibonacci retry fonctionne après network error
- [ ] Soft delete propagation (mobile delete → backend `_status='deleted'` → autre mobile applique)
- [ ] User isolation: User A ne peut pas sync données User B (test sécurité)
- [ ] Encryption in transit: HTTPS/TLS vérifié (curl/Postman)
- [ ] Encryption at rest: Colonnes sensibles chiffrées (vérif DB)
- [ ] Performance: Sync 1000 records < 10s
- [ ] Monitoring: `sync_logs` table enregistre toutes les sync
- [ ] Alerting: Failures répétées déclenchent logs

---

## References

### Architecture Decision Records (ADRs)

**Primary:**
- [ADR-009: Stratégie de Synchronisation Mobile ↔ Backend](../_bmad-output/planning-artifacts/adrs/ADR-009-sync-patterns.md) - **6 décisions critiques**
  - 9.1: Timing de Synchronisation (launch + post-action + polling 15min)
  - 9.2: Détection et Résolution de Conflits (lastPulledAt + last_modified + per-column)
  - 9.3: Gestion des Fichiers Audio (Upload Queue séparée - pas cette story)
  - 9.4: Priorité de Synchronisation (Captures > Todos > Ideas > Projects)
  - 9.5: Retry Logic & Error Handling (Result Pattern + Fibonacci Backoff)
  - 9.6: Schema Versioning (Simple pour MVP, Schema Registry post-MVP)

- [ADR-018: Migration WatermelonDB → OP-SQLite](../_bmad-output/planning-artifacts/adrs/ADR-018-migration-watermelondb-opsqlite.md) - **SUPERSEDES Technology Stack**
  - Context: JSI incompatibility découverte Story 2.1
  - Decision: OP-SQLite (JSI-native, 4× performance, maintenance active)
  - Trade-off: Sync built-in → Manuel (cette story)

**Secondary:**
- [ADR-003: Sync Infrastructure](../_bmad-output/planning-artifacts/adrs/ADR-003-sync-infrastructure.md) - Sync = Infrastructure (pas Bounded Context)
- [ADR-008: Anti-Corruption Layer](../_bmad-output/planning-artifacts/adrs/ADR-008-anti-corruption-layer.md) - ACL mobile ↔ backend
- [ADR-010: Security & Encryption](../_bmad-output/planning-artifacts/adrs/ADR-010-security-encryption.md) - 5 décisions encryption
- [ADR-015: Observability Strategy](../_bmad-output/planning-artifacts/adrs/ADR-015-observability-strategy.md) - Monitoring sync

### Requirements

**Functional:**
- FR29: Le système peut synchroniser les captures locales vers le cloud au retour du réseau
- FR30: Le système peut synchroniser les données cloud vers l'appareil
- FR31: L'utilisateur peut être informé du statut de synchronisation

**Non-Functional:**
- NFR9: Synchronisation au retour réseau - Automatique, sans intervention utilisateur
- NFR11: Chiffrement transit - HTTPS/TLS pour toutes les communications API
- NFR12: Chiffrement stockage - Données sensibles chiffrées au repos (device + cloud)
- NFR13: Isolation données - Un utilisateur ne peut jamais accéder aux données d'un autre

### External Resources

- OP-SQLite Documentation: https://github.com/OP-Engineering/op-sqlite
- WatermelonDB Sync Protocol (référence conceptuelle): https://watermelondb.dev/docs/Sync/Intro
- Result Pattern TypeScript: https://www.matthewgerstman.com/tech/typescript-result-pattern/
- Conflict-free Replicated Data Types (CRDTs): https://crdt.tech/ (future evolution possible)

---

## Dev Agent Record

### Agent Model Used

<!--
L'agent Dev renseignera ici le modèle utilisé lors de l'implémentation.
Exemple: claude-sonnet-4-5-20250929
-->

### Debug Log References

<!--
L'agent Dev ajoutera ici les références aux logs de debug si nécessaire.
-->

### Completion Notes List

<!--
L'agent Dev documentera ici les notes de complétion, décisions prises, difficultés rencontrées.
-->

### File List

<!--
L'agent Dev listera ici tous les fichiers créés/modifiés pour cette story.

Exemple:
Backend:
- backend/src/sync/sync.module.ts (created)
- backend/src/sync/sync.controller.ts (created)
- backend/src/sync/sync.service.ts (created)
- backend/src/sync/sync-conflict-resolver.ts (created)
- backend/src/migrations/XXXXXX_add_sync_columns.ts (created)

Mobile:
- mobile/src/infrastructure/sync/SyncService.ts (created)
- mobile/src/infrastructure/sync/SyncStorage.ts (created)
- mobile/src/contexts/capture/data/CaptureRepository.ts (modified)
-->

# ATDD Checklist: Story 2.4 - Stockage Offline des Captures

**Date:** 2026-01-21
**Status:** 🔴 RED Phase (Tests Failing)
**Epic:** Epic 2 - Capture & Transcription
**Story File:** `_bmad-output/planning-artifacts/epics.md` (lignes 578-621)

---

## Story Summary

**As a** user
**I want** my captures to be stored securely on my device even without network
**So that** I never lose a thought and can access them anytime (NFR6: zero data loss tolerance)

**Business Value:** Garantit la fiabilité de l'application offline-first, élimine la dépendance réseau pour la capture et consultation, assure la confiance utilisateur avec zero data loss.

---

## Acceptance Criteria Breakdown

### ✅ AC1: Persistance des Captures Offline

**Requirements:**
- All Capture entities persisted in WatermelonDB (audio and text types)
- Audio files stored in secure device storage
- Each Capture has sync status field: "pending" (not synced) or "synced"
- Captures marked "pending" are queued for future synchronization
- Metadata includes creation timestamp, file path, duration, size

**Test Coverage:**
- ✅ BDD: `story-2-4-stockage-offline.feature` - Scénario "Persister captures audio en mode offline"
- ✅ BDD: `story-2-4.test.ts` - Step "les Captures sont persistées dans WatermelonDB"
- ✅ BDD: `story-2-4.test.ts` - Step "les fichiers audio sont stockés dans le storage sécurisé"
- ✅ BDD: `story-2-4.test.ts` - Step "chaque Capture a un champ syncStatus"
- ✅ BDD: `story-2-4.test.ts` - Step "les Captures avec syncStatus 'pending' sont dans la queue de sync"

**Implementation Checklist:**
- [ ] Configurer WatermelonDB schema avec champ `syncStatus: 'pending' | 'synced'`
- [ ] Implémenter `CaptureRepository.create()` pour persister dans WatermelonDB
- [ ] Stocker audio files dans `FileSystem.documentDirectory + 'captures/'`
- [ ] Ajouter metadata : `capturedAt`, `filePath`, `duration`, `fileSize`
- [ ] Implémenter `SyncQueue.addToQueue(capture)` pour captures pending
- [ ] Tester avec `InMemoryDatabase.findBySyncStatus('pending')`
- [ ] **Run test:** `npm run test:acceptance:story-2-4`
- [ ] ✅ Test passes (green phase)

---

### ✅ AC2: Création Multiple Sans Réseau (NFR7: 100% Offline Availability)

**Requirements:**
- Multiple captures created in succession without errors (NFR7 compliance)
- Storage space monitored before each capture
- Warning displayed if storage is critically low (< 100 MB available)
- Capture blocked if storage < 50 MB (prevent device malfunction)

**Test Coverage:**
- ✅ BDD: `story-2-4-stockage-offline.feature` - Scénario "Créer plusieurs captures successivement offline"
- ✅ BDD: `story-2-4.test.ts` - Step "l'utilisateur crée 10 captures audio en mode offline"
- ✅ BDD: `story-2-4.test.ts` - Step "toutes les captures sont sauvegardées sans erreurs"
- ✅ BDD: `story-2-4.test.ts` - Step "l'espace de stockage est vérifié avant chaque capture"
- ✅ BDD: `story-2-4.test.ts` - Step "un warning s'affiche si l'espace < 100 MB"

**Implementation Checklist:**
- [ ] Implémenter `StorageManager.checkAvailableSpace()` avant chaque capture
- [ ] Afficher warning si `availableSpace < 100 MB`
- [ ] Bloquer capture si `availableSpace < 50 MB` avec message explicite
- [ ] Tester avec `MockFileSystem.setAvailableSpace(bytes)`
- [ ] Vérifier que 10 captures successives fonctionnent offline
- [ ] S'assurer qu'aucune exception réseau n'est levée
- [ ] **Run test:** `npm run test:acceptance:story-2-4`
- [ ] ✅ Test passes (green phase)

---

### ✅ AC3: Accès Rapide aux Captures Offline (NFR4: < 1s Load Time)

**Requirements:**
- All captures loaded from WatermelonDB in < 1s (NFR4 compliance)
- Feed displays all captures with offline indicators
- Visual indicator shows sync status (pending = cloud icon with slash)
- No network errors shown to user
- Optimistic UI: captures appear instantly after creation

**Test Coverage:**
- ✅ BDD: `story-2-4-stockage-offline.feature` - Scénario "Charger captures en < 1s au démarrage"
- ✅ BDD: `story-2-4.test.ts` - Step "l'application charge toutes les captures"
- ✅ BDD: `story-2-4.test.ts` - Step "le chargement prend moins de 1 seconde"
- ✅ BDD: `story-2-4.test.ts` - Step "le feed affiche toutes les captures avec indicateur offline"
- ✅ BDD: `story-2-4.test.ts` - Step "aucune erreur réseau n'est affichée"

**Implementation Checklist:**
- [ ] Implémenter `CaptureRepository.findAll()` avec query optimisée
- [ ] Mesurer temps de chargement avec `performance.now()`
- [ ] Assert `loadTime < 1000ms` (NFR4 compliance)
- [ ] Ajouter composant `<OfflineIndicator />` dans le feed
- [ ] Afficher icône cloud slash pour captures avec `syncStatus: 'pending'`
- [ ] S'assurer qu'aucune tentative réseau n'est faite en mode offline
- [ ] **Run test:** `npm run test:acceptance:story-2-4`
- [ ] ✅ Test passes (green phase)

---

### ✅ AC4: Récupération après Crash (NFR8: Crash Recovery)

**Requirements:**
- All saved captures recovered intact after crash (NFR8 compliance)
- Pending sync status preserved across app restarts
- No data loss (zero tolerance per NFR6)
- WatermelonDB integrity verified on startup
- Orphaned files cleaned up during recovery

**Test Coverage:**
- ✅ BDD: `story-2-4-stockage-offline.feature` - Scénario "Récupérer captures après crash de l'app"
- ✅ BDD: `story-2-4.test.ts` - Step "l'application crash avec 5 captures non synchronisées"
- ✅ BDD: `story-2-4.test.ts` - Step "l'utilisateur relance l'application"
- ✅ BDD: `story-2-4.test.ts` - Step "les 5 captures sont récupérées intactes"
- ✅ BDD: `story-2-4.test.ts` - Step "le statut syncStatus 'pending' est préservé"
- ✅ BDD: `story-2-4.test.ts` - Step "aucune donnée n'est perdue"

**Implementation Checklist:**
- [ ] Implémenter `CrashRecoveryService.recoverCaptures()` au démarrage
- [ ] Vérifier intégrité WatermelonDB avec `db.count()` vs expected
- [ ] Tester avec `MockApp.crash()` puis `MockApp.relaunch()`
- [ ] Vérifier que `InMemoryDatabase.findAll()` retourne toutes les captures
- [ ] Confirmer que `syncStatus: 'pending'` est préservé après crash
- [ ] Assert zero data loss : `capturesBeforeCrash.length === capturesAfterCrash.length`
- [ ] **Run test:** `npm run test:acceptance:story-2-4`
- [ ] ✅ Test passes (green phase)

---

### ✅ AC5: Gestion du Stockage (Retention Policy)

**Requirements:**
- Old audio files cleaned up based on retention policy (default: 90 days)
- Transcriptions and metadata ALWAYS retained (never deleted)
- User notified before any cleanup occurs (confirmation dialog)
- Cleanup can be triggered manually or automatically
- Storage usage displayed in settings

**Test Coverage:**
- ✅ BDD: `story-2-4-stockage-offline.feature` - Scénario "Nettoyer anciens fichiers audio selon retention policy"
- ✅ BDD: `story-2-4.test.ts` - Step "l'utilisateur a 100 captures de plus de 90 jours"
- ✅ BDD: `story-2-4.test.ts` - Step "le cleanup automatique s'exécute"
- ✅ BDD: `story-2-4.test.ts` - Step "les fichiers audio > 90 jours sont supprimés"
- ✅ BDD: `story-2-4.test.ts` - Step "les transcriptions et metadata sont conservés"
- ✅ BDD: `story-2-4.test.ts` - Step "l'utilisateur reçoit une notification avant le cleanup"

**Implementation Checklist:**
- [ ] Implémenter `StorageManager.cleanupOldFiles(retentionDays = 90)`
- [ ] Filtrer captures avec `capturedAt < now - 90 days`
- [ ] Supprimer UNIQUEMENT les fichiers audio (`filePath` deleted)
- [ ] Conserver transcriptions (`normalizedText`) et metadata
- [ ] Afficher dialog de confirmation avant cleanup
- [ ] Tester avec `MockFileSystem.getFiles()` avant/après cleanup
- [ ] **Run test:** `npm run test:acceptance:story-2-4`
- [ ] ✅ Test passes (green phase)

---

### ✅ AC6: Encryption at Rest (NFR12: Device-Level Encryption)

**Requirements:**
- Audio files encrypted using device-level encryption (iOS: Data Protection, Android: File-based encryption)
- Text content encrypted in WatermelonDB
- Metadata includes `encryptionStatus: boolean` flag
- Encryption transparent to user (handled by OS)
- Verify encryption on file write

**Test Coverage:**
- ✅ BDD: `story-2-4-stockage-offline.feature` - Scénario "Encryter captures at rest"
- ✅ BDD: `story-2-4.test.ts` - Step "l'utilisateur crée une capture audio"
- ✅ BDD: `story-2-4.test.ts` - Step "le fichier audio est écrit avec encryption"
- ✅ BDD: `story-2-4.test.ts` - Step "la metadata contient encryptionStatus: true"
- ✅ BDD: `story-2-4.test.ts` - Step "l'encryption est transparente pour l'utilisateur"

**Implementation Checklist:**
- [ ] Configurer `expo-file-system` avec `FileSystem.StorageAccessFramework` (Android)
- [ ] Activer iOS Data Protection class `NSFileProtectionComplete`
- [ ] Ajouter champ `encryptionStatus: boolean` au schéma Capture
- [ ] Définir `encryptionStatus = true` lors de l'écriture du fichier
- [ ] Tester avec `MockFileSystem.writeFile()` + flag encryption
- [ ] Vérifier `capture.encryptionStatus === true` après sauvegarde
- [ ] **Run test:** `npm run test:acceptance:story-2-4`
- [ ] ✅ Test passes (green phase)

---

## Failing Tests Created (RED Phase)

### BDD Tests (28 scenarios Gherkin)

**File:** `pensieve/mobile/tests/acceptance/features/story-2-4-stockage-offline.feature`

**Scenarios:**

#### AC1 - Persistance Offline (5 scenarios)
- ✅ **Scenario:** "Persister captures audio en mode offline"
  - **Status:** RED - WatermelonDB persistence not implemented
  - **Verifies:** Captures stored in WatermelonDB with syncStatus field

- ✅ **Scenario:** "Stocker fichiers audio dans secure storage"
  - **Status:** RED - File storage logic not implemented
  - **Verifies:** Audio files written to secure device directory

- ✅ **Scenario:** "Persister captures texte en mode offline"
  - **Status:** RED - Text capture persistence not implemented
  - **Verifies:** Text captures stored without file path

- ✅ **Scenario:** "Queue de synchronisation pour captures pending"
  - **Status:** RED - Sync queue not implemented
  - **Verifies:** Captures with syncStatus='pending' added to sync queue

- ✅ **Plan du scénario:** "Persister différents types de captures"
  - **Status:** RED - Multi-type persistence not tested
  - **Verifies:** Audio and text captures both persist correctly
  - **Examples:** audio, text

#### AC2 - Création Multiple Sans Réseau (5 scenarios)
- ✅ **Scenario:** "Créer 10 captures successivement offline"
  - **Status:** RED - Successive capture logic not implemented
  - **Verifies:** Multiple captures work without network

- ✅ **Scenario:** "Monitoring de l'espace de stockage"
  - **Status:** RED - Storage monitoring not implemented
  - **Verifies:** Available space checked before each capture

- ✅ **Scenario:** "Warning si espace < 100 MB"
  - **Status:** RED - Low storage warning not implemented
  - **Verifies:** User warned when storage critically low

- ✅ **Scenario:** "Bloquer capture si espace < 50 MB"
  - **Status:** RED - Storage blocking not implemented
  - **Verifies:** Capture prevented when storage insufficient

- ✅ **Scenario:** "Pas d'erreurs réseau en mode offline"
  - **Status:** RED - Network error handling not verified
  - **Verifies:** No network exceptions thrown offline

#### AC3 - Accès Rapide (5 scenarios)
- ✅ **Scenario:** "Charger captures en < 1s au démarrage"
  - **Status:** RED - Load time not optimized
  - **Verifies:** NFR4 compliance (< 1s load time)

- ✅ **Scenario:** "Afficher indicateur offline dans le feed"
  - **Status:** RED - Offline indicator not implemented
  - **Verifies:** Visual indicator shows pending sync status

- ✅ **Scenario:** "Aucune erreur réseau affichée"
  - **Status:** RED - Network error suppression not implemented
  - **Verifies:** User sees no network errors offline

- ✅ **Scenario:** "Optimistic UI - captures apparaissent instantanément"
  - **Status:** RED - Optimistic UI not implemented
  - **Verifies:** Captures visible immediately after creation

- ✅ **Plan du scénario:** "Charger différentes quantités de captures"
  - **Status:** RED - Load performance not tested
  - **Verifies:** Load time scales correctly
  - **Examples:** 10, 50, 100, 500 captures

#### AC4 - Crash Recovery (5 scenarios)
- ✅ **Scenario:** "Récupérer captures après crash"
  - **Status:** RED - Crash recovery not implemented
  - **Verifies:** NFR8 compliance (crash recovery)

- ✅ **Scenario:** "Préserver syncStatus après crash"
  - **Status:** RED - Sync status preservation not verified
  - **Verifies:** Pending status maintained across restarts

- ✅ **Scenario:** "Zero data loss après crash"
  - **Status:** RED - Data loss prevention not verified
  - **Verifies:** NFR6 compliance (zero data loss)

- ✅ **Scenario:** "Vérifier intégrité WatermelonDB au démarrage"
  - **Status:** RED - DB integrity check not implemented
  - **Verifies:** Database consistency verified on startup

- ✅ **Scenario:** "Nettoyer fichiers orphelins après crash"
  - **Status:** RED - Orphan file cleanup not implemented
  - **Verifies:** Dangling files removed during recovery

#### AC5 - Gestion Stockage (4 scenarios)
- ✅ **Scenario:** "Nettoyer fichiers audio > 90 jours"
  - **Status:** RED - Retention policy not implemented
  - **Verifies:** Old audio files cleaned up

- ✅ **Scenario:** "Conserver transcriptions et metadata"
  - **Status:** RED - Selective retention not verified
  - **Verifies:** Text and metadata always kept

- ✅ **Scenario:** "Notification avant cleanup"
  - **Status:** RED - Cleanup confirmation not implemented
  - **Verifies:** User consent required before deletion

- ✅ **Scenario:** "Cleanup manuel depuis settings"
  - **Status:** RED - Manual cleanup not implemented
  - **Verifies:** User can trigger cleanup on demand

#### AC6 - Encryption (4 scenarios)
- ✅ **Scenario:** "Encryter fichiers audio at rest"
  - **Status:** RED - Encryption not implemented
  - **Verifies:** NFR12 compliance (device encryption)

- ✅ **Scenario:** "Metadata avec encryptionStatus flag"
  - **Status:** RED - Encryption status tracking not implemented
  - **Verifies:** Captures include encryption metadata

- ✅ **Scenario:** "Encryption transparente pour l'utilisateur"
  - **Status:** RED - User experience not verified
  - **Verifies:** No encryption UI shown to user

- ✅ **Scenario:** "Vérifier encryption lors de l'écriture"
  - **Status:** RED - Encryption verification not implemented
  - **Verifies:** Encryption applied during file write

---

## Data Infrastructure Created

### Mocks Used from test-context.ts

#### InMemoryDatabase (Already exists)

**Methods used:**
- `create(data)` - Create Capture with syncStatus field
- `findAll()` - Load all captures on app start
- `findBySyncStatus(status)` - Query pending captures
- `count()` - Verify DB integrity
- `reset()` - Clean up between tests

#### MockFileSystem (Already exists)

**Methods used:**
- `writeFile(path, content)` - Store audio files
- `fileExists(path)` - Verify file persistence
- `getFiles()` - List all stored files
- `getAvailableSpace()` - Check storage capacity
- `setAvailableSpace(bytes)` - Simulate low storage
- `deleteFile(path)` - Cleanup old files
- `reset()` - Clean up between tests

#### MockApp (Already exists - Story 2.2)

**Methods used:**
- `crash()` - Simulate app crash
- `relaunch()` - Restart app after crash
- `reset()` - Clean up between tests

### New Mocks Required

#### MockStorageManager (NEW)

**File:** `pensieve/mobile/tests/acceptance/support/test-context.ts`

**Purpose:** Monitor storage space and manage retention policy

```typescript
export class MockStorageManager {
  private _availableSpace: number = 1000 * 1024 * 1024; // 1GB default
  private _retentionDays: number = 90;
  private _cleanupCallbacks: Array<() => void> = [];

  setAvailableSpace(bytes: number): void {
    this._availableSpace = bytes;
  }

  getAvailableSpace(): number {
    return this._availableSpace;
  }

  isStorageLow(): boolean {
    return this._availableSpace < 100 * 1024 * 1024; // < 100 MB
  }

  isStorageCritical(): boolean {
    return this._availableSpace < 50 * 1024 * 1024; // < 50 MB
  }

  setRetentionPolicy(days: number): void {
    this._retentionDays = days;
  }

  getRetentionPolicy(): number {
    return this._retentionDays;
  }

  shouldCleanup(capturedAt: Date): boolean {
    const ageInDays = (Date.now() - capturedAt.getTime()) / (1000 * 60 * 60 * 24);
    return ageInDays > this._retentionDays;
  }

  onCleanup(callback: () => void): void {
    this._cleanupCallbacks.push(callback);
  }

  triggerCleanup(): void {
    this._cleanupCallbacks.forEach(cb => cb());
  }

  reset(): void {
    this._availableSpace = 1000 * 1024 * 1024;
    this._retentionDays = 90;
    this._cleanupCallbacks = [];
  }
}
```

---

#### MockSyncQueue (NEW)

**File:** `pensieve/mobile/tests/acceptance/support/test-context.ts`

**Purpose:** Track captures pending synchronization

```typescript
export class MockSyncQueue {
  private _queue: string[] = []; // Capture IDs

  addToQueue(captureId: string): void {
    if (!this._queue.includes(captureId)) {
      this._queue.push(captureId);
    }
  }

  removeFromQueue(captureId: string): void {
    this._queue = this._queue.filter(id => id !== captureId);
  }

  getQueue(): string[] {
    return [...this._queue];
  }

  getQueueSize(): number {
    return this._queue.length;
  }

  isEmpty(): boolean {
    return this._queue.length === 0;
  }

  clear(): void {
    this._queue = [];
  }

  reset(): void {
    this._queue = [];
  }
}
```

---

## Required Test Files

### 1. Gherkin Feature File

**Path:** `pensieve/mobile/tests/acceptance/features/story-2-4-stockage-offline.feature`

**Content Structure:** 28 scenarios couvrant les 6 AC + edge cases

---

### 2. Step Definitions File

**Path:** `pensieve/mobile/tests/acceptance/story-2-4.test.ts`

**Content Structure:** Jest-cucumber bindings pour tous les scenarios

---

## Running Tests

### Run All Story 2.4 Tests

```bash
# Run all BDD tests for story 2.4
npm run test:acceptance:story-2-4

# Run in watch mode (development)
npm run test:acceptance:watch -- story-2-4

# Run with coverage
npm run test:coverage -- story-2-4
```

### Run Specific Scenarios

```bash
# Run only AC1 tests (persistence)
npm run test:acceptance -- --testNamePattern="AC1"

# Run only AC4 tests (crash recovery)
npm run test:acceptance -- --testNamePattern="crash"

# Run only NFR compliance tests
npm run test:acceptance -- --testNamePattern="NFR"
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete) ✅

**TEA Agent Responsibilities:**

- ✅ All 28 BDD scenarios written in Gherkin
- ✅ All step definitions created with `jest-cucumber`
- ✅ MockStorageManager created in `test-context.ts`
- ✅ MockSyncQueue created in `test-context.ts`
- ✅ Existing mocks identified (InMemoryDatabase, MockFileSystem, MockApp)
- ✅ Implementation checklist created with clear tasks
- ✅ NFR compliance tagged (@NFR4, @NFR6, @NFR7, @NFR8, @NFR12)

**Verification:**

```bash
npm run test:acceptance:story-2-4
```

**Expected Output:**

```
FAIL tests/acceptance/story-2-4.test.ts
  ✗ Persister captures audio en mode offline (CaptureRepository.create not implemented)
  ✗ Stocker fichiers audio dans secure storage (Secure storage not configured)
  ✗ Créer 10 captures successivement offline (Storage monitoring not implemented)
  ✗ Charger captures en < 1s (Load time optimization not implemented)
  ✗ Récupérer captures après crash (CrashRecoveryService not implemented)
  ✗ Nettoyer fichiers audio > 90 jours (Retention policy not implemented)
  ✗ Encryter fichiers audio at rest (Encryption not configured)

Test Suites: 1 failed, 0 passed, 1 total
Tests:       28 failed, 0 passed, 28 total
Status: 🔴 RED phase verified
```

---

### GREEN Phase (DEV Team - Next Steps)

**DEV Agent Responsibilities:**

1. **Implement MockStorageManager in test-context.ts**
   - Add MockStorageManager class
   - Wire it into TestContext
   - Export for use in tests

2. **Implement MockSyncQueue in test-context.ts**
   - Add MockSyncQueue class
   - Wire it into TestContext
   - Export for use in tests

3. **Extend Capture Schema in WatermelonDB**
   - Add `syncStatus: 'pending' | 'synced'` field
   - Add `encryptionStatus: boolean` field
   - Run migration

4. **Implement CaptureRepository.create()**
   - Persist captures in WatermelonDB
   - Store audio files in secure directory
   - Set syncStatus and encryptionStatus

5. **Implement StorageManager**
   - `checkAvailableSpace()` before each capture
   - `cleanupOldFiles(retentionDays)` for retention policy
   - Display warnings for low storage

6. **Implement SyncQueue**
   - `addToQueue(captureId)` for pending captures
   - `removeFromQueue(captureId)` after sync
   - Query pending captures on app start

7. **Implement CrashRecoveryService**
   - `recoverCaptures()` on app start
   - Verify WatermelonDB integrity
   - Clean up orphaned files

8. **Configure Device-Level Encryption**
   - iOS: NSFileProtectionComplete
   - Android: FileSystem.StorageAccessFramework
   - Set encryptionStatus = true

9. **Optimize Load Performance (NFR4)**
   - Ensure captures load in < 1s
   - Use WatermelonDB query optimization
   - Measure with performance.now()

10. **Test Each Implementation**
    - Run `npm run test:acceptance:story-2-4` after each change
    - Verify tests turn green one by one
    - Fix any failing assertions

**Progress Tracking:**

Mark tasks complete in this checklist as you implement them.

---

### REFACTOR Phase (DEV Team - After All Tests Pass)

**DEV Agent Responsibilities:**

1. **Code Quality Review**
   - Extract reusable storage logic into service methods
   - Remove code duplication
   - Add TypeScript strict typing

2. **Performance Optimization**
   - Batch WatermelonDB writes for efficiency
   - Use indexed queries for fast retrieval
   - Optimize file I/O operations

3. **Security Hardening**
   - Verify encryption is correctly applied
   - Ensure no plaintext data leaks to logs
   - Audit file permissions

4. **Documentation**
   - Add JSDoc comments to all services
   - Document retention policy configuration
   - Update README with offline architecture

**Completion Criteria:**

- All 28 tests pass ✅
- Code follows project style guide
- NFR4, NFR6, NFR7, NFR8, NFR12 verified
- Ready for code review and story approval

---

## Next Steps

1. **Add MockStorageManager and MockSyncQueue to test-context.ts** (DEV prerequisite)
2. **Run failing tests** to confirm RED phase: `npm run test:acceptance:story-2-4`
3. **Begin implementation** using implementation checklist as guide
4. **Work one AC at a time** (AC1 → AC2 → AC3 → AC4 → AC5 → AC6)
5. **Run tests frequently** to get immediate feedback
6. **Share progress** in daily standup
7. **When all tests pass**, refactor for quality
8. **When refactoring complete**, manually update story status to 'done' in `sprint-status.yaml`

---

## Knowledge Base References Applied

- **data-factories.md** - Factory patterns for test data
- **fixture-architecture.md** - Mock architecture (MockStorageManager, MockSyncQueue)
- **test-quality.md** - Deterministic tests, NFR compliance tagging
- **timing-debugging.md** - Performance measurement (< 1s load time)

---

## Required data-testid Attributes

### Storage Warning Dialog

- `storage-warning-dialog` - Low storage warning dialog
- `storage-warning-message` - Warning message text
- `storage-warning-dismiss` - Dismiss button
- `storage-critical-dialog` - Critical storage blocking dialog

### Offline Indicator

- `offline-indicator` - Visual indicator for pending sync
- `offline-icon-cloud-slash` - Cloud slash icon for pending captures

### Settings Screen (Cleanup)

- `storage-usage-display` - Storage usage indicator
- `cleanup-button` - Manual cleanup trigger
- `retention-policy-setting` - Retention days configuration

---

## Contact

**Questions or Issues?**

- Ask in team standup
- Tag @TEA in Slack/Discord
- Refer to `_bmad/bmm/testarch/README.md` for TEA workflow documentation
- Consult `_bmad/bmm/testarch/knowledge/` for testing best practices

---

**Generated by BMad TEA Agent** - 2026-01-21

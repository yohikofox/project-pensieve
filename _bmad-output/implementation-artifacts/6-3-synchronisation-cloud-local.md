# Story 6.3: Synchronisation Cloud → Local

Status: ready-for-dev

---

## Story

As a **user**,
I want **cloud data to automatically sync to my device when I install the app on a new device or log in**,
So that **I can access all my captures from any device** (FR30, NFR9).

---

## Acceptance Criteria

### AC1: Initial Full Sync on New Device Login

**Given** I install the app on a new device and log in
**When** authentication completes
**Then** an initial full sync is automatically triggered
**And** all my cloud data (Captures, Thoughts, Ideas, Todos) is downloaded
**And** a progress indicator shows "Syncing your data..." with percentage
**And** the sync happens automatically without user intervention (NFR9 compliance)

### AC2: Metadata First, Audio Files Background Download

**Given** the initial sync is downloading data
**When** downloading my captures
**Then** metadata is downloaded first (fast)
**And** audio files are downloaded in the background (priority queue)
**And** I can start browsing captures with transcriptions immediately
**And** audio files download progressively as needed (lazy loading)

### AC3: Real-Time Sync Between Devices

**Given** I'm using the app on multiple devices
**When** I create a capture on Device A
**Then** Device B automatically receives the new capture within seconds (when online)
**And** the sync happens in the background without user action
**And** the feed on Device B updates in real-time with the new capture
**And** a subtle animation indicates the new content

### AC4: Incremental Sync (Only Changes)

**Given** cloud data has changed (new captures, completed todos)
**When** the app performs a sync check
**Then** only changes since the last sync are downloaded (incremental sync)
**And** the download is efficient and minimizes bandwidth usage
**And** changes are applied to the local OP-SQLite database
**And** the UI updates reactively to show the new data

### AC5: Deletion Propagation Across Devices

**Given** I delete a capture on one device
**When** the deletion syncs to the cloud
**Then** other devices receive the deletion in the next sync
**And** the capture is removed from local storage on all devices
**And** the deletion is processed as a soft delete (sync protocol)

### AC6: Network Error Retry with Backoff

**Given** sync download fails due to network interruption
**When** the failure is detected
**Then** the sync is automatically retried with Fibonacci backoff
**And** partially downloaded data is preserved for resumption
**And** the app continues to work with locally available data
**And** I see a "Sync paused" indicator

### AC7: Conflict Resolution (Last-Write-Wins)

**Given** cloud data conflicts with local changes (rare edge case)
**When** both devices modified the same record
**Then** the last-write-wins strategy is applied based on server timestamp
**And** the winning version is applied locally
**And** the losing version is logged (for post-MVP conflict UI)
**And** no data is silently lost

### AC8: Background Sync After Offline Period

**Given** I open the app after being offline for days
**When** network becomes available
**Then** a background sync automatically starts
**And** all cloud changes since last sync are downloaded
**And** large downloads are chunked to avoid memory issues
**And** the app remains responsive during sync

### AC9: Sync Completion Confirmation

**Given** sync download completes successfully
**When** all cloud changes are applied
**Then** local sync timestamps are updated
**And** the UI reflects all new data
**And** I see a brief "Synced" confirmation
**And** the app is ready for normal use

---

## Tasks / Subtasks

### Task 1: Initial Full Sync on First Login (AC1)

- [ ] **1.1** Détecter premier login (pas de `lastPulledAt` dans AsyncStorage)
- [ ] **1.2** Trigger full sync automatiquement après authentification
- [ ] **1.3** Créer `InitialSyncService` avec progress tracking
- [ ] **1.4** Implémenter progress indicator UI (percentage-based)
- [ ] **1.5** Download ALL entities: captures, thoughts, ideas, todos
- [ ] **1.6** Populate local OP-SQLite database with downloaded data
- [ ] **1.7** Set initial `lastPulledAt` timestamp après succès
- [ ] **1.8** Tests unitaires InitialSyncService

**Références:**
- Story 6.1: Backend `/api/sync/pull` endpoint déjà implémenté
- Story 6.2: SyncService.performPull() existe, étendre pour initial sync
- NFR9: Synchronisation automatique sans intervention utilisateur

---

### Task 2: Metadata First, Audio Lazy Loading (AC2)

- [ ] **2.1** Modifier PULL response pour séparer metadata vs audio URLs
- [ ] **2.2** Download captures metadata en priorité (sans audio binaire)
- [ ] **2.3** Store captures avec `audio_url` (MinIO S3) mais sans fichier local
- [ ] **2.4** Créer `LazyAudioDownloader` service pour download on-demand
- [ ] **2.5** Hook dans CaptureDetailScreen: download audio quand capture ouverte
- [ ] **2.6** Implémenter priority queue pour audio downloads (most recent first)
- [ ] **2.7** Cache audio téléchargé localement (OP-SQLite ou filesystem)
- [ ] **2.8** Tests unitaires LazyAudioDownloader

**Références:**
- Story 6.2: AudioUploadService + ChunkedUploadService déjà implémentés
- ADR-014: Storage Management (MinIO S3, lazy loading)
- Performance: Métadonnées < 1s, audio background progressif

---

### Task 3: Real-Time Sync Between Devices (AC3)

- [ ] **3.1** Vérifier polling 15min existe (ADR-009.1: launch + post-action + polling)
- [ ] **3.2** Implémenter `PeriodicSyncService` avec interval 15 minutes
- [ ] **3.3** Trigger `SyncService.sync()` périodiquement quand app active
- [ ] **3.4** Implémenter reactive UI update via WatermelonDB observables (ou OP-SQLite change events)
- [ ] **3.5** Créer subtle animation pour nouveaux items dans feed (fade-in, slide)
- [ ] **3.6** Tests unitaires PeriodicSyncService + animation tests

**Références:**
- ADR-009.1: Timing de Synchronisation (launch + post-action + **polling 15min**)
- Story 6.2: NetworkMonitor + AutoSyncOrchestrator déjà implémentés
- UX: Animation subtile, pas distrayante

---

### Task 4: Incremental Sync Optimization (AC4)

- [ ] **4.1** Vérifier `SyncService.performPull()` envoie `lastPulledAt` (déjà fait Story 6.1)
- [ ] **4.2** Backend retourne seulement delta records depuis `lastPulledAt`
- [ ] **4.3** Apply changes to local OP-SQLite (INSERT new, UPDATE modified, soft DELETE)
- [ ] **4.4** Implémenter batching pour large datasets (100 records max par batch)
- [ ] **4.5** Update `lastPulledAt` après chaque batch success
- [ ] **4.6** Tests: 500 cloud changes → 5 batches de 100 → local DB updated

**Références:**
- Story 6.1: Backend `/api/sync/pull` déjà retourne delta
- Story 6.2: Batching logic déjà implémenté pour PUSH, réutiliser pour PULL
- ADR-009.4: Chunking 100 records max

---

### Task 5: Deletion Propagation (AC5)

- [ ] **5.1** Vérifier backend inclut soft deletes dans PULL response (`_status = 'deleted'`)
- [ ] **5.2** Mobile PULL handler détecte `_status = 'deleted'`
- [ ] **5.3** Apply soft delete localement: `UPDATE [table] SET _status = 'deleted' WHERE id = ?`
- [ ] **5.4** UI filter deleted items: `WHERE _status != 'deleted'`
- [ ] **5.5** Tests: Delete on Device A → sync → Device B removes from UI

**Références:**
- ADR-009.2: Soft deletes pour sync consistency
- Story 6.2: Soft delete handling déjà implémenté pour PUSH
- WatermelonDB/OP-SQLite: `_status` column convention

---

### Task 6: Network Error Retry (AC6)

- [ ] **6.1** Vérifier `retryWithFibonacci()` existe (déjà fait Story 6.1)
- [ ] **6.2** Wrap `SyncService.performPull()` avec retry logic
- [ ] **6.3** Preserve partially downloaded batch (commit après chaque batch)
- [ ] **6.4** Update SyncStatusStore: `setPending()` si retry en cours
- [ ] **6.5** Tests: Network failure mid-sync → retry → eventual success

**Références:**
- Story 6.1: `retry-logic.ts` implémente Fibonacci backoff
- Story 6.2: SyncStatusStore + SyncStatusIndicator déjà implémentés
- ADR-009.5: Fibonacci Backoff + Result Pattern

---

### Task 7: Conflict Resolution Integration (AC7)

- [ ] **7.1** Vérifier backend détecte conflicts via `lastPulledAt` vs `last_modified` (déjà fait Story 6.1)
- [ ] **7.2** Vérifier ConflictHandler.ts existe (créé Story 6.2)
- [ ] **7.3** Implémenter conflict resolution pour PULL (currently only PUSH)
- [ ] **7.4** Apply server version localement si server wins (last-write-wins)
- [ ] **7.5** Log conflicts locally: `console.log('[ConflictHandler] PULL conflict: ...')`
- [ ] **7.6** Tests: Device A offline modifie → Device B modifie → A pull → server wins

**Références:**
- Story 6.2: ConflictHandler créé pour PUSH, étendre pour PULL
- ADR-009.2: Conflict Resolution (last-write-wins MVP)
- Story 6.1: Backend conflict detection déjà implémenté

---

### Task 8: Background Sync After Offline Period (AC8)

- [ ] **8.1** Vérifier AutoSyncOrchestrator démarre au boot (fait Story 6.2)
- [ ] **8.2** Trigger sync automatique quand network revient (déjà fait Story 6.2)
- [ ] **8.3** Implémenter chunking pour large downloads (réutiliser batching Task 4)
- [ ] **8.4** Keep UI responsive: sync en background thread (fire-and-forget)
- [ ] **8.5** Tests: Offline 7 days → 1000 cloud changes → sync success sans freeze UI

**Références:**
- Story 6.2: NetworkMonitor + AutoSyncOrchestrator déjà implémentés
- ADR-009.4: Chunking pour éviter timeout
- Performance: App reste responsive pendant sync

---

### Task 9: Sync Completion UI Updates (AC9)

- [ ] **9.1** Vérifier SyncStatusStore.setSynced() existe (fait Story 6.2)
- [ ] **9.2** Trigger `setSynced(Date.now())` après PULL success
- [ ] **9.3** Update UI reactively via SyncStatusIndicator (déjà intégré Story 6.2)
- [ ] **9.4** Show brief toast notification "Synced" (optional, 2s auto-dismiss)
- [ ] **9.5** Tests: PULL success → SyncStatusStore updated → UI shows "Synced"

**Références:**
- Story 6.2: SyncStatusStore + SyncStatusIndicator déjà créés
- Story 6.2: CustomStackHeader déjà intègre indicator
- UX: Brief confirmation, pas intrusif

---

### Task 10: Integration Testing & BDD Scenarios (All ACs)

- [ ] **10.1** Créer fichier Gherkin: `mobile/tests/acceptance/features/story-6-3-sync-cloud-local.feature`
- [ ] **10.2** Scénarios BDD (9 scénarios pour AC1-AC9):
  - **AC1:** Initial full sync on first login
  - **AC2:** Metadata first, audio lazy loading
  - **AC3:** Real-time sync between devices (periodic 15min)
  - **AC4:** Incremental sync (delta only)
  - **AC5:** Deletion propagation
  - **AC6:** Network error retry
  - **AC7:** Conflict resolution PULL
  - **AC8:** Background sync after offline period
  - **AC9:** Sync completion UI updates
- [ ] **10.3** Créer step definitions: `mobile/tests/acceptance/story-6-3.test.ts`
- [ ] **10.4** Mock `axios`, `DatabaseConnection`, `AsyncStorage`, `ConflictHandler`, `SyncStatusStore`
- [ ] **10.5** Run tests: `npm run test:acceptance -- --testPathPattern="story-6-3"`

**Références:**
- Story 6.1: Pattern BDD déjà établi
- Story 6.2: 18 scénarios BDD créés (référence pour format)
- Tests unitaires Tasks 1-9: Base solide pour tests BDD

---

## Dev Notes

### ⚠️ CRITICAL: Story 6.3 Focus vs Story 6.1 + 6.2 Infrastructure

**Story 6.1 (DONE)** provided the **complete sync infrastructure**:
- Backend endpoints `/api/sync/pull` and `/api/sync/push` ✅
- Mobile `SyncService` with OP-SQLite queries ✅
- Conflict resolution logic (backend + mobile) ✅
- Encryption, monitoring, metrics ✅
- BDD tests for infrastructure ✅

**Story 6.2 (DONE)** focused on **Local → Cloud SYNC ORCHESTRATION**:
- **Network detection** → Auto-trigger sync when online ✅
- **Upload queue** → Large audio files separately from metadata ✅
- **Retry logic** → Fibonacci backoff for reliability ✅
- **Change tracking** → `_changed` flag management ✅
- **UI indicators** → User-facing sync status ✅

**Story 6.3 (THIS STORY)** focuses on **Cloud → Local PULL ORCHESTRATION**:
- **Initial full sync** → First login downloads everything
- **Lazy audio loading** → Metadata first, audio on-demand
- **Real-time sync** → Periodic polling (15min) + network change
- **Incremental PULL** → Only delta since last sync
- **Deletion propagation** → Soft deletes across devices
- **Conflict resolution PULL** → Apply server wins locally
- **Progress indicators** → Initial sync progress UI

**Key Distinction:**
- Story 6.1 = "How sync works" (protocol, endpoints, conflict resolution)
- Story 6.2 = "When PUSH triggers" (network events, debounce, queues, user feedback)
- Story 6.3 = "When PULL triggers" (first login, periodic, lazy loading, progress UI)

---

### Architecture Patterns from Story 6.1 & 6.2

**1. Sync Protocol (Already Implemented)**

```typescript
// PULL Phase (Cloud → Local) - Story 6.3 focus
GET /api/sync/pull
{
  lastPulledAt: 1736760600000,  // Last known server timestamp (null for initial sync)
  entities: ['captures', 'thoughts', 'ideas', 'todos']
}

Backend Response:
{
  changes: {
    captures: {
      created: [
        { id: "c1", raw_content: "...", _status: "active", last_modified: 1736760700000 }
      ],
      updated: [
        { id: "c2", transcription: "...", _status: "active", last_modified: 1736760800000 }
      ],
      deleted: [
        { id: "c3", _status: "deleted" }  // Soft delete
      ]
    },
    todos: { ... },
    thoughts: { ... },
    ideas: { ... }
  },
  timestamp: 1736760900000  // New lastPulledAt for client
}
```

**2. OP-SQLite Change Application (Story 6.3 Task 4)**

```typescript
// Apply PULL changes locally
async function applyPullChanges(changes: SyncPullResponse) {
  for (const [entity, delta] of Object.entries(changes.changes)) {
    // INSERT new records
    for (const record of delta.created) {
      await db.execute(
        `INSERT INTO ${entity} (id, ..., _status, last_modified)
         VALUES (?, ..., ?, ?)`,
        [record.id, ..., record._status, record.last_modified]
      );
    }

    // UPDATE existing records
    for (const record of delta.updated) {
      await db.execute(
        `UPDATE ${entity}
         SET ..., _status = ?, last_modified = ?
         WHERE id = ?`,
        [..., record._status, record.last_modified, record.id]
      );
    }

    // SOFT DELETE (set _status = 'deleted')
    for (const record of delta.deleted) {
      await db.execute(
        `UPDATE ${entity}
         SET _status = 'deleted', last_modified = ?
         WHERE id = ?`,
        [record.last_modified, record.id]
      );
    }
  }

  // Update lastPulledAt after all changes applied
  await AsyncStorage.setItem(`sync_last_pulled_${entity}`, timestamp.toString());
}
```

**3. Initial Sync vs Incremental Sync (Task 1 vs Task 4)**

```typescript
// Check if first sync (no lastPulledAt)
async function shouldPerformInitialSync(): Promise<boolean> {
  const lastPulled = await AsyncStorage.getItem('sync_last_pulled_captures');
  return lastPulled === null;  // True if first sync
}

async function performSync() {
  const isFirstSync = await shouldPerformInitialSync();

  if (isFirstSync) {
    // Initial full sync (AC1)
    await initialSyncService.downloadAllData({
      showProgress: true,
      entities: ['captures', 'thoughts', 'ideas', 'todos']
    });
  } else {
    // Incremental sync (AC4)
    const lastPulled = await getLastPulledTimestamp();
    await syncService.performPull({ lastPulledAt: lastPulled });
  }
}
```

**4. Lazy Audio Loading (Task 2)**

```typescript
// Capture stored with audio_url, but no local file
interface CaptureMetadata {
  id: string;
  type: 'audio';
  raw_content: string;  // Transcription text
  audio_url: string;    // MinIO S3 URL: https://minio.example.com/audio/userId/captureId.m4a
  audio_local_path?: string;  // Local cached path (null if not downloaded yet)
}

// Download audio on-demand (when user opens capture detail)
class LazyAudioDownloader {
  async downloadAudioIfNeeded(captureId: string): Promise<string | null> {
    const capture = await db.queryFirst('SELECT * FROM captures WHERE id = ?', [captureId]);

    // Already downloaded?
    if (capture.audio_local_path && await FileSystem.exists(capture.audio_local_path)) {
      return capture.audio_local_path;
    }

    // Download from MinIO S3
    const localPath = `${FileSystem.documentDirectory}/audio/${captureId}.m4a`;
    await FileSystem.downloadAsync(capture.audio_url, localPath);

    // Update DB with local path
    await db.execute(
      'UPDATE captures SET audio_local_path = ? WHERE id = ?',
      [localPath, captureId]
    );

    return localPath;
  }
}
```

**5. Real-Time Sync via Periodic Polling (Task 3)**

```typescript
// Periodic sync every 15 minutes (ADR-009.1)
class PeriodicSyncService {
  private intervalId: NodeJS.Timeout | null = null;

  start() {
    // Trigger sync every 15 minutes
    this.intervalId = setInterval(() => {
      if (networkMonitor.isConnected()) {
        syncService.sync({ priority: 'low', source: 'periodic' });
      }
    }, 15 * 60 * 1000);  // 15 minutes
  }

  stop() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }
}

// Usage: Start in bootstrap.ts, stop when app backgrounds
```

**Rationale 15min Polling:**
- Balance between real-time feel and battery efficiency
- Network change detection (Story 6.2) handles immediate sync when online
- Periodic polling catches missed changes (e.g., Device A creates while Device B is active but hasn't triggered sync)
- Aligns with ADR-009.1 timing strategy

---

### Tech Stack & Dependencies (Inherited)

**Mobile (React Native):**
- `@react-native-community/netinfo` - Already installed (Story 6.2)
- `@react-native-async-storage/async-storage` - Already installed
- `axios` - Already installed
- `op-sqlite` - Already installed (ADR-018)
- `expo-file-system` - For lazy audio download (Task 2)

**Backend (NestJS):**
- No new dependencies required - Story 6.1 endpoints already exist
- `/api/sync/pull` endpoint already implements delta logic

**Testing:**
- `axios-mock-adapter` - Already installed
- `jest-cucumber` - Already installed

---

### File Structure (New Files for Story 6.3)

**Mobile:**
```
mobile/src/
├── infrastructure/
│   ├── sync/
│   │   ├── InitialSyncService.ts          # NEW - Task 1 (first login full sync)
│   │   ├── LazyAudioDownloader.ts         # NEW - Task 2 (on-demand audio download)
│   │   ├── PeriodicSyncService.ts         # NEW - Task 3 (15min polling)
│   │   └── SyncService.ts                 # EXTEND - Add applyPullChanges()
│   └── ...
├── components/
│   └── InitialSyncProgressModal.tsx       # NEW - Task 1 (progress indicator UI)
└── ...
```

**Existing Files to Extend:**
- `SyncService.ts`: Add `applyPullChanges()` method for Task 4
- `ConflictHandler.ts`: Extend for PULL conflicts (Task 7)
- `AutoSyncOrchestrator.ts`: Integrate PeriodicSyncService (Task 3)

---

### Database Schema (No Changes Required)

**Mobile (OP-SQLite):**

All tables already have sync-compatible schema (from Story 6.1):
```sql
-- Captures table (example)
CREATE TABLE IF NOT EXISTS captures (
  id TEXT PRIMARY KEY,
  type TEXT NOT NULL,
  state TEXT NOT NULL,
  raw_content TEXT,
  audio_url TEXT,                  -- MinIO S3 URL (Story 6.2)
  audio_local_path TEXT,           -- NEW for Story 6.3 (lazy loading cache)
  _status TEXT DEFAULT 'active',   -- 'active' | 'deleted' (sync field)
  _changed INTEGER DEFAULT 0,      -- 0 | 1 (local modification flag)
  last_modified INTEGER NOT NULL,  -- Server timestamp (conflict resolution)
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);
```

**New Column (optional optimization):**
- `audio_local_path TEXT` - Cache local file path for downloaded audio (Task 2)
- Migration not strictly required (can use filesystem convention instead)

---

### Performance & Optimization

**1. Initial Sync Progress (Task 1)**

```typescript
// Show percentage-based progress for first login
async function initialSyncWithProgress() {
  const entities = ['captures', 'thoughts', 'ideas', 'todos'];
  let totalRecords = 0;
  let downloadedRecords = 0;

  // Fetch total counts from backend (optional endpoint)
  const counts = await api.get('/api/sync/counts');
  totalRecords = counts.captures + counts.thoughts + counts.ideas + counts.todos;

  for (const entity of entities) {
    const delta = await api.post('/api/sync/pull', { entity, lastPulledAt: null });

    for (const record of delta.changes[entity].created) {
      await insertRecord(entity, record);
      downloadedRecords++;

      // Update progress UI
      const percentage = Math.floor((downloadedRecords / totalRecords) * 100);
      updateProgressIndicator(percentage);
    }
  }
}
```

**2. Lazy Audio Download Priority Queue (Task 2)**

```typescript
// Download most recent captures first
class LazyAudioDownloader {
  private downloadQueue: string[] = [];  // captureIds sorted by created_at DESC

  async enqueueDownload(captureId: string) {
    if (!this.downloadQueue.includes(captureId)) {
      this.downloadQueue.push(captureId);
      this.processQueue();  // Fire-and-forget
    }
  }

  private async processQueue() {
    while (this.downloadQueue.length > 0) {
      const captureId = this.downloadQueue.shift();
      await this.downloadAudioIfNeeded(captureId!);
    }
  }
}
```

**3. Chunking Large Datasets (Task 4)**

```typescript
// Download in batches of 100 to avoid timeout
async function performIncrementalPull() {
  let offset = 0;
  const batchSize = 100;

  while (true) {
    const response = await api.post('/api/sync/pull', {
      lastPulledAt,
      limit: batchSize,
      offset
    });

    if (response.changes.length === 0) break;

    await applyPullChanges(response.changes);
    await updateLastPulledAt(response.timestamp);

    offset += batchSize;
  }
}
```

---

### Security & Privacy (Inherited from Story 6.1 & 6.2)

**User Isolation (Already Enforced):**
- Backend PULL endpoint filters by `userId` from JWT
- User A cannot pull User B's data
- NFR13 compliance already validated in Story 6.1

**Encryption (Already Enforced):**
- HTTPS/TLS for all API calls (NFR11)
- Encryption at rest (infrastructure-level) (NFR12)
- MinIO S3 audio files isolated by userId path

**No New Security Concerns:**
- PULL is read-only operation (no data modification risk)
- Same authentication flow as PUSH (Story 6.2)

---

### Known Issues & Edge Cases

**1. Large Initial Sync (1000+ Captures)**
- **Problème:** First login download timeout si trop de data
- **Solution:** Chunking (Task 4) + show progress indicator (Task 1)
- **Test:** Create 1000 captures on Device A → Login Device B → sync completes < 60s

**2. Audio Download Bandwidth**
- **Problème:** Download 100 audio files (5 GB) uses too much bandwidth
- **Solution:** Lazy loading (Task 2) - metadata first, audio on-demand
- **Test:** Initial sync downloads metadata < 10s, audio downloads progressively

**3. Periodic Sync Battery Drain**
- **Problème:** 15min polling drains battery
- **Solution:** Only poll when app is in foreground (stop when backgrounded)
- **Test:** App backgrounded → periodic sync stops → resume → sync resumes

**4. Conflict Loop (Same Record Modified on Both Devices)**
- **Problème:** Device A offline modifies → Device B modifies → A pulls → conflict → B pulls → conflict loop
- **Solution:** `last_modified` server timestamp breaks loop (ADR-009.2)
- **Test:** Both devices modify same Todo → sync → server wins → loop stopped

**5. Soft Delete Not Visible in UI**
- **Problème:** Deleted items still shown if UI doesn't filter `_status = 'deleted'`
- **Solution:** All OP-SQLite queries MUST filter: `WHERE _status != 'deleted'`
- **Test:** Delete on Device A → sync → Device B UI doesn't show deleted item

---

### Testing Strategy

**Unit Tests (Mobile):**
- `InitialSyncService.test.ts`: First login full sync, progress tracking
- `LazyAudioDownloader.test.ts`: On-demand download, priority queue
- `PeriodicSyncService.test.ts`: 15min interval, start/stop
- `SyncService.applyPullChanges.test.ts`: INSERT/UPDATE/DELETE logic

**BDD/Gherkin Tests (Mobile):**
- `story-6-3.feature`: 9 scenarios covering AC1-AC9
- Step definitions mock axios, DatabaseConnection, AsyncStorage

**Integration Tests (E2E):**
- First login → full sync → verify all data downloaded
- Device A creates → Device B periodic sync → verify real-time update
- Large dataset (500 records) → chunked download → verify performance
- Multi-device conflict → verify last-write-wins applied

**Performance Tests:**
- Initial sync 1000 records < 60s (NFR performance)
- Lazy audio download 50MB < 30s on 10 Mbps connection
- Periodic sync trigger < 5s (network overhead minimal)

---

### Acceptance Testing Checklist

- [ ] AC1: First login → auto full sync → progress indicator → all data downloaded
- [ ] AC2: Metadata first → audio lazy loading → can browse before audio downloads
- [ ] AC3: Device A creates → Device B receives within 15min → real-time feel
- [ ] AC4: Incremental sync → only delta downloaded → bandwidth efficient
- [ ] AC5: Delete on Device A → Device B removes from UI → soft delete propagated
- [ ] AC6: Network error → Fibonacci retry → eventual success → "Sync paused" indicator
- [ ] AC7: Conflict → last-write-wins → server version applied locally → logged
- [ ] AC8: Offline 7 days → 1000 cloud changes → sync success → app responsive
- [ ] AC9: Sync success → lastPulledAt updated → UI "Synced" confirmation
- [ ] User isolation: User A data not pulled by User B (security test)
- [ ] Performance: Initial sync 1000 records < 60s

---

## References

### Architecture Decision Records (ADRs)

**Primary:**
- [ADR-009: Stratégie de Synchronisation](../_bmad-output/planning-artifacts/adrs/ADR-009-sync-patterns.md) - 6 décisions sync critiques
  - 9.1: Timing de Synchronisation (**launch + post-action + polling 15min**) ← Story 6.3 Task 3
  - 9.2: Détection et Résolution de Conflits (lastPulledAt + last_modified) ← Story 6.3 Task 7
  - 9.4: Priorité de Synchronisation (Captures > Todos > Ideas) ← Story 6.3 Task 2
  - 9.5: Retry Logic & Error Handling (Fibonacci Backoff) ← Story 6.3 Task 6

- [ADR-014: Storage Management](../_bmad-output/planning-artifacts/adrs/ADR-014-storage-management.md) - MinIO S3, lazy loading ← **Story 6.3 Task 2**

**Secondary:**
- [ADR-018: Migration WatermelonDB → OP-SQLite](../_bmad-output/planning-artifacts/adrs/ADR-018-migration-watermelondb-opsqlite.md) - JSI-native, performance
- [ADR-023: Stratégie Unifiée de Gestion des Erreurs](../_bmad-output/planning-artifacts/adrs/ADR-023-error-handling-strategy.md) - Result Pattern partout

### Requirements

**Functional:**
- FR30: Le système peut synchroniser les données cloud vers le device au login
- FR31: L'utilisateur peut être informé du statut de synchronisation

**Non-Functional:**
- NFR6: Aucune perte de données - Tolérance zéro (retry logic + conflict resolution)
- NFR9: Synchronisation automatique sans intervention utilisateur (first login + periodic)
- NFR11: Chiffrement transit HTTPS/TLS (already done Story 6.1)
- NFR13: Isolation données utilisateur (JWT validation backend)

### Previous Story Context

**Story 6.1 (DONE)** - Infrastructure de Synchronisation
- Backend sync endpoints created ✅
- Mobile SyncService implemented ✅
- Conflict resolution logic ✅
- **Learnings:**
  - OP-SQLite queries performantes (4× faster than WatermelonDB)
  - Fibonacci backoff superior to exponential
  - Transaction wrapping CRITICAL for atomic operations

**Story 6.2 (DONE)** - Synchronisation Local → Cloud
- Network detection auto-sync ✅
- Upload queue for large audio files ✅
- Retry logic with Fibonacci backoff ✅
- UI sync status indicators ✅
- **Learnings:**
  - SyncTrigger debounce 3s prevents spam API calls
  - AutoSyncOrchestrator orchestrates NetworkMonitor + SyncService
  - Result Pattern partout (ADR-023 compliance)
  - DI container registration CRITICAL (crashes au boot si oublié)
  - Tests BDD nécessitent stricte TypeScript compliance

### Project Context

**Critical Rules from project-context.md:**
- Node 22, Expo SDK 54, React Native 0.81.5
- OP-SQLite (NOT WatermelonDB - ADR-018)
- TypeScript strict mode (non-negotiable)
- Result Pattern for error handling (ADR-023)
- BDD tests with jest-cucumber (Gherkin français)
- Clean Code standards (ADR-024)

**DI Bootstrap Sequence:**
```
index.ts → bootstrap() → App.tsx → MainApp.tsx
```
- All services MUST be registered in DI container BEFORE React renders
- AutoSyncOrchestrator + PeriodicSyncService démarrage au boot

**Testing Configuration:**
```bash
# Mobile
npm run test:unit                # babel-jest
npm run test:acceptance          # ts-jest, Gherkin scenarios
```

---

## Dev Agent Record

### Agent Model Used

*À compléter lors de l'implémentation*

### Debug Log References

*À compléter si nécessaire*

### Completion Notes List

*À compléter au fur et à mesure*

### File List

**To be created:**
- `mobile/src/infrastructure/sync/InitialSyncService.ts`
- `mobile/src/infrastructure/sync/LazyAudioDownloader.ts`
- `mobile/src/infrastructure/sync/PeriodicSyncService.ts`
- `mobile/src/components/InitialSyncProgressModal.tsx`
- `mobile/tests/acceptance/features/story-6-3-sync-cloud-local.feature`
- `mobile/tests/acceptance/story-6-3.test.ts`

**To be modified:**
- `mobile/src/infrastructure/sync/SyncService.ts` (add applyPullChanges)
- `mobile/src/infrastructure/sync/ConflictHandler.ts` (extend for PULL)
- `mobile/src/infrastructure/sync/AutoSyncOrchestrator.ts` (integrate PeriodicSyncService)
- `mobile/src/infrastructure/di/container.ts` (register new services)
- `mobile/src/config/bootstrap.ts` (start PeriodicSyncService)

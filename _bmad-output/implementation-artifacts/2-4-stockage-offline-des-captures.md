# Story 2.4: Stockage Offline des Captures

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **user**,
I want **my captures to be stored securely on my device even without network**,
so that **I never lose a thought and can access them anytime** (NFR6: zero data loss tolerance).

## Acceptance Criteria

### AC1: Persist All Captures Locally with Sync Status
**Given** I have created captures (audio or text) while offline
**When** the captures are saved
**Then** all Capture entities are persisted in WatermelonDB
**And** audio files are stored in secure device storage
**And** each Capture has a sync status field (pending/synced)
**And** captures marked "pending" are queued for future synchronization

### AC2: Handle Multiple Successive Offline Captures
**Given** the app has no network connectivity
**When** I create multiple captures in succession
**Then** all captures are saved locally without errors (NFR7: 100% offline availability)
**And** storage space is monitored to prevent overflow
**And** if storage is critically low, I receive a warning before capturing

### AC3: Fast Local Cache Access
**Given** I have offline captures stored locally
**When** I reopen the app while still offline
**Then** all my captures are immediately accessible (NFR4: < 1s load time)
**And** the feed displays all captures with offline indicators
**And** no network errors are shown

### AC4: Crash Recovery with Zero Data Loss
**Given** the app crashes with unsynchronized captures
**When** I relaunch the app
**Then** all saved captures are recovered intact (NFR8: crash recovery)
**And** pending sync status is preserved
**And** no data is lost

### AC5: Storage Management with Retention Policy
**Given** I have accumulated many offline captures
**When** storage management runs
**Then** old audio files can be cleaned up based on retention policy
**And** transcriptions and metadata are always retained
**And** I am notified before any cleanup occurs (Additional Requirement: Storage Management)

### AC6: Encryption at Rest
**Given** captures are encrypted at rest (NFR12)
**When** captures are written to device storage
**Then** audio files and text content are encrypted using device-level encryption
**And** metadata includes encryption status flag

## Tasks / Subtasks

- [ ] **Task 1: Configure WatermelonDB for Offline-First** (AC: 1, 3, 4)
  - [ ] Subtask 1.1: Verify Capture schema includes sync fields
    - Ensure syncStatus field ('pending' | 'synced')
    - Add _status field for WatermelonDB soft deletes
    - Add _changed flag for tracking modifications
    - Add last_modified_at timestamp
  - [ ] Subtask 1.2: Configure WatermelonDB optimistic locking
    - Enable conflict resolution (last-write-wins for MVP)
    - Configure schema migrations for sync compatibility
  - [ ] Subtask 1.3: Implement local cache queries
    - Create efficient queries for feed display
    - Index by capturedAt for chronological sorting
    - Optimize query performance for < 1s load (NFR4)

- [ ] **Task 2: Implement Secure Device Storage for Audio Files** (AC: 1, 6)
  - [ ] Subtask 2.1: Configure secure storage directory
    - Use expo-file-system documentDirectory (encrypted by OS)
    - Verify iOS/Android encryption at rest enabled
    - Set proper file permissions (user-only access)
  - [ ] Subtask 2.2: Implement audio file save with encryption flag
    - Save audio files with .m4a extension
    - Store file path in Capture.rawContent
    - Set encryption status in Capture metadata
  - [ ] Subtask 2.3: Verify device-level encryption
    - Check iOS Keychain / Android Keystore availability
    - Log encryption status in Capture metadata
    - Fallback warning if encryption unavailable

- [ ] **Task 3: Implement Sync Queue Management** (AC: 1)
  - [ ] Subtask 3.1: Create SyncQueueService
    - Track all captures with syncStatus='pending'
    - Maintain FIFO queue order
    - Persist queue state to survive app restarts
  - [ ] Subtask 3.2: Mark captures for sync
    - Automatically set syncStatus='pending' for offline captures
    - Update syncStatus='synced' after successful cloud sync (future story)
    - Track retry attempts for failed syncs

- [ ] **Task 4: Implement Storage Space Monitoring** (AC: 2)
  - [ ] Subtask 4.1: Create StorageMonitorService
    - Query available device storage via expo-file-system
    - Calculate total size of captured audio files
    - Set threshold for "critically low" (e.g., < 100MB free)
  - [ ] Subtask 4.2: Warn user before capture if storage low
    - Check storage before starting audio recording
    - Show alert: "Storage low (XMB remaining). Continue?"
    - Allow user to proceed or cancel
  - [ ] Subtask 4.3: Handle out-of-storage scenario
    - Catch storage errors during file save
    - Show user-friendly error message
    - Do not create orphaned Capture entities

- [ ] **Task 5: Implement Crash Recovery Mechanism** (AC: 4)
  - [ ] Subtask 5.1: Create CrashRecoveryService (extends Story 2.1)
    - Scan for incomplete Capture records on app launch
    - Check for orphaned audio files (file exists but no DB record)
    - Check for missing audio files (DB record but file missing)
  - [ ] Subtask 5.2: Recover valid captures
    - Match orphaned files to partial DB records
    - Create Capture records for valid orphaned files
    - Preserve syncStatus='pending' for recovered captures
  - [ ] Subtask 5.3: Clean up corrupted data
    - Delete truly corrupted/unrecoverable files
    - Remove invalid DB records
    - Log all recovery actions for debugging

- [ ] **Task 6: Implement Storage Retention Policy** (AC: 5)
  - [ ] Subtask 6.1: Create RetentionPolicyService
    - Define retention rules: Keep audio for X days (e.g., 30 days)
    - Always keep transcriptions and metadata
    - Only delete synced audio files (not pending)
  - [ ] Subtask 6.2: Implement cleanup logic
    - Run retention check periodically (e.g., weekly)
    - Identify eligible audio files for cleanup
    - Delete old files while preserving metadata
  - [ ] Subtask 6.3: Notify user before cleanup
    - Show notification: "Old audio files will be cleaned up (XMB freed)"
    - Provide option to disable auto-cleanup
    - Allow manual cleanup trigger in settings

- [ ] **Task 7: Add Offline Indicators to UI** (AC: 3)
  - [ ] Subtask 7.1: Show sync status badges on captures
    - Display "Pending sync" badge for offline captures
    - Use cloud icon with slash for offline state
    - Update badge when capture syncs successfully
  - [ ] Subtask 7.2: Show global offline indicator
    - Display offline mode banner in app header
    - Show count of pending syncs
    - Provide reassurance: "Your captures are safe locally"

- [ ] **Task 8: Write Comprehensive Tests** (AC: All)
  - [ ] Subtask 8.1: Unit tests for storage services
    - Test SyncQueueService (queue management, persistence)
    - Test StorageMonitorService (space calculation, thresholds)
    - Test CrashRecoveryService (recovery scenarios)
    - Test RetentionPolicyService (cleanup rules)
  - [ ] Subtask 8.2: Integration tests for offline scenarios
    - Test creating 10+ captures while offline
    - Test app restart with pending captures
    - Test crash recovery with partial data
    - Test storage low warning trigger
    - Test retention policy execution
  - [ ] Subtask 8.3: Performance tests
    - Verify < 1s load time for feed with 100+ captures (NFR4)
    - Test storage monitoring overhead
    - Test crash recovery speed
  - [ ] Subtask 8.4: Data integrity tests
    - Verify zero data loss across all scenarios (NFR6)
    - Test concurrent writes (race conditions)
    - Test atomic file operations

## Dev Notes

### Architecture Context

**Bounded Context:** Capture Context (Supporting Domain)
**Supporting Services:**
- SyncQueueService (manages pending syncs)
- StorageMonitorService (tracks device storage)
- CrashRecoveryService (recovers data after crashes)
- RetentionPolicyService (manages old file cleanup)

**Critical NFRs:**
- **NFR6:** 0 capture perdue, jamais (tolérance zéro) - TOP PRIORITY
- **NFR7:** Disponibilité capture offline = 100%
- **NFR4:** Chargement liste < 1s (cache local)
- **NFR8:** Récupération après crash automatique
- **NFR12:** Chiffrement au repos

### Technical Stack

**From Story 2.1:**
- WatermelonDB (offline-first with sync protocol)
- expo-file-system (secure storage)
- Capture model (already supports syncStatus)

**Additional:**
- Device storage APIs (FileSystem.getFreeDiskStorageAsync)
- Background task scheduling (for retention policy - optional)
- Notification API (for cleanup warnings)

### Offline-First Architecture Pattern

**Storage Layers:**
1. **WatermelonDB (SQLite):** Capture metadata, transcriptions, todos
2. **File System:** Audio files (.m4a), images (future)
3. **Sync Queue:** Pending operations for cloud sync

**Data Flow:**
```
Capture Created → WatermelonDB (syncStatus='pending') → Sync Queue → Cloud Sync (future)
                ↓
             File System (audio files)
```

### Performance Optimization

**NFR4 Compliance (< 1s load):**
- Index WatermelonDB on capturedAt
- Lazy load audio files (metadata only in feed)
- Pagination for large lists (50+ captures)
- Cache query results

**Storage Efficiency:**
- Compress audio files (m4a format)
- Cleanup old audio after retention period
- Keep transcriptions/metadata forever

### Security & Encryption

**NFR12 Compliance:**
- **iOS:** Data Protection API (enabled by default for documentDirectory)
- **Android:** EncryptedSharedPreferences + File-based Encryption
- **Verify encryption:** Check device settings, log status
- **Fallback:** Warn user if encryption unavailable

**No custom encryption implementation needed - rely on OS-level encryption.**

### Storage Management Strategy

**Retention Policy:**
- Audio files: 30 days (configurable)
- Transcriptions: Forever
- Metadata: Forever
- Only delete **synced** audio (not pending)

**Monitoring:**
- Check storage before audio capture
- Warn if < 100MB free
- Periodic cleanup (weekly background task)

### Reuse from Stories 2.1-2.3

**Already implemented:**
- Capture WatermelonDB model with syncStatus
- RecordingService with file storage
- CrashRecoveryService (partial implementation in 2.1)
- File management with expo-file-system

**Extend:**
- Add SyncQueueService (new)
- Add StorageMonitorService (new)
- Enhance CrashRecoveryService (recovery logic)
- Add RetentionPolicyService (new)

### File Structure

```
apps/mobile/
├── src/
│   ├── contexts/
│   │   └── Capture/
│   │       ├── services/
│   │       │   ├── RecordingService.ts  # From 2.1
│   │       │   ├── CrashRecoveryService.ts  # EXTEND from 2.1
│   │       │   ├── SyncQueueService.ts  # NEW
│   │       │   ├── StorageMonitorService.ts  # NEW
│   │       │   └── RetentionPolicyService.ts  # NEW
│   │       └── ui/
│   │           └── OfflineIndicator.tsx  # NEW
```

### Testing Standards

- **Unit tests:** All services (Sync, Storage, Crash, Retention)
- **Integration tests:** Full offline scenarios, crash recovery
- **Performance tests:** < 1s load, storage monitoring overhead
- **Data integrity tests:** Zero data loss verification
- **Edge cases:** Out of storage, concurrent writes, corrupted files

### Dependencies

**Already installed:**
- `@nozbe/watermelondb`
- `expo-file-system`

**No new dependencies required.**

### Previous Story Intelligence (Stories 2.1-2.3)

**Learnings:**
- WatermelonDB sync protocol works well
- expo-file-system reliable for audio storage
- Crash recovery pattern established
- Offline-first UX patterns defined

**Code patterns to reuse:**
- Service architecture (stateless, injectable)
- WatermelonDB query patterns
- File operations with error handling
- Offline UI indicators

### References

- [Source: _bmad-output/planning-artifacts/epics.md#Story 2.4]
- [Source: _bmad-output/planning-artifacts/architecture.md#Offline-First Architecture]
- [Source: _bmad-output/planning-artifacts/architecture.md#Storage Management]
- [Source: _bmad-output/implementation-artifacts/2-1-capture-audio-1-tap.md#Crash Recovery]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5

### Debug Log References

<!-- Populate during implementation -->

### Completion Notes List

<!-- Populate during implementation -->

### File List

<!-- Populate during implementation -->

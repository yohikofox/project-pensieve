---
adr: ADR-021
title: "DI Lifecycle Strategy - Transient First (Révision ADR-017)"
date: 2026-01-24
status: "✅ Accepted"
supersedes: "ADR-017 (partiel - lifecycle uniquement)"
context: "Story 2.5 - Transcription On-Device avec Whisper"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
  - Amelia (Dev Agent)
---

# ADR-021: DI Lifecycle Strategy - Transient First (Révision ADR-017)

**Date:** 2026-01-24
**Status:** ✅ Accepted
**Context:** Story 2.5 - Transcription On-Device avec Whisper
**Decision Makers:** yohikofox (Product Owner), Winston (Architect), Amelia (Dev)
**Supersedes:** ADR-017 (lifecycle strategy uniquement - TSyringe choice preserved)

---

## Context & Problem

**ADR-017 recommendation initiale:**

ADR-017 ("Dependency Injection & IoC Container Strategy") recommandait **Singleton pour TOUS les services** (ligne 91-95):

```typescript
export function registerServices() {
  container.registerSingleton(TOKENS.ICaptureRepository, CaptureRepository);
  container.registerSingleton(TOKENS.IAudioRecorder, ExpoAudioAdapter);
  container.registerSingleton(TOKENS.IFileSystem, ExpoFileSystemAdapter);
  container.registerSingleton(TOKENS.IPermissionService, PermissionService);
}
```

**Problème détecté lors de Story 2.5:**

Singleton = **state in-memory** = problème critique pour mobile:

```typescript
// ❌ MAUVAIS: State in-memory avec Singleton
@injectable()
export class TranscriptionQueueService {
  private queue: QueuedCapture[] = []; // ⚠️ In-memory state
  private paused: boolean = false;     // ⚠️ In-memory state

  enqueue(capture: QueuedCapture): void {
    this.queue.push(capture); // Mutate in-memory state
  }

  // Si app crash = queue perdue ❌
  // Si app kill = queue perdue ❌
  // Si device reboot = queue perdue ❌
}
```

**Challenge de yohikofox:**

> "Je ne suis pas fan de la règle singleton first. Cela met une énorme pression sur le développement fonctionnel, aucun state persistant. Si on est en mode un resolve = une nouvelle instance, on s'assure de ne pas avoir de mémoire partagée. De plus le state management doit être persisté en base localement, pas dans une instance d'un objet, ces derniers ne doivent pas persister in memory, ce qui va dans le sens des contraintes mobile."

**Contraintes mobile critiques:**
- **Crash-proof:** App peut crasher à tout moment (OOM, background kill)
- **Memory pressure:** RAM limitée sur devices anciens
- **Process termination:** iOS/Android kill background apps agressivement
- **State isolation:** Tests doivent avoir instances isolées
- **DB-first:** Single source of truth = database, pas mémoire

**Contraintes testabilité:**
- Singleton = state partagé entre tests = side effects
- Transient = instance propre par test = isolation parfaite

---

## Decision

**Principe architectural révisé: Transient First, Singleton Exceptions**

### Règle de Décision

```
IF service has technical cache (Whisper model, native session, RxJS Subject)
  THEN registerSingleton()
ELSE IF service state belongs in DB
  THEN register() + DB persistence (Transient)
ELSE IF service is stateless
  THEN register() (Transient)
```

### Classification Services

| Service Type | Lifecycle | Justification |
|--------------|-----------|---------------|
| **Repository (CaptureRepository)** | **Transient** | State en DB, pas in-memory ✅ |
| **Queue Service** | **Transient** | Queue en OP-SQLite (ADR-022), pas in-memory ✅ |
| **State Machine (CaptureStateService)** | **Transient** | Pure logic, pas de state ✅ |
| **File System Adapter** | **Transient** | Stateless wrapper sur Expo APIs ✅ |
| **Permission Service** | **Transient** | Stateless wrapper sur permissions ✅ |
| **EventBus** | **Singleton** | RxJS Subject = état global (subscribers) ⚠️ |
| **Whisper Service (TranscriptionService)** | **Singleton** | Model context = 500MB RAM cache ⚠️ |
| **Whisper Model Service** | **Singleton** | Model download state = cache technique ⚠️ |
| **Audio Recorder** | **Singleton** | Native recording session = état global ⚠️ |

### Architecture Révisée

```typescript
// src/infrastructure/di/container.ts
import 'reflect-metadata';
import { container } from 'tsyringe';

/**
 * Register Capture Context services
 */
export function registerCaptureServices() {
  // TRANSIENT: Repositories (state en DB)
  container.register(
    TOKENS.ICaptureRepository,
    CaptureRepository // Nouvelle instance par resolve
  );

  // SINGLETON: Audio recorder (native session = état global)
  container.registerSingleton(
    TOKENS.IAudioRecorder,
    ExpoAudioAdapter
  );

  // TRANSIENT: File system (stateless wrapper)
  container.register(
    TOKENS.IFileSystem,
    ExpoFileSystemAdapter
  );

  // TRANSIENT: Permission service (stateless wrapper)
  container.register(
    TOKENS.IPermissionService,
    PermissionService
  );

  // TRANSIENT: Recording service (orchestration stateless)
  container.register(
    TOKENS.IRecordingService,
    RecordingService
  );
}

/**
 * Register Normalization Context services
 */
export function registerNormalizationServices() {
  // SINGLETON: Whisper model service (model cache = état technique)
  container.registerSingleton(
    NORMALIZATION_TOKENS.IWhisperModelService,
    WhisperModelService // 500MB model context en RAM
  );

  // SINGLETON: Transcription service (Whisper model context caché)
  container.registerSingleton(
    NORMALIZATION_TOKENS.ITranscriptionService,
    TranscriptionService
  );

  // TRANSIENT: Queue service (queue en OP-SQLite, pas in-memory)
  container.register(
    NORMALIZATION_TOKENS.ITranscriptionQueueService,
    TranscriptionQueueService
  );

  // TRANSIENT: Queue processor (orchestration, pas de state)
  container.register(
    NORMALIZATION_TOKENS.ITranscriptionQueueProcessor,
    TranscriptionQueueProcessor
  );

  // TRANSIENT: State machine (pure logic)
  container.register(
    NORMALIZATION_TOKENS.ICaptureStateService,
    CaptureStateService
  );
}

/**
 * Register Infrastructure services
 */
export function registerInfrastructureServices() {
  // SINGLETON: EventBus (RxJS Subject = état global subscribers)
  container.registerSingleton(
    TOKENS.IEventBus,
    EventBus
  );

  // TRANSIENT: Database (stateless wrapper sur OP-SQLite)
  container.register(
    TOKENS.IDatabase,
    Database
  );
}
```

---

## Rationale

### Pourquoi Transient First?

**1. Crash-Proof Architecture**

```typescript
// ✅ BON: State en DB, service stateless
@injectable()
export class TranscriptionQueueService {
  constructor(
    @inject(TOKENS.IDatabase) private db: Database
  ) {}

  // Chaque operation = read from DB
  async enqueue(capture: QueuedCapture): Promise<void> {
    await this.db.insert('transcription_queue', {
      capture_id: capture.id,
      audio_path: capture.audioPath,
      status: 'pending',
      created_at: Date.now(),
    });
  }

  async getNextCapture(): Promise<QueuedCapture | null> {
    const row = await this.db.queryOne(`
      SELECT * FROM transcription_queue
      WHERE status = 'pending'
      ORDER BY created_at ASC
      LIMIT 1
    `);
    return row ? mapRowToQueuedCapture(row) : null;
  }

  // Pas de state in-memory = survive app crash ✅
  // Pas de state in-memory = survive app kill ✅
  // Pas de state in-memory = survive device reboot ✅
}
```

**2. Test Isolation Parfaite**

```typescript
// ✅ Avec Transient: Chaque test = instance propre
beforeEach(() => {
  container.clearInstances(); // Reset transient instances

  mockDb = createMockDatabase();
  container.register(TOKENS.IDatabase, { useValue: mockDb });

  // Nouvelle instance pour chaque test = isolation parfaite
  service = container.resolve(TranscriptionQueueService);
});

it('should enqueue capture in DB', async () => {
  await service.enqueue({ captureId: '1', audioPath: '/audio.m4a' });

  // Vérifier DB (source of truth)
  const rows = await mockDb.query('SELECT * FROM transcription_queue');
  expect(rows).toHaveLength(1);
});

// Pas de side effects entre tests ✅
// Pas de mock cleanup nécessaire ✅
```

**3. Mobile Memory Constraints**

| Pattern | Memory Impact | Crash Resistance |
|---------|---------------|------------------|
| **Singleton + in-memory state** | ❌ State accumule en RAM | ❌ Perdu au crash |
| **Transient + DB persistence** | ✅ Minimal (instance temporaire) | ✅ Persiste en DB |

**4. Single Source of Truth = Database**

```
Principe: DB = source of truth, pas objets in-memory

❌ MAUVAIS:
  Queue en mémoire → DB miroir → Sync issues, data loss au crash

✅ BON:
  DB = truth → Services = stateless readers/writers → Crash-proof
```

### Pourquoi Singleton pour Whisper/EventBus?

**Exception 1: Whisper Model Service**

```typescript
// Singleton justifié: Cache technique (500MB model context)
@injectable()
export class TranscriptionService {
  private modelContext: any | null = null; // ⚠️ Technical cache (not business state)

  async loadModel(modelPath: string): Promise<void> {
    if (this.modelContext) {
      return; // Model already loaded (cached)
    }

    this.modelContext = await whisperRn.initWhisper({ filePath: modelPath });
  }

  // Singleton = 1 seul model context chargé en RAM (500MB)
  // Transient = N model contexts = OOM crash ❌
}
```

**Exception 2: EventBus**

```typescript
// Singleton justifié: RxJS Subject global (subscribers)
@injectable()
export class EventBus {
  private eventStream = new Subject<DomainEvent>(); // ⚠️ Global event stream

  // Singleton = subscribers partagés globalement
  // Transient = subscribers isolés = pas de communication ❌
}
```

**Exception 3: Audio Recorder**

```typescript
// Singleton justifié: Native recording session
@injectable()
export class ExpoAudioAdapter {
  private recording: Audio.Recording | null = null; // ⚠️ Native session

  // Singleton = 1 seule session native
  // Transient = N sessions = conflicts ❌
}
```

### Comparaison Patterns

| Critère | Singleton First (ADR-017) | Transient First (ADR-021) |
|---------|---------------------------|---------------------------|
| **Crash resistance** | ❌ State perdu | ✅ **State en DB** |
| **Memory pressure** | ❌ State accumule | ✅ **Instances temporaires** |
| **Test isolation** | ❌ Side effects | ✅ **Isolation parfaite** |
| **Single source of truth** | ⚠️ Split (DB + memory) | ✅ **DB only** |
| **Mobile-first** | ⚠️ Risqué | ✅ **Optimal** |
| **Performance** | ✅ Cache en mémoire | ⚠️ DB reads (+latency) |
| **Simplicity** | ✅ Simple (singleton partout) | ⚠️ Décision par service |

**Trade-off accepté:** Latency légère (+5-10ms par DB read) vs crash-proof architecture

---

## Consequences

### ✅ Bénéfices

1. **Crash-proof:** State persiste en DB, survive crashes/kills/reboots
2. **Test isolation:** Chaque test = instance propre, pas de side effects
3. **Memory efficient:** Instances temporaires, garbage collected
4. **Single source of truth:** DB = authoritative, pas split memory/DB
5. **Mobile-first:** Optimisé pour contraintes RAM/process termination
6. **ADR-022 compliant:** State persistence en OP-SQLite (pas AsyncStorage)

### ⚠️ Trade-offs acceptés

1. **Latency DB reads:** +5-10ms par operation vs in-memory cache
   - Mitigation: OP-SQLite ultra-rapide (ADR-022)
   - Acceptable: Crash-proof > latency microscopique

2. **Complexité décision:** Développeur doit choisir Singleton vs Transient
   - Mitigation: Règle claire + exemples documentés
   - Acceptable: Décision architecturale explicite > "tout Singleton"

3. **Performance cache:** Pas de cache in-memory (sauf exceptions)
   - Mitigation: Exceptions pour caches techniques (Whisper, EventBus)
   - Acceptable: DB = fast enough pour mobile

### 🔄 Impact sur ADR-017

**Ce qui CHANGE:**
- ❌ Singleton first → ✅ **Transient first**
- ❌ Repositories = Singleton → ✅ **Repositories = Transient**
- ❌ State in-memory OK → ✅ **State en DB obligatoire**

**Ce qui RESTE:**
- ✅ TSyringe (choix container)
- ✅ Decorators `@injectable()` + `@inject()`
- ✅ Test containers avec mocks
- ✅ NestJS native pour backend

**ADR-017 status:** ✅ Accepted (container choice) + 🔄 Superseded (lifecycle strategy)

---

## Implementation

### Étapes de mise en œuvre

1. ⏳ Réviser tous registerSingleton() existants (ADR-017)
2. ⏳ Identifier services avec state in-memory → migrer vers DB
3. ⏳ Convertir CaptureRepository: Singleton → Transient
4. ⏳ Convertir PermissionService: Singleton → Transient
5. ⏳ Convertir FileSystem: Singleton → Transient
6. ⏳ Garder Singleton: EventBus, TranscriptionService, AudioRecorder
7. ⏳ Créer nouveaux services Transient: TranscriptionQueueService, CaptureStateService
8. ⏳ Ajouter tests vérifiant isolation (clearInstances entre tests)

### Files Modified

```
mobile/
├── src/
│   ├── infrastructure/
│   │   └── di/
│   │       └── container.ts            # Modified: Transient first
│   ├── contexts/capture/
│   │   ├── data/
│   │   │   └── CaptureRepository.ts    # Modified: State en DB
│   │   └── services/
│   │       └── PermissionService.ts    # Modified: Stateless
│   └── contexts/Normalization/services/
│       ├── TranscriptionQueueService.ts # New: Transient + DB
│       └── CaptureStateService.ts      # New: Transient stateless
```

**Effort estimé:** 3-4 heures (refactoring existant + nouveaux services)

---

## Validation Criteria

ADR considéré succès SI :

- ⏳ Tous services Transient sauf exceptions documentées
- ⏳ Aucun state in-memory (sauf caches techniques)
- ⏳ Tous repositories = Transient + state en DB
- ⏳ Tests unitaires: clearInstances() entre tests
- ⏳ Tests vérifient isolation (pas de side effects)
- ⏳ App survive crash: queue/state persist en DB
- ⏳ Performance acceptable: DB reads < 10ms

**Review Date:** 2026-02 (après Story 2.5 + validation crash scenarios)

---

## References

- ADR-017 (IoC DI Strategy): `./ADR-017-ioc-di-strategy.md`
- ADR-022 (State Persistence): `./ADR-022-state-persistence-opsqlite.md`
- TSyringe Lifecycles: https://github.com/microsoft/tsyringe#injection-scopes
- Mobile Memory Management: https://reactnative.dev/docs/performance#memory
- DDD Repositories: https://martinfowler.com/eaaCatalog/repository.html

---

## Decision Log

**2026-01-24** - Discussion yohikofox, Winston, Amelia

→ **Problème:** ADR-017 recommend Singleton pour tous = state in-memory = crash risk
→ **Challenge yohikofox:** "Pas fan de singleton first, state doit être en DB, pas in-memory"
→ **Analyse:** Mobile constraints = crash/kill fréquent → DB-first obligatoire
→ **Options:** Singleton first (ADR-017) ❌ vs Transient first + exceptions ✅
→ **Décision:** Transient first, Singleton pour caches techniques uniquement
→ **Validation:** yohikofox confirme DB-first = principe mobile correct

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)
- Amelia (Dev Agent)

---

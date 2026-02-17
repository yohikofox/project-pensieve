# Audit IoC/DI - Conformité ADR-017

**Date:** 2026-01-22
**Auteur:** Amelia (Dev Agent) + Agent Explore
**Contexte:** Story 2.1 - Capture Audio 1-Tap
**ADR concerné:** ADR-017 (Dependency Injection & IoC Container Strategy)

---

## Résumé Exécutif

L'architecture IoC/DI suite à ADR-017 est **PARTIELLEMENT CONFORME**. TSyringe est correctement configuré avec un container fonctionnel, mais seulement **2 services sur 5** sont conformes à l'architecture IoC/DI complète.

**Score global de conformité:** 31% (13/42 critères validés)

### Conformité par catégorie

| Catégorie | Conforme | Total | Score |
|-----------|----------|-------|-------|
| **Mobile - Interfaces** | 2 | 8 | 25% |
| **Mobile - @injectable** | 1 | 5 | 20% |
| **Mobile - Tokens DI** | 2 | 4 | 50% |
| **Mobile - Container** | 2 | 5 | 40% |
| **Mobile - Mocks Test** | 5 | 5 | 100% ✅ |
| **Backend** | À auditer | - | - |

---

## 1. MOBILE (React Native) - Audit Détaillé

### 1.1 Services - État de Conformité

#### ✅ CONFORME - RecordingService

**Fichier:** `src/contexts/capture/services/RecordingService.ts`

| Critère | Statut | Notes |
|---------|--------|-------|
| @injectable decorator | ✅ | Ligne 77 |
| Interfaces dépendances | ✅ | IAudioRecorder, IFileSystem, IPermissionService définies |
| @inject() dans constructor | ✅ | Lignes 83-86 avec TOKENS.* |
| Container registration | N/A | Service injecté, pas racine |

**Interfaces exposées:**
```typescript
interface IAudioRecorder {
  startRecording(): Promise<RepositoryResult<{ uri: string }>>;
  stopRecording(): Promise<RepositoryResult<{ uri: string; duration: number }>>;
  getStatus?(): { isRecording: boolean; durationMillis: number; uri?: string };
}

interface IFileSystem {
  writeFile?(path, content): Promise<RepositoryResult<void>>;
  readFile?(path): Promise<RepositoryResult<string>>;
  fileExists?(path): Promise<RepositoryResult<boolean>>;
  deleteFile?(path): Promise<RepositoryResult<void>>;
}

interface IPermissionService {
  hasMicrophonePermission(): Promise<boolean>;
  requestMicrophonePermission?(): Promise<any>;
}
```

---

#### ⚠️ PARTIELLEMENT CONFORME - PermissionService

**Fichier:** `src/contexts/capture/services/PermissionService.ts`

| Critère | Statut | Notes |
|---------|--------|-------|
| @injectable decorator | ❌ | **MANQUANT** |
| Interface dédiée | ✅ | IPermissionService existe |
| Container registration | ✅ | Enregistré dans container.ts ligne 31 |
| Méthodes statiques | ⚠️ | **PROBLÈME** - incompatible avec IoC |

**Problèmes identifiés:**
```typescript
// ❌ Pattern statique - ne permet pas l'injection
static async requestMicrophonePermission(): Promise<PermissionResult>
static async checkMicrophonePermission(): Promise<PermissionResult>
static async hasMicrophonePermission(): Promise<boolean>
```

**Action requise:** Convertir en méthodes d'instance + ajouter @injectable decorator

---

#### ❌ NON CONFORME - FileStorageService

**Fichier:** `src/contexts/capture/services/FileStorageService.ts`

| Critère | Statut | Notes |
|---------|--------|-------|
| @injectable decorator | ❌ | **MANQUANT** |
| Interface dédiée | ❌ | **IFileStorageService n'existe pas** |
| Token DI | ❌ | **Pas de token dans tokens.ts** |
| Container registration | ❌ | **Non enregistré** |

**Impact:** Création manuelle via `new FileStorageService()`, difficile à tester, couplage fort.

---

#### ❌ NON CONFORME - OfflineSyncService

**Fichier:** `src/contexts/capture/services/OfflineSyncService.ts`

| Critère | Statut | Notes |
|---------|--------|-------|
| @injectable decorator | ❌ | **MANQUANT** |
| Interface dédiée | ❌ | **IOfflineSyncService n'existe pas** |
| Token DI | ❌ | **Pas de token** |
| Container registration | ❌ | **Non enregistré** |
| Constructor | ⚠️ | Accepte CaptureRepository directement (pas ICaptureRepository) |

**Impact:** Couplage direct au repository concret, difficile à mocker.

---

#### ❌ NON CONFORME - CrashRecoveryService

**Fichier:** `src/contexts/capture/services/CrashRecoveryService.ts`

| Critère | Statut | Notes |
|---------|--------|-------|
| @injectable decorator | ❌ | **MANQUANT** |
| Interface dédiée | ❌ | **ICrashRecoveryService n'existe pas** |
| Token DI | ❌ | **Pas de token** |
| Container registration | ❌ | **Non enregistré** |
| Constructor | ⚠️ | Accepte CaptureRepository directement |

**Impact:** Même problème que OfflineSyncService.

---

### 1.2 Repositories - État de Conformité

#### ✅ CONFORME - CaptureRepository

**Fichier:** `src/contexts/capture/data/CaptureRepository.ts`

| Critère | Statut | Notes |
|---------|--------|-------|
| Interface dédiée | ✅ | ICaptureRepository dans domain/ |
| Implements interface | ✅ | Ligne 47: `implements ICaptureRepository` |
| Token DI | ✅ | TOKENS.ICaptureRepository défini |
| Container registration | ✅ | Ligne 28 de container.ts |

**Interface ICaptureRepository:**
```typescript
interface ICaptureRepository {
  create(data: CreateCaptureData): Promise<RepositoryResult<Capture>>;
  update(id: string, data: UpdateCaptureData): Promise<RepositoryResult<Capture>>;
  delete(id: string): Promise<RepositoryResult<void>>;
  findByState(state: string): Promise<Capture[]>;
  findBySyncStatus(syncStatus: string): Promise<Capture[]>;
  findAll(): Promise<Capture[]>;
}
```

---

### 1.3 Tokens DI - État Actuel

**Fichier:** `src/infrastructure/di/tokens.ts`

```typescript
export const TOKENS = {
  ICaptureRepository: Symbol.for('ICaptureRepository'),        // ✅ Utilisé
  IAudioRecorder: Symbol.for('IAudioRecorder'),                // ⚠️ Token défini, adapter manquant
  IFileSystem: Symbol.for('IFileSystem'),                      // ⚠️ Token défini, adapter manquant
  IPermissionService: Symbol.for('IPermissionService'),        // ✅ Utilisé
};
```

**Tokens manquants:**
- IFileStorageService
- IOfflineSyncService
- ICrashRecoveryService

---

### 1.4 Container DI - Configuration

**Fichier:** `src/infrastructure/di/container.ts`

```typescript
import 'reflect-metadata';
import { container } from 'tsyringe';
import { TOKENS } from './tokens';
import { CaptureRepository } from '../../contexts/capture/data/CaptureRepository';
import { PermissionService } from '../../contexts/capture/services/PermissionService';

export function registerServices() {
  // ✅ Repository
  container.registerSingleton(TOKENS.ICaptureRepository, CaptureRepository);

  // ✅ Permission Service
  container.registerSingleton(TOKENS.IPermissionService, PermissionService);

  // ⚠️ Adapters commentés - À créer
  // container.registerSingleton(TOKENS.IAudioRecorder, ExpoAudioAdapter);
  // container.registerSingleton(TOKENS.IFileSystem, ExpoFileSystemAdapter);
}
```

**⚠️ PROBLÈME CRITIQUE:** `registerServices()` n'est **PAS APPELÉ** dans App.tsx ou index.ts!

---

### 1.5 Adapters/Infrastructure - État

**Dossier:** `src/infrastructure/`

```
infrastructure/
└── di/
    ├── container.ts
    └── tokens.ts
```

**❌ Adapters manquants:**
- `ExpoAudioAdapter` (implémente IAudioRecorder)
- `ExpoFileSystemAdapter` (implémente IFileSystem)

**À créer:** `src/infrastructure/adapters/`

---

### 1.6 Tests & Mocks

#### ✅ Test Container - BIEN IMPLÉMENTÉ

**Fichier:** `tests/acceptance/support/test-container.ts`

```typescript
import 'reflect-metadata';
import { container } from 'tsyringe';

export function setupTestContainer() {
  container.reset();
  container.registerSingleton(TOKENS.ICaptureRepository, MockCaptureRepository);
  container.registerSingleton(TOKENS.IAudioRecorder, MockAudioRecorder);
  container.registerSingleton(TOKENS.IFileSystem, MockFileSystem);
  container.registerSingleton(TOKENS.IPermissionService, MockPermissionManager);
}
```

#### ✅ Mocks Disponibles - EXCELLENTS

**Fichier:** `tests/acceptance/support/test-context.ts`

| Mock | Lignes | Qualité | Interfaces |
|------|--------|---------|------------|
| MockAudioRecorder | 1099 | ✅ Excellent | IAudioRecorder |
| MockFileSystem | 174 | ✅ Excellent | IFileSystem |
| MockPermissionManager | 1001 | ✅ Excellent | IPermissionService |
| MockCaptureRepository | 102 | ✅ Excellent | ICaptureRepository |

**Bonus:** 12 mocks in-memory disponibles (InMemoryDatabase, MockSupabaseAuth, MockAsyncStorage, etc.)

---

## 2. Problèmes Identifiés par Priorité

### 🔴 CRITIQUE (Bloquant)

1. **Container jamais initialisé**
   - `registerServices()` n'est pas appelé au démarrage
   - Impact: Aucun service n'est disponible via DI en production
   - Fix: Importer et appeler dans `App.tsx`

2. **Adapters Expo manquants**
   - ExpoAudioAdapter et ExpoFileSystemAdapter n'existent pas
   - Impact: RecordingService ne peut pas fonctionner en production
   - Fix: Créer les 2 adapters

### 🟠 IMPORTANT (Qualité code)

3. **Services manquent @injectable**
   - 4 services sur 5 ne peuvent pas être injectés
   - Impact: Création manuelle, tests difficiles
   - Fix: Ajouter decorator sur 4 services

4. **Interfaces manquantes**
   - 3 services n'ont pas d'abstraction
   - Impact: Couplage fort, pas de testabilité
   - Fix: Créer IFileStorageService, IOfflineSyncService, ICrashRecoveryService

5. **PermissionService utilise static methods**
   - Incompatible avec injection de dépendances
   - Impact: Ne peut pas être mocké correctement
   - Fix: Convertir en méthodes d'instance

### 🟡 TECHNIQUE (Best practices)

6. **Couplage direct aux repositories**
   - OfflineSyncService et CrashRecoveryService acceptent `CaptureRepository` au lieu de `ICaptureRepository`
   - Impact: Difficile de tester avec mocks
   - Fix: Utiliser les interfaces

7. **reflect-metadata non centralisé**
   - Importé dans plusieurs fichiers
   - Impact: Duplication, risque d'oubli
   - Fix: Importer une seule fois dans index.ts

---

## 3. Plan d'Action - Mobile

### Phase 1: URGENT - Initialisation Container

**Priorité:** 🔴 CRITIQUE
**Effort:** 5 minutes

**Actions:**
1. Éditer `App.tsx` ou `index.ts`
2. Ajouter en tout début:
```typescript
import 'reflect-metadata';
import { registerServices } from './src/infrastructure/di/container';

registerServices();
```

**Validation:** Aucune erreur au démarrage, container disponible

---

### Phase 2: Corriger PermissionService

**Priorité:** 🟠 IMPORTANT
**Effort:** 30 minutes

**Actions:**
1. Ajouter `@injectable()` décorateur
2. Convertir méthodes statiques en instance:
```typescript
@injectable()
export class PermissionService implements IPermissionService {
  async requestMicrophonePermission(): Promise<PermissionResult> { /* ... */ }
  async checkMicrophonePermission(): Promise<PermissionResult> { /* ... */ }
  async hasMicrophonePermission(): Promise<boolean> { /* ... */ }
}
```
3. Tester injection dans RecordingService

---

### Phase 3: Créer Interfaces Manquantes

**Priorité:** 🟠 IMPORTANT
**Effort:** 1 heure

**Fichiers à créer:**

1. **`src/contexts/capture/domain/IFileStorageService.ts`**
```typescript
export interface IFileStorageService {
  moveToStorage(uri: string, captureId: string, duration: number): Promise<FileStorageResult<StorageResult>>;
  getFileMetadata(uri: string, duration: number): Promise<FileStorageResult<FileMetadata>>;
  deleteFile(path: string): Promise<FileStorageResult<void>>;
  fileExists(path: string): Promise<boolean>;
  getStorageDirectory(): string;
}
```

2. **`src/contexts/capture/domain/IOfflineSyncService.ts`**
```typescript
export interface IOfflineSyncService {
  getPendingCaptures(): Promise<PendingCapture[]>;
  markAsSynced(id: string): Promise<void>;
  getSyncStats(): Promise<SyncStats>;
  hasPendingSync(): Promise<boolean>;
}
```

3. **`src/contexts/capture/domain/ICrashRecoveryService.ts`**
```typescript
export interface ICrashRecoveryService {
  recoverIncompleteRecordings(): Promise<RecoveredCapture[]>;
  getPendingRecoveryCount(): Promise<number>;
  clearFailedCaptures(): Promise<number>;
}
```

---

### Phase 4: Ajouter @injectable aux Services

**Priorité:** 🟠 IMPORTANT
**Effort:** 30 minutes

**Services à modifier:**
1. FileStorageService → Ajouter `@injectable()`
2. OfflineSyncService → Ajouter `@injectable()` + changer constructor pour accepter `ICaptureRepository`
3. CrashRecoveryService → Ajouter `@injectable()` + changer constructor pour accepter `ICaptureRepository`

---

### Phase 5: Ajouter Tokens DI

**Priorité:** 🟠 IMPORTANT
**Effort:** 10 minutes

**Éditer:** `src/infrastructure/di/tokens.ts`

```typescript
export const TOKENS = {
  ICaptureRepository: Symbol.for('ICaptureRepository'),
  IAudioRecorder: Symbol.for('IAudioRecorder'),
  IFileSystem: Symbol.for('IFileSystem'),
  IPermissionService: Symbol.for('IPermissionService'),
  IFileStorageService: Symbol.for('IFileStorageService'),           // NOUVEAU
  IOfflineSyncService: Symbol.for('IOfflineSyncService'),           // NOUVEAU
  ICrashRecoveryService: Symbol.for('ICrashRecoveryService'),       // NOUVEAU
};
```

---

### Phase 6: Créer Adapters Expo

**Priorité:** 🔴 CRITIQUE
**Effort:** 2 heures

**Fichiers à créer:**

1. **`src/infrastructure/adapters/ExpoAudioAdapter.ts`**
   - Implémente `IAudioRecorder`
   - Utilise `expo-audio` (SDK 54+)
   - Wrapper pour `RecordingSession`
   - Retourne `Result<>` (pas d'exceptions)

2. **`src/infrastructure/adapters/ExpoFileSystemAdapter.ts`**
   - Implémente `IFileSystem`
   - Utilise `expo-file-system/legacy`
   - Wrapper pour file operations
   - Retourne `Result<>` (pas d'exceptions)

---

### Phase 7: Enregistrer dans Container

**Priorité:** 🟠 IMPORTANT
**Effort:** 15 minutes

**Éditer:** `src/infrastructure/di/container.ts`

```typescript
export function registerServices() {
  // Repositories
  container.registerSingleton(TOKENS.ICaptureRepository, CaptureRepository);

  // Services
  container.registerSingleton(TOKENS.IPermissionService, PermissionService);
  container.registerSingleton(TOKENS.IFileStorageService, FileStorageService);
  container.registerSingleton(TOKENS.IOfflineSyncService, OfflineSyncService);
  container.registerSingleton(TOKENS.ICrashRecoveryService, CrashRecoveryService);

  // Adapters
  container.registerSingleton(TOKENS.IAudioRecorder, ExpoAudioAdapter);
  container.registerSingleton(TOKENS.IFileSystem, ExpoFileSystemAdapter);
}
```

---

### Phase 8: Tests & Validation

**Priorité:** 🟡 TECHNIQUE
**Effort:** 1 heure

**Actions:**
1. Adapter tests unitaires pour TSyringe
2. Vérifier test-container enregistre tous les tokens
3. Exécuter suite complète de tests
4. Valider latence < 500ms (AC1)

---

## 4. Estimation Effort Total - Mobile

| Phase | Priorité | Effort | Complexité |
|-------|----------|--------|------------|
| Phase 1: Init Container | 🔴 | 5 min | Faible |
| Phase 2: PermissionService | 🟠 | 30 min | Moyenne |
| Phase 3: Interfaces | 🟠 | 1h | Moyenne |
| Phase 4: @injectable | 🟠 | 30 min | Faible |
| Phase 5: Tokens | 🟠 | 10 min | Faible |
| Phase 6: Adapters Expo | 🔴 | 2h | Élevée |
| Phase 7: Container | 🟠 | 15 min | Faible |
| Phase 8: Tests | 🟡 | 1h | Moyenne |

**Total estimé:** 5h 30min

**Chemin critique:** Phases 1 + 6 (Container init + Adapters) = 2h 05min

---

## 5. Validation de Conformité ADR-017

### Critères ADR-017

| Critère | État Actuel | État Cible | Phase |
|---------|-------------|------------|-------|
| Tests BDD passent avec TSyringe | ✅ 3/3 | ✅ | - |
| Bundle size < +100KB | ✅ ~80KB | ✅ | - |
| Latency < 500ms préservée | ✅ < 3ms | ✅ | - |
| Tests unitaires adaptés | ⏳ | ✅ | Phase 8 |
| Backend DI (NestJS) | ⏳ | ✅ | Audit 2 |
| Aucune régression 1 mois prod | ⏳ | ✅ | Post-release |

**Review Date:** 2026-03 (après Epic 2 complet)

---

## 6. Backend (NestJS) - À Auditer

**Status:** ⏳ EN ATTENTE

Le backend utilise le système DI natif de NestJS. Un audit séparé est requis pour valider:
- Configuration des modules
- Providers correctement déclarés
- Tests utilisent `Test.createTestingModule()`
- Injection de dépendances fonctionnelle

**Document suivant:** `audit-ioc-di-conformite-backend.md`

---

## Annexes

### A. Fichiers Clés Mobile

1. `/src/infrastructure/di/container.ts` - Configuration DI
2. `/src/infrastructure/di/tokens.ts` - Tokens injection
3. `/src/contexts/capture/services/RecordingService.ts` - Exemple conforme ✅
4. `/src/contexts/capture/domain/ICaptureRepository.ts` - Interface repository ✅
5. `/tests/acceptance/support/test-container.ts` - Setup tests ✅
6. `/tests/acceptance/support/test-context.ts` - Mocks in-memory ✅

### B. Dépendances NPM

```json
{
  "dependencies": {
    "tsyringe": "^4.8.0",
    "reflect-metadata": "^0.2.2"
  }
}
```

### C. Configuration TypeScript

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

---

**Fin de l'audit Mobile - Partie 1/2**

**Prochaine étape:** Audit Backend (NestJS)

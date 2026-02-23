# Story 17.2: Chiffrement transparent des données sensibles dans OP-SQLite

Status: pending

## Story

En tant qu'**utilisateur**,
je veux **que mes captures et transcriptions soient chiffrées dans la base de données locale**,
afin de **rendre mes données inaccessibles même en cas d'extraction physique du fichier de base de données**.

## Context

**Dépend de Story 17.1** — `IEncryptionService` doit être disponible dans le conteneur DI avant d'implémenter cette story.

Les données sensibles stockées dans OP-SQLite :
- `captures.raw_content` — contenu brut (texte saisi ou retranscrit)
- `captures.normalized_text` — texte normalisé après transcription
- `thoughts.content` — contenu des pensées extraites par l'IA
- Chemins des fichiers audio (optionnel — à évaluer)

Un `EncryptionService` existait déjà (Story 6.1) avec les helpers `encryptCaptureRecord` / `decryptCaptureRecord`, mais ils utilisaient un singleton hors DI. Ces helpers sont supprimés en Story 17.1 et remplacés par une intégration directe dans les repositories.

**Stratégie** : Auto-encrypt à l'écriture, auto-decrypt à la lecture dans les repositories. Transparent pour les couches services/hooks.

**Migration** : Les données existantes non chiffrées doivent être migrées au premier démarrage après mise à jour (migration OP-SQLite).

## Acceptance Criteria

### AC1 — raw_content chiffré à l'écriture dans CaptureRepository

**Given** une capture est créée ou mise à jour avec un `raw_content`
**When** le repository persiste la donnée dans OP-SQLite
**Then** `raw_content` est stocké chiffré (format `U2FsdGVkX1...`)
**And** le serveur ne reçoit jamais le contenu en clair lors de la synchronisation

### AC2 — raw_content déchiffré à la lecture depuis CaptureRepository

**Given** une capture est lue depuis OP-SQLite
**When** le repository retourne l'entité domaine
**Then** `raw_content` est déchiffré de façon transparente
**And** les couches supérieures (services, hooks, UI) reçoivent le contenu en clair

### AC3 — normalized_text chiffré/déchiffré de façon identique

**Given** une capture possède un `normalized_text` (après transcription)
**When** le repository écrit ou lit la donnée
**Then** `normalized_text` suit le même cycle encrypt/decrypt que `raw_content`

### AC4 — thoughts.content chiffré dans KnowledgeRepository

**Given** une pensée (`Thought`) est créée par la digestion IA
**When** le repository persiste la donnée dans OP-SQLite
**Then** `thoughts.content` est stocké chiffré
**And** la lecture déchiffre transparentement

### AC5 — Données inaccessibles hors app

**Given** un attaquant accède au fichier `.db` d'OP-SQLite (extraction physique)
**When** il lit la table `captures`
**Then** les colonnes `raw_content` et `normalized_text` contiennent uniquement du ciphertext Base64
**And** sans la clé SecureStore, le contenu est illisible

### AC6 — Migration des données existantes non chiffrées

**Given** une installation existante avec des données en clair
**When** l'application est mise à jour et démarre pour la première fois
**Then** une migration OP-SQLite chiffre les données existantes en clair
**And** la migration est idempotente (re-run = no-op si données déjà chiffrées)
**And** aucune donnée n'est perdue durant la migration

### AC7 — Graceful handling si clé non disponible

**Given** l'utilisateur a besoin d'une clé de récupération (après réinstallation, avant Story 17.4)
**When** le repository tente de déchiffrer sans clé disponible
**Then** le service retourne un `Result.fail('ENCRYPTION_KEY_NOT_AVAILABLE')`
**And** l'UI affiche un message explicatif (non bloquant sur la navigation)

### AC8 — Conformité Result Pattern (ADR-023)

**Given** n'importe quelle opération de chiffrement dans les repositories
**When** une erreur survient (crypto erreur, clé absente)
**Then** le repository retourne un `Result.fail(...)` cohérent avec ADR-023

## Tasks / Subtasks

- [ ] **Task 1 — Modifier CaptureRepository** (AC: #1, #2, #3, #7, #8)
  - [ ] 1.1 — Injecter `IEncryptionService` dans `CaptureRepository` via DI
  - [ ] 1.2 — Chiffrer `raw_content` et `normalized_text` dans `save()` / `update()`
  - [ ] 1.3 — Déchiffrer `raw_content` et `normalized_text` dans `findById()` / `findAll()`
  - [ ] 1.4 — Gérer le cas `isReady() === false` → retourner `Result.fail('ENCRYPTION_KEY_NOT_AVAILABLE')`
  - [ ] 1.5 — Utiliser `isEncrypted()` pour détecter les données en clair (rétrocompatibilité migration)

- [ ] **Task 2 — Modifier KnowledgeRepository** (AC: #4, #8)
  - [ ] 2.1 — Injecter `IEncryptionService` dans le repository Knowledge
  - [ ] 2.2 — Chiffrer/déchiffrer `thoughts.content` à l'écriture/lecture

- [ ] **Task 3 — Évaluer le chiffrement des chemins audio** (AC: #5)
  - [ ] 3.1 — Analyser si les chemins de fichiers audio révèlent des informations sensibles
  - [ ] 3.2 — Si oui : chiffrer `audio_uri` dans CaptureRepository
  - [ ] 3.3 — Documenter la décision dans Dev Notes

- [ ] **Task 4 — Migration OP-SQLite** (AC: #6)
  - [ ] 4.1 — Créer la migration `migrate_encrypt_existing_captures.ts` dans `src/database/migrations/`
  - [ ] 4.2 — Lire toutes les captures non chiffrées (`WHERE raw_content NOT LIKE 'U2FsdGVkX1%' OR raw_content IS NOT NULL`)
  - [ ] 4.3 — Chiffrer et sauvegarder en batch (transactions pour atomicité)
  - [ ] 4.4 — Même logique pour `thoughts.content`
  - [ ] 4.5 — Vérifier l'idempotence (run deux fois = même résultat)

- [ ] **Task 5 — Tests** (AC: tous)
  - [ ] 5.1 — Tests unitaires `CaptureRepository.encryption.test.ts`
  - [ ] 5.2 — Mocker `IEncryptionService` dans les tests repository
  - [ ] 5.3 — Écrire `story-17-2.feature` avec scénarios Gherkin
  - [ ] 5.4 — Écrire les step definitions `story-17-2.test.ts`
  - [ ] 5.5 — Test de migration : données en clair → chiffrées

## Dev Notes

### Fichiers concernés

| Fichier | Changement |
|---------|-----------|
| `mobile/src/contexts/capture/data/CaptureRepository.ts` | Injecter IEncryptionService, encrypt/decrypt |
| `mobile/src/contexts/knowledge/data/KnowledgeRepository.ts` | Injecter IEncryptionService, encrypt/decrypt |
| `mobile/src/database/migrations/migrate_encrypt_existing_captures.ts` | Nouveau — migration chiffrement |
| `mobile/src/database/index.ts` | Enregistrer la nouvelle migration |
| `mobile/tests/acceptance/features/story-17-2.feature` | Nouveau — Gherkin |
| `mobile/tests/acceptance/story-17-2.test.ts` | Nouveau — Step definitions |

### Détection données en clair vs chiffrées

```typescript
// AES encrypted data from CryptoJS starts with "U2FsdGVkX1" (Base64 "Salted__")
const isEncrypted = (data: string) => data?.startsWith('U2FsdGVkX1') ?? false;
```

Utiliser cette détection pour la migration et pour éviter le double chiffrement.

### Champs à chiffrer

| Table | Colonne | Sensibilité | Chiffrer |
|-------|---------|------------|---------|
| `captures` | `raw_content` | Haute — contenu pensée | ✅ |
| `captures` | `normalized_text` | Haute — transcription | ✅ |
| `captures` | `audio_uri` | Faible — chemin local | À évaluer |
| `thoughts` | `content` | Haute — analyse IA | ✅ |
| `todos` | `description` | Moyenne — action extraite | Optionnel (à décider) |

### Pattern d'injection dans les repositories

```typescript
@injectable()
export class CaptureRepository implements ICaptureRepository {
  constructor(
    @inject(TOKENS.IEncryptionService)
    private readonly encryptionService: IEncryptionService,
  ) {}

  async save(capture: Capture): Promise<Result<void>> {
    const encRaw = this.encryptionService.encrypt(capture.rawContent);
    if (!encRaw.isSuccess) return Result.fail(encRaw.error);
    // ... persist
  }
}
```

### Synchronisation cloud (Story 6.x)

La synchronisation vers le backend doit synchroniser les données **chiffrées** (jamais en clair). Le backend ne peut pas déchiffrer → zero-knowledge maintenu. Vérifier le comportement du sync protocol OP-SQLite avec les champs chiffrés.

## Dev Agent Record

### Agent Model Used

TBD

### Completion Notes List

TBD

### File List

TBD

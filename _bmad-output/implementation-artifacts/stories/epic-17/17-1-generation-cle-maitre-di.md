# Story 17.1: Génération de la clé maître AES-256 et intégration dans le conteneur DI

Status: pending

## Story

En tant qu'**utilisateur**,
je veux **que mes données soient chiffrées avec une clé maitresse générée localement sur mon appareil**,
afin de **garantir que le serveur ne peut jamais accéder à mes données en clair (zero-knowledge)**.

## Context

Un `EncryptionService` existe déjà à `infrastructure/security/EncryptionService.ts` (154 lignes, AES-256 via `crypto-js` + `expo-secure-store`). Il génère et stocke une clé locale mais :
- Il est **en dehors du conteneur DI** (pattern singleton manuel hors tsyringe)
- Il n'existe pas d'**interface `IEncryptionService`** utilisable via les tokens
- Il n'est pas lié au **cycle de vie du compte** (pas de génération au signup, pas de suppression au delete account)
- Il n'est pas lié au bounded context `security/` ou `identity/`

Cette story refactore le service existant pour l'intégrer proprement dans l'architecture DDD/DI du projet.

**Clé SecureStore** : `pensine.master_key` (renommée depuis `pensine_encryption_master_key`)

**Contrainte zero-knowledge** : la clé ne doit jamais être transmise au serveur. Elle vit uniquement dans le SecureStore local (hardware-backed Keychain iOS / Keystore Android).

## Acceptance Criteria

### AC1 — Interface IEncryptionService définie

**Given** le bounded context `security/` (ou sous-module de `identity/`)
**When** un développeur veut utiliser le chiffrement
**Then** une interface `IEncryptionService` est disponible dans `domain/`
**And** elle expose au minimum : `encrypt(plaintext: string): Result<string>`, `decrypt(ciphertext: string): Result<string>`, `isReady(): boolean`

### AC2 — EncryptionService enregistré dans le conteneur DI

**Given** le bootstrap de l'application (`container.ts`)
**When** l'application démarre
**Then** `IEncryptionService` est résolvable via `container.resolve(TOKENS.IEncryptionService)`
**And** l'instance utilise `expo-secure-store` pour la clé maître
**And** l'enregistrement respecte ADR-021 (Transient First — vérifier si Transient ou Singleton est justifié ici)

### AC3 — Génération de la clé au signup

**Given** un utilisateur crée un nouveau compte (hook post-signup `onSignUp`)
**When** le compte est créé avec succès
**Then** une clé maître AES-256 (32 bytes aléatoires) est générée via `crypto-js`
**And** elle est stockée dans SecureStore sous la clé `pensine.master_key`
**And** aucune clé n'est envoyée au serveur à cette étape

### AC4 — Chargement de la clé existante au login

**Given** un utilisateur se reconnecte sur le même device
**When** l'application démarre et l'utilisateur est authentifié
**Then** la clé existante est chargée depuis SecureStore
**And** `IEncryptionService.isReady()` retourne `true`

### AC5 — Absence de clé détectée après réinstallation

**Given** un utilisateur réinstalle l'application
**When** l'application démarre et l'utilisateur se connecte
**Then** l'absence de clé locale est détectée (`SecureStore.getItemAsync` retourne `null`)
**And** un flag `needsKeyRecovery` est émis (événement domaine ou store Zustand) pour déclencher le flow Story 17.4

### AC6 — Suppression de la clé à la suppression de compte

**Given** un utilisateur supprime son compte
**When** la suppression est confirmée et traitée
**Then** la clé maître est supprimée du SecureStore (`deleteItemAsync`)
**And** `IEncryptionService.isReady()` retourne `false`

### AC7 — Conformité Result Pattern (ADR-023)

**Given** n'importe quelle opération de `IEncryptionService`
**When** une erreur survient (SecureStore inaccessible, crypto erreur)
**Then** le service retourne un `Result.fail(...)` au lieu de `throw`
**And** aucun `throw new Error(...)` non capturé n'est présent dans l'implémentation

## Tasks / Subtasks

- [ ] **Task 1 — Créer l'interface IEncryptionService** (AC: #1)
  - [ ] 1.1 — Créer `src/contexts/identity/domain/IEncryptionService.ts` (ou `security/domain/`)
  - [ ] 1.2 — Définir les méthodes : `encrypt`, `decrypt`, `isReady`, `deleteKey`
  - [ ] 1.3 — Typer les retours avec `Result<T>` depuis `shared/domain/Result.ts`

- [ ] **Task 2 — Refactorer EncryptionService** (AC: #2, #7)
  - [ ] 2.1 — Implémenter `IEncryptionService` dans `infrastructure/security/EncryptionService.ts`
  - [ ] 2.2 — Remplacer le singleton manuel par une classe injectable (`@injectable()`)
  - [ ] 2.3 — Wrapper toutes les opérations dans des `Result<T>` (ADR-023)
  - [ ] 2.4 — Renommer la clé SecureStore de `pensine_encryption_master_key` → `pensine.master_key`
  - [ ] 2.5 — Supprimer `getEncryptionService()` et les helpers `encryptCaptureRecord/decryptCaptureRecord` (ils seront dans les repositories — Story 17.2)

- [ ] **Task 3 — Ajouter le token DI et enregistrer** (AC: #2)
  - [ ] 3.1 — Ajouter `IEncryptionService: Symbol('IEncryptionService')` dans `tokens.ts`
  - [ ] 3.2 — Enregistrer dans `container.ts` : `container.register<IEncryptionService>(TOKENS.IEncryptionService, { useClass: EncryptionService })`
  - [ ] 3.3 — Décider Transient vs Singleton (Singleton justifié : la clé en mémoire évite les appels SecureStore répétés — documenter dans Dev Notes)

- [ ] **Task 4 — Hook cycle de vie compte** (AC: #3, #4, #5, #6)
  - [ ] 4.1 — Localiser le hook `onSignUp` dans `identity/` (ou `bootstrap.ts`)
  - [ ] 4.2 — Appeler `IEncryptionService.generateAndStoreKey()` après signup réussi
  - [ ] 4.3 — Appeler `IEncryptionService.loadKey()` au login (si clé absente → émettre `KeyMissingEvent`)
  - [ ] 4.4 — Appeler `IEncryptionService.deleteKey()` dans le flow de suppression de compte

- [ ] **Task 5 — Tests** (AC: tous)
  - [ ] 5.1 — Tests unitaires `EncryptionService.test.ts` : generate, load, delete, Result errors
  - [ ] 5.2 — Écrire `story-17-1.feature` avec scénarios Gherkin
  - [ ] 5.3 — Écrire les step definitions `story-17-1.test.ts`
  - [ ] 5.4 — Mocker `expo-secure-store` dans `test-context.ts`

## Dev Notes

### Fichiers concernés

| Fichier | Changement |
|---------|-----------|
| `mobile/src/contexts/identity/domain/IEncryptionService.ts` | Nouveau — interface |
| `mobile/src/infrastructure/security/EncryptionService.ts` | Refactoré — injectable + Result Pattern |
| `mobile/src/infrastructure/di/tokens.ts` | Ajouter `IEncryptionService` |
| `mobile/src/infrastructure/di/container.ts` | Enregistrer le service |
| `mobile/src/contexts/identity/hooks/useAuth.ts` | Hook signup/login/delete |
| `mobile/tests/acceptance/features/story-17-1.feature` | Nouveau — Gherkin |
| `mobile/tests/acceptance/story-17-1.test.ts` | Nouveau — Step definitions |
| `mobile/tests/acceptance/support/test-context.ts` | Ajouter mock SecureStore pour chiffrement |

### Décision Singleton vs Transient (ADR-021)

ADR-021 prescrit "Transient First" mais autorise les exceptions documentées. Pour `IEncryptionService` :
- **Singleton justifié** : la clé maître est chargée une fois depuis SecureStore (I/O lent) et mise en cache en mémoire pour performance
- **Documenter** cette exception dans le code avec un commentaire `// SINGLETON: master key cached in-memory, SecureStore I/O costly`

### Clés SecureStore utilisées

| Clé | Contenu | Lifecycle |
|-----|---------|-----------|
| `pensine.master_key` | Clé AES-256 raw (Base64) | Créé signup → Supprimé delete account |

### Dépendances existantes (déjà installées)

- `expo-secure-store@15.0.8` ✅
- `crypto-js@4.2.0` ✅

### Contrainte zéro-knowledge

La clé ne quitte jamais le device à cette étape. La synchronisation multi-device avec récupération après réinstallation est couverte par Story 17.4.

## Dev Agent Record

### Agent Model Used

TBD

### Completion Notes List

TBD

### File List

TBD

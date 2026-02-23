# Story 17.4: Sauvegarde chiffrée et récupération de la clé maître après réinstallation

Status: pending

## Story

En tant qu'**utilisateur**,
je veux **pouvoir retrouver l'accès à mes données après avoir désinstallé et réinstallé l'application**,
afin de **ne jamais perdre mes captures même sur un nouvel appareil ou après réinstallation**.

## Context

**Dépend de Story 17.1** — `IEncryptionService` doit exister dans le conteneur DI.

Après une réinstallation, SecureStore est vidé (iOS Keychain lié au bundle ID avec `keychainAccessible` par défaut). La clé maître est perdue → les données synchronisées depuis le cloud (chiffrées) seraient illisibles.

**Solution zero-knowledge** : Le mot de passe utilisateur sert à dériver une clé secondaire (`backupKey`) qui chiffre la clé maître. Le blob résultant est stocké sur le serveur. Le serveur ne voit jamais la clé maître.

**Mécanisme** :
```
Signup:
  masterKey = AES-256 random (32 bytes)
  salt = crypto random (16 bytes)
  backupKey = PBKDF2(password, salt, iterations=100000, hash=SHA-512)
  encryptedBlob = AES_ENCRYPT(masterKey, backupKey)
  → SecureStore.set('pensine.master_key', masterKey)  // local uniquement
  → POST /users/me/key-backup { blob: encryptedBlob, salt }  // serveur ne voit pas masterKey

Récupération après réinstall:
  → GET /users/me/key-backup → { blob, salt }
  → user saisit son mot de passe
  → backupKey = PBKDF2(password, salt, ...)
  → masterKey = AES_DECRYPT(blob, backupKey)
  → SecureStore.set('pensine.master_key', masterKey)
```

**Périmètre** : Mobile (flow signup + récupération) + Backend NestJS (nouveaux endpoints + nouvelle table).

## Acceptance Criteria

### AC1 — Backup blob créé au signup

**Given** un utilisateur crée un nouveau compte avec un mot de passe
**When** le signup est traité (Story 17.1 génère la clé maître)
**Then** un `backupKey` est dérivé via PBKDF2(password, salt, 100000 iterations, SHA-512)
**And** un `encryptedBlob = AES_ENCRYPT(masterKey, backupKey)` est créé
**And** `{ blob, salt }` est envoyé au backend via `POST /users/me/key-backup`
**And** le backend stocke uniquement le blob et le salt (jamais la masterKey, jamais le password)

### AC2 — Endpoint backend POST /users/me/key-backup

**Given** un utilisateur authentifié envoie `{ blob, salt }` au backend
**When** `POST /users/me/key-backup` est reçu
**Then** le backend stocke les données dans la table `user_key_backup` (`owner_id`, `blob`, `salt`, `created_at`, `updated_at`)
**And** une entrée existante pour cet utilisateur est remplacée (upsert)
**And** le backend retourne `201 Created`
**And** le backend ne déchiffre ni ne valide le contenu du blob

### AC3 — Endpoint backend GET /users/me/key-backup

**Given** un utilisateur authentifié demande son backup
**When** `GET /users/me/key-backup` est reçu
**Then** le backend retourne `{ blob, salt }` pour l'utilisateur authentifié
**And** si aucun backup n'existe : retourne `404 Not Found`
**And** un utilisateur ne peut accéder qu'à son propre backup (vérification `owner_id`)

### AC4 — Table backend `user_key_backup` avec migration TypeORM

**Given** le backend démarre après déploiement
**When** les migrations TypeORM sont exécutées
**Then** la table `user_key_backup` est créée avec les colonnes : `id` (UUID v7), `owner_id` (UUID, FK users), `blob` (TEXT, NOT NULL), `salt` (VARCHAR(64), NOT NULL), `created_at`, `updated_at`
**And** la migration est réversible (DOWN migration incluse)
**And** la table respecte ADR-026 (UUID v7, soft_deleted_at NON requis ici car données de sécurité)

### AC5 — Détection de clé absente au login (mobile)

**Given** un utilisateur se connecte après réinstallation
**When** l'application démarre et détecte l'absence de `pensine.master_key` dans SecureStore
**Then** le flow de récupération est déclenché automatiquement
**And** l'utilisateur est invité à saisir son mot de passe pour récupérer sa clé

### AC6 — Récupération complète de la clé maître

**Given** l'utilisateur saisit son mot de passe correct lors du flow de récupération
**When** `backupKey = PBKDF2(password, salt)` est recalculé
**Then** `masterKey = AES_DECRYPT(blob, backupKey)` est déchiffré avec succès
**And** la clé maître est restaurée dans SecureStore (`pensine.master_key`)
**And** `IEncryptionService.isReady()` retourne `true`
**And** les données synchronisées depuis le cloud sont à nouveau lisibles

### AC7 — Message d'erreur si mot de passe incorrect

**Given** l'utilisateur saisit un mot de passe incorrect lors du flow de récupération
**When** le déchiffrement AES échoue ou produit un résultat invalide
**Then** un message d'erreur clair est affiché ("Mot de passe incorrect — vos données restent protégées")
**And** l'utilisateur peut réessayer

### AC8 — Backup mis à jour lors d'un changement de mot de passe

**Given** un utilisateur change son mot de passe
**When** le nouveau mot de passe est confirmé
**Then** un nouveau `backupKey` est dérivé du nouveau mot de passe
**And** un nouveau blob est créé et envoyé via `POST /users/me/key-backup` (upsert)
**And** l'ancien backup est remplacé

### AC9 — Suppression du backup à la suppression de compte

**Given** un utilisateur supprime son compte
**When** la suppression est traitée côté backend
**Then** la ligne `user_key_backup` correspondante est supprimée (hard delete)

## Tasks / Subtasks

- [ ] **Task 1 — Implémenter PBKDF2 dans IEncryptionService** (AC: #1, #6)
  - [ ] 1.1 — Ajouter méthode `deriveBackupKey(password: string, salt: string): Result<string>` à `IEncryptionService`
  - [ ] 1.2 — Implémenter via `CryptoJS.PBKDF2(password, salt, { keySize: 32, iterations: 100000, hasher: CryptoJS.algo.SHA512 })`
  - [ ] 1.3 — Ajouter `generateSalt(): string` (16 bytes aléatoires, Base64)
  - [ ] 1.4 — Ajouter `createEncryptedBlob(masterKey: string, backupKey: string): Result<string>`
  - [ ] 1.5 — Ajouter `decryptBlob(blob: string, backupKey: string): Result<string>`

- [ ] **Task 2 — Service de sauvegarde mobile KeyBackupService** (AC: #1, #5, #6, #7, #8)
  - [ ] 2.1 — Créer `src/contexts/identity/services/KeyBackupService.ts`
  - [ ] 2.2 — Implémenter `backupKey(password: string): Result<void>` — PBKDF2 + AES + POST backend
  - [ ] 2.3 — Implémenter `recoverKey(password: string): Result<void>` — GET backend + PBKDF2 + AES decrypt + SecureStore
  - [ ] 2.4 — Enregistrer dans `container.ts`

- [ ] **Task 3 — Intégration dans le flow auth mobile** (AC: #1, #5, #6, #8)
  - [ ] 3.1 — Au signup : après génération clé (Story 17.1), appeler `KeyBackupService.backupKey(password)`
  - [ ] 3.2 — Au login sans clé locale : détecter `needsKeyRecovery` et afficher l'écran de récupération
  - [ ] 3.3 — Au changement de mot de passe : appeler `KeyBackupService.backupKey(newPassword)`

- [ ] **Task 4 — Écran de récupération mobile** (AC: #5, #6, #7)
  - [ ] 4.1 — Créer `src/screens/security/KeyRecoveryScreen.tsx`
  - [ ] 4.2 — Champ de saisie mot de passe + bouton "Récupérer mes données"
  - [ ] 4.3 — Spinner pendant le déchiffrement
  - [ ] 4.4 — Affichage d'erreur si mot de passe incorrect (AC7)
  - [ ] 4.5 — Redirection vers l'app principale après succès

- [ ] **Task 5 — Backend : entité et repository UserKeyBackup** (AC: #2, #3, #4)
  - [ ] 5.1 — Créer `UserKeyBackupEntity` dans `backend/src/modules/identity/domain/entities/`
  - [ ] 5.2 — Créer `UserKeyBackupRepository` dans `backend/src/modules/identity/infrastructure/`
  - [ ] 5.3 — Respecter ADR-026 (UUID v7 via `generateUuidV7()`, étendre `BaseEntity`)

- [ ] **Task 6 — Backend : contrôleur et migration TypeORM** (AC: #2, #3, #4, #9)
  - [ ] 6.1 — Créer `KeyBackupController` dans `backend/src/modules/identity/application/controllers/`
  - [ ] 6.2 — Implémenter `POST /users/me/key-backup` (upsert)
  - [ ] 6.3 — Implémenter `GET /users/me/key-backup`
  - [ ] 6.4 — Sécuriser avec `SupabaseAuthGuard` (uniquement l'utilisateur authentifié)
  - [ ] 6.5 — Créer migration TypeORM `CreateUserKeyBackupTable` (UP + DOWN)
  - [ ] 6.6 — Ajouter suppression cascade dans le flow RGPD (delete account)

- [ ] **Task 7 — Tests** (AC: tous)
  - [ ] 7.1 — Tests unitaires `KeyBackupService.test.ts` (mobile)
  - [ ] 7.2 — Tests unitaires backend `KeyBackupController.spec.ts`
  - [ ] 7.3 — Tests BDD backend `story-17-4.test.ts` (acceptance)
  - [ ] 7.4 — Écrire `story-17-4.feature` Gherkin (mobile + backend)
  - [ ] 7.5 — Scénario de réinstallation simulée : seed chiffré → suppression SecureStore → récupération

## Dev Notes

### Fichiers concernés — Mobile

| Fichier | Changement |
|---------|-----------|
| `mobile/src/contexts/identity/domain/IEncryptionService.ts` | Ajouter PBKDF2, generateSalt, createEncryptedBlob, decryptBlob |
| `mobile/src/infrastructure/security/EncryptionService.ts` | Implémenter les nouvelles méthodes |
| `mobile/src/contexts/identity/services/KeyBackupService.ts` | Nouveau |
| `mobile/src/infrastructure/di/tokens.ts` | Ajouter `IKeyBackupService` |
| `mobile/src/infrastructure/di/container.ts` | Enregistrer KeyBackupService |
| `mobile/src/screens/security/KeyRecoveryScreen.tsx` | Nouveau — écran récupération |
| `mobile/tests/acceptance/features/story-17-4.feature` | Nouveau — Gherkin |
| `mobile/tests/acceptance/story-17-4.test.ts` | Nouveau — Step definitions |

### Fichiers concernés — Backend

| Fichier | Changement |
|---------|-----------|
| `backend/src/modules/identity/domain/entities/user-key-backup.entity.ts` | Nouveau |
| `backend/src/modules/identity/infrastructure/user-key-backup.repository.ts` | Nouveau |
| `backend/src/modules/identity/application/controllers/key-backup.controller.ts` | Nouveau |
| `backend/src/migrations/[timestamp]-CreateUserKeyBackupTable.ts` | Nouveau |
| `backend/test/acceptance/features/story-17-4.feature` | Nouveau |
| `backend/test/acceptance/story-17-4.test.ts` | Nouveau |

### Schéma de la table `user_key_backup`

```sql
CREATE TABLE user_key_backup (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- UUIDv7 via generateUuidV7()
  owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  blob TEXT NOT NULL,           -- AES_ENCRYPT(masterKey, backupKey) — Base64
  salt VARCHAR(64) NOT NULL,    -- Salt PBKDF2 — Base64 (16 bytes)
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(owner_id)              -- Un seul backup par utilisateur
);
```

### Paramètres PBKDF2 justifiés

| Paramètre | Valeur | Justification |
|-----------|--------|--------------|
| Iterations | 100 000 | OWASP recommande ≥ 310 000 pour SHA-256 / ≥ 120 000 pour SHA-512 (2023). 100k est un minimum acceptable pour mobile (performance). Documenter et monitorer. |
| Hash | SHA-512 | Plus résistant aux attaques GPU que SHA-256 |
| Key size | 32 bytes (256 bits) | Correspond à la taille de la clé AES-256 |

**Note** : Si performance PBKDF2 inacceptable sur mobile (> 2s), envisager de déléguer à un worker thread ou d'abaisser les iterations avec avertissement.

### Sécurité du transport

- `blob` et `salt` transitent sur HTTPS (NFR11 — TLS obligatoire)
- Le backend ne peut pas reconstruire `masterKey` sans le mot de passe utilisateur → zero-knowledge maintenu
- Le `blob` est opaque pour le backend

### Gestion du changement de mot de passe

Le backup doit être mis à jour quand le mot de passe change, sinon la récupération utilisera l'ancien mot de passe. Si l'utilisateur oublie son mot de passe ET son backup est révoqué → **perte de données irréversible** (par design zero-knowledge). Afficher un avertissement clair lors de la configuration.

## Dev Agent Record

### Agent Model Used

TBD

### Completion Notes List

TBD

### File List

TBD

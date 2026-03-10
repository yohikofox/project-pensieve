# Story 27.1: PAT Backend — Table + PATService + PATGuard + Endpoints CRUD + Renew

Status: ready-for-dev

## Story

En tant qu'**utilisateur de l'API**,
je veux **pouvoir générer des Personal Access Tokens avec expiry absolue et scopes granulaires**,
afin de **permettre à des clients externes (MCP, scripts) d'accéder à mon compte de façon sécurisée et révocable**.

## Context & Motivation

Prérequis pour Epic 28 (MCP). Les PATs remplacent l'usage de tokens JWT Better Auth dans les contextes non-interactifs. Format : `pns_<32 bytes base64url>` — prefix identifiable pour détection de leaks.

**Renew = Option B** : nouveau token + révocation atomique de l'ancien.

## Acceptance Criteria

### AC1 : Création d'un PAT
**Given** un utilisateur authentifié (JWT Better Auth)
**When** il appelle `POST /api/auth/pat` avec `{ name, scopes, expiresInDays }`
**Then** un PAT est créé avec `expires_at = now() + expiresInDays`
**And** la réponse contient le token en clair **une seule fois** (`token: "pns_..."`)
**And** seul le hash SHA-256 est stocké en base (`token_hash`)
**And** le préfixe est stocké séparément pour affichage (`prefix: "pns_a1b2"`)

### AC2 : Scopes valides
**Given** une tentative de création avec un scope invalide
**When** la validation est effectuée
**Then** une erreur 400 est retournée avec la liste des scopes autorisés

**Scopes autorisés :**
- `captures:read`
- `thoughts:read`
- `ideas:read`
- `todos:read`
- `todos:write`

### AC3 : Listage des PATs
**Given** un utilisateur authentifié
**When** il appelle `GET /api/auth/pat`
**Then** la liste de ses PATs est retournée (jamais le token en clair — uniquement `prefix`, `name`, `scopes`, `expires_at`, `last_used_at`, `revoked_at`, `created_at`)

### AC4 : Modification d'un PAT (nom et scopes)
**Given** un utilisateur authentifié propriétaire du PAT
**When** il appelle `PATCH /api/auth/pat/:id` avec `{ name?, scopes? }`
**Then** le nom et/ou les scopes sont mis à jour
**And** le token lui-même n'est pas modifié

### AC5 : Renew d'un PAT (Option B — rotation)
**Given** un PAT actif appartenant à l'utilisateur
**When** il appelle `POST /api/auth/pat/:id/renew` avec `{ expiresInDays? }`
**Then** un nouveau token est généré (nouveau `token_hash`, nouveau `prefix`)
**And** l'ancien PAT est révoqué atomiquement (`revoked_at = now()`) dans la même transaction
**And** le nouveau token est retourné en clair **une seule fois**

### AC6 : Révocation d'un PAT
**Given** un PAT actif appartenant à l'utilisateur
**When** il appelle `DELETE /api/auth/pat/:id`
**Then** `revoked_at` est set à `now()` immédiatement
**And** toute requête ultérieure avec ce PAT reçoit 401

### AC7 : Authentification via PATGuard
**Given** une requête avec header `Authorization: Bearer pns_...`
**When** `PATGuard` est évalué
**Then** le token est hashé (SHA-256) et comparé à `token_hash` en base
**And** si le PAT est valide (non révoqué, non expiré) : la requête est autorisée
**And** `last_used_at` est mis à jour
**And** si le PAT est invalide : 401 Unauthorized

### AC8 : Vérification des scopes
**Given** un PAT valide avec scopes `['captures:read']`
**When** une requête accède à un endpoint protégé par scope `todos:write`
**Then** une erreur 403 Forbidden est retournée

### AC9 : Admin — gestion pour tout utilisateur
**Given** un admin authentifié (JWT admin)
**When** il appelle les mêmes endpoints avec query param `?userId=<uuid>`
**Then** les opérations s'appliquent au PAT de l'utilisateur ciblé
**And** un PAT admin ne peut pas agir sur les PATs d'autres admins

### AC10 : Traçabilité (dépend Story 26.1)
**Given** une requête authentifiée via PAT
**When** elle est traitée
**Then** le log d'ingress inclut `patId` et `userId` en plus du `traceId`

## Technical Specification

### Migration TypeORM

```sql
CREATE TABLE "personal_access_tokens" (
  "id"           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "user_id"      UUID NOT NULL REFERENCES "users"("id") ON DELETE CASCADE,
  "name"         VARCHAR(100) NOT NULL,
  "token_hash"   VARCHAR(64) NOT NULL UNIQUE,  -- SHA-256 hex
  "prefix"       VARCHAR(12) NOT NULL,          -- "pns_a1b2c3d4"
  "scopes"       TEXT[] NOT NULL DEFAULT '{}',
  "expires_at"   TIMESTAMPTZ NOT NULL,
  "last_used_at" TIMESTAMPTZ,
  "revoked_at"   TIMESTAMPTZ,
  "created_at"   TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX "IDX_PAT_USER_ID" ON "personal_access_tokens" ("user_id");
CREATE INDEX "IDX_PAT_TOKEN_HASH" ON "personal_access_tokens" ("token_hash");
```

### Format du token

```
pns_<32 bytes crypto.randomBytes().toString('base64url')>
Prefix affiché : 8 premiers chars après pns_ → "pns_a1b2c3d4"
Hash stocké : crypto.createHash('sha256').update(token).digest('hex')
```

### Module structure

```
backend/src/modules/pat/
├── pat.module.ts
├── domain/entities/personal-access-token.entity.ts
├── application/
│   ├── services/pat.service.ts
│   ├── controllers/pat.controller.ts
│   └── dto/
│       ├── create-pat.dto.ts
│       ├── update-pat.dto.ts
│       └── renew-pat.dto.ts
└── infrastructure/
    ├── guards/pat.guard.ts
    └── repositories/pat.repository.ts
```

## Tasks / Subtasks

### Task 1 : Migration + Entité (AC1, AC7)

- [ ] Subtask 1.1 : Créer migration TypeORM `1781000000000-CreatePersonalAccessTokensTable.ts`
- [ ] Subtask 1.2 : Créer entité `PersonalAccessToken` avec tous les champs
- [ ] Subtask 1.3 : Créer `PATRepository` avec méthodes `findByHash`, `findByUserId`, `findByIdAndUserId`

### Task 2 : PATService — génération et gestion (AC1, AC2, AC4, AC5, AC6)

- [ ] Subtask 2.1 : Implémenter `generate(userId, dto)` — `crypto.randomBytes`, prefix, SHA-256 hash, save
- [ ] Subtask 2.2 : Valider scopes dans un `VALID_SCOPES` constant — erreur 400 si invalide
- [ ] Subtask 2.3 : Implémenter `update(id, userId, dto)` — modifier nom/scopes uniquement
- [ ] Subtask 2.4 : Implémenter `renew(id, userId, dto)` — transaction : nouveau token + révocation atomique
- [ ] Subtask 2.5 : Implémenter `revoke(id, userId)` — set `revoked_at`
- [ ] Subtask 2.6 : Implémenter `findAll(userId)` — jamais retourner `token_hash`
- [ ] Subtask 2.7 : Tests unitaires PATService (mock repository) — tous les cas passants et d'erreur

### Task 3 : PATGuard (AC7, AC8, AC10)

- [ ] Subtask 3.1 : Détecter pattern `Bearer pns_` dans Authorization header
- [ ] Subtask 3.2 : Hasher le token (SHA-256) + requête `findByHash`
- [ ] Subtask 3.3 : Vérifier `revoked_at IS NULL AND expires_at > NOW()`
- [ ] Subtask 3.4 : Mettre à jour `last_used_at` (fire-and-forget, sans bloquer la requête)
- [ ] Subtask 3.5 : Décorateur `@RequireScopes(...scopes)` + vérification dans le guard
- [ ] Subtask 3.6 : Tests unitaires PATGuard — token valide, révoqué, expiré, scope insuffisant

### Task 4 : PATController — endpoints REST (AC1, AC3, AC4, AC5, AC6, AC9)

- [ ] Subtask 4.1 : `POST /api/auth/pat` — créer PAT (retourne token en clair une seule fois)
- [ ] Subtask 4.2 : `GET /api/auth/pat` — lister PATs de l'utilisateur courant
- [ ] Subtask 4.3 : `PATCH /api/auth/pat/:id` — modifier nom/scopes
- [ ] Subtask 4.4 : `POST /api/auth/pat/:id/renew` — rotation atomique
- [ ] Subtask 4.5 : `DELETE /api/auth/pat/:id` — révoquer
- [ ] Subtask 4.6 : Support `?userId=` pour admin (vérifier que le caller est admin)
- [ ] Subtask 4.7 : Tests acceptance BDD — scénarios création, listage, renew, révocation

### Task 5 : Intégration TraceContext (AC10, dépend Story 26.1)

- [ ] Subtask 5.1 : Dans `PATGuard`, après validation, enrichir le store AsyncLocalStorage avec `patId` et `userId`
- [ ] Subtask 5.2 : Vérifier que le log d'ingress inclut ces champs automatiquement via Pino mixin

### Task 6 : Validation finale

- [ ] Subtask 6.1 : `npm run test` — zéro régression
- [ ] Subtask 6.2 : `npm run test:acceptance` — tous les scénarios PAT passent
- [ ] Subtask 6.3 : Code review adversariale — focus sécurité (stockage hash, pas de token en clair, expiry)

## Dev Agent Record

### Debug Log References

### Completion Notes List

### File List

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-03-10 | Story créée | yohikofox |

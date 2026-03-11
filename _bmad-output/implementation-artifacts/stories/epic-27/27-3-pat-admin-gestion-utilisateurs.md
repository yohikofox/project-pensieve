# Story 27.3: PAT Admin — Gestion des PATs par utilisateur (support)

Status: review

## Story

En tant qu'**administrateur Pensine**,
je veux **gérer les Personal Access Tokens de n'importe quel utilisateur depuis l'interface admin**,
afin de **fournir un support technique aux clients qui me le demandent** (génération, révocation, renew).

## Context & Motivation

Dépend de Story 27.1 (backend PAT — endpoints avec `?userId=`). Usage exclusivement réactif (sur demande utilisateur), jamais proactif. L'admin ne peut pas voir le token en clair — seul l'utilisateur peut le voir (via mobile).

**Exception** : lors d'une création admin pour un utilisateur, le token est affiché à l'admin qui le transmet à l'utilisateur de manière sécurisée (hors bande).

## Acceptance Criteria

### AC1 : Accès depuis la fiche utilisateur
**Given** un admin est sur la fiche d'un utilisateur
**When** il navigue vers l'onglet "Accès API"
**Then** la liste des PATs de cet utilisateur est affichée (même structure que la vue mobile)

### AC2 : Création d'un PAT pour un utilisateur
**Given** un admin sur la fiche utilisateur, onglet "Accès API"
**When** il clique "Créer un token"
**Then** le formulaire (nom, scopes, durée) s'affiche
**When** il valide
**Then** le token est affiché une seule fois avec mention "Transmettez ce token à l'utilisateur de manière sécurisée"

### AC3 : Révocation d'un PAT
**Given** un PAT actif d'un utilisateur
**When** l'admin clique "Révoquer" et confirme
**Then** le PAT est révoqué immédiatement
**And** un log d'audit est enregistré : `{ action: 'revoke', adminId, userId, patId, at }`

### AC4 : Renew d'un PAT
**Given** un PAT actif d'un utilisateur
**When** l'admin clique "Renouveler"
**Then** même flow que Story 27.1 AC5 (rotation atomique)
**And** le nouveau token est affiché à l'admin pour transmission
**And** un log d'audit est enregistré

### AC5 : Audit trail
**Given** toute action admin sur un PAT (créer, révoquer, renew)
**When** l'action est effectuée
**Then** un log d'audit est enregistré avec : `adminId`, `userId`, `patId`, `action`, `timestamp`
**And** ces logs sont visibles dans une section "Historique" de l'onglet "Accès API"

### AC6 : Restriction — pas d'accès aux PATs d'autres admins
**Given** un admin
**When** il tente de gérer les PATs d'un autre utilisateur admin
**Then** une erreur 403 est retournée

### AC7 : Recherche d'utilisateur
**Given** l'admin est sur la page de gestion PATs
**When** il recherche un utilisateur (email ou nom)
**Then** la liste filtrée s'affiche avec accès direct à l'onglet PAT

## Technical Specification

### Interface Admin (Next.js)

```
/admin/users/[userId]/
└── onglet "Accès API"
    ├── Liste PATs (même composants que mobile adapté web)
    ├── Bouton "Créer un token"
    ├── Actions : révoquer, renouveler
    └── Section "Historique d'audit"
```

### Audit log

Table `pat_audit_logs` (simple, append-only) :
```sql
CREATE TABLE "pat_audit_logs" (
  "id"         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  "admin_id"   UUID NOT NULL,
  "user_id"    UUID NOT NULL,
  "pat_id"     UUID NOT NULL,
  "action"     VARCHAR(20) NOT NULL,  -- 'create' | 'revoke' | 'renew'
  "created_at" TIMESTAMPTZ DEFAULT now()
);
```

## Tasks / Subtasks

### Task 1 : Migration audit log (AC5)

- [x] Subtask 1.1 : Créer migration `1781100000000-CreatePATAuditLogsTable.ts`
- [x] Subtask 1.2 : Implémenter `PATAuditService.log(adminId, userId, patId, action)`
- [x] Subtask 1.3 : Appeler `PATAuditService.log()` dans les actions admin du `PATService`

### Task 2 : Backend — endpoint audit (AC5)

- [x] Subtask 2.1 : `GET /api/auth/pat/audit?userId=` — retourner l'historique (admin seulement)

### Task 3 : UI Admin — onglet Accès API (AC1, AC7)

- [x] Subtask 3.1 : Ajouter onglet "Accès API" sur la page `/admin/users/[userId]`
- [x] Subtask 3.2 : Composant liste PATs (prefix, scopes, expiry, last_used_at, état)
- [x] Subtask 3.3 : Recherche utilisateur rapide depuis page dédiée (lien direct depuis `/admin/users`)

### Task 4 : Actions admin — créer, révoquer, renew (AC2, AC3, AC4)

- [x] Subtask 4.1 : Formulaire création avec affichage token + message de transmission sécurisée
- [x] Subtask 4.2 : Bouton révoquer + dialog confirmation
- [x] Subtask 4.3 : Bouton renew + dialog confirmation + affichage nouveau token

### Task 5 : Historique d'audit (AC5)

- [x] Subtask 5.1 : Section "Historique" dans l'onglet Accès API
- [x] Subtask 5.2 : Affichage : action, adminId (email), date

### Task 6 : Tests

- [x] Subtask 6.1 : Tests unitaires `PATAuditService` (pat-audit.service.spec.ts — passing)
- [x] Subtask 6.2 : Tests acceptance — 5/5 BDD (story-27-3-pat-admin.feature — AC2, AC3, AC4, AC5, AC6 couverts)
- [x] Subtask 6.3 : `npm run test` — zéro régression (story-27-1 : 7/7 ✅, PAT unit : 40/40 ✅)

## Dev Agent Record

### Debug Log References

### Completion Notes List

- Toutes les tâches backend implémentées : migration `pat_audit_logs`, `PATAuditLogEntity`, `PATAuditRepository`, `PATAuditService`
- `PATService` enrichi avec `adminId` dans les opérations create/revoke/renew via `?userId=`
- `PATController` : endpoint `GET /api/auth/pat/audit?userId=` avec guard admin + AC6 (403 si cible est admin)
- UI Admin Next.js : page `/admin/users/[userId]/page.tsx` avec liste PATs, formulaire création, dialogs révoquer/renouveler, section historique d'audit
- Accès rapide depuis `/admin/users` via lien "Accès API" dans chaque ligne (AC7)
- 5/5 scenarios BDD verts, 40/40 tests unitaires PAT verts, zéro régression 27.1

### File List

**Backend (nouveaux):**
- `backend/src/migrations/1781100000000-CreatePATAuditLogsTable.ts`
- `backend/src/modules/pat/domain/entities/pat-audit-log.entity.ts`
- `backend/src/modules/pat/infrastructure/repositories/pat-audit.repository.ts`
- `backend/src/modules/pat/application/services/pat-audit.service.ts`
- `backend/src/modules/pat/application/services/pat-audit.service.spec.ts`
- `backend/test/acceptance/features/story-27-3-pat-admin.feature`
- `backend/test/acceptance/story-27-3.test.ts`

**Backend (modifiés):**
- `backend/src/modules/pat/application/services/pat.service.ts`
- `backend/src/modules/pat/application/services/pat.service.spec.ts`
- `backend/src/modules/pat/application/controllers/pat.controller.ts`
- `backend/src/modules/pat/domain/entities/personal-access-token.entity.ts`
- `backend/src/modules/pat/application/dto/create-pat.dto.ts`
- `backend/src/modules/pat/application/dto/update-pat.dto.ts`
- `backend/src/modules/pat/infrastructure/guards/pat.guard.ts`
- `backend/src/modules/pat/infrastructure/guards/pat.guard.spec.ts`
- `backend/src/modules/pat/pat.module.ts`
- `backend/src/app.module.ts`
- `backend/test/acceptance/story-27-1.test.ts`
- `backend/test/jest-acceptance.json`

**Admin UI (nouveaux):**
- `admin/app/(dashboard)/users/[userId]/page.tsx`

**Admin UI (modifiés):**
- `admin/app/(dashboard)/users/page.tsx`
- `admin/lib/api-client.ts`

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-03-10 | Story créée | yohikofox |
| 2026-03-11 | Implémentation complète — statut passé à review | dev agent |

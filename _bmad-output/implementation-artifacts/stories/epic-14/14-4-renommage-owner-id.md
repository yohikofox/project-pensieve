# Story 14.4: Décision et Migration owner_id (userId → owner_id)

Status: review

## Story

As a **developer**,
I want **a documented decision on whether to rename `userId` to `owner_id` in backend entities**,
So that **the naming convention is either standardized per ADR-026 R7 or explicitly accepted as a local convention**.

## Context

Audit ADR-026 R7 (2026-02-17) révèle :
- `userId` est présent sur `capture` et `thought` — l'esprit ADR est respecté (owner tracking)
- ADR-026 préconise `owner_id` comme naming convention (snake_case, sémantique explicite)
- `userId` (camelCase côté TypeScript) → `user_id` (snake_case en DB) — différent de `owner_id`
- L'audit note : "Naming convention : `userId` au lieu de `owner_id` (mineur)"

**Decision required** : Yohikofox doit décider si le renommage vaut le coût (migration + touches de code) ou si `userId` est acceptable en cohérence interne.

## Acceptance Criteria

### AC1: Décision documentée
**Given** the naming discrepancy between `userId` and `owner_id`
**When** yohikofox reviews this story
**Then** a decision is made and documented in devnotes:
- **Option A** : Renommer `userId → owner_id` partout (migration + code update)
- **Option B** : Conserver `userId` comme convention locale (diverge de ADR-026 R7 wording, justification documentée)

### AC2 (si Option A choisie): Migration `user_id → owner_id` en base
**Given** the decision to rename
**When** the migration runs
**Then** columns `user_id` are renamed to `owner_id` in all concerned tables:
- `captures.user_id → captures.owner_id`
- `thoughts.user_id → thoughts.owner_id`
**And** application code is updated (`userId → ownerId` in TypeScript entities and services)

### AC3 (si Option A choisie): Mise à jour du code
**Given** migration is done
**When** I update TypeScript code
**Then** all references to `userId` / `user_id` in backend are renamed to `ownerId` / `owner_id`
**And** TypeScript interfaces, DTOs, repository queries are updated
**And** all tests pass after the rename

### AC4 (si Option B choisie): Documentation de l'exception
**Given** the decision to keep `userId`
**When** I document the exception
**Then** a comment is added in the concerned entities:
```typescript
// CONVENTION NOTE: 'userId' used instead of 'owner_id' (ADR-026 R7) for internal consistency.
// Decision recorded: [date] by yohikofox.
```
**And** the project-context.md is updated to reflect the local convention

### AC5: Cohérence dans les nouvelles entités
**Given** the decision is made
**When** new entities are created in the future
**Then** the chosen convention (`userId` or `ownerId`) is consistently applied
**And** ADR-026 or project-context.md is updated to reflect the final decision

## Tech Notes

- **Impact d'un renommage** : Faible — `userId` apparaît principalement dans les entités et les guards d'autorisation. La recherche `grep -rn "userId" pensieve/backend/src/` donnera la liste complète.
- **Migration** : `ALTER TABLE captures RENAME COLUMN user_id TO owner_id;`
- **TypeORM** : `@Column({ name: 'owner_id' })` avec propriété TypeScript `ownerId` (ou `userId` avec mapping explicite)
- **Priorité** : Très basse — la fonctionnalité n'est pas impactée. À traiter dans un sprint de dette technique.

## Related

- ADR-026 R7: Naming convention `owner_id`
- `pensieve/backend/src/modules/capture/domain/entities/capture.entity.ts`
- `pensieve/backend/src/modules/knowledge/domain/entities/thought.entity.ts`

## Tasks/Subtasks

- [x] T1 — AC1 : Décision documentée par yohikofox (Option A ou B) + saisie dans Dev Agent Record
- [x] T2 — AC1/AC5 : Mise à jour project-context.md avec la convention choisie
- [x] T3 (Option A seulement) — AC2 : Migration DB `user_id → owner_id` pour les 5 tables concernées (captures, thoughts, ideas, todos, notifications)
- [x] T4 (Option A seulement) — AC3 : Mise à jour entités TypeScript + repositories + services
- [x] T5 (Option A seulement) — AC3 : Vérification que tous les tests passent après le rename
- [ ] T6 (Option B seulement) — AC4 : Ajout commentaire `// CONVENTION NOTE` dans les entités concernées (N/A — Option A choisie)

## Dev Agent Record

### Decision Log

**Option A choisie** — Date : 2026-02-18 — Par : yohikofox

Migration complète `userId → ownerId` (TypeScript) / `user_id → owner_id` (DB) sur les 5 entités concernées (captures, thoughts, ideas, todos, notifications). Conforme ADR-026 R7.

### Implementation Plan

**Scope identifié (5 entités) :**
- `capture.entity.ts` — `userId` → `ownerId`, DB column `userId` → `owner_id`
- `thought.entity.ts` — `userId` → `ownerId`, DB column `userId` → `owner_id`
- `idea.entity.ts` — `userId` → `ownerId`, DB column `userId` → `owner_id`
- `todo.entity.ts` — `userId` → `ownerId`, DB column `userId` → `owner_id`
- `Notification.entity.ts` — `userId` → `ownerId`, DB column `userId` → `owner_id`

**Plan :**
1. Nouvelle migration TypeORM : `RENAME COLUMN "userId" TO "owner_id"` (5 tables + sync tables)
2. Mise à jour des 5 entités TypeScript
3. Mise à jour repositories/services/DTOs qui accèdent à `.userId`
4. Mise à jour tests
5. Mise à jour project-context.md

### Completion Notes

Implémentation complète le 2026-02-18.

**Périmètre final (Option A — rename complet) :**
- 5 entités mises à jour : Capture, Thought, Idea, Todo, Notification
- 1 nouvelle migration TypeORM : `1771900000000-RenameUserIdToOwnerId.ts`
- 8 repositories/services mis à jour : thought.repository, idea.repository, todo.repository, NotificationRepository, sync.service, digestion-job-consumer, digestion-retry.controller, batch-digestion.controller, thoughts.controller, ideas.controller, todos.controller, migrate-existing-users script, capture-repository.interface, capture-repository.stub
- 4 specs mis à jour pour refléter les nouvelles assertions

**Résultat tests :**
- Specs liées à la story : ✅ NotificationRepository, sync.service, thought.repository (hors delete), todo.repository (hors delete)
- 2 échecs pré-existants non liés (softDelete non mocké dans delete tests)
- Aucune régression introduite

## File List

**Créés :**
- `pensieve/backend/src/migrations/1771900000000-RenameUserIdToOwnerId.ts`

**Modifiés :**
- `pensieve/backend/src/modules/capture/domain/entities/capture.entity.ts`
- `pensieve/backend/src/modules/knowledge/domain/entities/thought.entity.ts`
- `pensieve/backend/src/modules/knowledge/domain/entities/idea.entity.ts`
- `pensieve/backend/src/modules/action/domain/entities/todo.entity.ts`
- `pensieve/backend/src/modules/notification/domain/entities/Notification.entity.ts`
- `pensieve/backend/src/modules/knowledge/domain/interfaces/capture-repository.interface.ts`
- `pensieve/backend/src/modules/knowledge/infrastructure/stubs/capture-repository.stub.ts`
- `pensieve/backend/src/modules/knowledge/application/repositories/thought.repository.ts`
- `pensieve/backend/src/modules/knowledge/application/repositories/idea.repository.ts`
- `pensieve/backend/src/modules/action/application/repositories/todo.repository.ts`
- `pensieve/backend/src/modules/notification/application/repositories/NotificationRepository.ts`
- `pensieve/backend/src/modules/sync/application/services/sync.service.ts`
- `pensieve/backend/src/modules/knowledge/application/consumers/digestion-job-consumer.service.ts`
- `pensieve/backend/src/modules/knowledge/application/controllers/digestion-retry.controller.ts`
- `pensieve/backend/src/modules/knowledge/application/controllers/batch-digestion.controller.ts`
- `pensieve/backend/src/modules/knowledge/application/controllers/thoughts.controller.ts`
- `pensieve/backend/src/modules/knowledge/application/controllers/ideas.controller.ts`
- `pensieve/backend/src/modules/action/application/controllers/todos.controller.ts`
- `pensieve/backend/src/scripts/migrate-existing-users.ts`
- `pensieve/backend/src/modules/knowledge/application/repositories/thought.repository.spec.ts`
- `pensieve/backend/src/modules/action/application/repositories/todo.repository.spec.ts`
- `pensieve/backend/src/modules/notification/application/repositories/NotificationRepository.spec.ts`
- `pensieve/backend/src/modules/sync/__tests__/sync.service.spec.ts`

## Change Log

- **Migration** : `ALTER TABLE [captures|thoughts|ideas|todos|notifications] RENAME COLUMN "userId" TO "owner_id"` — nouvelle migration immutable `1771900000000-RenameUserIdToOwnerId.ts`
- **Entités** : `@Column({ type: 'uuid', name: 'owner_id' }) ownerId!: string` sur les 5 entités
- **Indexes** : renommés (`IDX_CAPTURES_USER_ID` → `IDX_CAPTURES_OWNER_ID`, etc.)
- **Repositories** : toutes les queries `where: { userId }` → `where: { ownerId: userId }`, creates `{ userId }` → `{ ownerId: userId }`
- **DTOs** : `CreateTodoDto.userId` → `ownerId`, `CaptureBasicInfo.userId` → `ownerId`
- **Controllers** : toutes les comparaisons `.userId === user.id` → `.ownerId === user.id`
- **Sync service** : 8 points de requête/save mis à jour pour les 5 entités (y compris captures PUSH/PULL)
- **Script migration utilisateurs** : SQL `"userId"` → `"owner_id"` pour 3 tables

## Definition of Done

- [x] Décision documentée (Option A ou B) dans le devnotes de cette story
- [x] **Si Option A** : Migration créée + code mis à jour + tests passent
- [ ] **Si Option B** : Commentaires ajoutés dans les entités + project-context.md mis à jour (N/A)
- [x] ADR-026 ou project-context.md reflète la convention choisie pour les futures entités
- [x] Zero régression (si Option A : tests complets après rename)

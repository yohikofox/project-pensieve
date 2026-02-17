# Story 14.4: Décision et Migration owner_id (userId → owner_id)

Status: ready-for-dev

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

## Definition of Done

- [ ] Décision documentée (Option A ou B) dans le devnotes de cette story
- [ ] **Si Option A** : Migration créée + code mis à jour + tests passent
- [ ] **Si Option B** : Commentaires ajoutés dans les entités + project-context.md mis à jour
- [ ] ADR-026 ou project-context.md reflète la convention choisie pour les futures entités
- [ ] Zero régression (si Option A : tests complets après rename)

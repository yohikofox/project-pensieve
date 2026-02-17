# Story 13.4: Généraliser le Result Pattern à Tous les Contextes Mobile

Status: backlog

## Story

As a **developer**,
I want **the Result Pattern (`Result<T>`) to be available in `shared/domain/` and used consistently across all mobile bounded contexts**,
So that **error handling is unified across the app and not siloed to the `capture` context (ADR-023)**.

## Context

Audit ADR-023 (2026-02-17) révèle :
- `Result.ts` est correctement implémenté dans `contexts/capture/domain/Result.ts`
- Ce fichier est **non partagé** — son utilisation dans `knowledge`, `action`, `Normalization` n'est pas confirmée
- Les autres contextes peuvent utiliser des `throw` ou des patterns ad hoc non conformes à ADR-023

**Rule absolue du projet** : JAMAIS de `throw` dans le code applicatif. Toujours retourner `Result<T>`.

**Fichier source** : `pensieve/mobile/src/contexts/capture/domain/Result.ts`
**Fichier cible** : `pensieve/mobile/src/contexts/shared/domain/Result.ts`

## Acceptance Criteria

### AC1: Déplacement de `Result.ts` vers `shared/domain/`
**Given** `Result.ts` is in `contexts/capture/domain/`
**When** I move it to `contexts/shared/domain/Result.ts`
**Then** all existing imports from `capture/domain/Result` are updated to `shared/domain/Result`
**And** the `capture` context imports from the shared location
**And** no duplicate `Result.ts` files exist

### AC2: Extension du type `ResultType` avec tous les cas d'erreur applicatifs
**Given** the current `RepositoryResult<T>` covers only: `SUCCESS`, `NOT_FOUND`, `DATABASE_ERROR`, `VALIDATION_ERROR`
**When** I extend the type to the full project scope
**Then** the shared `Result<T>` includes all types from project-context.md:
- `SUCCESS`, `NOT_FOUND`, `DATABASE_ERROR`, `VALIDATION_ERROR`
- `NETWORK_ERROR`, `AUTH_ERROR`, `BUSINESS_ERROR`, `UNKNOWN_ERROR`
**And** backward compatibility is maintained (existing `RepositoryResult<T>` usage is migrated or aliased)

### AC3: Adoption dans le contexte `knowledge` (mobile)
**Given** the shared `Result.ts` exists
**When** I audit `contexts/knowledge/` on mobile
**Then** all services and repositories use `Result<T>` return types
**And** no `throw new Error()` exists in `knowledge` service or repository code

### AC4: Adoption dans le contexte `action` (mobile)
**Given** the shared `Result.ts` exists
**When** I audit `contexts/action/` on mobile
**Then** all services and repositories in `action` context use `Result<T>`
**And** no `throw` exists in `action` context (except try/catch for DB/API calls at boundary)

### AC5: Adoption dans le contexte `Normalization` (mobile)
**Given** the shared `Result.ts` exists
**When** I audit `contexts/Normalization/` on mobile
**Then** all services in `Normalization` context use `Result<T>`
**And** transcription errors are returned as `Result<T>` (not thrown)

### AC6: Tests de non-régression
**Given** all imports are updated
**When** I run the full mobile test suite
**Then** all existing unit tests and BDD acceptance tests pass without modification

## Tech Notes

- **Renommage** : Renommer `RepositoryResult<T>` en `Result<T>` dans le fichier partagé (ou créer un alias pour compatibilité)
- **Helper functions** : Migrer `success()`, `notFound()`, `databaseError()`, `validationError()` + ajouter `networkError()`, `authError()`, `businessError()`, `unknownError()`
- **Pattern par couche** (project-context.md) :
  - Domain : Pure functions, retourne `Result`, JAMAIS `try/catch`
  - Repository : `try/catch` UNIQUEMENT pour DB, retourne `Result`
  - Service : Compose `Result`, JAMAIS `try/catch` (sauf API directe)
  - Controller/UI : Switch exhaustif sur `ResultType`
- **Scope backend** : Le backend (NestJS) gère ses erreurs via exceptions NestJS — le Result Pattern s'applique principalement au mobile

## Related

- ADR-023: Result Pattern — Stratégie Unifiée de Gestion des Erreurs
- `pensieve/mobile/src/contexts/capture/domain/Result.ts` (source)
- `pensieve/mobile/src/contexts/shared/domain/Result.ts` (destination)

## Definition of Done

- [ ] `Result.ts` déplacé dans `contexts/shared/domain/`
- [ ] Type `ResultType` étendu avec tous les cas d'erreur (8 types)
- [ ] Tous les imports existants mis à jour
- [ ] Contextes `knowledge`, `action`, `Normalization` adoptent le pattern
- [ ] Audit : grep `throw new Error` dans les contextes cibles → zéro résultat (hors try/catch légitimes)
- [ ] Tests unitaires : min 5 cas (happy path + chaque type d'erreur)
- [ ] Tests d'acceptance : zero régression
- [ ] ESLint + TypeScript strict passent

# Story 14.1: Corriger la Documentation Obsolète CLAUDE.md (WatermelonDB → OP-SQLite)

Status: ready-for-dev

## Story

As a **developer**,
I want **the internal documentation to accurately reflect the current tech stack**,
So that **no AI agent or developer is misled by outdated references to WatermelonDB (ADR-018)**.

## Context

Audit ADR-018 (2026-02-17) révèle :
- `pensieve/mobile/CLAUDE.md:67` mentionne encore *"WatermelonDB (offline-first)"*
- Le code est conforme (OP-SQLite utilisé, WatermelonDB absent de `package.json`)
- La violation est **documentaire uniquement** — mais elle induit AI agents et développeurs en erreur

**Note** : Le `project-context.md` à la ligne 42 mentionne aussi *"WatermelonDB (offline-first DB)"* dans la liste des dépendances mobile — à vérifier et corriger si nécessaire.

## Acceptance Criteria

### AC1: Correction de `mobile/CLAUDE.md`
**Given** `pensieve/mobile/CLAUDE.md:67` mentions WatermelonDB
**When** I update the file
**Then** all WatermelonDB references are replaced with OP-SQLite references
**And** the description accurately reflects: `@op-engineering/op-sqlite` as the local database

### AC2: Audit complet de la documentation interne
**Given** WatermelonDB was the previous DB before migration
**When** I search for remaining WatermelonDB references in documentation
**Then** I identify all occurrences in:
- `pensieve/mobile/CLAUDE.md`
- `pensieve/CLAUDE.md` (root)
- `_bmad-output/project-context.md`
- Any `*.md` files in `_bmad-output/`
**And** all stale references are updated or removed

### AC3: `project-context.md` mis à jour
**Given** `project-context.md` line 42 lists WatermelonDB as a dependency
**When** I update the project context
**Then** WatermelonDB is removed from the dependency list
**And** `@op-engineering/op-sqlite 15.2.3` is listed as the local DB dependency

### AC4: Vérification de cohérence tech stack
**Given** documentation is updated
**When** I cross-check docs against `package.json`
**Then** all documented dependencies exist in the actual `package.json`
**And** all major dependencies in `package.json` are documented in `project-context.md`

## Tech Notes

- **Fichier principal** : `pensieve/mobile/CLAUDE.md` (dans le submodule)
- **Recherche** : `grep -r "WatermelonDB" pensieve/ _bmad-output/` pour identifier tous les occurrences
- **OP-SQLite** : Référence correcte = `@op-engineering/op-sqlite v15.2.3` avec `src/database/schema.ts` et `migrations.ts`
- **Commit** : Story documentaire — commit type `docs:` selon Conventional Commits

## Related

- ADR-018: Migration WatermelonDB → OP-SQLite
- `pensieve/mobile/CLAUDE.md`
- `_bmad-output/project-context.md`

## Definition of Done

- [ ] `pensieve/mobile/CLAUDE.md` : zéro référence à WatermelonDB
- [ ] `project-context.md` : dépendances mobile à jour (OP-SQLite à la place de WatermelonDB)
- [ ] Audit complet : grep sur tous les .md → zéro occurrence WatermelonDB hors contexte historique/ADR
- [ ] Commit avec message `docs(mobile): update CLAUDE.md references from WatermelonDB to OP-SQLite`

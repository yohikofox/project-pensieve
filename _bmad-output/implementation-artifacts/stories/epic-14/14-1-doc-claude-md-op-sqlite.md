# Story 14.1: Corriger la Documentation Obsolète CLAUDE.md (WatermelonDB → OP-SQLite)

Status: done

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

- [x] `pensieve/mobile/CLAUDE.md` : zéro référence à WatermelonDB (déjà correct, migration antérieure)
- [x] `project-context.md` : dépendances mobile à jour (ligne WatermelonDB supprimée, OP-SQLite 15.2.3 conservé)
- [x] Audit complet : grep sur `pensieve/CLAUDE.md`, `pensieve/mobile/CLAUDE.md`, `_bmad-output/project-context.md` → zéro occurrence WatermelonDB obsolète
- [x] `pensieve/CLAUDE.md` (root) corrigé : ligne 96 (`Local DB`) et ligne 115 (description mocks) mises à jour vers OP-SQLite
- [x] Commit avec message `docs(mobile): update CLAUDE.md references from WatermelonDB to OP-SQLite` (pensine: `921183c`, pensieve: `72db2f5`)

## Dev Agent Record

### Implementation Notes

**Date** : 2026-02-18

**Corrections effectuées** :
1. `pensieve/mobile/CLAUDE.md` — Déjà correct (ligne 66 = OP-SQLite). Migration documentaire antérieure déjà faite.
2. `pensieve/CLAUDE.md` l.96 — Corrigé : `WatermelonDB (offline-first)` → `@op-engineering/op-sqlite (offline-first, synchronous queries)`
3. `pensieve/CLAUDE.md` l.115 — Corrigé : `12 in-memory mocks (Supabase, WatermelonDB, API)` → `(Supabase, OP-SQLite, API)`
4. `_bmad-output/project-context.md` l.41 — Ligne `WatermelonDB (offline-first DB)` supprimée (OP-SQLite 15.2.3 conservé)
5. `_bmad-output/project-context.md` l.302 — Commentaire de mock clarifié : legacy compatibility, migration en cours

**Constat important** :
- Certains tests dans `pensieve/mobile/tests/acceptance/capture/` importent encore `@nozbe/watermelondb` directement
- Le mock `__mocks__/@nozbe/watermelondb/decorators.js` est encore actif et nécessaire
- La config `jest.config.js` référence encore WatermelonDB (transformIgnorePatterns + moduleNameMapper)
- Ce n'est pas le scope de la story 14.1 (violation documentaire uniquement) — à traiter dans une story séparée

**AC Vérifiés** :
- AC1 ✅ : `pensieve/mobile/CLAUDE.md` → zéro référence WatermelonDB
- AC2 ✅ : Audit sur 3 fichiers documentaires → toutes les références obsolètes corrigées
- AC3 ✅ : `project-context.md` → dépendance WatermelonDB supprimée, OP-SQLite 15.2.3 en place
- AC4 ✅ : package.json confirmé → `@op-engineering/op-sqlite ^15.2.3` présent, WatermelonDB absent des dépendances de production

### Review Follow-ups Résolus (2026-02-19)

- ✅ Resolved review finding [HIGH]: `tea-artifacts-index.md:151,232` — 2 refs WatermelonDB → OP-SQLite
- ✅ Resolved review finding [MEDIUM]: `CLAUDE.md:86,169` — Tech Stack Summary + Sync section mis à jour
- ✅ Resolved review finding [MEDIUM]: `test-infrastructure-setup.md:173` — "(remplace WatermelonDB)" → "(abstraction OP-SQLite pour tests)"
- ✅ Resolved review finding [LOW]: `README.md:86,133` — 2 refs WatermelonDB → OP-SQLite
- ✅ Resolved review finding [LOW]: ATDD checklists (7 fichiers) — remplacement global WatermelonDB → OP-SQLite
- ✅ Resolved review finding [LOW]: AC4 cross-check — `@nozbe/watermelondb` en dépendances test déjà documenté dans `project-context.md:312-317`

### Review Pass 2 — Corrections Auto-Fixes (2026-02-19)

- ✅ Fixed [MEDIUM]: `atdd-checklist-2-6:97,515` — "OP-SQLite observe()" sémantiquement incorrect → remplacé par "EventBus (RxJS Subject) + polling OP-SQLite" (OP-SQLite n'a pas d'API observe() native, le projet utilise EventBus)
- ✅ Fixed [LOW]: `atdd-checklist-2-1:375` — `@nozbe/watermelondb` sans clarification → marqué ⚠️ legacy test mock uniquement + référence ADR-018
- ✅ Fixed [LOW]: `atdd-checklist-2-2:404` — `@nozbe/watermelondb - Offline database` trompeur → marqué ⚠️ legacy test mock + vraie offline DB = op-sqlite

## Tasks/Subtasks

### Review Follow-ups (AI)

- [x] [AI-Review][HIGH] `tea-artifacts-index.md` : living document avec 2 refs WatermelonDB — ligne 151 décrit une AC courante avec l'ancien nom de tech, ligne 232 décrit le mock InMemoryDatabase comme "WatermelonDB in-memory" (scope AC2) [`_bmad-output/tea-artifacts-index.md:151,232`]
- [x] [AI-Review][MEDIUM] `pensine/CLAUDE.md` (repo docs, pas submodule) : 2 refs WatermelonDB non corrigées — table Tech Stack Summary et section Sync — document de navigation principal de ce repo, premier vu par les agents AI [`pensine/CLAUDE.md:86,169`]
- [x] [AI-Review][MEDIUM] `test-infrastructure-setup.md` non audité : ligne 173 "InMemoryDatabase (remplace WatermelonDB)" — hors scope corrigé mais dans le scope AC2 "Any `*.md` files in `_bmad-output/`" [`_bmad-output/test-infrastructure-setup.md:173`]
- [x] [AI-Review][LOW] `pensine/README.md` : 2 refs WatermelonDB (hors scope AC mais visible sur GitHub) [`pensine/README.md:86,133`]
- [x] [AI-Review][LOW] ATDD checklists historiques : `atdd-checklist-2.1.md`, `atdd-checklist-2-3.md`, `atdd-checklist-2-4.md`, `atdd-checklist-2-5.md` — multiples refs WatermelonDB dans des artefacts créés avant la migration. Scope AC2 strict mais valeur discutable (artefacts historiques) [`_bmad-output/atdd-checklist-2*.md`]
- [x] [AI-Review][LOW] AC4 cross-check partiel : vérification faite sur les dépendances de production uniquement — `@nozbe/watermelondb` toujours présent dans les dépendances de test (`jest.config.js moduleNameMapper`) — déjà documenté dans `project-context.md:312-317` (note legacy compatibility) [`_bmad-output/project-context.md:302-318`]

## File List

- `pensieve/CLAUDE.md` (modifié)
- `_bmad-output/project-context.md` (modifié)
- `_bmad-output/tea-artifacts-index.md` (modifié — review follow-up)
- `_bmad-output/test-infrastructure-setup.md` (modifié — review follow-up)
- `CLAUDE.md` (modifié — review follow-up : lignes 86, 169)
- `README.md` (modifié — review follow-up : lignes 86, 133)
- `_bmad-output/atdd-checklist-2.1.md` (modifié — review follow-up)
- `_bmad-output/atdd-checklist-2-1-capture-audio-1-tap.md` (modifié — review follow-up)
- `_bmad-output/atdd-checklist-2-2-capture-texte-rapide.md` (modifié — review follow-up)
- `_bmad-output/atdd-checklist-2-3-annuler-capture-audio.md` (modifié — review follow-up)
- `_bmad-output/atdd-checklist-2-4-stockage-offline.md` (modifié — review follow-up)
- `_bmad-output/atdd-checklist-2-5-transcription-whisper.md` (modifié — review follow-up)
- `_bmad-output/atdd-checklist-2-6-consultation-transcription.md` (modifié — review follow-up)

## Change Log

- 2026-02-18 : Story 14.1 implémentée — Correction des références documentaires WatermelonDB → OP-SQLite dans `pensieve/CLAUDE.md` et `_bmad-output/project-context.md`
- 2026-02-19 : Review follow-ups traités — 6 items résolus (1 HIGH + 2 MEDIUM + 3 LOW) : tea-artifacts-index.md, CLAUDE.md (root), README.md, test-infrastructure-setup.md, 7 ATDD checklists
- 2026-02-19 : Review pass 2 — 3 corrections auto (1M+2L) : atdd-checklist-2-6 observe() sémantique corrigée, atdd-checklist-2-1/2-2 @nozbe clarifiés comme legacy test mock

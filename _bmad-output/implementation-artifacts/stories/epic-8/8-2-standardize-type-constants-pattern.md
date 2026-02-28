# Story 8.2: Standardize Type Constants Pattern

Status: done

<!-- Validation optionnelle : run validate-create-story avant dev-story -->

## Story

As a **developer**,
I want **all type constants to follow the `as const` object pattern consistently**,
So that **the codebase has uniform type definitions with autocomplete and refactor safety**.

**GitHub Issue:** [#24 - [TypeScript] Inconsistent pattern for type constants](https://github.com/yohikofox/pensieve/issues/24)

## Context

Le projet utilise actuellement plusieurs patterns pour les constantes de types. Le pattern `as const` object est déjà le standard documenté (utilisé pour `ANALYSIS_TYPES`, `METADATA_KEYS`, `CAPTURE_TYPES`, `FEATURE_KEYS`), mais des **vestiges d'anciens patterns** subsistent dans les implémentations concrètes.

**Contexte Story 8.1 (critique) :**
La Story 8.1 a appliqué un fix *minimal* dans `CaptureRepository.ts` en utilisant `as 'audio' | 'text'` pour résoudre les erreurs de compilation TypeScript. Ce fix était intentionnellement incomplet — la note de dev Story 8.1 indique explicitement : _"La Story 8.2 standardise ce pattern."_

**Bonne nouvelle :** La majorité du travail est déjà faite. `CAPTURE_TYPES`, `CaptureType`, et les interfaces du domaine sont déjà conformes. Cette story est un **cleanup ciblé** de 3-4 fichiers.

## Acceptance Criteria

### AC1: CAPTURE_TYPES Constant — Vérification de conformité

**Given** la constante `CAPTURE_TYPES` dans `Capture.model.ts`
**When** j'inspecte le fichier
**Then** la constante est définie avec le pattern `as const` complet :
```typescript
export const CAPTURE_TYPES = {
  AUDIO: "audio",
  TEXT: "text",
  IMAGE: "image",
  URL: "url",
} as const;
export type CaptureType = (typeof CAPTURE_TYPES)[keyof typeof CAPTURE_TYPES];
```
**And** le type `CaptureType` est exporté pour usage dans tout le codebase
**Note Dev :** ✅ **DÉJÀ CONFORME** — ligne 16-23 de `Capture.model.ts`. Vérification uniquement.

### AC2: Interface Capture — Vérification de conformité

**Given** l'interface `Capture` dans `Capture.model.ts`
**When** j'inspecte le champ `type`
**Then** le champ utilise `CaptureType` (pas `string`) : `type: CaptureType;`
**Note Dev :** ✅ **DÉJÀ CONFORME** — ligne 47 de `Capture.model.ts`. Vérification uniquement.

### AC3: CaptureEvents — Vérification de conformité

**Given** les interfaces d'événements dans `CaptureEvents.ts`
**When** j'inspecte les payloads
**Then** `captureType` utilise `CaptureType` (pas les unions inline)
**And** l'import est : `import { type CaptureType } from '../domain/Capture.model'`
**Note Dev :** ✅ **DÉJÀ CONFORME** — lignes 13, 29, 51 de `CaptureEvents.ts`. Vérification uniquement.

### AC4: Supprimer les type assertions incomplètes dans CaptureRepository

**Given** `CaptureRepository.ts` publie des événements avec `captureType`
**When** j'inspecte les 3 emplacements où `capture.type` est casté
**Then** les assertions `as 'audio' | 'text'` sont remplacées par `as CaptureType`
**And** `CaptureType` est importé depuis `'../domain/Capture.model'`
**And** l'IDE n'affiche plus d'avertissement sur les types incohérents
**And** les 4 types (audio, text, image, url) sont correctement supportés dans les événements

**Fichier cible :** `mobile/src/contexts/capture/data/CaptureRepository.ts`
**Lignes :** 176, 306, 463 (résidu du fix Story 8.1)

```typescript
// AVANT (Story 8.1 — fix incomplet)
captureType: capture.type as 'audio' | 'text',

// APRÈS (Story 8.2 — fix complet)
captureType: capture.type as CaptureType,
```

### AC5: Supprimer les magic strings dans WaveformExtractionService

**Given** `WaveformExtractionService.ts` compare un type de capture
**When** j'inspecte la condition ligne 135
**Then** le string literal `'audio'` est remplacé par `CAPTURE_TYPES.AUDIO`
**And** `CAPTURE_TYPES` est importé depuis `'../../capture/domain/Capture.model'` (ou path approprié)
**And** le comportement runtime reste identique

```typescript
// AVANT
if (capture.type !== 'audio') { ... }

// APRÈS
if (capture.type !== CAPTURE_TYPES.AUDIO) { ... }
```

### AC6: Mettre à jour les tests — remplacer les literals par les constantes

**Given** des fichiers de tests utilisent des string literals pour les types de capture
**When** je recherche toutes les occurrences
**Then** tous les `'audio'`, `'text'`, `'image'`, `'url'` en tant que valeurs de type capture sont remplacés par `CAPTURE_TYPES.AUDIO`, `.TEXT`, `.IMAGE`, `.URL`
**And** les imports `CAPTURE_TYPES` sont ajoutés aux fichiers de test concernés
**And** tous les tests passent après les modifications

### AC7: Vérification TypeScript et tests de non-régression

**Given** toutes les modifications AC4-AC6 sont appliquées
**When** j'exécute `npx tsc --noEmit` dans `pensieve/mobile/`
**Then** zéro nouvelle erreur TypeScript introduite par cette story
**And** `npm run test:acceptance` passe sans régression
**And** `npm run test:unit` passe sans régression
**And** `npm run test:architecture` passe (conformité ADR)

## Tasks / Subtasks

### Task 1: Audit de conformité (AC1-AC3) — lecture seule
- [x] Subtask 1.1 : Vérifier `Capture.model.ts` — confirmer lignes 16-23 (CAPTURE_TYPES) et ligne 47 (`type: CaptureType`) ✅
- [x] Subtask 1.2 : Vérifier `CaptureEvents.ts` — confirmer import CaptureType + usage lignes 29, 51 ✅
- [x] Subtask 1.3 : Confirmer que RecordingService et TextCaptureService utilisent déjà CAPTURE_TYPES.AUDIO/TEXT ✅
- [x] Subtask 1.4 : Si tout est conforme, passer à Task 2 sans modification

### Task 2: Fix CaptureRepository.ts — 3 type assertions (AC4)
- [x] Subtask 2.1 : Ouvrir `mobile/src/contexts/capture/data/CaptureRepository.ts`
- [x] Subtask 2.2 : Vérifier la ligne d'import en haut du fichier — `CaptureType` déjà importé (ligne 21)
- [x] Subtask 2.3 : Ligne 176 — changer `capture.type as 'audio' | 'text'` → `capture.type as CaptureType`
- [x] Subtask 2.4 : Ligne 306 — même correction (méthode `update()`)
- [x] Subtask 2.5 : Ligne 463 — même correction (méthode `delete()`)
- [x] Subtask 2.6 : Vérifier qu'il n'y a pas d'autre occurrence de `'audio' | 'text'` dans ce fichier — aucune trouvée ✅

### Task 3: Fix WaveformExtractionService.ts (AC5)
- [x] Subtask 3.1 : Ouvrir `mobile/src/contexts/capture/services/WaveformExtractionService.ts`
- [x] Subtask 3.2 : Trouver le fichier exact — trouvé ligne 135
- [x] Subtask 3.3 : Ajouter l'import `CAPTURE_TYPES` depuis `'../domain/Capture.model'`
- [x] Subtask 3.4 : Remplacer `!== 'audio'` par `!== CAPTURE_TYPES.AUDIO`
- [x] Subtask 3.5 : Vérifier si d'autres comparaisons de ce type existent — aucune autre

### Task 4: Recherche exhaustive des magic strings restants (AC6)
- [x] Subtask 4.1 : Exécuter la recherche complète — 5 fichiers src/ + 5 fichiers tests/ identifiés
- [x] Subtask 4.2 : Fichiers sources corrigés : TranscriptionQueueProcessor.ts (×2), TranscriptionModelService.ts, useCaptureActions.ts — plus fix casse `Capture→capture` dans les imports
- [x] Subtask 4.3 : Fichiers tests corrigés : MockCaptureRepository.ts, story-2-7 (×10), story-6-5, capture-model.test.ts, text-capture-service.test.ts
- [x] Subtask 4.4 : Fichiers SQL/migrations non touchés ✅

### Task 5: Validation finale (AC7)
- [x] Subtask 5.1 : `npx tsc --noEmit` — zéro NOUVELLE erreur (erreurs pré-existantes uniquement)
- [x] Subtask 5.2 : `npm run test:acceptance` — zéro régression (18 FAILs pré-existants, story-6-5 bonus PASS)
- [x] Subtask 5.3 : `npm run test:unit` — zéro régression (57 suites FAIL identiques avant/après)
- [x] Subtask 5.4 : `npm run test:architecture` — zéro régression (6 FAILs pré-existants)
- [ ] Subtask 5.5 : Fermer l'issue GitHub #24 après merge (commenter : "Fixed in story 8.2")

## Dev Notes

### 🎯 Périmètre Exact — Ce qui Change vs Ce qui Reste

#### ✅ DÉJÀ CONFORME — Ne PAS modifier

| Fichier | Statut | Raison |
|---------|--------|--------|
| `capture/domain/Capture.model.ts` | ✅ Conforme | CAPTURE_TYPES + CaptureType + `type: CaptureType` déjà présents |
| `capture/events/CaptureEvents.ts` | ✅ Conforme | Importe et utilise `CaptureType` (lignes 13, 29, 51) |
| `capture/services/RecordingService.ts` | ✅ Conforme | Utilise `CAPTURE_TYPES.AUDIO` |
| `capture/services/TextCaptureService.ts` | ✅ Conforme | Utilise `CAPTURE_TYPES.TEXT` |

#### ❌ À CORRIGER

| Fichier | Problème | Fix |
|---------|---------|-----|
| `capture/data/CaptureRepository.ts` (×3) | `as 'audio' \| 'text'` — union narrow incomplète | → `as CaptureType` |
| `WaveformExtractionService.ts:135` | `!== 'audio'` — magic string | → `!== CAPTURE_TYPES.AUDIO` |
| Fichiers tests (TBD via grep) | String literals directes | → `CAPTURE_TYPES.X` |

### 🔍 Héritage Story 8.1

La Story 8.1 a appliqué des type assertions minimales dans `CaptureRepository.ts` pour corriger les erreurs de compilation TypeScript (issues #22, #21, #20). La note de dev Story 8.1 spécifie :

> _"Le fix minimal et sûr est la type assertion `as 'audio' | 'text'`. Une refactorisation complète (utiliser `CAPTURE_TYPES` dans les interfaces d'événements ou modifier `Capture.model.ts`) serait hors scope de cette story. La Story 8.2 standardise ce pattern."_

**Conséquence :** `CaptureRepository.ts` a intentionnellement 3 casts incomplets qui sont exactement le scope de cette story.

### 🏗️ Pattern de Référence — as const

Le pattern standard du projet (déjà appliqué dans `ANALYSIS_TYPES`, `METADATA_KEYS`, `CAPTURE_STATES`, `FEATURE_KEYS`) :

```typescript
// 1. Définition constante (dans le fichier domain/)
export const MY_TYPES = {
  TYPE_A: "value_a",
  TYPE_B: "value_b",
} as const;

// 2. Type dérivé (dans le même fichier)
export type MyType = (typeof MY_TYPES)[keyof typeof MY_TYPES];

// 3. Utilisation dans interfaces
interface MyEntity {
  type: MyType;  // ✅ Pas string
}

// 4. Utilisation dans le code
if (entity.type === MY_TYPES.TYPE_A) { ... }  // ✅ Pas 'value_a'
```

### 🔧 Fix CaptureRepository — Détail Technique

Les 3 casts dans `CaptureRepository.ts` existent parce que la donnée vient de OP-SQLite (layer DB → string brut) mais le payload de l'événement attend `CaptureType`. Le cast est légitime, mais doit être vers le bon type :

```typescript
// Ligne 176 (méthode create() — CaptureRecordedEvent)
captureType: capture.type as CaptureType,  // ✅ (était: as 'audio' | 'text')

// Ligne 306 (méthode update() — CaptureRecordedEvent)
captureType: capture.type as CaptureType,  // ✅

// Ligne 463 (méthode delete() — CaptureDeletedEvent)
captureType: capture.type as CaptureType,  // ✅
```

**Pourquoi `as CaptureType` et non supprimer le cast ?**
La donnée issue de OP-SQLite est typée `string` dans la couche data. Le cast est nécessaire pour satisfaire le type attendu par l'interface d'événement. Utiliser `CaptureType` (qui inclut les 4 valeurs) est plus correct que l'union narrow `'audio' | 'text'` qui excluait IMAGE et URL.

**Import à vérifier/ajouter :**
```typescript
// Dans CaptureRepository.ts, l'import depuis Capture.model.ts doit inclure CaptureType :
import { type CaptureType, /* autres imports */ } from '../domain/Capture.model';
```

### 🔬 Recherche Exhaustive — Commandes Utiles

```bash
# Depuis pensieve/mobile/

# Trouver toutes les occurrences de cast incomplet
grep -rn "as 'audio' | 'text'" src/

# Trouver tous les string literals de types de capture
grep -rn "'audio'\|'text'\|'image'\|'url'" src/contexts/capture/ tests/acceptance/

# Identifier les fichiers de tests à mettre à jour
grep -rln "'audio'\|'text'" tests/acceptance/

# Vérifier WaveformExtractionService
grep -rn "!== 'audio'\|=== 'audio'\|=== 'text'\|!== 'text'" src/
```

### ⚠️ Garde-Fous Critiques

1. **NE PAS toucher les migrations SQL** — les valeurs 'audio', 'text', etc. dans les migrations sont des données DB, pas des constantes TypeScript
2. **NE PAS modifier `Capture.model.ts` ni `CaptureEvents.ts`** — ils sont déjà conformes
3. **NE PAS introduire de nouvelles dépendances** — cette story est un cleanup, pas une feature
4. **NE PAS changer le comportement runtime** — les valeurs en base de données restent "audio", "text", etc. (les constantes TypeScript ont les mêmes valeurs string)

### 📐 Architecture Compliance

- **ADR-024 (Clean Code)** : Supprimer les magic strings → utiliser les constantes nommées — ✅ Ce que fait cette story
- **ADR-023 (Result Pattern)** : Pas impacté par cette story
- **Tests d'architecture** : `npm run test:architecture` doit passer — vérifie les dépendances circulaires et conformité ADR

### Project Structure Notes

- **Bounded Context Capture** : `mobile/src/contexts/capture/`
  - `domain/` — Capture.model.ts, interfaces pures
  - `data/` — CaptureRepository.ts (OP-SQLite)
  - `events/` — CaptureEvents.ts (domain events)
  - `services/` — WaveformExtractionService.ts, RecordingService.ts, etc.
- **Tests** : `mobile/tests/acceptance/features/*.feature` + `mobile/tests/acceptance/story-*.test.ts`
- **DI tokens** : `mobile/src/infrastructure/di/tokens.ts`
- **Test infrastructure** : `mobile/tests/acceptance/support/test-context.ts` (12 mocks in-memory)

### References

- [Source: pensieve issue #24](https://github.com/yohikofox/pensieve/issues/24) — Spécification complète avec exemples de code
- [Source: story 8.1 dev notes] — Contexte du fix incomplet dans CaptureRepository.ts
- [Source: mobile/src/contexts/capture/domain/Capture.model.ts#1-36] — CAPTURE_TYPES et CaptureType (référence)
- [Source: mobile/src/contexts/capture/events/CaptureEvents.ts#12-51] — CaptureEvents conforme (référence)
- [Source: _bmad-output/planning-artifacts/architecture.md] — DDD Bounded Context Capture

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

Story 8.2 implémentée le 2026-02-28 :

1. **AC1-AC3 (audit)** : Confirmé conforme sans modification — CAPTURE_TYPES, CaptureType, CaptureEvents.ts tous conformes.

2. **AC4 (CaptureRepository)** : 3 casts `as 'audio' | 'text'` remplacés initialement par `as CaptureType`, puis **supprimés entièrement** lors du code review (cast redondant — `mapRowToCapture()` renvoie déjà `Capture` avec `type: CaptureType`). `captureType: capture.type` suffit.

3. **AC5 (WaveformExtractionService)** : Import `CAPTURE_TYPES` ajouté + `!== 'audio'` → `!== CAPTURE_TYPES.AUDIO` ligne 135.

4. **AC6 (magic strings exhaustif)** : 5 fichiers src/ corrigés (TranscriptionQueueProcessor ×2, TranscriptionModelService, useCaptureActions) + 5 fichiers tests/ acceptances (MockCaptureRepository, story-2-7 ×10, story-6-5, capture-model, text-capture-service). Bonus : casse `'../../Capture/...'` → `'../../capture/...'` corrigée dans TranscriptionQueueProcessor et son test.

5. **AC7 (validation)** : Zéro nouvelle erreur TypeScript. Zéro régression (tests:acceptance 18 FAILs pré-existants, unit 57 FAILs pré-existants, architecture 6 FAILs pré-existants). story-6-5 bonus : passé de FAIL à PASS grâce au fix AC6.

Code review (2026-02-28) — 7 findings (2H + 3M + 2L), 5 HIGH+MEDIUM fixés :
- **H1 corrigé** : `capture-integration.test.ts` (19 magic strings) + `capture-performance.test.ts` (15 magic strings) — CAPTURE_TYPES importé + remplacements
- **H2 corrigé** : `story-2-7` 5 filter lambdas restants (`c.type === 'audio'`) → `CAPTURE_TYPES.AUDIO`
- **M1 corrigé** : Casts `as CaptureType` redondants supprimés dans CaptureRepository.ts (lignes 176, 306, 463) — `capture.type` est déjà `CaptureType` via mapper
- **M2 corrigé** : `TranscriptionModelService.ts` — `CAPTURE_STATES` importé + `capture.state === 'captured'` → `CAPTURE_STATES.CAPTURED`
- **M3 corrigé** : Messages de log `[TranscriptionModelService] AC6: ...` nettoyés (référence story supprimée des logs runtime)
- **L1 non fixé** : `useCaptureActions.ts` titres partage non-i18n → déféré (hors scope story 8.2)
- **L2 non fixé** : `MockCaptureRepository.ts` type union inline états → déféré (hors scope story 8.2)

### File List

**Fichiers modifiés (code source) :**
- `pensieve/mobile/src/contexts/capture/data/CaptureRepository.ts` (AC4 : cast redondant supprimé × 3)
- `pensieve/mobile/src/contexts/capture/services/WaveformExtractionService.ts` (AC5 : import + magic string)
- `pensieve/mobile/src/contexts/Normalization/processors/TranscriptionQueueProcessor.ts` (AC6 : import CAPTURE_TYPES + 2 comparaisons + fix casse)
- `pensieve/mobile/src/contexts/Normalization/services/TranscriptionModelService.ts` (AC6 + code review M2/M3 : CAPTURE_STATES + log cleanup)
- `pensieve/mobile/src/hooks/useCaptureActions.ts` (AC6 : import CAPTURE_TYPES + 1 comparaison)
- `pensieve/mobile/src/contexts/Normalization/processors/__tests__/TranscriptionQueueProcessor.test.ts` (fix casse import)

**Fichiers modifiés (tests) :**
- `pensieve/mobile/tests/acceptance/support/mocks/MockCaptureRepository.ts` (AC6 : CaptureType)
- `pensieve/mobile/tests/acceptance/story-2-7-guide-config-modele.test.ts` (AC6 : 10 occ. + code review H2 : 5 filter lambdas)
- `pensieve/mobile/tests/acceptance/story-6-5.test.ts` (AC6 : 1 occurrence CAPTURE_TYPES.AUDIO)
- `pensieve/mobile/tests/acceptance/capture/capture-model.test.ts` (AC6 : CAPTURE_TYPES.AUDIO × 7)
- `pensieve/mobile/tests/acceptance/capture/text-capture-service.test.ts` (AC6 : CAPTURE_TYPES.TEXT × 1)
- `pensieve/mobile/src/contexts/capture/__tests__/capture-integration.test.ts` (code review H1 : 19 occ. CAPTURE_TYPES.AUDIO)
- `pensieve/mobile/src/contexts/capture/__tests__/capture-performance.test.ts` (code review H1 : 15 occ. CAPTURE_TYPES.AUDIO)

**Fichiers de documentation :**
- `_bmad-output/implementation-artifacts/sprint-status.yaml` (status → review)
- `_bmad-output/implementation-artifacts/stories/epic-8/8-2-standardize-type-constants-pattern.md` (ce fichier)

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-02-28 | Story créée depuis issue GitHub #24 — contexte Story 8.1 analysé, scope ciblé sur 3 fichiers | yohikofox |
| 2026-02-28 | Implémentation complète — AC1-AC7 satisfaits — cleanup magic strings exhaustif + fix casse imports | claude-sonnet-4-6 |
| 2026-02-28 | Code review adversariale — 7 findings (2H+3M+2L), 5 fixés : H1 2 fichiers test manqués, H2 story-2-7 filter lambdas, M1 casts redondants supprimés, M2 CAPTURE_STATES, M3 logs AC6 nettoyés | claude-sonnet-4-6 |

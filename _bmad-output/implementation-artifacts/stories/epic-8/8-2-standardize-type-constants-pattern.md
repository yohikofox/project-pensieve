# Story 8.2: Standardize Type Constants Pattern

Status: ready-for-dev

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
- [ ] Subtask 1.1 : Vérifier `Capture.model.ts` — confirmer lignes 16-23 (CAPTURE_TYPES) et ligne 47 (`type: CaptureType`) ✅
- [ ] Subtask 1.2 : Vérifier `CaptureEvents.ts` — confirmer import CaptureType + usage lignes 29, 51 ✅
- [ ] Subtask 1.3 : Confirmer que RecordingService et TextCaptureService utilisent déjà CAPTURE_TYPES.AUDIO/TEXT ✅
- [ ] Subtask 1.4 : Si tout est conforme, passer à Task 2 sans modification

### Task 2: Fix CaptureRepository.ts — 3 type assertions (AC4)
- [ ] Subtask 2.1 : Ouvrir `mobile/src/contexts/capture/data/CaptureRepository.ts`
- [ ] Subtask 2.2 : Vérifier la ligne d'import en haut du fichier — si `CaptureType` n'est pas importé, ajouter dans l'import existant depuis `'../domain/Capture.model'`
- [ ] Subtask 2.3 : Ligne 176 — changer `capture.type as 'audio' | 'text'` → `capture.type as CaptureType`
- [ ] Subtask 2.4 : Ligne 306 — même correction (méthode `update()`)
- [ ] Subtask 2.5 : Ligne 463 — même correction (méthode `delete()`)
- [ ] Subtask 2.6 : Vérifier qu'il n'y a pas d'autre occurrence de `'audio' | 'text'` dans ce fichier avec `grep "as 'audio'" mobile/src/contexts/capture/data/CaptureRepository.ts`

### Task 3: Fix WaveformExtractionService.ts (AC5)
- [ ] Subtask 3.1 : Ouvrir `mobile/src/contexts/capture/services/WaveformExtractionService.ts` (ou chemin exact)
- [ ] Subtask 3.2 : Trouver le fichier exact : `grep -rn "!== 'audio'" mobile/src/`
- [ ] Subtask 3.3 : Ajouter l'import `CAPTURE_TYPES` depuis le bon chemin relatif
- [ ] Subtask 3.4 : Remplacer `!== 'audio'` par `!== CAPTURE_TYPES.AUDIO`
- [ ] Subtask 3.5 : Vérifier si d'autres comparaisons de ce type existent dans le fichier

### Task 4: Recherche exhaustive des magic strings restants (AC6)
- [ ] Subtask 4.1 : Exécuter la recherche complète :
  ```bash
  grep -rn "captureType.*:.*['\"]audio['\"]" mobile/src/ mobile/tests/
  grep -rn "type.*:.*['\"]audio['\"]" mobile/src/contexts/capture/ mobile/tests/
  grep -rn "=== 'audio'\|!== 'audio'\|=== 'text'\|!== 'text'" mobile/src/ mobile/tests/
  ```
- [ ] Subtask 4.2 : Pour chaque occurrence dans les fichiers sources (`src/`) : remplacer par `CAPTURE_TYPES.AUDIO/TEXT/IMAGE/URL`
- [ ] Subtask 4.3 : Pour chaque occurrence dans les fichiers tests (`tests/`) : remplacer par `CAPTURE_TYPES.AUDIO/TEXT/IMAGE/URL` avec import approprié
- [ ] Subtask 4.4 : **NE PAS** toucher les fichiers SQL ou les migrations — ces valeurs sont des données persistées, pas des constantes TypeScript

### Task 5: Validation finale (AC7)
- [ ] Subtask 5.1 : `cd pensieve/mobile && npx tsc --noEmit` — zéro nouvelle erreur
- [ ] Subtask 5.2 : `npm run test:acceptance` — tous les tests BDD passent
- [ ] Subtask 5.3 : `npm run test:unit` — aucune régression
- [ ] Subtask 5.4 : `npm run test:architecture` — conformité ADR maintenue
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

### File List

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-02-28 | Story créée depuis issue GitHub #24 — contexte Story 8.1 analysé, scope ciblé sur 3 fichiers | yohikofox |

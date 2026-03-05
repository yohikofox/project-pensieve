# Story 16.1: Verrouillage des actions et feedback visuel pendant le traitement d'une capture

Status: done

## Story

En tant qu'**utilisateur**,
je veux **que les actions non pertinentes soient desactivees et qu'un indicateur visuel m'informe qu'un traitement est en cours sur une capture**,
afin de **ne pas declencher d'actions conflictuelles et de comprendre l'etat de ma capture a tout moment**.

## Context

Constat terrain : lorsqu'une transcription est lancee depuis la liste des captures, aucun feedback visuel immediat n'apparait. L'utilisateur peut relancer une transcription, supprimer la capture, ou declencher une analyse alors qu'un traitement est en cours. Depuis la vue detail, les boutons d'action (analyse, retry, post-traitement, debug) restent actifs pendant le traitement.

**Regle** : Pendant un traitement actif (state === 'processing'), seule la lecture audio doit rester accessible. Le retry n'est autorise que si le statut est 'failed'.

**Etats concernes** (Capture.model.ts) :
- `processing` -> toutes les actions verrouillees sauf lecture
- `captured` -> transcription et suppression autorisees
- `ready` -> toutes les actions autorisees
- `failed` -> retry et suppression autorises

## Acceptance Criteria

### AC1 — Liste : Actions desactivees pendant le traitement

**Given** une capture est a l'etat `processing`
**When** l'utilisateur visualise la carte dans la liste
**Then** les actions de swipe (supprimer, partager) sont desactivees
**And** le bouton "Transcrire" n'est pas affiche
**And** le bouton Play/Pause reste fonctionnel

### AC2 — Liste : Feedback visuel de traitement en cours

**Given** une capture vient de passer a l'etat `processing`
**When** l'utilisateur visualise la carte dans la liste
**Then** un spinner est visible sur la carte
**And** un badge "En cours..." est affiche
**And** les elements d'action desactives apparaissent visuellement comme tels (opacite reduite ou grises)

### AC3 — Detail : Actions desactivees pendant le traitement

**Given** une capture est a l'etat `processing`
**When** l'utilisateur est sur l'ecran de detail de cette capture
**Then** les boutons Post-traitement, Analyse, Re-transcrire (debug), et Supprimer sont desactives
**And** le bouton de lecture audio reste fonctionnel
**And** les boutons desactives montrent un etat visuel "disabled" clair

### AC4 — Detail : Feedback visuel de traitement en cours

**Given** une capture est a l'etat `processing`
**When** l'utilisateur est sur l'ecran de detail
**Then** un indicateur de traitement (spinner + texte) est visible dans la section d'actions
**And** la transition vers `ready` ou `failed` met a jour l'UI automatiquement (event-driven, deja en place via useCaptureDetailListener)

### AC5 — Retry autorise uniquement en etat failed

**Given** une capture est a l'etat `failed`
**When** l'utilisateur visualise la capture (liste ou detail)
**Then** le bouton Retry est actif
**And** les actions de suppression sont autorisees
**And** les autres actions (analyse, post-traitement) restent desactivees

### AC6 — Pas de double declenchement de transcription

**Given** une capture est a l'etat `processing` ou en file d'attente (`isInQueue`)
**When** l'utilisateur tente de lancer une transcription (via bouton ou autre chemin)
**Then** l'action est ignoree ou le bouton est absent/desactive

## Tasks / Subtasks

- [x] **Task 1 — Guard des actions dans CaptureCard (liste)** (AC: #1, #2, #6)
  - [x] 1.1 — Ajouter helper `isProcessing(capture)` : retourne `true` si `state === 'processing'` ou `isInQueue === true`
  - [x] 1.2 — Desactiver/masquer le bouton Transcrire quand `isProcessing`
  - [x] 1.3 — Desactiver le bouton Retry sauf si `state === 'failed'`
  - [x] 1.4 — Verifier que le badge "En cours" avec spinner est bien visible (deja partiellement implemente)
  - [x] 1.5 — Ajouter opacite reduite sur la zone d'actions quand processing

- [x] **Task 2 — Desactiver le swipe pendant le traitement** (AC: #1)
  - [x] 2.1 — Passer `enabled={!isProcessing}` a `SwipeableCard` depuis `CaptureListItem`
  - [x] 2.2 — Le prop `enabled` existe deja sur SwipeableCard (verifie)

- [x] **Task 3 — Guard des actions dans l'ecran Detail** (AC: #3, #4, #5)
  - [x] 3.1 — Identifier tous les boutons d'action dans CaptureDetailContent et ActionsSection
  - [x] 3.2 — Desactiver Post-traitement, Analyse, Re-transcrire, Supprimer quand `state === 'processing'`
  - [x] 3.3 — Laisser Retry actif uniquement si `state === 'failed'`
  - [x] 3.4 — Ajouter un indicateur visuel (spinner + "Traitement en cours...") dans la section actions
  - [x] 3.5 — Verifier que useCaptureDetailListener met bien a jour l'UI (deja event-driven)

- [x] **Task 4 — Guard dans ActionBar (detail)** (AC: #3)
  - [x] 4.1 — Desactiver Supprimer dans l'ActionBar quand `state === 'processing'`
  - [x] 4.2 — Partager et Copier restent actifs (lecture seule, pas de conflit)

- [x] **Task 5 — Tests** (AC: tous)
  - [x] 5.1 — Tests unitaires pour le helper `isProcessing`
  - [x] 5.2 — Verifier les tests existants ne regressent pas

## Dev Notes

### Etats et matrice d'actions

| Action | captured | processing | ready | failed |
|--------|----------|-----------|-------|--------|
| Play/Pause | ✅ | ✅ | ✅ | ✅ |
| Transcrire | ✅ | ❌ | ❌ | ❌ |
| Retry | ❌ | ❌ | ❌ | ✅ |
| Supprimer | ✅ | ❌ | ✅ | ✅ |
| Partager | ✅ | ❌ | ✅ | ✅ |
| Post-traitement | ❌ | ❌ | ✅ | ❌ |
| Analyse | ❌ | ❌ | ✅ | ❌ |
| Re-transcrire (debug) | ❌ | ❌ | ✅ | ✅ |

### Fichiers concernes

| Fichier | Changement |
|---------|-----------|
| `mobile/src/components/captures/CaptureCard.tsx` | Desactiver actions selon etat, feedback visuel |
| `mobile/src/components/captures/CaptureListItem.tsx` | Passer `isProcessing` a SwipeableCard.enabled |
| `mobile/src/components/cards/SwipeableCard.tsx` | Deja le prop `enabled` — juste consumer |
| `mobile/src/screens/captures/CaptureDetailContent.tsx` | Guard actions, spinner processing |
| `mobile/src/components/capture/ActionsSection.tsx` | Desactiver boutons pendant processing |
| `mobile/src/components/capture/ActionBar.tsx` (a verifier) | Desactiver Supprimer pendant processing |

### Pattern existant a reutiliser

Le `CaptureCard` affiche deja un badge "En cours" avec `ActivityIndicator` quand `state === 'processing'`. Il faut juste completer :
1. Desactiver les boutons d'action correspondants
2. Propager l'info au `SwipeableCard`

L'event-driven update via `useCaptureDetailListener` fonctionne deja — pas besoin de toucher au systeme d'evenements.

### Contraintes

- Ne pas bloquer la lecture audio (Play/Pause) — c'est un acces en lecture seule
- Le Copier et Partager (contenu texte deja transcrit) restent actifs dans le detail si `state === 'ready'`
- Le systeme de retry avec fenetre 20min et limite 3 essais existe deja (RetryLimitService) — ne pas le modifier

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Implementation Plan

Audit complet de l'état existant avant implémentation :
- Tasks 2, 1.2, 1.3, 1.4, 3.2 (partiel), 3.4, 4.1 : déjà implémentées dans les stories précédentes
- Manquant : helper `isProcessing` extractable (1.1), opacité zone actions (1.5), tests unitaires (5.1)

**Choix d'implémentation :**
- Helper créé dans `contexts/capture/utils/capture.guards.ts` (pattern existant avec `actionItemParser.ts`)
- Import avec alias `isCaptureProcessing` pour éviter collision avec la variable locale `isProcessing`
- Opacité 0.6 appliquée à la View des boutons d'action — Play/Pause reste tappable (opacity ne bloque pas l'interaction)
- `AnalysisCard` n'est rendu que si `isReady` → boutons Analyse naturellement inaccessibles pendant processing

### Completion Notes List

- ✅ `capture.guards.ts` créé avec `isProcessing(capture)` helper — 8/8 tests unitaires verts
- ✅ `AudioCaptureCard` refactorisé pour utiliser le helper partagé + opacité 0.6 sur zone actions (Task 1.5)
- ✅ `CaptureListItem` refactorisé pour utiliser le helper partagé
- ✅ Tous les AC vérifiés : swipe désactivé (AC1), opacité (AC2), ActionsSection (AC3/AC4), Retry (AC5), pas de double déclenchement (AC6)
- ✅ Aucune régression introduite — échecs pré-existants confirmés (Capture.retry.test.ts, etc.)

### File List

- `pensieve/mobile/src/contexts/capture/utils/capture.guards.ts` (créé)
- `pensieve/mobile/src/contexts/capture/utils/__tests__/capture.guards.test.ts` (créé)
- `pensieve/mobile/src/components/captures/AudioCaptureCard.tsx` (modifié)
- `pensieve/mobile/src/components/captures/CaptureListItem.tsx` (modifié)
- `pensieve/mobile/src/components/capture/ActionsSection.tsx` (modifié — guard partagé, vérifié conforme AC3/AC4)
- `pensieve/mobile/src/components/capture/ActionBar.tsx` (modifié — guard partagé, vérifié conforme AC3 Supprimer)
- `pensieve/mobile/tests/acceptance/features/story-16-1-capture-processing-guards.feature` (créé)
- `pensieve/mobile/tests/acceptance/story-16-1.test.ts` (créé)

### Change Log

- 2026-03-05: Story 16.1 implémentée — helper `isProcessing` extractable + opacité zone actions (Tasks 1.1, 1.5, 5.1, 5.2). Tasks 1.2, 1.3, 1.4, 2, 3, 4 déjà présentes dans le codebase.
- 2026-03-05: Code review adversarial — 6 issues HIGH/MEDIUM corrigées : (H1) Play/Pause sorti de la zone opacifiée dans AudioCaptureCard ; (H2/H3) RetryLimitService dans useMemo + typage CaptureWithRetry explicite sans as any ; (M1) guard partagé `isCaptureProcessingGuard` adopté dans ActionsSection et ActionBar ; (M2) feature Gherkin + step definitions BDD créés (7 scénarios, 7 passants) ; (M3) File List complétée avec ActionsSection, ActionBar, tests BDD. Fixes mineurs L2 (remainingTime != null).

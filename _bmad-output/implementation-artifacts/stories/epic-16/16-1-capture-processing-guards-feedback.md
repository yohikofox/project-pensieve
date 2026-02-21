# Story 16.1: Verrouillage des actions et feedback visuel pendant le traitement d'une capture

Status: ready-for-dev

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

- [ ] **Task 1 — Guard des actions dans CaptureCard (liste)** (AC: #1, #2, #6)
  - [ ] 1.1 — Ajouter helper `isProcessing(capture)` : retourne `true` si `state === 'processing'` ou `isInQueue === true`
  - [ ] 1.2 — Desactiver/masquer le bouton Transcrire quand `isProcessing`
  - [ ] 1.3 — Desactiver le bouton Retry sauf si `state === 'failed'`
  - [ ] 1.4 — Verifier que le badge "En cours" avec spinner est bien visible (deja partiellement implemente)
  - [ ] 1.5 — Ajouter opacite reduite sur la zone d'actions quand processing

- [ ] **Task 2 — Desactiver le swipe pendant le traitement** (AC: #1)
  - [ ] 2.1 — Passer `enabled={!isProcessing}` a `SwipeableCard` depuis `CaptureListItem`
  - [ ] 2.2 — Le prop `enabled` existe deja sur SwipeableCard (verifie)

- [ ] **Task 3 — Guard des actions dans l'ecran Detail** (AC: #3, #4, #5)
  - [ ] 3.1 — Identifier tous les boutons d'action dans CaptureDetailContent et ActionsSection
  - [ ] 3.2 — Desactiver Post-traitement, Analyse, Re-transcrire, Supprimer quand `state === 'processing'`
  - [ ] 3.3 — Laisser Retry actif uniquement si `state === 'failed'`
  - [ ] 3.4 — Ajouter un indicateur visuel (spinner + "Traitement en cours...") dans la section actions
  - [ ] 3.5 — Verifier que useCaptureDetailListener met bien a jour l'UI (deja event-driven)

- [ ] **Task 4 — Guard dans ActionBar (detail)** (AC: #3)
  - [ ] 4.1 — Desactiver Supprimer dans l'ActionBar quand `state === 'processing'`
  - [ ] 4.2 — Partager et Copier restent actifs (lecture seule, pas de conflit)

- [ ] **Task 5 — Tests** (AC: tous)
  - [ ] 5.1 — Tests unitaires pour le helper `isProcessing`
  - [ ] 5.2 — Verifier les tests existants ne regressent pas

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

TBD

### Completion Notes List

TBD

### File List

TBD

# Story 3.5: Vue Détail d'une Capture Texte — Layout Dédié & Édition Inline

Status: done

## Contexte

Retour utilisateur (test naïf, profil non-tech) : modifier le texte d'une capture textuelle nécessite actuellement 3+ actions non-intuitives — ouvrir le détail, trouver l'onglet "Transcription" (terme audio), éditer, fermer le clavier pour accéder au bouton save. Ce flux est un non-sens UX.

L'AC3 de la Story 3.2 spécifie "the UI adapts to show text-specific layout (no audio player)" — cette story livre cet engagement avec un layout dédié et une édition inline.

## Story

As a **user**,
I want **the detail screen of a text capture to display my text directly and let me edit it with a clear and explicit action**,
So that **I can read and correct my content without navigating through unintuitive tabs** (FR22, FR24).

## Acceptance Criteria

### AC1: Layout dédié pour les captures texte
**Given** I open the detail of a text capture
**When** the detail screen loads
**Then** the text content is displayed inline in the zone where the audio player appears for audio captures
**And** the "Analyse" section is displayed below (no tab navigation required)
**And** there is no "Transcription" tab in the interface

### AC2: Mode lecture (défaut)
**Given** I am viewing the detail of a text capture
**When** the screen loads
**Then** the text is displayed in read-only mode
**And** a "Modifier" button is visible near the text zone
**And** the ActionBar shows: Copy, Share, Delete

### AC3: Passage en mode édition
**Given** I am in read mode on a text capture detail
**When** I tap the "Modifier" button
**Then** the text zone switches to an editable TextInput with a distinct visual style (border or background change)
**And** the keyboard opens automatically with focus on the text
**And** the ActionBar changes to show only: Cancel, Save
**And** the Analyse section is no longer visible (pushed by keyboard or hidden)

### AC4: Sauvegarde de la modification
**Given** I am in edit mode and have modified the text
**When** I tap "Save" in the ActionBar
**Then** the text is persisted (normalizedText updated)
**And** the screen returns to read mode
**And** the updated text is displayed

### AC5: Annulation de la modification
**Given** I am in edit mode
**When** I tap "Cancel" in the ActionBar
**Then** the text reverts to its original value
**And** the screen returns to read mode
**And** no changes are persisted

### AC6: Pas de régression sur les captures audio
**Given** I open the detail of an audio capture
**When** the detail screen loads
**Then** the layout is identical to the existing implementation (player + tabs Analyse/Transcription)
**And** no existing functionality is broken

## Tasks / Subtasks

- [x] **Task 1: Créer CaptureTextDetailContent** (AC: 1, 2)
  - [x] Subtask 1.1: Extraire le layout texte depuis `CaptureDetailContent.tsx` dans un composant dédié `CaptureTextDetailContent.tsx`
  - [x] Subtask 1.2: Afficher le texte brut inline (zone équivalente au player audio)
  - [x] Subtask 1.3: Afficher la section Analyse directement sous le texte (sans onglet)
  - [x] Subtask 1.4: Router vers `CaptureTextDetailContent` ou `CaptureAudioDetailContent` selon `capture.type` dans `CaptureDetailContent.tsx`

- [x] **Task 2: Implémenter le mode édition inline** (AC: 2, 3)
  - [x] Subtask 2.1: Ajouter bouton "Modifier" dans la zone texte (mode lecture)
  - [x] Subtask 2.2: State `isEditingText` dans `captureDetailStore` (partagé avec ActionBar)
  - [x] Subtask 2.3: Switcher vers `TextInput` avec style distinct (border primary[500] + bg neutral)
  - [x] Subtask 2.4: Focus automatique + curseur positionné en début de texte via selection + delay
  - [x] Subtask 2.5: Masquer la section Analyse en mode édition

- [x] **Task 3: Adapter l'ActionBar** (AC: 2, 3, 4, 5)
  - [x] Subtask 3.1: En mode lecture → Copy, Share, Delete (comportement actuel)
  - [x] Subtask 3.2: En mode édition → Annuler + Enregistrer uniquement
  - [x] Subtask 3.3: "Annuler" revert le texte et sort du mode édition sans save
  - [x] Subtask 3.4: "Enregistrer" persiste via `useTextEditor.handleSave()` et sort du mode édition

- [x] **Task 4: Tests BDD** (AC: 1–6)
  - [x] Subtask 4.1: Feature Gherkin `story-3-5-text-capture-detail.feature`
  - [x] Subtask 4.2: Step definitions couvrant les 6 AC
  - [x] Subtask 4.3: Vérifier non-régression sur le flow audio (AC6)

## Notes Techniques

- `CaptureDetailContent.tsx` devient un router de layout : `capture.type === 'audio'` → `CaptureAudioDetailContent`, sinon → `CaptureTextDetailContent`
- `useTextEditor.ts` reste inchangé — la logique save/discard/hasChanges est réutilisée telle quelle
- `ActionBar.tsx` : le comportement `hasChanges` → save/cancel est déjà implémenté, il suffit de s'assurer qu'il est consommé correctement depuis le nouveau composant
- Ne pas modifier `ContentSection.tsx` directement — composer depuis le nouveau composant texte

## Definition of Done

- [x] Layout texte dédié affiché pour les captures texte
- [x] Bouton "Modifier" visible en mode lecture
- [x] Mode édition avec visuel distinct et focus automatique
- [x] ActionBar contextuelle (lecture vs édition)
- [x] Save et Cancel fonctionnels
- [x] Zéro régression sur les captures audio
- [x] Tests BDD écrits et passants
- [x] Code review adversariale complétée

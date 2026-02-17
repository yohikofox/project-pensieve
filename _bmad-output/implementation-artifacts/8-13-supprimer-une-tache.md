# Story 8.13: Supprimer une Tâche

Status: backlog

## Story

As a **user**,
I want **to permanently delete an action/todo from the Actions tab**,
So that **I can keep my task list clean by removing tasks that are no longer relevant**.

## Context

Quick win identifié lors d'une session discovery avec un power user (2026-02-17).
L'absence de suppression de tâche est une friction UX notable : les utilisateurs ne peuvent pas nettoyer leur liste d'actions des tâches obsolètes.

**Distinction importante avec la Story 8.14 (Abandonner) :**
- **Supprimer** = suppression hard, la tâche disparaît définitivement
- **Abandonner** = changement d'état sémantique, la tâche reste visible comme "abandonnée"

## Acceptance Criteria

### AC1: Swipe pour supprimer (geste rapide)
**Given** I am viewing a Todo in the Actions tab
**When** I swipe left on the Todo card
**Then** a red delete action is revealed
**And** a trash icon with label "Supprimer" is displayed
**And** tapping the delete action triggers a confirmation dialog

### AC2: Confirmation avant suppression
**Given** I triggered the delete action via swipe or detail view
**When** the confirmation dialog appears
**Then** I see a message like "Supprimer cette tâche ? Cette action est irréversible."
**And** I can confirm ("Supprimer") or cancel ("Annuler")
**And** cancelling returns to the normal Todo card state

### AC3: Suppression effective (hard delete)
**Given** I confirmed the deletion
**When** the deletion is processed
**Then** the Todo is permanently removed from the local database (hard delete)
**And** the Todo is smoothly animated out of the list (slide + fade)
**And** haptic feedback confirms the action
**And** the task count in the Actions tab updates immediately

### AC4: Bouton supprimer dans la vue détail
**Given** I am in the Todo detail view
**When** I look for delete options
**Then** I see a "Supprimer" button (e.g., in a contextual menu or footer destructive button)
**And** tapping it triggers the same confirmation dialog as AC2
**And** after confirmation, I am returned to the Actions tab list

### AC5: Synchronisation de la suppression
**Given** I deleted a Todo
**When** the device is connected to the network
**Then** the deletion is propagated to the cloud/backend
**And** the Todo disappears on all synced devices
**And** if offline, the deletion is queued and propagated on next sync

## Tech Notes

- **Type**: Hard delete — `DELETE FROM todos WHERE id = ?`
- **Sync**: Utiliser le mécanisme WatermelonDB (soft-delete au niveau DB pour sync, hard-delete logique côté UI)
- **UI**: Swipe action via `react-native-gesture-handler` (déjà utilisé dans le projet)
- **Tab concerné**: Tab "Actions" — tous les filtres (À faire, Complétées, etc.)
- **Epic parent**: Epic 8 (Bug fixes & Quick Wins)

## Related

- Story 8.14: Abandonner une tâche (état sémantique alternatif)
- Story 5.2: Tab Actions Centralisé (composant liste existant)
- Story 5.4: Complétion et Navigation des Actions (vue détail existante)

## Definition of Done

- [ ] Swipe-to-delete fonctionnel dans le Tab Actions
- [ ] Bouton "Supprimer" dans la vue détail de la tâche
- [ ] Dialog de confirmation implémenté
- [ ] Hard delete en base locale avec propagation sync
- [ ] Tests BDD (3 scénarios minimum)
- [ ] Haptic feedback et animation de sortie

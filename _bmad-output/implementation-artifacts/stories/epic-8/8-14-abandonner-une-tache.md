# Story 8.14: Abandonner une Tâche

Status: backlog

## Story

As a **user**,
I want **to mark an action/todo as "abandoned" without permanently deleting it**,
So that **I can preserve the history of my decisions while keeping my active task list clean**.

## Context

Quick win identifié lors d'une session discovery avec un power user (2026-02-17).
L'"abandon" est un état sémantique distinct de la complétion et de la suppression : il conserve la traçabilité des décisions ("j'avais cette idée, j'ai choisi de ne pas la poursuivre").

**Distinction importante avec la Story 8.13 (Supprimer) :**
- **Supprimer** = suppression hard, la tâche disparaît définitivement
- **Abandonner** = changement d'état, la tâche reste visible avec le marqueur "Abandonnée" (historique préservé)

**Valeur clé** : L'utilisateur peut retrouver les actions abandonnées et comprendre pourquoi elles n'ont pas été menées à terme — utile pour les décisions répétitives et l'auto-réflexion.

## Acceptance Criteria

### AC1: Action "Abandonner" dans le swipe contextuel
**Given** I am viewing a Todo in the Actions tab
**When** I swipe left on the Todo card
**Then** the swipe menu shows at minimum two actions : "Abandonner" et "Supprimer"
**And** "Abandonner" is visually distinct from "Supprimer" (e.g., grey/orange vs red)
**And** tapping "Abandonner" transitions the Todo to abandoned state

### AC2: État visuel distinct "Abandonnée"
**Given** a Todo is in "abandoned" state
**When** I view it in the Actions tab
**Then** it displays a visual marker indicating "Abandonnée" (badge, icon, or muted styling)
**And** the styling is distinct from "Complétée" (strikethrough) and "À faire" (normal)
**And** an abandoned icon (e.g., 🚫 or ✗) distinguishes the state

### AC3: Bouton "Abandonner" dans la vue détail
**Given** I am in the Todo detail view of an active Todo
**When** I look for action options
**Then** I see an "Abandonner cette tâche" option (contextual menu or secondary button)
**And** tapping it transitions the Todo to abandoned state immediately
**And** I remain in the detail view, now showing the updated state

### AC4: Filtre dédié pour les tâches abandonnées
**Given** I am in the Actions tab
**When** I browse filter options
**Then** a "Abandonnées" filter is available alongside "À faire", "Complétées"
**And** the filter shows the count of abandoned tasks
**And** selecting the filter displays only abandoned tasks

### AC5: Réactiver une tâche abandonnée (annuler abandon)
**Given** I am viewing an abandoned Todo (in detail or list)
**When** I tap "Réactiver" or "Reprendre cette tâche"
**Then** the Todo transitions back to "todo" (active) state
**And** it reappears in the "À faire" filter
**And** the abandoned marker is removed

### AC6: Synchronisation de l'état abandonné
**Given** I marked a Todo as abandoned
**When** the device is connected to the network
**Then** the state change is propagated to the cloud/backend
**And** the abandoned state appears on all synced devices
**And** if offline, the state change is queued and propagated on next sync

## Tech Notes

- **Type**: Soft state change — Ajout d'un statut `abandoned` dans le modèle Todo
- **Modèle Todo** : Ajouter `status: 'todo' | 'completed' | 'abandoned'` (ou équivalent)
- **Sync**: L'état `abandoned` est synchronisé normalement via WatermelonDB
- **UI**: Swipe action contextuelle (partage le menu swipe avec Story 8.13)
- **Tab concerné**: Tab "Actions" — impact sur les filtres existants
- **Epic parent**: Epic 8 (Bug fixes & Quick Wins)

## Related

- Story 8.13: Supprimer une tâche (suppression définitive)
- Story 5.2: Tab Actions Centralisé (composant liste + filtres existants)
- Story 5.3: Filtres et Tri des Actions (filtre à étendre)
- Story 5.4: Complétion et Navigation des Actions (vue détail à étendre)

## Definition of Done

- [ ] Nouvel état `abandoned` ajouté au modèle Todo (migration BDD)
- [ ] Swipe-to-abandon fonctionnel dans le Tab Actions
- [ ] Bouton "Abandonner" dans la vue détail de la tâche
- [ ] Styling visuel distinct pour les tâches abandonnées
- [ ] Filtre "Abandonnées" ajouté dans le Tab Actions
- [ ] Bouton "Réactiver" pour annuler l'abandon
- [ ] Propagation sync de l'état
- [ ] Tests BDD (4 scénarios minimum)

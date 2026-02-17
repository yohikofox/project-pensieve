# Story 8.15: Priorisation Visuelle des Tâches

Status: backlog

## Story

As a **user**,
I want **to immediately identify the priority of my tasks through visual color coding**,
So that **I can focus on what matters most without having to read every task to assess urgency**.

## Context

Quick win identifié lors d'une session discovery avec un power user (2026-02-17).
Le Tab Actions manque de hiérarchisation visuelle : toutes les tâches se ressemblent visuellement quelle que soit leur urgence. Les utilisateurs doivent lire chaque tâche pour évaluer sa priorité.

**Système de couleurs proposé :**
- 🔴 **En retard** : deadline passée (critique)
- 🟠 **Prioritaire** : marquée comme importante manuellement
- 🟡 **Approchante** : deadline dans les 48h (urgence modérée)
- 🟢 **Normale** : tâche standard sans urgence

**Question ouverte (à spec lors de l'implémentation) :** L'assignation de priorité est-elle automatique (basée sur deadline uniquement) ou manuelle (tag de priorité) ou hybride ?
→ **Choix retenu pour cette story** : Hybride — priorité automatique basée sur deadline + flag "prioritaire" manuel pour les tâches sans deadline.

## Acceptance Criteria

### AC1: Indicateur visuel "En retard" (rouge)
**Given** a Todo has a deadline that is in the past
**When** I view it in the Actions tab
**Then** the Todo card displays a red visual indicator (left border, badge, or background tint)
**And** the label "En retard" or an icon (e.g., 🔴) is visible without opening the detail view
**And** the styling is distinct from all other priority levels

### AC2: Indicateur visuel "Prioritaire" (orange)
**Given** a Todo has been manually flagged as "prioritaire"
**When** I view it in the Actions tab
**Then** the Todo card displays an orange visual indicator
**And** a star or flag icon indicates the manual priority
**And** if the task is also "En retard", the red "En retard" styling takes precedence

### AC3: Indicateur visuel "Approchante" (jaune)
**Given** a Todo has a deadline within the next 48 hours (but not yet past)
**When** I view it in the Actions tab
**Then** the Todo card displays a yellow/amber visual indicator
**And** a clock or warning icon indicates the approaching deadline
**And** red "En retard" takes precedence if the deadline has passed

### AC4: Tâche normale (vert ou neutre)
**Given** a Todo has no deadline and is not flagged as prioritaire
**When** I view it in the Actions tab
**Then** the Todo card displays neutral or subtle green styling
**And** no urgent indicator is shown (clean, standard appearance)

### AC5: Marquer une tâche comme "Prioritaire" (flag manuel)
**Given** I am viewing a Todo in the list or detail view
**When** I tap a priority flag/star button on the Todo card
**Then** the Todo is marked as "prioritaire"
**And** the orange indicator appears immediately (optimistic UI)
**And** tapping again removes the priority flag (toggle)

### AC6: Tri automatique par priorité
**Given** I am in the Actions tab
**When** the list is displayed (default sort or "Priorité" sort)
**Then** tasks are ordered: En retard → Prioritaires → Approchantes → Normales
**And** within each group, tasks are sorted by deadline (nearest first) then by creation date

### AC7: Cohérence dans tous les contextes
**Given** a Todo has a priority indicator
**When** I view it inline in the Feed (story 5.1) OR in the Tab Actions
**Then** the same visual indicator is applied consistently
**And** the priority state is reflected in both views

## Tech Notes

- **Modèle Todo** : Ajouter champ `isPriority: boolean` (ou `priorityFlag`)
- **Calcul deadline status** : Côté UI, calculer dynamiquement `isOverdue` et `isApproaching` à partir de la deadline existante — pas de champ DB supplémentaire
- **UI**: Barre colorée latérale gauche (style Notion) — minimal et propre
- **Question spec** : Valider l'approche hybride (auto + manuel) en implémentation
- **Tab concerné**: Tab "Actions" + inline Feed (AC7)
- **Epic parent**: Epic 8 (Bug fixes & Quick Wins)

## Open Questions (à résoudre lors du dev)

1. **Seuil "approchante"** : 48h est-il le bon seuil, ou configurable (24h, 72h) ?
2. **Inline Feed** : Appliquer les indicateurs aussi dans le feed (AC7) ou seulement dans le Tab Actions pour commencer ?
3. **Performance** : Le tri par priorité nécessite-t-il un index sur `deadline` + `isPriority` ?

## Related

- Story 5.2: Tab Actions Centralisé (liste à enrichir)
- Story 5.3: Filtres et Tri des Actions (tri à étendre avec "Priorité")
- Story 5.4: Complétion et Navigation des Actions (vue détail — flag priorité éditable)
- Story 8.13: Supprimer une tâche (même epic, cohérence swipe gestures)
- Story 8.14: Abandonner une tâche (même epic)

## Definition of Done

- [ ] Indicateurs visuels couleur implémentés pour les 4 états de priorité
- [ ] Champ `isPriority` ajouté au modèle Todo (migration BDD)
- [ ] Toggle "Prioritaire" accessible depuis la liste et la vue détail
- [ ] Tri automatique par priorité fonctionnel
- [ ] Cohérence visuell dans le Feed inline (AC7)
- [ ] Tests BDD (4 scénarios minimum : rouge, orange, jaune, vert)
- [ ] Questions ouvertes documentées dans la story si non résolues avant dev

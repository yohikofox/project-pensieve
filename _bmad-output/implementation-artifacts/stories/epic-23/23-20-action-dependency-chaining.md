# Story 23.20 — Action Dependency Chaining

Status: backlog

## Story

As a **user describing multi-step processes**,
I want **Pensine to detect sequential dependencies between actions**,
So that **"appelle Paul pour le fichier, puis envoie-le à Marie" creates two ordered todos**.

## Context

Détecte les séquences dans une capture : connecteurs temporels ("puis", "ensuite",
"après avoir", "une fois que") → crée des Todos avec lien de dépendance.
La Todo 2 devient active uniquement quand la Todo 1 est complétée.

## Acceptance Criteria

### AC1 — Détection de séquences dans le prompt LLM
**Given** le prompt `delegation_items` enrichi avec la notion de séquence
**When** le LLM détecte un connecteur de séquence
**Then** les actions sont retournées avec un champ `depends_on_index: number | null`

### AC2 — Todos liées dans Task BC
**Given** des actions avec dépendances
**When** Task BC crée les Todos
**Then** la Todo dépendante a un état `blocked` jusqu'à completion de la Todo parente

### AC3 — UI : affichage de la chaîne
**Given** des Todos liées
**When** l'utilisateur consulte l'onglet Actions
**Then** les Todos dépendantes sont visuellement indentées sous leur parente
**And** elles passent automatiquement en `active` quand la parente est complétée

## Dev Notes
Modification du prompt `delegation_items` :
Ajouter champ `sequence_index: number` et `depends_on_index: number | null` au schéma JSON.

## References
- ADR-034 (modèle sémantique Action/Todo)
- ADR-035 (choreography CaptureDigested)
- Story 19-3 (Task BC — materialisation Todo)

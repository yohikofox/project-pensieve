# Story 23.14 — Follow-up Tracker

Status: backlog

## Story

As a **user who makes commitments verbally**,
I want **Pensine to detect unresolved promises and nudge me**,
So that **"j'allais appeler Paul" from 5 days ago doesn't fall through the cracks**.

## Context

Analyse les captures passées pour détecter des engagements sans action de suivi.
Différent d'un rappel à date — basé sur l'absence d'action détectée, pas sur un timer.

## Acceptance Criteria

### AC1 — Détection d'engagement sans suivi
**Given** une capture contenant une promesse ("je vais", "je dois", "il faut que je")
**When** aucun Todo lié n'est complété dans les N jours configurés
**Then** Pensine génère un nudge : "Il y a 5 jours tu as mentionné appeler Paul — aucune action détectée"

### AC2 — Nudge contextuel (pas un rappel bête)
**Given** un engagement non suivi détecté
**When** le nudge est déclenché
**Then** il propose directement les actions : [Créer un Todo] [Marquer comme fait] [Ignorer]

### AC3 — Seuil configurable
**Given** les Settings
**When** l'utilisateur configure le Follow-up Tracker
**Then** il peut définir le délai avant nudge (défaut : 3 jours) et activer/désactiver la feature

## References
- ADR-034 (sémantique Action/Todo — détection d'engagements)
- Story 23-12 (Project Clustering — contexte des captures liées)

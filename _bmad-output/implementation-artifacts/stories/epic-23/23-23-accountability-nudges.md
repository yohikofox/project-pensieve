# Story 23.23 — Accountability Nudges Intelligents

Status: backlog

## Story

As a **user with an overloaded todo list**,
I want **smart, context-aware nudges instead of dumb scheduled reminders**,
So that **I'm pushed to act at the right moment, not just at the right time**.

## Context

La différence avec un rappel classique : le nudge est déclenché par le **contexte**,
pas par une heure. Exemples :
- Créneau libre détecté dans Google Calendar → nudge sur la todo la plus prioritaire
- L'utilisateur ouvre Pensine → si todo vieille de 7+ jours, la mettre en avant
- Fin de journée (18h) avec todos critiques non avancées → nudge

## Acceptance Criteria

### AC1 — Nudge sur créneau libre
**Given** un créneau libre détecté dans Google Calendar (1h+)
**When** des todos prioritaires existent
**Then** une notification contextuelle : "Tu as 1h30 libre — [Todo la plus prioritaire]"

### AC2 — Nudge à l'ouverture de l'app
**Given** une todo vieille de N jours (configurable, défaut 5)
**When** l'utilisateur ouvre Pensine
**Then** la todo ancienne est mise en avant dans le feed avec un indicateur d'ancienneté

### AC3 — Nudge de fin de journée
**Given** des todos critiques non avancées
**When** 18h arrive (ou heure de fin de journée configurée)
**Then** une notification résumée : "3 todos non avancées aujourd'hui — dont 1 deadline demain"

### AC4 — Apprentissage des patterns de productivité
**Given** l'historique de completion des todos
**When** Pensine détecte un créneau habituellement productif pour l'utilisateur
**Then** les nudges sont préférentiellement envoyés dans ces créneaux

## References
- Story 23-13 (Meeting Prep — accès Google Calendar)
- Story 23-15 (Daily Briefing — même infrastructure de notifications)
- ADR-020 (Background Processing)

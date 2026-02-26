# Story 23.15 — Daily Briefing

Status: backlog

## Story

As a **user starting my day**,
I want **a daily automated summary of what's pending and upcoming**,
So that **I have full situational awareness in 30 seconds without opening multiple apps**.

## Context

Capture synthétique générée automatiquement chaque matin.
Agrège : captures non digérées, todos du jour, réunions à venir, ExecutionJobs en attente.
Peut être lue vocalement (TTS) ou consultée comme une carte dans le feed.

## Acceptance Criteria

### AC1 — Génération automatique à heure configurée
**Given** l'heure de briefing configurée (défaut : 7h30)
**When** l'heure arrive
**Then** Pensine génère un briefing en tâche de fond (expo-task-manager)

### AC2 — Contenu du briefing
**Given** le briefing généré
**Then** il contient :
- Captures non digérées (si applicable)
- Todos ouvertes + deadline aujourd'hui
- Réunions du jour (Google Calendar)
- ExecutionJobs en `pending_validation`
- Rappel des 3 todos les plus vieilles (follow-up tracker)

### AC3 — Lecture vocale (TTS)
**Given** le briefing disponible
**When** l'utilisateur appuie sur "Écouter"
**Then** Pensine lit le briefing via TTS on-device (expo-speech)

### AC4 — Notification push avec résumé
**Given** le briefing généré
**When** l'heure de briefing arrive
**Then** une notification push affiche : "3 todos, 2 réunions, 1 action en attente" avec accès en 1 tap

## References
- ADR-020 (Background Processing)
- Story 23-14 (Follow-up Tracker — source de données)
- Epic 19 (ExecutionJobs — données source)

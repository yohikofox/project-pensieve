# Story 23.13 — Meeting Prep Assistant

Status: backlog

## Story

As a **user with upcoming meetings**,
I want **Pensine to prepare a contextual brief before each meeting**,
So that **I arrive prepared without manual effort**.

## Context

Lit Google Calendar (scope `calendar.readonly` — déjà dans story 19-8),
croise avec les captures existantes sur le même sujet/contact,
génère un mini-brief 1h avant chaque réunion.

## Acceptance Criteria

### AC1 — Détection des réunions à venir
**Given** l'accès Google Calendar (OAuth story 19-7)
**When** une réunion est dans moins de 2h
**Then** Pensine recherche dans l'historique des captures liées aux participants ou au titre de la réunion

### AC2 — Génération du brief
**Given** des captures pertinentes trouvées
**When** le brief est généré (LLM local)
**Then** il contient : contexte (captures liées), todos ouvertes concernant les participants, dernière interaction capturée

### AC3 — Notification proactive
**Given** le brief généré
**When** 1h avant la réunion
**Then** une notification push est envoyée : "Réunion avec Paul dans 1h — 3 éléments contextuels disponibles"

## References
- Story 19-7 (OAuth Google — scope calendar.readonly)
- Story 19-8 (Google Calendar integration)
- Story 23-12 (Project Clustering — source de contexte)

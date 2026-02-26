# Story 23.5 — Android Auto

Status: backlog

## Story

As a **user driving**,
I want **to capture ideas and validate ExecutionJobs hands-free via Android Auto**,
So that **I can use Pensine safely while driving without touching my phone**.

## Context

La voiture est un contexte particulièrement propice aux captures (idées, rappels, appels à passer).
Android Auto expose une interface simplifiée sur l'écran du tableau de bord.

**Deux fonctions principales :**
1. Capture vocale mains-libres
2. Validation/rejet des ExecutionJobs en attente (lecture vocale + réponse oui/non)

**SDK** : Android for Cars App Library (`androidx.car.app`).

## Acceptance Criteria

### AC1 — App Pensine visible dans Android Auto
**Given** Pensine installée et Android Auto connecté
**When** l'utilisateur ouvre Android Auto
**Then** Pensine apparaît dans la liste des apps disponibles

### AC2 — Capture vocale depuis Android Auto
**Given** Pensine active dans Android Auto
**When** l'utilisateur appuie sur le bouton micro
**Then** une capture audio est démarrée et traitée en arrière-plan

### AC3 — Lecture et validation d'ExecutionJobs en attente
**Given** des ExecutionJobs en état `pending_validation`
**When** l'utilisateur ouvre Pensine dans Android Auto
**Then** les jobs en attente sont lus à voix haute et l'utilisateur peut répondre "Oui" / "Non" / "Plus tard"

## References
- ADR-020 (Background Processing)
- Epic 19 (ExecutionJob — trust model N1 validation)

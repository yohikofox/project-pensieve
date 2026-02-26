# Story 23.6 — Wear OS

Status: backlog

## Story

As a **user with an Android smartwatch**,
I want **to start captures and validate ExecutionJobs from my wrist**,
So that **I have zero-friction capture at any moment without taking out my phone**.

## Context

Friction minimale absolue — le cas d'usage Pensine par excellence. Un tap sur la montre
suffit pour capturer. Les ExecutionJobs en attente de validation peuvent aussi être
approuvés/rejetés au poignet.

**SDK** : Wear OS (Jetpack Compose for Wear OS + Wearable Data Layer API pour comm avec le téléphone).

## Acceptance Criteria

### AC1 — Complication Pensine sur le watchface
**Given** Pensine installée (téléphone + montre)
**When** l'utilisateur configure son watchface
**Then** une complication Pensine (tap → capture) est disponible

### AC2 — Capture vocale depuis la montre
**Given** la complication ou l'app Pensine Wear
**When** l'utilisateur tape → parle dans le micro de la montre
**Then** l'audio est transféré au téléphone pour traitement (Wearable Data Layer)

### AC3 — Notification d'ExecutionJob sur la montre
**Given** un ExecutionJob en `pending_validation`
**When** le téléphone reçoit l'ExecutionJob
**Then** une notification apparaît sur la montre avec actions Swipe : ✅ Valider / ❌ Rejeter

## References
- ADR-020 (Background Processing)
- Epic 19 (ExecutionJob trust model)
- Epic 20 (Triggers alternatifs)

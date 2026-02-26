# Story 23.22 — Recurring Pattern Automation

Status: backlog

## Story

As a **user with repetitive routines**,
I want **Pensine to detect my recurring todos and offer to automate them**,
So that **"envoyer le rapport hebdo" becomes a recurring task without me configuring it manually**.

## Context

Analyse l'historique des Todos complétées → détecte les patterns de fréquence
(hebdomadaire, mensuel, quotidien) → propose de créer une récurrence automatique.
Apprentissage passif — aucune configuration initiale requise.

## Acceptance Criteria

### AC1 — Détection de patterns récurrents
**Given** un historique de Todos complétées
**When** une Todo similaire apparaît 3+ fois avec un intervalle régulier
**Then** Pensine détecte le pattern et le signale à l'utilisateur

### AC2 — Proposition de récurrence
**Given** un pattern détecté
**When** Pensine notifie l'utilisateur
**Then** il peut choisir : [Créer récurrence automatique] [Ignorer] [Me redemander dans 1 mois]

### AC3 — Todo récurrente créée automatiquement
**Given** une récurrence configurée
**When** l'intervalle est atteint
**Then** une nouvelle Todo est créée automatiquement sans capture vocale
**And** l'utilisateur en est notifié (pas en mode silencieux)

## References
- ADR-020 (Background Processing — tâche périodique de détection)
- Story 19-3 (Task BC — création de Todo)

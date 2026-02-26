# Story 23.3 — Location & Geofencing

Status: backlog

## Story

As a **user**,
I want **to create location-triggered reminders and have my captures enriched with location context**,
So that **"rappelle-moi quand j'arrive au bureau" works automatically**.

## Context

Deux usages complémentaires :
1. **Enrichissement passif** : ajouter automatiquement la localisation aux captures (contexte "au bureau", "en voiture", "chez le client")
2. **Triggers géographiques** : `ReminderCreate` avec déclencheur "à l'arrivée à X"

Cohérent avec Epic 20 (triggers de délégation) — le geofencing est un trigger alternatif
au Wake Word pour les scénarios de déplacement.

**Expo** : `expo-location` + `expo-task-manager` (déjà prévu ADR-020).

## Acceptance Criteria

### AC1 — Enrichissement contextuel des captures
**Given** l'autorisation de localisation accordée (en arrière-plan)
**When** une capture est créée
**Then** les métadonnées incluent `location: { label: "Bureau", lat, lng }` si disponible

### AC2 — ReminderCreate avec trigger géographique
**Given** une capture contenant "quand j'arrive à [lieu]" ou "au bureau, rappelle-moi de"
**When** le LLM classifie l'action
**Then** un `ExecutionJob` `ReminderCreate` est créé avec `params.trigger: { type: "geofence", location: "..." }`

### AC3 — Geofence management
**Given** des lieux fréquents (bureau, domicile)
**When** l'utilisateur les configure dans les Settings
**Then** Pensine peut résoudre "au bureau" → coordonnées GPS sans demander à l'utilisateur

### AC4 — Arrière-plan géré via expo-task-manager
**Given** l'app en arrière-plan
**When** l'utilisateur entre dans une zone geofencée
**Then** le rappel est déclenché (conforme ADR-020)

## References
- ADR-020 (Background Processing — expo-task-manager)
- Epic 20 (Delegation Triggers)
- Story 19-12 (Trust model N2/N3)

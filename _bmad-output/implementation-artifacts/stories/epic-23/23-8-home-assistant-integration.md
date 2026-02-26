# Story 23.8 — Home Assistant Integration

Status: backlog

## Story

As a **user with a home automation setup**,
I want **Pensine to control my smart home from a voice capture**,
So that **"éteins les lumières du salon" or "mets le thermostat à 19°" work as ExecutionJobs**.

## Context

Home Assistant expose une API REST locale — zero cloud, zero coût, cohérent avec
local-first (ADR-036). Particulièrement pertinent ici : tu as déjà un homelab.

**Capability** : `HomeAutomation` — appelle l'API REST HA locale.

**Cas d'usage :** lumières, thermostats, volets, scènes, scripts HA.

## Acceptance Criteria

### AC1 — Configuration Home Assistant dans les Settings
**Given** Pensine Settings > Intégrations
**When** l'utilisateur configure Home Assistant
**Then** il peut saisir l'URL locale (ex: `http://homeassistant.local:8123`) et un Long-Lived Access Token

### AC2 — Capability HomeAutomation dans Delegation BC
**Given** une capture contenant "éteins X", "allume X", "mets [appareil] à [valeur]"
**When** le LLM classifie l'intent
**Then** un `ExecutionJob` `HomeAutomation` est créé avec :
- `params.entity_id` : identifiant HA de l'appareil
- `params.action` : `turn_on | turn_off | set_value`
- `params.value` : valeur optionnelle (température, luminosité)

### AC3 — Résolution entity_id depuis le nom naturel
**Given** `target: "lumières du salon"`
**When** Pensine cherche dans la liste des entités HA configurées
**Then** l'entity_id `light.salon` est résolu (mapping nom naturel → entity_id configurable)

### AC4 — Exécution via API REST HA
**Given** un `ExecutionJob HomeAutomation` validé
**When** Pensine appelle `POST /api/services/{domain}/{service}`
**Then** l'action est exécutée en local sans passer par le cloud

## Dev Notes
- API HA : `POST http://homeassistant.local:8123/api/services/light/turn_off`
  `Authorization: Bearer {long_lived_token}`
- Pas d'OAuth nécessaire — Long-Lived Access Token suffit
- Stockage du token : `expo-secure-store` (conforme ADR-022)

## References
- ADR-036 (local-first, zero cloud)
- ADR-033 (Delegation BC — capabilities)
- Epic 19 (ExecutionJob pipeline)

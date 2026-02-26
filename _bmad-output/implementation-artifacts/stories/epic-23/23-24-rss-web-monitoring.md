# Story 23.24 — Veille Automatique (RSS & Web Monitoring)

Status: backlog

## Story

As a **user wanting to stay informed on specific topics**,
I want **Pensine to monitor RSS feeds and web sources and create captures automatically**,
So that **relevant information becomes an actionable capture without me browsing manually**.

## Context

Tâche planifiée backend → scrape des flux RSS ou URLs configurées → LLM filtre
par pertinence → crée des captures automatiquement si le seuil de pertinence est atteint.

Cas d'usage : veille concurrentielle, prix marché, actualités tech, secteur métier.

## Acceptance Criteria

### AC1 — Configuration des sources de veille
**Given** Settings > Veille
**When** l'utilisateur configure une source
**Then** il peut ajouter : URL RSS, URL de page à surveiller, mots-clés de filtrage, fréquence

### AC2 — Monitoring backend planifié
**Given** des sources configurées
**When** la fréquence configurée est atteinte (backend cron)
**Then** les nouvelles entrées sont récupérées et analysées par le LLM

### AC3 — Filtrage par pertinence
**Given** un article ou entrée récupérée
**When** le LLM l'analyse
**Then** une capture n'est créée que si le score de pertinence dépasse le seuil configuré
(évite le bruit — qualité > quantité)

### AC4 — Capture créée dans le feed
**Given** un contenu pertinent détecté
**When** la capture est créée
**Then** elle apparaît dans le feed avec la source, le résumé LLM et les actions extraites

## Dev Notes
- Traitement backend (NestJS cron) — le mobile reçoit les captures via sync
- LLM de filtrage : peut utiliser le modèle cloud (si BYOK Epic 22 activé) pour les tâches planifiées

## References
- ADR-020 (Background Processing — côté mobile pour les sources locales)
- Epic 6 (Sync — réception des captures créées en backend)

# Story 23.25 — GitHub / Linear Issues → Todos

Status: backlog

## Story

As a **developer user**,
I want **issues assigned to me on GitHub or Linear to automatically become Todos in Pensine**,
So that **my technical task management is centralized without copy-pasting**.

## Context

Intégration bidirectionnelle :
- GitHub/Linear issue assignée → Todo créée dans Pensine
- Todo complétée dans Pensine → issue close (optionnel, configurable)
- Commentaire mentionnant @username → Todo "à répondre"

## Acceptance Criteria

### AC1 — Configuration OAuth GitHub / Linear
**Given** Settings > Intégrations > Dev Tools
**When** l'utilisateur connecte son compte GitHub ou Linear
**Then** l'OAuth est complété et le token stocké dans expo-secure-store

### AC2 — Issues assignées → Todos
**Given** une issue GitHub/Linear assignée à l'utilisateur
**When** le polling détecte la nouvelle issue (ou webhook reçu)
**Then** un Todo est créé dans Task BC avec :
- Titre = titre de l'issue
- Lien vers l'issue
- Label de source "GitHub" ou "Linear"

### AC3 — Complétion bidirectionnelle (opt-in)
**Given** un Todo lié à une issue
**When** l'utilisateur complète le Todo dans Pensine
**Then** si l'option est activée, l'issue est close via l'API correspondante

### AC4 — Mention @username → Todo "à répondre"
**Given** un commentaire mentionnant l'utilisateur
**When** Pensine le détecte
**Then** un Todo "Répondre au commentaire de [auteur] sur [issue]" est créé

## References
- Story 19-3 (Task BC — création Todo)
- ADR-025 (HTTP client — fetch natif pour les APIs)

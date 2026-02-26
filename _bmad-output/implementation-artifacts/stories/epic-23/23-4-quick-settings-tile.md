# Story 23.4 — Quick Settings Tile (Android)

Status: backlog

## Story

As a **user**,
I want **a Quick Settings tile to start a capture instantly**,
So that **I can capture without unlocking my phone or opening the app**.

## Context

Tuile dans le panneau de notifications Android (glisser depuis le haut de l'écran).
Un tap → capture audio lancée immédiatement, même depuis l'écran de veille.

Complémentaire au widget (Epic 20) mais accessible différemment — cas d'usage :
mains occupées, phone dans la poche, besoin rapide.

**Android** : `TileService` API. Pas d'équivalent iOS natif (Control Center est limité).

## Acceptance Criteria

### AC1 — Tile disponible dans Quick Settings
**Given** Pensine installée sur Android
**When** l'utilisateur édite ses Quick Settings
**Then** une tuile Pensine est disponible à ajouter

### AC2 — Tap sur la tile → capture immédiate
**Given** la tile ajoutée
**When** l'utilisateur tape la tile (écran veille ou déverrouillé)
**Then** Pensine démarre une capture audio immédiatement (même flux que le bouton principal)

### AC3 — Long press → choix du type de capture
**Given** la tile active
**When** l'utilisateur fait un long press
**Then** il peut choisir : audio / texte / photo

## References
- Epic 20 (Widget — trigger alternatif)
- ADR-020 (Background Processing)

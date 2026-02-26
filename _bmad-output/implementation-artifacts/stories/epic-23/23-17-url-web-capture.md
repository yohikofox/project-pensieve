# Story 23.17 — URL / Web Page Capture

Status: backlog

## Story

As a **user browsing the web**,
I want **to share any URL to Pensine and get an instant summary + extracted actions**,
So that **articles, product pages and documentation become actionable captures**.

## Context

Partager une URL depuis le navigateur Android → intent share → Pensine.
Scrape le contenu, passe au LLM local, génère résumé + actions.
Zero cloud — scraping + LLM on-device.

## Acceptance Criteria

### AC1 — Pensine apparaît dans le menu "Partager" Android
**Given** Pensine installée
**When** l'utilisateur appuie sur "Partager" dans Chrome/Firefox
**Then** Pensine apparaît comme cible de partage

### AC2 — Scraping et extraction du contenu
**Given** une URL partagée
**When** Pensine reçoit l'intent
**Then** le contenu textuel de la page est extrait (sans images ni tracking)
**And** le pipeline LLM local génère : résumé, highlights, action_items

### AC3 — Capture créée dans le feed
**Given** le contenu extrait
**When** le digest est terminé
**Then** une capture apparaît dans le feed avec l'URL source, le résumé et les actions extraites

### AC4 — Gestion des pages protégées / vides
**Given** une URL inaccessible ou sans contenu textuel
**When** le scraping échoue
**Then** une capture est créée avec l'URL uniquement + notification "Contenu non extractible"

## References
- ADR-036 (local-first)
- ADR-004 (Single LLM call pipeline)

# Story 23.26 — Multi-Capture Synthesis

Status: backlog

## Story

As a **user with months of captures on a project**,
I want **to synthesize all captures related to a topic into a structured document**,
So that **scattered voice notes become an actionable project summary**.

## Context

Sélectionner N captures (ou un cluster automatique story 23-12) → LLM génère un
document de synthèse structuré : décisions prises, idées accumulées, actions ouvertes,
chronologie. Export vers Notion ou Google Doc.

## Acceptance Criteria

### AC1 — Sélection manuelle ou depuis un cluster
**Given** le feed ou un cluster (story 23-12)
**When** l'utilisateur sélectionne des captures (multi-select) ou un cluster entier
**Then** un bouton "Synthétiser" est disponible

### AC2 — Génération du document de synthèse
**Given** les captures sélectionnées
**When** le LLM génère la synthèse (appel local ou cloud BYOK si activé)
**Then** le document contient :
- Résumé exécutif (3-5 phrases)
- Décisions identifiées
- Idées accumulées
- Actions ouvertes (Todos non complétées liées)
- Chronologie des captures (date + résumé court)

### AC3 — Export vers Notion ou Google Doc
**Given** la synthèse générée
**When** l'utilisateur choisit d'exporter
**Then** elle est créée dans Notion (story 23-9) ou Google Docs (story 23-11)
avec la liste des captures sources en annexe

### AC4 — Synthèse accessible dans le feed
**Given** la synthèse générée
**When** elle est consultable dans Pensine
**Then** elle apparaît comme une capture spéciale "Synthèse — Projet X (12 captures)"

## References
- Story 23-12 (Project Clustering — source de clusters)
- Story 23-9 (Notion sync — export)
- Story 23-11 (Google Drive — export)
- ADR-004 (Single LLM call — à adapter pour multi-capture)

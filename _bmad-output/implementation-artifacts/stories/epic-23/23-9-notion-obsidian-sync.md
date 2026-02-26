# Story 23.9 — Sync Ideas vers Notion / Obsidian

Status: backlog

## Story

As a **user building a personal knowledge base**,
I want **my extracted ideas to sync automatically to Notion or Obsidian**,
So that **Pensine feeds my existing knowledge management workflow**.

## Context

Les `ideas` extraites des captures (prompt `ideas` existant) n'ont nulle part où aller
dans Pensine au-delà du feed. Les syncer vers une base de connaissance existante
crée un pipeline "capture vocale → PKM" complet.

**Deux cibles :**
- **Notion** : API officielle, bien documentée, OAuth2
- **Obsidian** : fichiers Markdown sur Google Drive / iCloud (pas d'API directe)

## Acceptance Criteria

### AC1 — Configuration Notion dans les Settings
**Given** Settings > Intégrations > Notion
**When** l'utilisateur connecte son compte Notion (OAuth2)
**Then** il peut choisir la database ou page cible pour les idées

### AC2 — Sync automatique des ideas vers Notion
**Given** une capture digérée avec au moins une `idea` extraite
**When** la digestion se termine
**Then** chaque idée est créée comme une entrée dans la Notion database configurée
avec : titre, date de capture, lien vers la capture source

### AC3 — Sync vers Obsidian via Google Drive
**Given** un vault Obsidian synchronisé sur Google Drive
**When** une idea est extraite
**Then** un fichier Markdown est créé dans le dossier Obsidian configuré
(format : `YYYY-MM-DD - {titre}.md` avec contenu + backlinks)

### AC4 — Contrôle granulaire
**Given** la sync activée
**When** l'utilisateur consulte une capture
**Then** il peut choisir manuellement si une idée spécifique doit être synchée ou non

## Dev Notes
- Notion API : `https://api.notion.com/v1/pages` — POST pour créer une entrée
- Obsidian : pas d'API → écrire un fichier .md dans Google Drive via Drive API
- Les deux peuvent coexister

## References
- `analysisPrompts.ts` — prompt `ideas` (données source)
- ADR-025 (HTTP client strategy — fetch natif)

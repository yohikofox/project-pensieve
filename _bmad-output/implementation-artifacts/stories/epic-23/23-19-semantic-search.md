# Story 23.19 — Semantic Search

Status: backlog

## Story

As a **user with months of captures**,
I want **to search my captures by meaning, not just keywords**,
So that **"trouve mes idées sur la croissance" returns captures about revenue, acquisition and scaling even if those words aren't used**.

## Context

Embeddings vectoriels générés on-device pour chaque capture → stockés dans OP-SQLite
(extension vectorielle) ou index local. Recherche par similarité cosinus.

Modèle d'embeddings léger : `all-MiniLM-L6-v2` (~22MB) ou équivalent via litert-lm.
Zero cloud, zero coût, conforme local-first (ADR-036).

## Acceptance Criteria

### AC1 — Index d'embeddings généré en arrière-plan
**Given** des captures existantes
**When** la feature est activée pour la première fois
**Then** Pensine génère les embeddings de toutes les captures en tâche de fond (expo-task-manager)
**And** les nouvelles captures sont indexées automatiquement après digestion

### AC2 — Interface de recherche sémantique
**Given** l'écran de recherche
**When** l'utilisateur saisit une requête
**Then** les captures sont classées par pertinence sémantique (pas par date ou fulltext)

### AC3 — Résultats contextualisés
**Given** des résultats retournés
**When** l'utilisateur les consulte
**Then** chaque résultat affiche le passage le plus pertinent de la capture (snippet)

### AC4 — Recherche hybride (sémantique + fulltext)
**Given** une requête contenant des noms propres ou dates exactes
**When** la recherche est lancée
**Then** les résultats combinent similarité sémantique ET correspondance exacte (BM25 + cosine)

## Dev Notes
- OP-SQLite supporte sqlite-vec pour la recherche vectorielle
- Modèle d'embeddings à valider : all-MiniLM-L6-v2 via litert-lm (même runtime que gemma3n-2b)

## References
- ADR-036 (local-first)
- ADR-018 (OP-SQLite)
- ADR-022 (State persistence — stockage des embeddings)

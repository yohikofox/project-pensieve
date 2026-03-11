# Story 29.3 — Indexation repo et manifest discovery

Status: backlog

## Story

As a **user with multiple repos linked to a project**,
I want **Pensine to understand what each repo is about**,
So that **the AI can suggest the most relevant repo when I create a GitHub issue**.

## Context

Pour que le sélecteur de repo (Story 29-5) puisse proposer le repo le plus pertinent
via LLM, il faut que chaque repo dispose d'un **manifest** : un texte court décrivant
son périmètre et sa responsabilité.

**Stratégie en 3 tiers (ordre de priorité décroissant) :**

```
Tier 1 — Découverte passive (pas de LLM)
  Le backend fetch l'un de ces fichiers via l'API GitHub Contents :
  1. .pensine/context.md   ← fichier dédié Pensine (idéal)
  2. AGENTS.md             ← manifest AI standard (Claude Code, Gemini, etc.)
  3. README.md             ← fallback généraliste

Tier 2 — Indexation backend LLM (si Tier 1 insuffisant)
  Backend lit les fichiers clés du repo (README, package.json / Cargo.toml /
  go.mod, structure des dossiers racine, CLAUDE.md si présent)
  → Cloud LLM (GPT-4o-mini ou Claude Haiku) génère un résumé en 150-300 mots
  → Stocké en DB : manifestCache + manifestCachedAt (TTL 7 jours)
  → Déclenché on-demand (première utilisation) ou refresh TTL expiré

Tier 3 — Commit du manifest (post-MVP)
  Après validation utilisateur, le backend commit .pensine/context.md dans le repo
  → Nécessite PAT avec scope contents:write
  → Bénéficie à tous les outils (Claude Code, autres agents)
```

**Pas de Tier 3 en MVP** — la story couvre uniquement Tier 1 et Tier 2.

**Note sécurité :** le PAT est déjà stocké en backend (Story 29-2). Aucune nouvelle
surface de sécurité pour les appels GitHub API depuis le backend.

## Acceptance Criteria

### AC1 — Tier 1 : Découverte passive
**Given** un `CodeRepository` vient d'être configuré
**When** le backend déclenche la découverte (après création ou à la demande)
**Then** :
- Il tente de fetcher `.pensine/context.md` → `AGENTS.md` → `README.md` dans cet ordre
- Le premier fichier trouvé est utilisé comme manifest
- `manifestSource` est mis à jour (`auto`), `manifestCache` stocké, `manifestCachedAt` = now

### AC2 — Tier 2 : Indexation LLM si manifest insuffisant
**Given** Tier 1 n'a rien trouvé (404) ou le contenu est trop court (< 100 caractères)
**When** le backend déclenche l'indexation
**Then** :
- Le backend fetch les fichiers clés du repo (root README, package.json/Cargo.toml/go.mod,
  liste des dossiers racine, CLAUDE.md si présent) — max 10 fichiers
- Un prompt LLM est envoyé (cloud : GPT-4o-mini ou Claude Haiku) pour générer un résumé
  de 150-300 mots décrivant le périmètre et la responsabilité du repo
- `manifestCache` est stocké avec `manifestCachedAt` = now (TTL 7 jours)

### AC3 — TTL et refresh
**Given** un manifest en cache dont `manifestCachedAt` est > 7 jours
**When** le RepoSelector est affiché (Story 29-5)
**Then** un refresh silencieux est déclenché en arrière-plan (non bloquant pour l'UX)

### AC4 — Refresh manuel
**Given** la page de gestion d'un CodeRepository
**When** l'utilisateur clique "Actualiser le manifest"
**Then** une nouvelle indexation est déclenchée (Tier 1 → Tier 2 si nécessaire)
et `manifestCachedAt` est mis à jour

### AC5 — Manifest absent (graceful fallback)
**Given** un repo sans manifest (Tier 1 et Tier 2 ont échoué ou LLM indisponible)
**When** le RepoSelector s'affiche (Story 29-5)
**Then** le repo est présenté avec son label et `owner/repo` uniquement — la sélection
manuelle reste possible

### AC6 — Pas de lecture de code métier
**Given** le processus d'indexation Tier 2
**When** le backend sélectionne les fichiers à envoyer au LLM
**Then** seuls les fichiers de métadonnées (README, manifests, configs de build) sont
inclus — jamais le code source applicatif (`.ts`, `.js`, `.py`, etc.)

## Notes techniques

**Nouveau service backend :**
```
Integration BC — Backend
  RepoIndexingService
    ├── discoverManifest(repoId): Promise<Result<string>>   // Tier 1
    ├── indexRepository(repoId): Promise<Result<string>>    // Tier 2
    └── getManifest(repoId): Promise<Result<string>>        // Tier 1 → 2 auto

  RepoIndexingJob
    trigger: on-demand | TTL expired
    queue: RabbitMQ (non bloquant)
```

**Prompt LLM Tier 2 (indicatif) :**
```
Tu analyses un repository Git. Voici ses fichiers de configuration et de description :
[contenu des fichiers]

Génère un résumé de 150-300 mots décrivant :
1. La responsabilité principale de ce repo
2. Les technologies principales
3. Le domaine métier couvert
Réponds uniquement avec le résumé, sans titre ni formatage markdown.
```

- Le TTL de 7 jours est configurable via variable d'environnement `MANIFEST_CACHE_TTL_DAYS`.
- Le Tier 2 est activé uniquement si `MANIFEST_INDEXING_ENABLED=true` (désactivé par défaut
  en développement pour économiser les tokens LLM).

## References

- Story 29-2 (CodeRepository configuré en prérequis — PAT disponible)
- Story 29-5 (consumer du manifest — RepoSelector)
- ADR-025 (HTTP client — fetch natif côté backend)

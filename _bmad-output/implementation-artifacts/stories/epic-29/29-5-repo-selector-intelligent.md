# Story 29.5 — RepoSelector intelligent (1 repo → auto, N repos → suggestion LLM)

Status: backlog

## Story

As a **user creating a GitHub issue from a capture**,
I want **Pensine to automatically pick the right repository or help me choose**,
So that **I don't have to manually navigate through a list of repos every time**.

## Context

Point d'entrée UX de tout le flow issue creation. La logique de sélection dépend du
nombre de repos configurés sur le Project lié à la capture :

```
Capture (liée à un Project)
        │
        ├─ Project.codeRepositories.length === 0
        │    └─ bouton "Créer une issue" masqué (non affiché)
        │
        ├─ Project.codeRepositories.length === 1
        │    └─ repo sélectionné automatiquement → IssueTypeSelector
        │
        └─ Project.codeRepositories.length > 1
             └─ RepoSelectorModal
                  ├─ Manifest(s) disponible(s)
                  │    └─ LLM compare manifests + capture → suggestion + score
                  └─ Manifest(s) absent(s)
                       └─ Liste manuelle avec label + owner/repo
```

**Le bouton "Créer une issue"** est visible sur la CaptureDetail uniquement si :
1. La capture est liée à un Project (non orpheline)
2. Le Project a au moins 1 `CodeRepository` configuré
3. L'utilisateur a la permission `project:issue:create`

## Acceptance Criteria

### AC1 — Bouton visible sous conditions
**Given** une capture liée à un Project avec au moins 1 repo configuré
**And** l'utilisateur a la permission `project:issue:create`
**When** il consulte la CaptureDetail
**Then** un bouton "Créer une issue GitHub" (ou icône) est visible

### AC2 — Bouton masqué si capture orpheline ou projet sans repo
**Given** une capture sans Project associé, ou un Project sans CodeRepository
**When** l'utilisateur consulte la CaptureDetail
**Then** aucun bouton de création d'issue n'est visible

### AC3 — Sélection automatique si 1 repo
**Given** le Project lié a exactement 1 CodeRepository
**When** l'utilisateur clique "Créer une issue"
**Then** le repo est sélectionné automatiquement — l'IssueTypeSelector s'ouvre directement

### AC4 — RepoSelectorModal si N repos avec manifests
**Given** le Project lié a 2+ CodeRepositories avec manifests disponibles
**When** l'utilisateur clique "Créer une issue"
**Then** un modal s'ouvre avec :
- La liste des repos (label + owner/repo)
- Une suggestion LLM mise en évidence : "Suggéré : Backend API — pertinence élevée"
- L'utilisateur peut valider la suggestion ou choisir un autre repo

### AC5 — RepoSelectorModal si N repos sans manifests
**Given** le Project lié a 2+ CodeRepositories sans manifest disponible
**When** l'utilisateur clique "Créer une issue"
**Then** un modal s'ouvre avec la liste des repos (label + owner/repo) — sélection manuelle
uniquement, sans suggestion LLM

### AC6 — IssueTypeSelector
**Given** un repo est sélectionné (automatiquement ou manuellement)
**When** l'IssueTypeSelector s'affiche
**Then** l'utilisateur peut choisir le type : `bug` | `feature` | `analysis` | `other`
(chips ou liste, design à définir)

### AC7 — Déclenchement de la génération
**Given** le type est sélectionné
**When** l'utilisateur confirme
**Then** la tâche `IssueGeneratorTask` est déclenchée (Story 29-4)
et l'utilisateur voit un indicateur de chargement

### AC8 — Suggestion LLM pour le repo (logique backend)
**Given** N repos avec manifests disponibles et le contenu de la capture
**When** le backend calcule la suggestion
**Then** il compare le contenu de la capture avec chaque manifest via un appel LLM
et retourne `{ repoId, score, reason }` pour le repo le plus pertinent

## Notes techniques

**Appel LLM pour la suggestion de repo (AC8) :**
```
Prompt :
Tu compares le contenu d'une capture avec des descriptions de repositories Git.

Capture : [rawContent / summary]

Repositories :
1. [label] — [manifest]
2. [label] — [manifest]
...

Quel repository est le plus pertinent pour créer une issue à partir de cette capture ?
Réponds en JSON : { "repoIndex": 1, "score": 0.87, "reason": "..." }
```

- L'appel est fait **uniquement si N ≥ 2 et au moins 1 manifest disponible**.
- Si l'appel échoue → sélection manuelle (AC5 comportement).
- Le `reason` est affiché dans le modal pour aider l'utilisateur à comprendre la suggestion.
- La sélection finale reste toujours à l'utilisateur (jamais auto-sélection avec N repos).

**Mobile :**
- `RepoSelectorModal` : component sheet bottom (pattern existant dans l'app)
- `IssueTypeSelectorSheet` : chips simples
- Pas de navigation vers un nouvel écran — tout en modal/sheet

## References

- Story 29-2 (CodeRepositories configurés)
- Story 29-3 (manifests — disponibles pour AC4/AC8)
- Story 29-4 (IssueGeneratorTask — déclenchée en AC7)
- Story 29-6 (IssueReviewScreen — étape suivante)

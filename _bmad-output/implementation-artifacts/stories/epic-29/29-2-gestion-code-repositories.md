# Story 29.2 — Gérer les CodeRepositories d'un Project

Status: backlog

## Story

As a **user managing a project**,
I want **to link one or more Git repositories to my project**,
So that **I can later create GitHub issues from my captures without leaving the app**.

## Context

Un `Project` peut être associé à **0..N** `CodeRepository`. La relation est N:M :
le même repo peut être partagé entre plusieurs Projects.

**Nouveau BC : Integration**

Cette story introduit le BC `Integration` côté backend et mobile :

```
Integration BC
├── CodeRepository (aggregate)
│   ├── id:               string (uuidv7)
│   ├── projectId:        string
│   ├── label:            string       ← ex: "Frontend", "Backend API"
│   ├── provider:         'github'     ← MVP. Interface prête pour gitlab|bitbucket|azuredevops|custom
│   ├── repoOwner:        string
│   ├── repoName:         string
│   ├── patRef:           string       ← référence au PAT stocké en backend (non exposé mobile)
│   ├── patScope:         'read' | 'write'
│   ├── manifestSource:   'auto' | 'custom_url' | 'none'
│   ├── manifestUrl?:     string
│   ├── createdAt:        timestamptz
│   └── updatedAt:        timestamptz
```

**Sécurité PAT (MVP) :** le PAT est stocké **en clair** dans la base backend (chiffrement
prévu en Story 29-8, à implémenter après cette story). Le mobile ne reçoit jamais le PAT
brut — il reçoit uniquement un `patRef` (UUID) pour référencer le repo dans les appels API.

**Scopes GitHub recommandés (documentés, non validés programmatiquement en MVP) :**
- Lecture manifest + repo tree : `contents:read`
- Création d'issues : `issues:write`
- Commit manifest (Tier 3, post-MVP) : `contents:write`

**Permission requise :** `project:repository:manage` (RBAC).

## Acceptance Criteria

### AC1 — Accès à la gestion des repos
**Given** l'utilisateur est sur la page détail d'un Project
**When** il navigue vers l'onglet "Repositories" (ou section dédiée)
**Then** la liste des CodeRepositories liés s'affiche (vide si aucun)

### AC2 — Ajouter un repo
**Given** l'utilisateur clique "Ajouter un repo"
**When** le formulaire s'ouvre
**Then** il peut renseigner :
- Label (ex: "Backend API" — optionnel, défaut : `owner/repo`)
- Owner GitHub (ex: `yohikofox`)
- Nom du repo (ex: `pensieve`)
- PAT GitHub (champ masqué, jamais réaffiché)
- Scope PAT : `read` ou `write`
- Source manifest : `auto` (défaut) | URL custom | aucun

### AC3 — Validation du PAT
**Given** l'utilisateur soumet le formulaire
**When** le backend reçoit la requête
**Then** :
- Une requête test est effectuée sur l'API GitHub avec le PAT (`GET /repos/{owner}/{repo}`)
- Si succès → `CodeRepository` créé, `patRef` retourné au mobile
- Si échec (401/403/404) → message d'erreur explicite affiché

### AC4 — Plusieurs repos sur un même Project
**Given** un Project avec un repo existant
**When** l'utilisateur ajoute un second repo
**Then** les deux repos sont listés et indépendants

### AC5 — Même repo sur plusieurs Projects
**Given** deux Projects différents
**When** l'utilisateur configure le même `owner/repo` sur chacun
**Then** deux entrées `CodeRepository` distinctes sont créées (pas de déduplication forcée en MVP)

### AC6 — Supprimer un repo
**Given** un CodeRepository listé
**When** l'utilisateur le supprime (confirmation obligatoire)
**Then** le repo est dissocié du Project (soft delete) — les issues déjà créées conservent
leur référence locale

### AC7 — Permission manquante
**Given** l'utilisateur n'a pas `project:repository:manage`
**When** il tente d'accéder à la gestion des repos
**Then** une erreur 403 est retournée et le bouton "Ajouter" n'est pas visible

## Notes techniques

- Le PAT est transmis en clair sur HTTPS dans le body de la requête — acceptable en MVP.
  Chiffrement backend à implémenter en Story 29-8.
- La validation PAT (AC3) doit avoir un timeout de 5s maximum pour ne pas bloquer l'UX.
- Le mobile stocke uniquement le `patRef` (UUID) en local — jamais le PAT lui-même.
- Prévoir la colonne `credential_ref` dans la table `code_repositories` pour la migration
  vers chiffrement (Story 29-8) sans breaking change.

## References

- Story 29-1 (Project créé en prérequis)
- Story 29-8 (chiffrement credentials — à faire après cette story)
- ADR-026 (BaseEntity + soft delete)
- ADR-023 (Result Pattern)
- Architecture GitHub API : https://docs.github.com/en/rest/repos/repos

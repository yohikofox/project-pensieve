# Story 29.6 — Push GitHub Issue + persistance locale

Status: backlog

## Story

As a **user who has reviewed the AI-generated issue draft**,
I want **to publish it to GitHub and keep a local reference**,
So that **I can track it from Pensine without losing the link to the original capture**.

## Context

Étape finale du flow : l'utilisateur a relu et éventuellement édité le draft (Story 29-4).
Il confirme → l'issue est créée sur GitHub via l'API, et une référence locale
(`ProjectIssue`) est persistée en OP-SQLite.

**Nouveau agrégat : `ProjectIssue`**

```
ProjectIssue {
  id:                 string (uuidv7)
  projectId:          string
  codeRepositoryId:   string
  captureId?:         string          // source optionnelle
  type:               IssueType       // 'bug' | 'feature' | 'analysis' | 'other'
  title:              string
  body:               string
  syncStatus:         'draft' | 'pending' | 'synced' | 'error'

  remoteRef?: {
    provider:         'github'
    issueNumber:      number
    url:              string
    status:           'open' | 'closed' | 'in_progress'
    labels:           string[]
    lastSyncAt:       Date
  }

  createdAt:          timestamptz
  updatedAt:          timestamptz
}
```

**Cycle de vie :**
```
[IssueReviewScreen] → syncStatus: 'draft'
      ↓ Publier
syncStatus: 'pending'  (call GitHub API en cours)
      ↓ Succès
syncStatus: 'synced'   + remoteRef rempli
      ↓ Échec
syncStatus: 'error'    + possibilité de retry
```

**Permission requise :** `project:issue:create` (RBAC).

## Acceptance Criteria

### AC1 — Publication de l'issue
**Given** l'utilisateur clique "Publier l'issue" sur l'IssueReviewScreen
**When** l'app envoie la requête au backend
**Then** :
- Le backend appelle `POST /repos/{owner}/{repo}/issues` avec le PAT du CodeRepository
- L'issue est créée sur GitHub avec le titre et le body finaux
- Les labels sélectionnés sont appliqués (uniquement ceux qui existent dans le repo)

### AC2 — Persistance locale immédiate
**Given** l'utilisateur clique "Publier"
**When** la requête est envoyée
**Then** un `ProjectIssue` est créé localement avec `syncStatus: 'pending'`
avant même la réponse GitHub (optimistic local write)

### AC3 — Mise à jour après succès
**Given** l'API GitHub retourne 201 Created
**When** le backend reçoit la réponse
**Then** :
- `syncStatus` passe à `'synced'`
- `remoteRef.issueNumber`, `remoteRef.url`, `remoteRef.status: 'open'` sont renseignés
- Un message de succès avec le lien vers l'issue est affiché à l'utilisateur
- Le lien est ouvrable dans le browser natif

### AC4 — Gestion d'erreur
**Given** l'API GitHub retourne une erreur (401, 403, 422, 5xx)
**When** l'erreur est reçue
**Then** :
- `syncStatus` passe à `'error'`
- Un message d'erreur explicite est affiché : "Erreur GitHub : [message]"
- Un bouton "Réessayer" permet de relancer la publication sans repasser par le formulaire

### AC5 — Vue liste des issues d'un Project
**Given** un Project avec des `ProjectIssue` créées
**When** l'utilisateur consulte la page détail du Project
**Then** une section "Issues" liste les issues avec :
- Titre
- Type (badge)
- Statut (`open` / `closed` / `pending` / `error`)
- Lien vers l'issue GitHub (si `synced`)
- Date de création

### AC6 — Lien capture → issue
**Given** une capture qui a généré une issue
**When** l'utilisateur consulte la CaptureDetail
**Then** un indicateur "Issue créée" est affiché avec le lien vers l'issue GitHub

### AC7 — Labels GitHub uniquement existants
**Given** des `labelsHint` suggérés par le LLM
**When** l'issue est publiée
**Then** seuls les labels qui existent dans le repo sont appliqués (vérification avant push)
— les labels inexistants sont ignorés silencieusement en MVP

## Notes techniques

**Backend :**
```
Integration BC
  GitHubIssueAdapter implements IIssueTrackerPort
    createIssue(params: CreateIssueParams, patRef: string): Promise<Result<IssueRef>>

  IssueTrackerAdapterFactory
    getAdapter(provider: RepositoryProvider): IIssueTrackerPort
```

**Types d'issues extensibles (post-MVP) :**
Le type `IssueType` est une union string avec les valeurs par défaut + les
`customIssueTypes` du Project. Le backend valide que le type est dans la liste autorisée.
```typescript
type IssueType = 'bug' | 'feature' | 'analysis' | 'other' | string
```

**Labels :**
- Avant le push, faire `GET /repos/{owner}/{repo}/labels` pour récupérer la liste.
- Matcher `labelsHint` (case-insensitive) contre la liste — appliquer uniquement les matches.
- Mettre en cache la liste de labels par repo (TTL 1h).

**Draft survie au crash :**
Le `ProjectIssue` avec `syncStatus: 'draft'` est créé dès que l'IssueReviewScreen
est ouverte — ainsi, même si l'app crash, le brouillon est récupérable.

## References

- Story 29-2 (CodeRepository + patRef)
- Story 29-4 (IssueGeneratorTask — source du draft)
- Story 29-5 (RepoSelector — déclenche le flow)
- Story 29-7 (sync status — mise à jour post-publication)
- Story 29-8 (chiffrement PAT — à implémenter après cette story)
- ADR-023 (Result Pattern)
- GitHub API : POST /repos/{owner}/{repo}/issues

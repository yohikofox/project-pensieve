# Story 29.7 — Sync status des issues (polling manuel, MVP)

Status: backlog

## Story

As a **user tracking a GitHub issue from Pensine**,
I want **to refresh the status of my issues without leaving the app**,
So that **I know if an issue has been closed or progressed without switching to GitHub**.

## Context

Une fois une issue créée (Story 29-6), son statut peut évoluer sur GitHub indépendamment
de Pensine (close, reopen, ajout de labels). Cette story implémente un mécanisme de
**polling manuel** pour mettre à jour les `ProjectIssue` locales.

**MVP : polling on-demand** (l'utilisateur déclenche le refresh). Pas de webhook,
pas de polling automatique en background — trop complexe pour le MVP.

**Post-MVP (non implémenté ici) :**
- GitHub webhook → backend → push mobile (via WebSocket ou RabbitMQ)
- Polling automatique périodique (background task)

**Champs mis à jour lors du sync :**
```
remoteRef.status       // 'open' | 'closed'
remoteRef.labels       // labels actuels sur l'issue
remoteRef.lastSyncAt   // timestamp du dernier sync réussi
```

## Acceptance Criteria

### AC1 — Refresh depuis la liste des issues d'un Project
**Given** l'utilisateur consulte la section "Issues" d'un Project
**When** il tire vers le bas (pull-to-refresh) ou clique un bouton "Actualiser"
**Then** :
- Toutes les issues `synced` du Project sont recheckées via l'API GitHub
- Les statuts et labels sont mis à jour localement
- `lastSyncAt` est mis à jour

### AC2 — Refresh depuis une CaptureDetail
**Given** une capture avec une issue liée (`ProjectIssue` en `synced`)
**When** l'utilisateur clique le badge "Issue #42 · open" sur la CaptureDetail
**Then** :
- L'app ouvre un panel/sheet avec les détails de l'issue (titre, statut, labels, lien)
- Un bouton "Actualiser" dans ce panel déclenche le sync de cette issue spécifique

### AC3 — Issue fermée
**Given** une issue qui était `open` et qui a été fermée sur GitHub
**When** le sync est déclenché
**Then** :
- `remoteRef.status` passe à `'closed'`
- Le badge sur la CaptureDetail et la liste des issues reflètent le nouveau statut
- Aucune notification push n'est envoyée en MVP

### AC4 — Gestion d'erreur sync
**Given** le sync échoue (timeout, 401, réseau)
**When** l'erreur est reçue
**Then** :
- `syncStatus` reste à `'synced'` (pas de régression)
- `lastSyncAt` n'est pas mis à jour
- Un message d'erreur discret est affiché (toast ou inline) sans bloquer l'UX

### AC5 — Issues en erreur
**Given** des `ProjectIssue` avec `syncStatus: 'error'` (échec de publication Story 29-6)
**When** l'utilisateur déclenche un refresh
**Then** :
- Ces issues ne sont pas incluses dans le sync (elles n'ont pas de `remoteRef`)
- Un bouton "Réessayer" reste visible pour chaque issue en erreur

### AC6 — Indicateur de fraîcheur
**Given** une issue dont le `lastSyncAt` est > 24h
**When** l'utilisateur consulte la liste des issues
**Then** un indicateur discret "Actualisé il y a X jours" est visible
(pas d'alerte bloquante — juste une information)

## Notes techniques

**Backend :**
```
Integration BC
  IssueSyncService
    syncIssue(projectIssueId): Promise<Result<ProjectIssue>>
    syncProjectIssues(projectId): Promise<Result<ProjectIssue[]>>
```

**Appel GitHub API :**
`GET /repos/{owner}/{repo}/issues/{issue_number}`
→ Champs lus : `state` (open|closed), `labels[].name`, `closed_at`

**Batch sync :**
Pour un Project avec N issues, les appels GitHub sont faits en séquence (pas en parallèle)
pour éviter le rate limiting GitHub (5000 req/h par PAT — large marge, mais bonne pratique).

**Mapping statuts GitHub → Pensine :**
```
GitHub state: 'open'   → Pensine: 'open'
GitHub state: 'closed' → Pensine: 'closed'
GitHub labels contient 'in progress' / 'wip' → Pensine: 'in_progress' (best effort)
```

## References

- Story 29-6 (ProjectIssue créée en prérequis)
- GitHub API : GET /repos/{owner}/{repo}/issues/{issue_number}
- ADR-023 (Result Pattern)
- ADR-025 (HTTP client — fetch natif)

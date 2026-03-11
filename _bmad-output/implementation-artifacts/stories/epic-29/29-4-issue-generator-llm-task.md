# Story 29.4 — Issue Generator : LLM Task depuis une capture

Status: backlog

## Story

As a **user reviewing a capture linked to a project**,
I want **Pensine to automatically draft a GitHub issue title and body from my capture**,
So that **I don't have to manually rewrite what I already said or typed**.

## Context

Cette story implémente la tâche LLM qui génère un draft d'issue GitHub à partir du
contenu d'une capture. C'est une **tâche asynchrone** déclenchée depuis l'UI (Story 29-5
ou directement depuis la CaptureDetail).

**Types d'issues (MVP) :**
```
'bug' | 'feature' | 'analysis' | 'other'
```
La liste est extensible : un champ `customIssueTypes: string[]` est prévu sur l'entité
`Project` pour les types personnalisés par client (post-MVP).

**Input de la tâche LLM :**
- `capture.rawContent` (texte brut ou transcription)
- `capture.summary` (résumé IA si disponible)
- `capture.ideas` (idées extraites si disponibles)
- `issueType` sélectionné par l'utilisateur
- `repoManifest` (manifest du repo cible — contexte optionnel pour adapter le ton)

**Output :**
```
IssueGeneratorResult {
  title: string        // max 72 caractères (convention GitHub)
  body:  string        // Markdown — structure adaptée au type
  labelsHint: string[] // suggestions de labels (non imposées)
}
```

**Corps de l'issue selon le type :**
- `bug`      → reproduction steps, comportement attendu, comportement observé
- `feature`  → contexte utilisateur, solution proposée, critères d'acceptation
- `analysis` → contexte, questions clés, données disponibles
- `other`    → structure libre

## Acceptance Criteria

### AC1 — Déclenchement de la tâche
**Given** l'utilisateur a sélectionné un type d'issue et un repo (Story 29-5)
**When** il confirme
**Then** une tâche `IssueGeneratorTask` est créée et mise en queue (non bloquant)
avec un indicateur de chargement dans l'UI

### AC2 — Génération du draft
**Given** la tâche est en cours
**When** le LLM (backend : GPT-4o-mini ou Claude Haiku) retourne le résultat
**Then** :
- Le titre généré respecte les 72 caractères maximum
- Le body est formaté en Markdown selon le type sélectionné
- Les `labelsHint` sont proposés (non appliqués automatiquement)

### AC3 — Affichage du draft (IssueReviewScreen)
**Given** le draft est prêt
**When** l'utilisateur arrive sur la vue de relecture
**Then** :
- Le titre est éditable
- Le body est éditable (textarea Markdown)
- Les labels suggérés sont affichés comme chips sélectionnables/désélectionnables
- Un bouton "Publier l'issue" est présent
- Un bouton "Annuler" permet de revenir sans créer l'issue

### AC4 — Contenu minimal garanti
**Given** une capture sans transcription ni résumé (ex: capture texte très courte)
**When** le LLM génère le draft
**Then** :
- Le titre est généré à partir du contenu brut disponible
- Le body contient a minima une section "Description" avec le contenu de la capture
- Aucun champ n'est laissé vide (placeholder si vraiment aucun contenu exploitable)

### AC5 — Timeout et fallback
**Given** le LLM backend ne répond pas dans les 15 secondes
**When** le timeout est atteint
**Then** :
- Un draft vide pré-structuré est affiché (template Markdown selon le type)
- Un message "Génération impossible — vous pouvez éditer manuellement" est affiché
- L'utilisateur peut remplir le formulaire manuellement et publier

### AC6 — Langue de l'issue
**Given** la langue de communication de l'utilisateur (config)
**When** le LLM génère le draft
**Then** le titre et le body sont générés dans la langue de travail du Project
(défaut : anglais pour les issues GitHub, quelle que soit la langue de l'app)

## Notes techniques

**Nouveau service mobile + backend :**
```
Integration BC — Backend
  IssueGeneratorService
    generateDraft(captureId, issueType, repoManifest?): Promise<Result<IssueGeneratorResult>>

Integration BC — Mobile
  IssueGeneratorTask        // appel au backend
  IssueReviewStore          // Zustand — draft en cours + état édition
```

**Prompt LLM (indicatif, type `bug`) :**
```
Tu génères une GitHub issue à partir d'une note vocale ou textuelle.

Type d'issue : bug
Contenu de la capture : [rawContent / summary]
Contexte du repo : [repoManifest ou "non fourni"]

Génère :
1. Un titre précis (max 72 caractères, en anglais)
2. Un body Markdown avec les sections :
   - ## Description
   - ## Steps to reproduce
   - ## Expected behavior
   - ## Actual behavior
3. Une liste de 1-3 labels GitHub suggérés

Réponds en JSON : { "title": "...", "body": "...", "labelsHint": [...] }
```

- L'appel LLM est fait côté **backend** uniquement — le mobile déclenche une tâche et
  poll le résultat (ou WebSocket si disponible).
- Le draft est stocké temporairement en local (OP-SQLite, table `issue_drafts`, TTL 24h)
  pour survivre à un crash ou une mise en arrière-plan.
- `labelsHint` sont des suggestions textuelles — ils ne créent pas de labels dans GitHub
  si ceux-ci n'existent pas déjà dans le repo.

## References

- Story 29-1 (Project en prérequis)
- Story 29-2 (CodeRepository — `patRef` disponible)
- Story 29-3 (manifest repo — contexte LLM optionnel)
- Story 29-5 (consumer — déclenche cette tâche)
- Story 29-6 (consumer — reçoit le draft pour push)
- ADR-023 (Result Pattern)

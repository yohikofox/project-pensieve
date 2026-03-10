# Story 28.1: Backend — Endpoints MCP-ready (captures, thoughts, ideas, todos via PAT)

Status: ready-for-dev

## Story

En tant que **client MCP**,
je veux **accéder aux captures, thoughts, ideas et todos d'un utilisateur via son PAT**,
afin de **permettre à Claude Code (ou tout autre client MCP) d'interagir avec le contenu de Pensine**.

## Context & Motivation

Dépend de Story 27.1 (PATGuard) et Story 26.1 (TraceContext). Ces endpoints sont conçus pour la consommation par le serveur MCP — ils doivent être stables, paginés, filtrables et respecter les scopes PAT.

## Acceptance Criteria

### AC1 : Listage des captures
**Given** un PAT valide avec scope `captures:read`
**When** `GET /api/mcp/captures` est appelé
**Then** la liste paginée des captures de l'utilisateur est retournée
**And** chaque capture inclut : `id`, `clientId`, `type`, `state`, `normalizedText`, `duration`, `createdAt`, `lastModifiedAt`

**Paramètres de filtrage :**
- `?state=` — filtrer par état (transcribed, digested, etc.)
- `?type=` — filtrer par type (audio, text)
- `?from=` / `?to=` — plage de dates (ISO 8601)
- `?limit=` (défaut: 20, max: 100) + `?cursor=` (pagination curseur)

### AC2 : Détail d'une capture
**Given** un PAT valide avec scope `captures:read`
**When** `GET /api/mcp/captures/:id` est appelé
**Then** la capture complète est retournée avec `rawContent` (texte brut) et `normalizedText`

### AC3 : Recherche fulltext dans les captures
**Given** un PAT valide avec scope `captures:read`
**When** `GET /api/mcp/captures/search?q=<terme>` est appelé
**Then** les captures dont `normalizedText` contient le terme sont retournées (ILIKE)
**And** les résultats sont triés par pertinence / date décroissante

### AC4 : Listage des thoughts
**Given** un PAT valide avec scope `thoughts:read`
**When** `GET /api/mcp/thoughts` est appelé
**Then** la liste paginée des thoughts de l'utilisateur est retournée
**And** chaque thought inclut : `id`, `captureId`, `content`, `summary`, `status`, `createdAt`

### AC5 : Détail d'un thought (avec ses ideas)
**Given** un PAT valide avec scopes `thoughts:read` + `ideas:read`
**When** `GET /api/mcp/thoughts/:id` est appelé
**Then** le thought complet est retourné avec sa liste d'ideas associées

### AC6 : Listage des todos
**Given** un PAT valide avec scope `todos:read`
**When** `GET /api/mcp/todos` est appelé
**Then** la liste paginée des todos de l'utilisateur est retournée

**Paramètres de filtrage :**
- `?completed=true|false`
- `?priority=high|medium|low`
- `?from=` / `?to=` (deadline)

### AC7 : Vérification stricte des scopes
**Given** un PAT avec scope `captures:read` uniquement
**When** `GET /api/mcp/thoughts` est appelé
**Then** une erreur 403 est retournée avec message explicite sur le scope manquant

### AC8 : Traçabilité complète (dépend Story 26.1)
**Given** une requête MCP authentifiée via PAT
**When** elle est traitée
**Then** le traceId propagé depuis le header `X-Trace-ID` est présent dans tous les logs
**And** la source est `mcp`

### AC9 : Sécurité — isolation stricte par utilisateur
**Given** un PAT appartenant à l'utilisateur A
**When** une requête tente d'accéder aux données de l'utilisateur B (en manipulant les IDs)
**Then** une erreur 404 est retournée (pas 403 — ne pas révéler l'existence)

## Technical Specification

### Routes

```
GET  /api/mcp/captures              # AC1
GET  /api/mcp/captures/search       # AC3
GET  /api/mcp/captures/:id          # AC2
GET  /api/mcp/thoughts              # AC4
GET  /api/mcp/thoughts/:id          # AC5
GET  /api/mcp/todos                 # AC6
```

### Module structure

```
backend/src/modules/mcp/
├── mcp.module.ts
└── application/
    ├── controllers/
    │   ├── mcp-captures.controller.ts
    │   ├── mcp-thoughts.controller.ts
    │   └── mcp-todos.controller.ts
    └── dto/
        ├── mcp-capture-filters.dto.ts
        ├── mcp-thought-filters.dto.ts
        └── mcp-todo-filters.dto.ts
```

### Pagination curseur

```typescript
interface PaginatedResponse<T> {
  data: T[];
  nextCursor: string | null;  // ID du dernier élément
  hasMore: boolean;
  total?: number;
}
```

### Guard stack sur les routes MCP

```typescript
@UseGuards(PATGuard)
@RequireScopes('captures:read')
@Get()
async listCaptures(@CurrentUser() user, @Query() filters) { ... }
```

## Tasks / Subtasks

### Task 1 : Module MCP + structure (AC7, AC9)

- [ ] Subtask 1.1 : Créer `mcp.module.ts` avec import `PATModule`, `CaptureModule`, `KnowledgeModule`, `ActionModule`
- [ ] Subtask 1.2 : Implémenter filtre global `ownerId` sur toutes les queries MCP (isolation utilisateur)

### Task 2 : Endpoints captures (AC1, AC2, AC3)

- [ ] Subtask 2.1 : `GET /api/mcp/captures` — liste paginée (curseur) avec filtres state/type/dates
- [ ] Subtask 2.2 : `GET /api/mcp/captures/:id` — détail (vérification ownerId)
- [ ] Subtask 2.3 : `GET /api/mcp/captures/search?q=` — ILIKE sur normalizedText
- [ ] Subtask 2.4 : Tests acceptance BDD — listage, filtrage, recherche, isolation utilisateur

### Task 3 : Endpoints thoughts + ideas (AC4, AC5)

- [ ] Subtask 3.1 : `GET /api/mcp/thoughts` — liste paginée avec filtres
- [ ] Subtask 3.2 : `GET /api/mcp/thoughts/:id` — thought + ideas (scope conditionnel)
- [ ] Subtask 3.3 : Tests acceptance BDD

### Task 4 : Endpoints todos (AC6)

- [ ] Subtask 4.1 : `GET /api/mcp/todos` — liste paginée avec filtres completed/priority/deadline
- [ ] Subtask 4.2 : Tests acceptance BDD

### Task 5 : Vérification scopes + isolation (AC7, AC9)

- [ ] Subtask 5.1 : Tests — accès avec scope insuffisant → 403
- [ ] Subtask 5.2 : Tests — accès à ressource d'un autre utilisateur → 404

### Task 6 : Validation finale

- [ ] Subtask 6.1 : `npm run test:acceptance` — tous les scénarios MCP passent
- [ ] Subtask 6.2 : `npm run test` — zéro régression
- [ ] Subtask 6.3 : Code review adversariale — focus sécurité et isolation

## Dev Agent Record

### Debug Log References

### Completion Notes List

### File List

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-03-10 | Story créée | yohikofox |

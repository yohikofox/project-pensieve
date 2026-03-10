# Story 28.2: MCP Server — Package `mcp/` + tools + propagation X-Trace-ID

Status: ready-for-dev

## Story

En tant qu'**utilisateur de Claude Code (ou tout client MCP)**,
je veux **interagir avec mes captures et knowledge Pensine via des outils MCP**,
afin de **pouvoir rechercher, lire et analyser mes pensées directement depuis un contexte IA**.

## Context & Motivation

Dépend de Story 28.1 (endpoints backend) et Story 27.1 (PAT auth). Le serveur MCP est un package standalone `pensieve/mcp/` qui centralise toute la logique dans le backend — le MCP ne fait que transposer les appels REST en outils MCP.

**Deux modes de transport :**
- **stdio** : pour usage local (Claude Code, `claude mcp add`)
- **HTTP+SSE** : pour clients distants (futurs clients)

## Acceptance Criteria

### AC1 : Initialisation et authentification
**Given** le MCP server est configuré avec un PAT valide (`PENSINE_PAT` env var)
**When** un client MCP se connecte
**Then** le serveur est opérationnel et les tools sont listés

**Given** aucun PAT n'est configuré
**When** le serveur démarre
**Then** une erreur explicite est loguée et le process s'arrête

### AC2 : Tool `list_captures`
**Given** le client MCP appelle `list_captures`
**When** le serveur traite la requête
**Then** les captures de l'utilisateur sont retournées formatées pour Claude

**Paramètres optionnels :** `state`, `type`, `from`, `to`, `limit`, `cursor`

**Format de sortie :** chaque capture inclut `id`, `type`, `state`, `normalizedText` (extrait), `createdAt`

### AC3 : Tool `get_capture`
**Given** le client MCP appelle `get_capture` avec un `id`
**When** le serveur traite la requête
**Then** la capture complète est retournée (`rawContent`, `normalizedText` complet)

### AC4 : Tool `search_captures`
**Given** le client MCP appelle `search_captures` avec `query`
**When** le serveur traite la requête
**Then** les captures matching sont retournées avec contexte

### AC5 : Tool `list_thoughts`
**Given** le client MCP appelle `list_thoughts`
**When** le serveur traite la requête
**Then** les thoughts récents sont retournés avec `summary` et `captureId`

**Paramètres optionnels :** `limit`, `cursor`, `from`, `to`

### AC6 : Tool `get_thought`
**Given** le client MCP appelle `get_thought` avec un `id`
**When** le serveur traite la requête
**Then** le thought complet est retourné avec ses ideas associées (si scope `ideas:read` disponible)

### AC7 : Tool `list_todos`
**Given** le client MCP appelle `list_todos`
**When** le serveur traite la requête
**Then** les todos sont retournés

**Paramètres optionnels :** `completed`, `priority`, `from`, `to` (deadline)

### AC8 : Propagation X-Trace-ID (dépend Story 26.1)
**Given** un tool MCP est invoqué
**When** le serveur appelle le backend
**Then** un UUID est généré pour cet appel et envoyé via `X-Trace-ID`
**And** `X-Request-Source: mcp` est envoyé
**And** le `traceId` est logué localement côté MCP : `[abc-123] tool:list_captures → backend`

### AC9 : Mode stdio fonctionnel
**Given** le MCP est configuré dans Claude Code via `claude mcp add`
**When** Claude invoque un tool
**Then** la communication stdio fonctionne sans erreur

### AC10 : Mode HTTP+SSE fonctionnel
**Given** le MCP server est lancé en mode HTTP (`MCP_TRANSPORT=http`)
**When** un client distant se connecte via SSE
**Then** les tools sont disponibles et fonctionnels

### AC11 : Gestion des erreurs
**Given** le backend retourne une erreur (401, 403, 500)
**When** le tool MCP reçoit cette erreur
**Then** un message d'erreur clair est retourné au client MCP (pas de crash serveur)
**And** le traceId est inclus dans le message d'erreur pour débogage

## Technical Specification

### Package structure

```
pensieve/mcp/
├── package.json
├── tsconfig.json
├── .env.example
├── src/
│   ├── server.ts              # Entry point — stdio ou HTTP selon MCP_TRANSPORT
│   ├── client/
│   │   └── pensine-client.ts  # Wrapper HTTP vers backend (fetch + PAT + X-Trace-ID)
│   ├── tools/
│   │   ├── captures.tools.ts  # list_captures, get_capture, search_captures
│   │   ├── thoughts.tools.ts  # list_thoughts, get_thought
│   │   └── todos.tools.ts     # list_todos
│   └── utils/
│       └── trace.ts           # Génération traceId + logging local
└── Dockerfile
```

### Dependencies

```json
{
  "@modelcontextprotocol/sdk": "latest",
  "dotenv": "^17.x",
  "uuid": "^11.x"
}
```

### Variables d'environnement

```env
PENSINE_API_URL=https://api.pensine.example.local
PENSINE_PAT=pns_...
MCP_TRANSPORT=stdio        # stdio | http
MCP_HTTP_PORT=3100         # si transport=http
```

### Intégration Claude Code (stdio)

```json
// ~/.claude/settings.json ou via claude mcp add
{
  "mcpServers": {
    "pensine": {
      "command": "node",
      "args": ["/path/to/pensieve/mcp/dist/server.js"],
      "env": {
        "PENSINE_API_URL": "https://api.pensine.example.local",
        "PENSINE_PAT": "pns_..."
      }
    }
  }
}
```

### Dockerfile (pour déploiement HTTP)

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY dist ./dist
ENV MCP_TRANSPORT=http
CMD ["node", "dist/server.js"]
```

## Tasks / Subtasks

### Task 1 : Setup package (AC1)

- [ ] Subtask 1.1 : Initialiser `pensieve/mcp/package.json` (Node 22, TypeScript, `@modelcontextprotocol/sdk`)
- [ ] Subtask 1.2 : `tsconfig.json` + scripts (`build`, `start`, `dev`)
- [ ] Subtask 1.3 : Créer `PensineClient` — wrapper fetch avec PAT header + X-Trace-ID + X-Request-Source
- [ ] Subtask 1.4 : Validation au démarrage — vérifier `PENSINE_PAT` présent et non vide

### Task 2 : Utilitaire trace (AC8)

- [ ] Subtask 2.1 : `generateTraceId()` — UUID v4
- [ ] Subtask 2.2 : Logger local structuré `[traceId] tool:xxx → backend`

### Task 3 : Tools captures (AC2, AC3, AC4)

- [ ] Subtask 3.1 : `list_captures` — mapping paramètres + appel backend + format réponse
- [ ] Subtask 3.2 : `get_capture` — appel backend + format réponse complet
- [ ] Subtask 3.3 : `search_captures` — appel backend search + format réponse

### Task 4 : Tools thoughts + ideas (AC5, AC6)

- [ ] Subtask 4.1 : `list_thoughts` — mapping paramètres + appel backend
- [ ] Subtask 4.2 : `get_thought` — thought + ideas conditionnels

### Task 5 : Tool todos (AC7)

- [ ] Subtask 5.1 : `list_todos` — mapping paramètres + appel backend

### Task 6 : Entry point stdio + HTTP (AC9, AC10)

- [ ] Subtask 6.1 : `server.ts` — détection `MCP_TRANSPORT` et branchement stdio/HTTP
- [ ] Subtask 6.2 : Mode stdio via `StdioServerTransport` du SDK MCP
- [ ] Subtask 6.3 : Mode HTTP+SSE via `SSEServerTransport` du SDK MCP

### Task 7 : Gestion erreurs (AC11)

- [ ] Subtask 7.1 : Try/catch sur chaque tool — retourner message d'erreur structuré avec traceId
- [ ] Subtask 7.2 : Cas spéciaux : 401 → message "PAT invalide ou révoqué", 403 → "Scope insuffisant"

### Task 8 : Dockerfile + Makefile (déploiement)

- [ ] Subtask 8.1 : Créer `pensieve/mcp/Dockerfile`
- [ ] Subtask 8.2 : Ajouter `build-mcp`, `release-mcp`, `deploy-mcp` dans `Makefile`
- [ ] Subtask 8.3 : Ajouter `mcp` comme composant dans `deploy.sh --only=mcp`

### Task 9 : Tests + documentation

- [ ] Subtask 9.1 : Tests unitaires PensineClient (mock fetch) — propagation headers, gestion erreurs
- [ ] Subtask 9.2 : Tests unitaires tools (mock PensineClient) — mapping paramètres/réponses
- [ ] Subtask 9.3 : Créer `.env.example` avec toutes les variables documentées
- [ ] Subtask 9.4 : Documentation d'intégration Claude Code (instructions `claude mcp add`)

## Dev Agent Record

### Debug Log References

### Completion Notes List

### File List

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-03-10 | Story créée | yohikofox |

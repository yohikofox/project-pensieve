# Story 26.1: Distributed Tracing — TraceMiddleware + TraceContext + Pino enrichissement

Status: done

## Story

En tant que **développeur / ops**,
je veux **que chaque requête backend soit identifiée par un traceId unique propagé dans tous les logs**,
afin de **pouvoir reconstituer la vie complète d'une requête cross-app en filtrant par traceId**.

## Context & Motivation

Prérequis critique pour Epic 27 (PAT) et Epic 28 (MCP). Sans tracing, il est impossible de corréler les logs d'une requête MCP traversant le backend.

**Pattern cible :**
```
[abc-123] [mcp]    → POST /api/captures/search (45.2.3.1)
[abc-123] [mcp]    PATGuard: token validated, user xyz
[abc-123] [mcp]    CaptureService: querying 3 captures
[abc-123] [mcp]    ← 200 OK (45ms)
```

## Acceptance Criteria

### AC1 : Extraction ou génération du traceId
**Given** une requête entrante sans header `X-Trace-ID`
**When** le middleware de tracing est exécuté
**Then** un UUID v4 est généré et utilisé comme traceId pour toute la durée de la requête

**Given** une requête entrante avec header `X-Trace-ID: abc-123`
**When** le middleware de tracing est exécuté
**Then** `abc-123` est utilisé comme traceId (propagation cross-app)

### AC2 : Header retour
**Given** n'importe quelle requête traitée
**When** la réponse est envoyée
**Then** le header `X-Trace-ID` est présent dans la réponse avec le traceId utilisé

### AC3 : Extraction de l'origine
**Given** une requête avec header `X-Request-Source: mcp`
**When** le middleware est exécuté
**Then** la source `mcp` est stockée dans le contexte de tracing

**Given** une requête sans header `X-Request-Source`
**When** le middleware est exécuté
**Then** la source est `unknown`

**Valeurs acceptées :** `mcp`, `mobile`, `web`, `admin`, `unknown`

### AC4 : Log d'ingress
**Given** une requête entrante
**When** le middleware est exécuté
**Then** un log est émis avec : `traceId`, `source`, méthode HTTP, path, IP cliente

**Format attendu :**
```json
{ "traceId": "abc-123", "source": "mcp", "method": "POST", "path": "/api/captures/search", "ip": "1.2.3.4", "msg": "incoming request" }
```

### AC5 : Propagation via AsyncLocalStorage
**Given** le traceId est extrait ou généré en middleware
**When** n'importe quel service NestJS est exécuté dans le contexte de cette requête
**Then** `TraceContext.getTraceId()` retourne le traceId correct
**And** `TraceContext.getSource()` retourne la source correcte
**And** aucun passage explicite du traceId en paramètre n'est nécessaire

### AC6 : Enrichissement automatique des logs Pino
**Given** le traceId est stocké dans AsyncLocalStorage
**When** n'importe quel `logger.log()`, `logger.error()`, etc. est appelé
**Then** chaque ligne de log inclut automatiquement `{ traceId, source }`
**And** ceci s'applique sans modification des call sites existants

### AC7 : Compatibilité avec les requêtes sans contexte (hors HTTP)
**Given** un consumer RabbitMQ exécuté hors contexte HTTP
**When** `TraceContext.getTraceId()` est appelé
**Then** retourne `null` ou `undefined` sans throw d'erreur

### AC8 : Tests
**Given** le middleware est en place
**When** les tests sont exécutés
**Then** tous les tests unitaires du middleware passent
**And** les tests d'intégration vérifient la propagation end-to-end (middleware → service → log)

## Technical Specification

### Fichiers à créer

```
backend/src/common/trace/
├── trace.context.ts          # AsyncLocalStorage singleton + getTraceId/getSource
├── trace.middleware.ts       # NestJS middleware global
├── trace.module.ts           # Module exportant TraceContext
└── trace.context.spec.ts     # Tests unitaires
```

### `TraceContext` (AsyncLocalStorage)

```typescript
import { AsyncLocalStorage } from 'async_hooks';
import { Injectable } from '@nestjs/common';

interface TraceStore {
  traceId: string;
  source: string;
}

const storage = new AsyncLocalStorage<TraceStore>();

@Injectable()
export class TraceContext {
  static getTraceId(): string | undefined {
    return storage.getStore()?.traceId;
  }

  static getSource(): string | undefined {
    return storage.getStore()?.source;
  }

  static run<T>(store: TraceStore, fn: () => T): T {
    return storage.run(store, fn);
  }
}
```

### `TraceMiddleware`

```typescript
@Injectable()
export class TraceMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const traceId = (req.headers['x-trace-id'] as string) ?? uuidv4();
    const source = (req.headers['x-request-source'] as string) ?? 'unknown';

    res.setHeader('X-Trace-ID', traceId);

    TraceContext.run({ traceId, source }, () => {
      this.logger.log({ traceId, source, method: req.method, path: req.path, ip: req.ip, msg: 'incoming request' });
      next();
    });
  }
}
```

### Pino `mixin`

Dans `app.module.ts` / config pino :
```typescript
mixin: () => ({
  traceId: TraceContext.getTraceId(),
  source: TraceContext.getSource(),
})
```

### Enregistrement middleware global

Dans `AppModule` :
```typescript
configure(consumer: MiddlewareConsumer) {
  consumer.apply(TraceMiddleware).forRoutes('*');
}
```

## Tasks / Subtasks

### Task 1 : TraceContext + AsyncLocalStorage (AC5, AC7)

- [x] Subtask 1.1 : Créer `backend/src/common/trace/trace.context.ts` avec `AsyncLocalStorage<TraceStore>` + méthodes statiques `getTraceId()`, `getSource()`, `run()`
- [x] Subtask 1.2 : Écrire tests unitaires `trace.context.spec.ts` — vérifier isolation entre contextes parallèles, retour `undefined` hors contexte

### Task 2 : TraceMiddleware (AC1, AC2, AC3, AC4)

- [x] Subtask 2.1 : Créer `trace.middleware.ts` — extraire `X-Trace-ID` ou générer UUID v4, extraire `X-Request-Source`, appeler `TraceContext.run()`
- [x] Subtask 2.2 : Émettre log d'ingress avec `{ traceId, source, method, path, ip }`
- [x] Subtask 2.3 : Setter header `X-Trace-ID` en réponse
- [x] Subtask 2.4 : Écrire tests unitaires middleware — propagation header, génération UUID, source fallback

### Task 3 : TraceModule + enregistrement global (AC5)

- [x] Subtask 3.1 : Créer `trace.module.ts` exportant `TraceContext` et `TraceMiddleware`
- [x] Subtask 3.2 : Enregistrer `TraceMiddleware` globalement dans `AppModule.configure()`

### Task 4 : Enrichissement Pino (AC6)

- [x] Subtask 4.1 : Configurer le `mixin` Pino pour injecter `traceId` et `source` dans chaque log
- [x] Subtask 4.2 : Vérifier que les logs existants (PinoLogger) incluent automatiquement les champs sans modification des call sites

### Task 5 : Tests d'intégration (AC8)

- [x] Subtask 5.1 : Test d'intégration — requête HTTP avec `X-Trace-ID` fourni → même traceId dans les logs du service appelé
- [x] Subtask 5.2 : Test d'intégration — requête sans header → traceId généré présent dans logs + header réponse
- [x] Subtask 5.3 : `npm run test` — zéro régression

## Dev Agent Record

### Debug Log References

### Completion Notes List

### File List

**Fichiers créés :**
- `backend/src/common/trace/trace.context.ts`
- `backend/src/common/trace/trace.middleware.ts`
- `backend/src/common/trace/trace.middleware.spec.ts`
- `backend/src/common/trace/trace.module.ts`
- `backend/src/common/trace/trace.context.spec.ts`
- `backend/src/common/trace/index.ts`
- `backend/test/acceptance/features/story-26-1-distributed-tracing.feature`
- `backend/test/acceptance/story-26-1.test.ts`

**Fichiers modifiés :**
- `backend/src/app.module.ts`
- `backend/src/config/logger.config.ts`

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-03-10 | Story créée — distributed tracing, prérequis MCP/PAT | yohikofox |

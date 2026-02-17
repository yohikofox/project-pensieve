# Story 14.3: Intégration Observability — Sentry et Structured Logging Backend

Status: backlog

## Story

As a **developer**,
I want **proper observability tools integrated (Sentry for error tracking, structured logging for backend)**,
So that **the system meets ADR-015 requirements for monitoring, alerting, and debugging in production**.

## Context

Audit ADR-015 (2026-02-17) révèle :
- `LoggerService` présent en mobile ✅
- `queue-monitoring.service.ts` et `metrics.controller.ts` présents en backend ✅
- **Manques** :
  - Intégration Sentry non trouvée dans les dépendances
  - Stack Prometheus/Grafana non confirmée dans le code backend
  - Logger structuré NestJS (winston ou pino) non visible dans les dépendances backend

**Décision à prendre** : Le scope exact de cette story dépend de la priorité pour le sprint actuel. La story est en **Priorité BASSE** — elle peut attendre la fin d'Epic 6.

## Acceptance Criteria

### AC1: Logger structuré NestJS (backend)
**Given** the backend uses NestJS built-in Logger
**When** I integrate a structured logger
**Then** either `winston` or `pino` is added as a NestJS logger provider
**And** all log entries are JSON-formatted with: `timestamp`, `level`, `message`, `context`, `requestId`
**And** existing `logger.info()` / `logger.error()` calls remain compatible

### AC2: Sentry intégration (mobile)
**Given** Sentry is not in mobile dependencies
**When** I evaluate Sentry integration
**Then** if Sentry is required for the current sprint:
  - `@sentry/react-native` is added with Expo SDK 54 compatibility verified
  - Sentry.init() is called in `index.ts` before app bootstrap
  - Uncaught errors are captured automatically
  - Manual `Sentry.captureException()` wraps known failure points
**Or** if Sentry is deferred: this AC is documented as out-of-scope with a ticket for the next sprint

### AC3: Sentry intégration (backend)
**Given** Sentry is not in backend dependencies
**When** I evaluate Sentry integration
**Then** if Sentry is required:
  - `@sentry/nestjs` is added
  - NestJS exception filter captures unhandled exceptions → Sentry
  - DSN configured via environment variable `SENTRY_DSN`
**Or** if deferred: documented as out-of-scope

### AC4: Métriques Prometheus exposées (backend)
**Given** `metrics.controller.ts` exists in the backend
**When** I verify Prometheus compatibility
**Then** the metrics endpoint (`/metrics`) returns Prometheus-compatible format
**And** the following metrics are exposed:
  - HTTP request duration (histogram)
  - RabbitMQ queue depth
  - Active connections
**Or** if Prometheus stack is not deployed: document the decision

### AC5: Documentation de la décision observability
**Given** this is a low-priority story
**When** implementation begins
**Then** the team documents in devnotes which observability tools are in scope for V1
**And** deferred items are tracked in `_bmad-output/implementation-artifacts/observability-decisions.md`

## Tech Notes

- **Compatibilité Sentry + Expo SDK 54** : Vérifier `@sentry/react-native` ≥ 5.x pour Expo SDK 54 avant installation
- **Logger NestJS** : Préférer `pino` (plus performant) ou `winston` (plus de plugins)
  - Pattern : `WinstonModule.forRoot({...})` dans `AppModule`
- **Prometheus** : Si le stack n'est pas déployé en homelab, la story peut se limiter à vérifier que `metrics.controller.ts` expose le bon format
- **Priorité** : Implémenter AC1 (structured logging) en priorité — faible risque, valeur immédiate
- **Sentry DSN** : Configurer via `.env.example` (ne jamais committer le DSN réel)

## Related

- ADR-015: Observability Strategy
- `pensieve/backend/src/modules/knowledge/application/controllers/metrics.controller.ts`
- `pensieve/mobile/src/infrastructure/logging/LoggerService.ts`

## Definition of Done

- [ ] Logger structuré JSON en backend (winston ou pino) — AC1
- [ ] Décision documentée sur Sentry (intégré ou déféré avec ticket)
- [ ] Décision documentée sur Prometheus (intégré ou déféré avec justification)
- [ ] Si Sentry intégré : tests de capture d'erreur en dev
- [ ] `SENTRY_DSN` dans `.env.example` (pas de DSN réel committé)
- [ ] `observability-decisions.md` créé listant les décisions prises
- [ ] Zero régression sur tests existants

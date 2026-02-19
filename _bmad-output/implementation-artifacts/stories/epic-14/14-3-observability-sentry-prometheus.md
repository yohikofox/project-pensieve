# Story 14.3: Intégration Observability — Sentry et Structured Logging Backend

Status: done

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
**Then** `nestjs-pino` is added as a NestJS logger provider (amendement ADR-015 §15.1 — 2026-02-18)
**And** all log entries are JSON-formatted with: `time` (Unix ms), `level`, `msg`, `context`
**And** HTTP-scoped logs additionally include `reqId` (injected by pino-http middleware only — not present on non-HTTP logs such as workers, startup logs)
**And** sensitive fields are automatically redacted: `authorization`, `cookie`, `email`, `password`, `token`, `transcription`
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

## Tasks / Subtasks

### Task 1: Documentation des décisions observability (AC2, AC3, AC4, AC5)
- [x] T1.1: Évaluer compatibilité `@sentry/react-native` avec Expo SDK 54 — décider intégrer ou déférer
- [x] T1.2: Évaluer intégration `@sentry/nestjs` backend — décider intégrer ou déférer
- [x] T1.3: Vérifier que `/metrics` expose un format Prometheus valide — documenter métriques manquantes
- [x] T1.4: Créer `observability-decisions.md` avec toutes les décisions V1

### Task 2: Logger structuré NestJS avec pino (AC1)
- [x] T2.1: Installer `nestjs-pino`, `pino-http` (prod) et `pino-pretty` (dev)
- [x] T2.2: Configurer `LoggerModule` pino dans `AppModule` (JSON structuré + pretty en dev)
- [x] T2.3: Mettre à jour `main.ts` — `bufferLogs: true` + `app.useLogger(app.get(Logger))`
- [x] T2.4: Ajouter `LOG_LEVEL`, `SENTRY_DSN`, `LOG_FILE_PATH` dans `.env.example`
- [x] T2.5: Écrire tests unitaires pour la configuration du logger (11 tests)
- [x] T2.6: Ajouter `redact` pino pour filtrage PII (ADR-015 §15.1 — H1 code review)
- [x] T2.7: Ajouter transport fichier optionnel via `LOG_FILE_PATH` (ADR-015 rotation — M3 code review)

### Task 3: Tests BDD acceptance (AC1)
- [x] T3.1: Créer `test/acceptance/features/story-14-3.feature` (Gherkin, 4 scénarios)
- [x] T3.2: Créer `test/acceptance/story-14-3.test.ts` (step definitions, 4 tests)

## Dev Notes

### Décisions prises (2026-02-18)

**Sentry (AC2 mobile + AC3 backend) → DIFFÉRÉ V1.5**
- Raison : `@sentry/react-native` ≥ 6.x requiert Expo config plugin natif + DSN Sentry project actif
- Pas de compte Sentry configuré pour ce sprint
- Ticket créé : voir `observability-decisions.md`

**Prometheus (AC4) → PARTIELLEMENT VÉRIFIÉ**
- `metrics.controller.ts` expose déjà un format Prometheus valide (queue depth, jobs)
- HTTP request duration histogram et active connections : déféré car `prom-client` non installé
- Le stack Prometheus homelab n'est pas confirmé comme déployé
- Décision documentée dans `observability-decisions.md`

**Logger pino (AC1) → IMPLÉMENTÉ**
- `nestjs-pino` v4.x + `pino-http` v10.x installés
- `pino-pretty` en devDependency
- `LoggerModule.forRootAsync()` dans `AppModule`
- `main.ts` mis à jour avec `bufferLogs: true`

## Dev Agent Record

### Implementation Plan

1. **Task 1** : Documentation et décisions — création `observability-decisions.md`
2. **Task 2** : Installation pino + configuration `AppModule` + `main.ts` + `.env.example`
3. **Task 3** : Tests BDD acceptance

### Debug Log

*(vide)*

### Completion Notes

**AC1 — Logger structuré pino** : Implémenté avec `nestjs-pino` v4.x + `pino-http` v11.x. La configuration produit des logs JSON avec les champs requis : `time`, `level`, `msg`, `context`, `reqId`. En dev, `pino-pretty` est utilisé pour une sortie lisible en console. `main.ts` mis à jour avec `bufferLogs: true` pour capturer les logs au démarrage.

**AC2 / AC3 — Sentry** : Différé V1.5 — compte Sentry non disponible, intégration native Expo requise. Décision documentée dans `observability-decisions.md`.

**AC4 — Prometheus** : Format validé — l'endpoint `/metrics` existant expose déjà un format Prometheus valide. HTTP histogram et active connections différés (prom-client non installé, stack homelab non confirmé).

**AC5** : `observability-decisions.md` créé avec toutes les décisions V1.

**Tests** : 8 tests unitaires (logger config) + 4 BDD acceptance. Zéro régression (mêmes 4 tests pré-existants échouent avant et après).

## File List

**Backend — Nouveaux fichiers :**
- `pensieve/backend/src/config/logger.config.ts`
- `pensieve/backend/src/config/logger.config.spec.ts`
- `pensieve/backend/src/config/logger.integration.spec.ts` (intégration NestJS — H2 code review)
- `pensieve/backend/test/acceptance/features/story-14-3-observability-logger.feature`
- `pensieve/backend/test/acceptance/story-14-3.test.ts`

**Backend — Fichiers modifiés :**
- `pensieve/backend/src/app.module.ts` (import LoggerModule + LOG_FILE_PATH)
- `pensieve/backend/src/main.ts` (bufferLogs + useLogger + void bootstrap)
- `pensieve/backend/.env.example` (LOG_LEVEL + SENTRY_DSN + LOG_FILE_PATH)
- `pensieve/backend/package.json` (nestjs-pino, pino-http, pino-pretty)

**Documentation :**
- `_bmad-output/implementation-artifacts/observability-decisions.md` (nouveau)

## Change Log

| Date | Author | Change |
|------|--------|--------|
| 2026-02-18 | dev-agent | Story enrichie avec Tasks/Subtasks, Dev Notes, démarrage implémentation |
| 2026-02-18 | dev-agent | AC1 implémenté : nestjs-pino + pino-http configurés, 12 tests écrits. AC2/AC3/AC4 partiels documentés. |
| 2026-02-19 | code-review | 8 issues (2H+4M+2L) — 6 fixes appliqués : SensitiveDataFilter redact (H1), intégration NestJS réelle logger.integration.spec.ts (H2), champs AC1 corrigés time/msg/reqId (M1+M2), transport fichier LOG_FILE_PATH (M3), mock RabbitMQ BDD (M4), version pino-http v11 doc (L2), ServerResponse inutilisé retiré (L1). 11 unit tests + 4 integration + 4 BDD. |

## Definition of Done

- [x] Logger structuré JSON en backend (winston ou pino) — AC1
- [x] Décision documentée sur Sentry (intégré ou déféré avec ticket)
- [x] Décision documentée sur Prometheus (intégré ou déféré avec justification)
- [ ] Si Sentry intégré : tests de capture d'erreur en dev (N/A — Sentry différé)
- [x] `SENTRY_DSN` dans `.env.example` (pas de DSN réel committé)
- [x] `observability-decisions.md` créé listant les décisions prises
- [x] Zero régression sur tests existants

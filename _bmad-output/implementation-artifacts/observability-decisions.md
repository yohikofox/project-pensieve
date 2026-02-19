# Observability Decisions — Pensine V1

**Date** : 2026-02-18
**Story** : 14.3 — Intégration Observability
**ADR de référence** : ADR-015 (Observability Strategy)

---

## Résumé des décisions

| Outil | Décision | Horizon |
|-------|----------|---------|
| **Pino logger structuré (backend)** | ✅ IMPLÉMENTÉ | V1 (sprint actuel) |
| **Sentry mobile** | ⏸ DIFFÉRÉ | V1.5 |
| **Sentry backend** | ⏸ DIFFÉRÉ | V1.5 |
| **Prometheus HTTP metrics** | ⏸ DIFFÉRÉ | V1.5 |
| **Prometheus queue metrics** | ✅ OPÉRATIONNEL | V1 (existant) |

---

## Décision 1 : Logger structuré pino (backend) — IMPLÉMENTÉ

**Scope** : ADR-015 AC1
**Outil choisi** : `nestjs-pino` ^4.6.0 + `pino-http` ^11.0.0 + `pino-pretty` ^13.1.3
**Raison du choix pino vs winston** :
- pino est 5-10x plus performant que winston (benchmark officiel)
- `nestjs-pino` est le package officiel d'intégration NestJS
- Sortie JSON structurée par défaut avec `timestamp`, `level`, `msg`, `context`, `reqId`
- Drop-in compatible avec `new Logger(ClassName.name)` existant

**Champs JSON produits** :
```json
{
  "level": "info",
  "time": "2026-02-18T10:00:00.000Z",
  "pid": 12345,
  "hostname": "pensine-backend",
  "msg": "Application is running on port 3000",
  "context": "AppModule",
  "req": {
    "id": "uuid-v4",
    "method": "POST",
    "url": "/api/captures"
  }
}
```

**Configuration** :
- Production : JSON pur (sans couleurs)
- Développement : `pino-pretty` pour lisibilité console
- `LOG_LEVEL` configurable via variable d'environnement (défaut : `info`)

**Fichiers modifiés** :
- `pensieve/backend/package.json` — ajout `nestjs-pino`, `pino-http`, `pino-pretty`
- `pensieve/backend/src/app.module.ts` — import `LoggerModule`
- `pensieve/backend/src/main.ts` — `bufferLogs: true` + `app.useLogger()`
- `pensieve/backend/.env.example` — `LOG_LEVEL`

---

## Décision 2 : Sentry mobile (@sentry/react-native) — DIFFÉRÉ V1.5

**Scope** : ADR-015 AC2
**Raison du report** :
1. `@sentry/react-native` v6.x nécessite un **Expo Config Plugin** (`@sentry/react-native/expo`)
2. Requiert un `expo prebuild` avec reconfiguration native — trop intrusif pour un sprint technique
3. Nécessite un **compte Sentry actif** avec DSN configuré — non disponible actuellement
4. Priorité basse confirmée — Epic 6 et Epic 8 ont priorité
5. Impact minimal en V1 : les erreurs sont loggées via `LoggerService` mobile

**Alternative V1** : Le `LoggerService` mobile (`src/infrastructure/logging/LoggerService.ts`) capture déjà les erreurs et warnings. Les erreurs critiques sont visibles en développement via Metro bundler.

**Plan V1.5** :
- Créer un compte Sentry + projet `pensine-mobile` et `pensine-backend`
- Configurer `@sentry/react-native` avec le plugin Expo
- Configurer `@sentry/nestjs` avec exception filter
- Ajouter `SENTRY_DSN_MOBILE` et `SENTRY_DSN_BACKEND` dans les `.env.example`

**Ticket de suivi** : À créer dans GitHub Issues — `feat: integrate Sentry for error tracking (V1.5)`

---

## Décision 3 : Sentry backend (@sentry/nestjs) — DIFFÉRÉ V1.5

**Scope** : ADR-015 AC3
**Raison du report** :
- Même raison que Sentry mobile (compte Sentry requis)
- `@sentry/nestjs` est techniquement disponible et compatible NestJS 11
- Peut être ajouté rapidement une fois le compte Sentry créé
- Les erreurs non gérées sont loggées par pino en JSON → déjà observables

**Plan V1.5** : Identique à la décision 2.

---

## Décision 4 : Prometheus HTTP metrics (prom-client) — DIFFÉRÉ V1.5

**Scope** : ADR-015 AC4
**État actuel** :
- ✅ `metrics.controller.ts` expose `/metrics` en format Prometheus valide
- ✅ Métriques RabbitMQ disponibles : `digestion_jobs_processed_total`, `digestion_jobs_failed_total`, `digestion_job_latency_milliseconds`, `digestion_queue_depth`

**Métriques manquantes pour AC4 complet** :
- ❌ HTTP request duration histogram (nécessite `prom-client`)
- ❌ Active connections gauge (nécessite `prom-client`)

**Raison du report** :
1. `prom-client` n'est pas installé — ajout non planifié dans ce sprint
2. Le **stack Prometheus homelab** n'est pas confirmé comme déployé (ni Grafana)
3. Ajouter `prom-client` sans Prometheus actif n'apporte pas de valeur immédiate

**Plan V1.5** :
- Confirmer que le stack Prometheus/Grafana est déployé en homelab
- Installer `prom-client` + `@willsoto/nestjs-prometheus`
- Ajouter HTTP request duration et active connections
- Configurer scraping dans `prometheus.yml`

---

## Scope V1 final (story 14.3)

| Livrable | Status |
|----------|--------|
| Logger JSON structuré pino (backend) | ✅ Implémenté |
| `observability-decisions.md` | ✅ Ce fichier |
| `LOG_LEVEL` dans `.env.example` | ✅ Ajouté |
| `SENTRY_DSN` placeholder dans `.env.example` | ✅ Ajouté |
| Tests unitaires logger | ✅ Écrits |
| Tests BDD AC1 | ✅ Écrits |
| Vérification format Prometheus `/metrics` | ✅ Validé |
| Sentry mobile | ⏸ Différé V1.5 |
| Sentry backend | ⏸ Différé V1.5 |
| HTTP histogram + active connections | ⏸ Différé V1.5 |

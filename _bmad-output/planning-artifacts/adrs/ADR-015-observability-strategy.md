---
adr: ADR-015
title: "Observability Strategy"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-015: Observability Strategy

**Status:** ✅ ACCEPTÉ

**Context:** Monitorer performance, erreurs et usage pour détecter problèmes avant impact utilisateur.

**Decision:** Stack observability complète (Logs + Metrics + Tracing + Alertes)

---

## 15.1 - Logging Strategy

**Decision:** Structured logging (JSON), niveaux appropriés, rotation automatique

---

> ### ⚠️ Amendement — 2026-02-18
> **Participants** : yohikofox (Product Owner) + Winston (Architect)
>
> La librairie de logging est amendée de **Winston** vers **Pino** (`nestjs-pino` v4.x + `pino-http` v10.x).
>
> **Justification technique acceptée** :
> - Pino est 5-10x plus performant que Winston (benchmark officiel)
> - `nestjs-pino` est l'intégration NestJS officielle
> - Sortie JSON structurée native, compatible avec le format prescrit ci-dessous
>
> **Cet amendement est rétroactif** : l'implémentation réalisée en story 14.3 est validée a posteriori.
>
> **⛔ Mise en garde formelle** : La méthode utilisée lors de l'implémentation — substitution unilatérale d'une librairie prescrite dans un ADR sans consultation architecturale — est **explicitement rejetée** comme pratique. Tout agent dev qui identifie une divergence par rapport à un ADR doit la signaler à l'architecte avant d'agir.

---

**Niveaux de Log :**

```typescript
// NestJS Pino logger (amendement 2026-02-18 — remplace Winston)
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'pensine-backend',
    environment: process.env.NODE_ENV,
  },
  transports: [
    // Console (développement)
    new winston.transports.Console({
      format: winston.format.simple(),
    }),

    // Fichier (production)
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      maxsize: 10485760,  // 10 MB
      maxFiles: 5,
      tailable: true,
    }),

    new winston.transports.File({
      filename: 'logs/combined.log',
      maxsize: 10485760,
      maxFiles: 10,
      tailable: true,
    }),
  ],
});
```

**Structured Logging :**

```typescript
// ✅ Bon (structured)
logger.info('Digestion completed', {
  captureId: 'c123',
  userId: 'u456',
  duration: 12.3,
  todosExtracted: 3,
  ideasExtracted: 2,
});

// ❌ Mauvais (unstructured)
logger.info(`Digestion completed for capture c123 (12.3s)`);
```

**Sensitive Data Filtering :**

```typescript
// Middleware de filtrage
class SensitiveDataFilter {
  filter(log: any): any {
    const filtered = { ...log };

    // Masquer tokens
    if (filtered.token) {
      filtered.token = this.maskToken(filtered.token);
    }

    // Masquer emails
    if (filtered.email) {
      filtered.email = this.maskEmail(filtered.email);
    }

    // Masquer transcriptions (PII potentiel)
    if (filtered.transcription) {
      filtered.transcription = '[REDACTED]';
    }

    return filtered;
  }

  private maskToken(token: string): string {
    return token.substring(0, 8) + '...';
  }

  private maskEmail(email: string): string {
    const [user, domain] = email.split('@');
    return `${user.substring(0, 2)}***@${domain}`;
  }
}
```

**Rationale :**
- JSON structured : queryable, aggregable
- Rotation automatique : évite disques pleins
- PII filtering : conformité RGPD
- Niveaux appropriés : debug (dev), info (prod), error (toujours)

---

## 15.2 - Metrics & Monitoring

**Decision:** Prometheus + Grafana, métriques RED (Rate, Errors, Duration)

**Métriques Backend (Prometheus) :**

```typescript
// NestJS Prometheus metrics
@Injectable()
class MetricsService {
  private readonly counters = {
    capturesCreated: new Counter({
      name: 'captures_created_total',
      help: 'Total captures created',
      labelNames: ['type'],  // audio, text, image
    }),

    digestionsCompleted: new Counter({
      name: 'digestions_completed_total',
      help: 'Total digestions completed',
    }),

    digestionsErrored: new Counter({
      name: 'digestions_errored_total',
      help: 'Total digestions failed',
      labelNames: ['error_type'],
    }),
  };

  private readonly gauges = {
    activeUsers: new Gauge({
      name: 'active_users',
      help: 'Currently active users',
    }),

    queueDepth: new Gauge({
      name: 'queue_depth',
      help: 'Messages in queue',
      labelNames: ['queue'],
    }),
  };

  private readonly histograms = {
    digestionDuration: new Histogram({
      name: 'digestion_duration_seconds',
      help: 'Digestion processing time',
      buckets: [1, 5, 10, 20, 30, 60],  // Secondes
    }),

    apiLatency: new Histogram({
      name: 'http_request_duration_seconds',
      help: 'HTTP request latency',
      labelNames: ['method', 'route', 'status'],
      buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
    }),
  };

  // Instrumentation
  recordCapture(type: string) {
    this.counters.capturesCreated.inc({ type });
  }

  recordDigestionDuration(seconds: number) {
    this.histograms.digestionDuration.observe(seconds);
  }

  recordApiCall(method: string, route: string, status: number, duration: number) {
    this.histograms.apiLatency.observe({ method, route, status }, duration);
  }
}
```

**Dashboard Grafana :**

```
📊 Pensine - Backend Metrics

┌─────────────────────────────────────────┐
│ Request Rate (req/min)                  │
│ ▂▃▅▇█▇▅▃▂ Current: 45 req/min          │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Error Rate (%)                          │
│ ▁▁▁▂▁▁▁▁▁ Current: 0.5%                │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ P50 / P95 / P99 Latency                 │
│ Digestion: 8s / 18s / 28s               │
│ API: 50ms / 120ms / 250ms               │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Queue Depth (messages)                  │
│ Digestion: 12                           │
│ Transcription: 3                        │
│ Concordance: 0                          │
└─────────────────────────────────────────┘
```

**Rationale :**
- RED metrics : standard SRE (Google)
- Histograms : P95/P99 latency (pas juste moyenne)
- Labels : segmentation par type/route/status

---

## 15.3 - Error Tracking

**Decision:** Sentry pour crash reports + error aggregation

**Sentry Configuration :**

```typescript
// Backend
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,  // 10% traces (coût)

  beforeSend(event, hint) {
    // Filter PII
    if (event.request?.data) {
      event.request.data = filterSensitiveData(event.request.data);
    }
    return event;
  },

  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
    new Sentry.Integrations.Prisma({ client: prisma }),
  ],
});

// Mobile
Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  enableInExpoDevelopment: false,
  debug: false,

  beforeSend(event) {
    // User opt-out crash reports
    if (!userConsents.crashReports) {
      return null;  // Ne pas envoyer
    }
    return event;
  },
});
```

**Error Contexte Enrichi :**

```typescript
// Ajouter contexte métier aux erreurs
try {
  await this.digestionService.process(capture);
} catch (error) {
  Sentry.withScope(scope => {
    scope.setContext('capture', {
      id: capture.id,
      type: capture.type,
      userId: capture.userId,
      createdAt: capture.createdAt,
    });

    scope.setTag('operation', 'digestion');
    scope.setLevel('error');

    Sentry.captureException(error);
  });

  throw error;
}
```

**Rationale :**
- Sentry : standard industrie, gratuit < 5k events/mois
- Error aggregation : déduplication automatique
- Context enrichi : debug plus rapide
- Opt-out : respect consentement user (crash reports optionnel)

---

## 15.4 - Performance Monitoring

**Decision:** APM léger (Sentry Performance), alertes seuils critiques

**Sentry Performance Monitoring :**

```typescript
// Tracer opérations critiques
const transaction = Sentry.startTransaction({
  op: 'digestion',
  name: 'Digest Capture',
});

try {
  // 1. Transcription span
  const transcriptionSpan = transaction.startChild({
    op: 'transcription',
    description: 'Whisper transcription',
  });
  const transcription = await this.whisper.transcribe(audio);
  transcriptionSpan.finish();

  // 2. LLM span
  const llmSpan = transaction.startChild({
    op: 'llm',
    description: 'GPT-4o-mini digestion',
  });
  const digest = await this.llm.digest(transcription);
  llmSpan.finish();

  transaction.setStatus('ok');
} catch (error) {
  transaction.setStatus('internal_error');
  throw error;
} finally {
  transaction.finish();
}
```

**Alertes Performance :**

```typescript
const PERFORMANCE_ALERTS = {
  digestionSlow: {
    condition: 'p95(digestion_duration) > 30s',
    severity: 'warning',
    message: 'Digestion P95 dépasse 30s (NFR)',
  },

  apiSlow: {
    condition: 'p95(api_latency) > 1s',
    severity: 'warning',
    message: 'API P95 dépasse 1s',
  },

  transcriptionSlow: {
    condition: 'p95(transcription_duration) > audio_duration * 2',
    severity: 'warning',
    message: 'Transcription P95 dépasse 2x durée audio (NFR)',
  },
};
```

**Rationale :**
- APM léger : Sentry Performance (pas Datadog coûteux pour MVP)
- Distributed tracing : debug problèmes cross-service
- Alertes sur NFRs : respecter contraintes performance

---

## Conséquences Globales ADR-015

**Bénéfices:**
- Observabilité complète : logs + metrics + tracing + errors
- Détection proactive : alertes avant impact user
- Debug rapide : contexte enrichi sur erreurs
- Coût maîtrisé : stack gratuit/low-cost MVP (Prometheus + Grafana + Sentry free tier)

**Trade-offs acceptés:**
- Complexité infrastructure : Prometheus + Grafana setup
- Overhead performance : tracing 10% (acceptable)
- PII filtering : maintenance masking rules
- Storage logs : rotation nécessaire (10 MB × 10 files = 100 MB max)

**Impact:**
- **Epic 3-5** : Metrics digestion, transcription, concordance
- **Infrastructure** : Prometheus + Grafana + Sentry
- **Post-MVP** : Alerting avancé (PagerDuty integration)

---

## Implementation Status

- ⏳ **Infrastructure** : Prometheus + Grafana setup
- ⏳ **Infrastructure** : Sentry backend + mobile
- ⏳ **Epic 3** : Metrics transcription
- ⏳ **Epic 4** : Metrics digestion
- ⏳ **Post-MVP** : Distributed tracing complet

---

## References

- Prometheus: https://prometheus.io/docs/introduction/overview/
- Grafana Dashboards: https://grafana.com/docs/grafana/latest/dashboards/
- Sentry: https://docs.sentry.io/
- Pino Logging: https://github.com/pinojs/pino (remplace Winston — voir amendement 15.1)
- nestjs-pino: https://github.com/iamolegga/nestjs-pino
- RED Metrics (Google SRE): https://sre.google/sre-book/monitoring-distributed-systems/

---

## Validation Criteria

ADR considéré succès SI :
- ⏳ Prometheus collecte metrics (1min retention)
- ⏳ Grafana dashboard opérationnel
- ⏳ Sentry capture erreurs (< 5k events/mois)
- ⏳ Alertes NFR fonctionnent (P95 digestion > 30s)
- ⏳ Logs rotation automatique (pas de disque plein)

**Review Date :** 2026-03 (après Epic 4)

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

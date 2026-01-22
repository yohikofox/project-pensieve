---
adr: ADR-012
title: "Queue Management avec RabbitMQ"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-012: Queue Management avec RabbitMQ

**Status:** ✅ ACCEPTÉ

**Context:** Gérer les tâches asynchrones (transcription, digestion IA, concordance) avec RabbitMQ pour isolation pannes et résilience.

**Decision:** 4 décisions validées pour gestion complète des queues.

---

## 12.1 - Dead Letter Queues (DLQ)

**Decision:** DLQ systématique pour chaque queue métier

**Architecture :**

```typescript
// Configuration RabbitMQ
const queueConfig = {
  // Queue principale
  digestion: {
    name: 'digestion.queue',
    durable: true,
    arguments: {
      'x-dead-letter-exchange': 'dlx',
      'x-dead-letter-routing-key': 'digestion.dlq',
      'x-message-ttl': 300000,  // 5 min max processing
    }
  },

  // Dead Letter Queue
  digestion_dlq: {
    name: 'digestion.dlq',
    durable: true,
    // Pas de retry automatique depuis DLQ
  }
};

// Transcription queue
const transcriptionConfig = {
  name: 'transcription.queue',
  durable: true,
  arguments: {
    'x-dead-letter-exchange': 'dlx',
    'x-dead-letter-routing-key': 'transcription.dlq',
    'x-message-ttl': 600000,  // 10 min max (audio long)
  }
};

// Concordance queue
const concordanceConfig = {
  name: 'concordance.queue',
  durable: true,
  arguments: {
    'x-dead-letter-exchange': 'dlx',
    'x-dead-letter-routing-key': 'concordance.dlq',
    'x-message-ttl': 60000,  // 1 min max
  }
};
```

**Consumer avec ACK manuel :**

```typescript
@Injectable()
class DigestionConsumer {
  @RabbitSubscribe({
    exchange: 'pensine',
    routingKey: 'digestion.queue',
    queue: 'digestion.queue',
  })
  async handleDigestion(msg: DigestionMessage, context: RabbitContext) {
    const channel = context.getChannelRef();
    const originalMsg = context.getMessage();

    try {
      // Traitement
      const result = await this.digestionService.process(msg);

      // ACK success
      channel.ack(originalMsg);

      // Publier résultat
      await this.eventBus.publish(new ThoughtDigested(result));

    } catch (error) {
      // Log erreur
      this.logger.error('Digestion failed', { msg, error });

      // NACK → va en DLQ
      channel.nack(originalMsg, false, false);

      // Notifier erreur
      await this.notificationService.sendError(msg.userId, 'digestion_failed');
    }
  }
}
```

**Monitoring DLQ :**

```typescript
// Cron job : monitorer DLQ toutes les 10 minutes
@Cron('*/10 * * * *')
async monitorDeadLetters() {
  const dlqStats = await this.rabbitService.getQueueStats([
    'digestion.dlq',
    'transcription.dlq',
    'concordance.dlq'
  ]);

  for (const [queue, stats] of Object.entries(dlqStats)) {
    if (stats.messages > 0) {
      // Alerte si messages en DLQ
      await this.alertService.send({
        severity: 'warning',
        message: `${stats.messages} messages in ${queue}`,
        queue,
        count: stats.messages
      });
    }
  }
}
```

**Rationale :**
- DLQ évite perte de messages échoués
- TTL empêche blocage infini (timeout)
- NACK sans requeue → DLQ directement
- Monitoring DLQ = détection erreurs systémiques

---

## 12.2 - Retry Logic & Exponential Backoff

**Decision:** Retry avec Fibonacci backoff, max 5 attempts

**Retry Headers (RabbitMQ) :**

```typescript
// Message avec retry count dans headers
interface MessageWithRetry {
  payload: any;
  headers: {
    'x-retry-count': number;
    'x-first-attempt': number;  // Timestamp
    'x-last-attempt': number;
  };
}

// Publisher ajoute headers
await this.rabbitService.publish('digestion.queue', {
  captureId: 'c123',
  userId: 'u456',
}, {
  headers: {
    'x-retry-count': 0,
    'x-first-attempt': Date.now(),
    'x-last-attempt': Date.now(),
  }
});
```

**Consumer avec retry logic :**

```typescript
@Injectable()
class DigestionConsumer {
  private readonly MAX_RETRIES = 5;
  private readonly FIBONACCI_DELAYS = [1, 1, 2, 3, 5, 8];  // Secondes

  async handleDigestion(msg: MessageWithRetry, context: RabbitContext) {
    const channel = context.getChannelRef();
    const originalMsg = context.getMessage();
    const retryCount = msg.headers['x-retry-count'] || 0;

    try {
      await this.digestionService.process(msg.payload);
      channel.ack(originalMsg);

    } catch (error) {
      // Vérifier si erreur retryable
      const isRetryable = this.isRetryableError(error);

      if (!isRetryable || retryCount >= this.MAX_RETRIES) {
        // Non retryable ou max retries → DLQ
        this.logger.error('Max retries reached or non-retryable', { msg, error });
        channel.nack(originalMsg, false, false);  // → DLQ
        return;
      }

      // Retry avec backoff
      const delaySeconds = this.FIBONACCI_DELAYS[retryCount];

      // Re-publier avec delay
      await this.retryWithDelay(msg, retryCount + 1, delaySeconds);

      // ACK message original (sera re-traité via republication)
      channel.ack(originalMsg);
    }
  }

  private async retryWithDelay(
    msg: MessageWithRetry,
    newRetryCount: number,
    delaySeconds: number
  ) {
    await this.rabbitService.publish(
      'digestion.queue',
      msg.payload,
      {
        headers: {
          'x-retry-count': newRetryCount,
          'x-first-attempt': msg.headers['x-first-attempt'],
          'x-last-attempt': Date.now(),
        },
        // Delayed message exchange plugin
        delay: delaySeconds * 1000,
      }
    );
  }

  private isRetryableError(error: Error): boolean {
    // Erreurs temporaires = retryable
    const retryableErrors = [
      'ECONNREFUSED',     // LLM API down
      'ETIMEDOUT',        // Timeout réseau
      'ENOTFOUND',        // DNS temporaire
      '429',              // Rate limit LLM
      '503',              // Service unavailable
    ];

    return retryableErrors.some(code =>
      error.message.includes(code) || error.name.includes(code)
    );
  }
}
```

**RabbitMQ Delayed Message Exchange :**

```bash
# Installation plugin
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# Configuration exchange
{
  "name": "delayed_exchange",
  "type": "x-delayed-message",
  "durable": true,
  "arguments": {
    "x-delayed-type": "direct"
  }
}
```

**Rationale :**
- Fibonacci backoff : progression douce (1s, 1s, 2s, 3s, 5s, 8s)
- Max 5 retries : évite boucle infinie
- Erreurs retryables vs permanentes : stratégie différenciée
- Delayed exchange : évite polling (natif RabbitMQ)

---

## 12.3 - Queue Prioritization

**Decision:** Queues séparées avec consommateurs prioritaires

**Architecture multi-queues :**

```typescript
// Queues par priorité métier
const QUEUES = {
  // CRITICAL : impact user immédiat
  transcription: {
    name: 'transcription.queue',
    priority: 'critical',
    consumers: 3,  // 3 workers dédiés
  },

  // HIGH : expérience user
  digestion: {
    name: 'digestion.queue',
    priority: 'high',
    consumers: 2,
  },

  // MEDIUM : background
  concordance: {
    name: 'concordance.queue',
    priority: 'medium',
    consumers: 1,
  },

  // LOW : batch
  analytics: {
    name: 'analytics.queue',
    priority: 'low',
    consumers: 1,
  },
};
```

**Consumer avec scaling dynamique :**

```typescript
// NestJS worker scalable
@Module({
  imports: [
    RabbitMQModule.forRoot({
      exchanges: [{ name: 'pensine', type: 'topic' }],
      uri: process.env.RABBITMQ_URI,
      connectionInitOptions: { wait: false },
    }),
  ],
})
class WorkersModule implements OnModuleInit {
  constructor(private rabbitService: AmqpConnection) {}

  onModuleInit() {
    // Spawn consumers selon config
    for (const [name, config] of Object.entries(QUEUES)) {
      for (let i = 0; i < config.consumers; i++) {
        this.spawnConsumer(name, i);
      }
    }
  }

  private spawnConsumer(queueName: string, workerId: number) {
    this.logger.log(`Spawning consumer ${queueName}:${workerId}`);
    // Consumer s'enregistre automatiquement via @RabbitSubscribe
  }
}
```

**Metrics & Auto-scaling (Post-MVP) :**

```typescript
// Monitorer queue depth
@Cron('*/1 * * * *')  // Toutes les minutes
async monitorQueueDepth() {
  const stats = await this.rabbitService.getQueueStats('digestion.queue');

  if (stats.messages > 100) {
    // Queue saturée → augmenter consumers
    await this.scalingService.scaleUp('digestion', targetConsumers: 4);
  }

  if (stats.messages < 10 && stats.consumers > 2) {
    // Queue vide → réduire consumers
    await this.scalingService.scaleDown('digestion', targetConsumers: 2);
  }
}
```

**Rationale :**
- Queues séparées : isolation pannes (transcription down ≠ digestion bloquée)
- Consumers dédiés : garantie traitement prioritaire
- Scaling par queue : optimisation ressources
- MVP : consumers fixes, Post-MVP : auto-scaling

---

## 12.4 - Monitoring & Alerting

**Decision:** Métriques RabbitMQ + alertes proactives

**Métriques RabbitMQ à tracker :**

```typescript
interface QueueMetrics {
  // Volume
  messages: number;           // Messages en attente
  messagesReady: number;      // Prêts à consommer
  messagesUnacked: number;    // En cours de traitement

  // Performance
  publishRate: number;        // Msgs/sec publiés
  consumeRate: number;        // Msgs/sec consommés
  ackRate: number;            // Msgs/sec acknowledgés

  // Consumers
  consumers: number;          // Consumers actifs
  consumerUtilisation: number; // % utilisation

  // Durée
  avgProcessingTime: number;  // Temps moyen traitement
}
```

**Collecte métriques (Prometheus) :**

```typescript
// NestJS Prometheus exporter
@Injectable()
class RabbitMetricsCollector {
  private readonly gauges = {
    queueDepth: new Gauge({
      name: 'rabbitmq_queue_messages',
      help: 'Messages in queue',
      labelNames: ['queue'],
    }),
    consumers: new Gauge({
      name: 'rabbitmq_queue_consumers',
      help: 'Active consumers',
      labelNames: ['queue'],
    }),
  };

  @Cron('*/30 * * * * *')  // Toutes les 30s
  async collectMetrics() {
    for (const queueName of Object.keys(QUEUES)) {
      const stats = await this.rabbitService.getQueueStats(queueName);

      this.gauges.queueDepth.set({ queue: queueName }, stats.messages);
      this.gauges.consumers.set({ queue: queueName }, stats.consumers);
    }
  }
}
```

**Alertes (critères) :**

```typescript
const ALERTS = {
  queueSaturated: {
    condition: 'queue_depth > 500',
    severity: 'warning',
    message: 'Queue saturée, scaling nécessaire',
  },

  consumerDown: {
    condition: 'consumers == 0 && queue_depth > 0',
    severity: 'critical',
    message: 'Aucun consumer actif',
  },

  dlqNotEmpty: {
    condition: 'dlq_depth > 0',
    severity: 'warning',
    message: 'Messages en DLQ nécessitent investigation',
  },

  slowProcessing: {
    condition: 'avg_processing_time > 60000',  // 60s
    severity: 'warning',
    message: 'Traitement lent détecté',
  },
};
```

**Dashboard Grafana (KPIs) :**

```
📊 RabbitMQ Dashboard

┌─────────────────────────────────────────┐
│ Queue Depth (real-time)                 │
│ ▂▃▅▇█▇▅▃▂ Digestion: 23 msgs           │
│ ▁▁▂▃▂▁▁▁▁ Transcription: 5 msgs        │
│ ▁▁▁▁▁▁▁▁▁ Concordance: 0 msgs          │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Throughput (msgs/min)                   │
│ Published: 120 msg/min                  │
│ Consumed: 115 msg/min                   │
│ DLQ: 2 msg/min ⚠️                      │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Processing Time (avg)                   │
│ Digestion: 12.3s                        │
│ Transcription: 45s                      │
│ Concordance: 3.2s                       │
└─────────────────────────────────────────┘
```

**Rationale :**
- Prometheus : métriques time-series standard
- Grafana : visualisation temps réel
- Alertes proactives : détection avant incident
- KPIs métier : digestion < 30s, transcription < 2x durée

---

## Conséquences Globales ADR-012

**Bénéfices:**
- Résilience : DLQ + retry évitent perte messages
- Performance : queues prioritaires + scaling
- Observabilité : métriques + alertes proactives
- Maintenance : isolation pannes par queue
- Coût : RabbitMQ léger (< 512 MB RAM pour MVP)

**Trade-offs acceptés:**
- Complexité infrastructure : RabbitMQ + monitoring
- Overhead réseau : messages retry
- Latence retry : Fibonacci backoff (max 8s avant DLQ)

**Impact:**
- **Epic 3** : Transcription queue (Story 3.1-3.2)
- **Epic 4** : Digestion queue (Story 4.1-4.3)
- **Epic 5** : Concordance queue (Story 5.1-5.3)
- **Infrastructure** : RabbitMQ + Prometheus + Grafana

---

## Implementation Status

- ⏳ **Epic 3** : Transcription queue
- ⏳ **Epic 4** : Digestion queue
- ⏳ **Epic 5** : Concordance queue
- ⏳ **Infrastructure** : RabbitMQ setup
- ⏳ **Post-MVP** : Auto-scaling consumers

---

## References

- RabbitMQ Documentation: https://www.rabbitmq.com/documentation.html
- Dead Letter Queues: https://www.rabbitmq.com/dlx.html
- Delayed Message Plugin: https://github.com/rabbitmq/rabbitmq-delayed-message-exchange
- NestJS RabbitMQ: https://docs.nestjs.com/microservices/rabbitmq
- Prometheus Client: https://github.com/siimon/prom-client

---

## Validation Criteria

ADR considéré succès SI :
- ⏳ DLQ fonctionne (messages échoués récupérables)
- ⏳ Retry logic fonctionne (max 5 attempts)
- ⏳ Queues prioritaires respectées (transcription > digestion > concordance)
- ⏳ Alertes Grafana opérationnelles
- ⏳ 0 perte de messages en production (1 mois monitoring)

**Review Date :** 2026-03 (après Epic 5)

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

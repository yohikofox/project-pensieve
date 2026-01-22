---
adr: ADR-009
title: "Stratégie de Synchronisation Mobile ↔ Backend"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-009: Stratégie de Synchronisation Mobile ↔ Backend

**Status:** ✅ ACCEPTÉ

**Context:** Définir comment la synchronisation offline-first fonctionne entre mobile et backend.

**Decision:** 6 décisions validées pour la stratégie de sync complète.

---

## 9.1 - Timing de Synchronisation

**Decision:** Option C - Balanced (launch + post-action + polling 15min)

**Triggers de sync:**
1. **Au lancement app** (priorité haute)
2. **Après actions critiques** (capture, digestion completed, todo completed)
3. **Polling background 15min** (si app ouverte)
4. **Retour réseau** (après période offline)

**Rationale:**
- Balance entre réactivité et consommation batterie
- Données critiques (captures) synchronisées rapidement
- Polling 15min raisonnable pour updates non-critiques
- Pas de sync agressive permanente

**Implémentation:**
```typescript
// 1. Au lancement
useEffect(() => {
  syncService.sync({ priority: 'high' });
}, []);

// 2. Après action critique
onCaptureComplete(() => {
  syncService.sync({ priority: 'high', entity: 'captures' });
});

// 3. Polling background
setInterval(() => {
  if (appState === 'active') {
    syncService.sync({ priority: 'low' });
  }
}, 15 * 60 * 1000);

// 4. Retour réseau
NetInfo.addEventListener(state => {
  if (state.isConnected) {
    syncService.sync({ priority: 'medium' });
  }
});
```

---

## 9.2 - Détection et Résolution de Conflits

**Decision:** Utiliser ~~le mécanisme standard de WatermelonDB~~ le pattern `lastPulledAt` + `last_modified` per-record.

⚠️ **UPDATE (2026-01-22):** OP-SQLite remplace WatermelonDB. Pattern `lastPulledAt` + `last_modified` reste valide - implémentation manuelle Epic 6. Voir [ADR-018](./ADR-018-migration-watermelondb-opsqlite.md)

**Comment ça fonctionne:**

```typescript
// 1. PULL PHASE (Backend → Mobile)
GET /sync/pull?last_pulled_at=2026-01-13T09:00:00Z

Backend retourne:
{
  changes: {
    captures: {
      updated: [
        { id: "c1", last_modified: 1736759400000, ... }  // Modifié après lastPulledAt
      ]
    }
  },
  timestamp: 1736760600000  // Nouveau lastPulledAt pour le client
}

Mobile enregistre: lastPulledAt = 1736760600000

// 2. PUSH PHASE (Mobile → Backend)
POST /sync/push
{
  last_pulled_at: 1736760600000,
  changes: {
    captures: {
      updated: [
        { id: "c1", ... }
      ]
    }
  }
}

Backend vérifie pour CHAQUE record:
IF server.last_modified > request.last_pulled_at
THEN → CONFLIT (donnée modifiée côté serveur depuis dernier pull)
ELSE → OK (accepter push)
```

**Résolution des conflits détectés:**

```typescript
// Stratégie per-column client-wins avec logique métier
class SyncConflictResolver {
  resolve(serverRecord, clientRecord, entity) {
    switch(entity) {
      case 'capture':
        // Métadonnées techniques: serveur gagne
        return {
          ...clientRecord,
          normalized_text: serverRecord.normalized_text,  // Serveur
          state: serverRecord.state,                      // Serveur

          // Données user: client gagne
          tags: clientRecord.tags,                        // Client
          projectId: clientRecord.projectId               // Client
        };

      case 'todo':
        // État métier: client gagne (user a agi localement)
        return {
          ...serverRecord,
          state: clientRecord.state,                      // Client
          completed_at: clientRecord.completed_at,        // Client

          // Métadonnées: serveur gagne
          priority: serverRecord.priority                 // Serveur (calculé par IA)
        };

      default:
        return clientRecord;  // Client-wins par défaut
    }
  }
}
```

**Gestion multi-clients:**

Chaque client a son propre `lastPulledAt`, le système fonctionne correctement:

```
Timeline:
10:00 - Record créé (last_modified = 10:00)
10:15 - MobileA pull (lastPulledAt_A = 10:15)
10:30 - MobileB pull (lastPulledAt_B = 10:30)
10:31 - MobileA modifie + push
        → Backend: last_modified = 10:31
10:32 - MobileB modifie + push
        → Backend détecte: server.last_modified (10:31) > client.lastPulledAt_B (10:30)
        → CONFLIT correctement détecté
```

**Support backoffice & API externes:**

```typescript
// Modification directe en DB (backoffice, cron job, API externe)
UPDATE captures SET normalized_text = '...', last_modified = NOW();

// Prochain pull mobile détecte automatiquement:
GET /sync/pull?last_pulled_at=2026-01-13T10:00:00Z
→ Backend retourne record avec last_modified > lastPulledAt
→ Mobile applique update correctement
```

**Rationale:**
- Mécanisme éprouvé ~~de WatermelonDB~~ pattern standard
- `lastPulledAt` client-specific permet multi-clients
- `last_modified` server-side source de vérité
- Fonctionne avec backoffice et modifications externes
- Per-column resolution permet business logic fine

---

## 9.3 - Gestion des Fichiers (Audio)

**Decision:** Option C - Upload Queue séparée

**Architecture:**

```typescript
// 1. Capture audio sauvegardée localement immédiatement
const capture = await captureService.create({
  type: 'audio',
  audioUri: 'file://local/audio123.m4a',  // Local file
  state: 'captured'
});

// 2. Ajout à upload queue
await uploadQueue.enqueue({
  captureId: capture.id,
  fileUri: capture.audioUri,
  priority: 'high'
});

// 3. Upload asynchrone indépendant du sync
uploadQueue.process({
  onSuccess: (captureId, remoteUrl) => {
    // Update capture avec URL remote
    db.captures.update(captureId, {
      audioRemoteUrl: remoteUrl,
      uploadState: 'completed'
    });
  },
  onError: (captureId, error) => {
    // Retry avec backoff
    uploadQueue.retry(captureId, { delay: fibonacci(attemptCount) });
  }
});

// 4. Sync metadata séparément (pas bloqué par upload)
syncService.sync();  // Sync capture metadata même si upload en cours
```

**Retry Upload Strategy:**

```typescript
class UploadQueue {
  async retry(item, options) {
    const delays = [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]; // Fibonacci en secondes
    const maxDelay = 5 * 60;  // Cap à 5 minutes

    const delay = Math.min(
      delays[item.attemptCount] * 1000,
      maxDelay * 1000
    );

    await sleep(delay);
    return this.upload(item);
  }
}
```

**Rationale:**
- Upload fichiers lents/volumineux ne bloque pas sync metadata
- Retry indépendant avec backoff
- Capture utilisable immédiatement (fichier local)
- Transcription Whisper fonctionne sur fichier local
- Upload asynchrone en arrière-plan

**Conséquences:**
- Complexité légèrement augmentée (2 systèmes)
- Gestion états: `pending`, `uploading`, `completed`, `failed`
- Cleanup fichiers locaux après upload réussi

---

## 9.4 - Priorité de Synchronisation

**Decision:** Option B - Priority-based (Captures > Todos > Ideas > Projects)

**Ordre de sync:**

```typescript
const SYNC_PRIORITY = {
  captures: 1,     // HIGHEST - données critiques user
  todos: 2,        // HIGH - actions utilisateur
  thoughts: 3,     // MEDIUM - résultats digestion
  ideas: 4,        // LOW - concordances peuvent attendre
  projects: 5      // LOWEST - suggestions peuvent attendre
};

async function sync() {
  const entities = Object.keys(SYNC_PRIORITY)
    .sort((a, b) => SYNC_PRIORITY[a] - SYNC_PRIORITY[b]);

  for (const entity of entities) {
    await syncEntity(entity);
  }
}
```

**Sync incrémental si timeout:**

```typescript
// Si connexion lente ou timeout, sync par batch prioritaires
async function syncWithTimeout(timeoutMs = 10000) {
  const startTime = Date.now();

  for (const entity of prioritizedEntities) {
    if (Date.now() - startTime > timeoutMs) {
      console.log(`Timeout: synced up to ${entity}`);
      break;  // Reste sera synced au prochain trigger
    }

    await syncEntity(entity);
  }
}
```

**Rationale:**
- Captures = données critiques (NFR6: "0 capture perdue")
- Todos = actions user importantes
- Ideas/Projects = peuvent attendre (non-bloquant)
- Connexion lente ou timeout ne perd pas données critiques

---

## 9.5 - Retry Logic & Error Handling

**Decision:** Result Pattern + Fibonacci Backoff (cap 5min)

**Result Pattern (pas try/catch):**

```typescript
// Enum de résultats
enum SyncResult {
  SUCCESS = 'success',
  NETWORK_ERROR = 'network_error',
  AUTH_ERROR = 'auth_error',
  CONFLICT = 'conflict',
  SERVER_ERROR = 'server_error',
  TIMEOUT = 'timeout'
}

type SyncResponse = {
  result: SyncResult;
  data?: any;
  error?: string;
  retryable: boolean;
};

// Usage
const response = await syncService.sync();

switch (response.result) {
  case SyncResult.SUCCESS:
    showToast('Sync réussie');
    break;

  case SyncResult.NETWORK_ERROR:
    if (response.retryable) {
      scheduleRetry({ delay: fibonacci(attemptCount) });
    }
    break;

  case SyncResult.AUTH_ERROR:
    redirectToLogin();  // Non retryable
    break;

  case SyncResult.CONFLICT:
    resolveConflict(response.data);
    break;
}
```

**Fibonacci Backoff:**

```typescript
const fibonacciDelays = [1, 1, 2, 3, 5, 8, 13, 21, 34, 55];  // Secondes
const maxDelay = 5 * 60;  // Cap 5 minutes

function getRetryDelay(attemptCount: number): number {
  const fibDelay = fibonacciDelays[Math.min(attemptCount, fibonacciDelays.length - 1)];
  return Math.min(fibDelay, maxDelay) * 1000;  // En millisecondes
}

// Exemple timeline:
// Attempt 1: 1s
// Attempt 2: 1s
// Attempt 3: 2s
// Attempt 4: 3s
// Attempt 5: 5s
// Attempt 6: 8s
// Attempt 7: 13s
// Attempt 8: 21s
// Attempt 9: 34s
// Attempt 10: 55s
// Attempt 11+: 5min (capped)
```

**Rationale (Fibonacci > Exponential):**

```
Fibonacci: 1, 1, 2, 3, 5, 8, 13, 21, 34, 55 → 5min cap
Exponential (base 2): 2, 4, 8, 16, 32, 64, 128, 256 → trop agressif

Avantages Fibonacci:
- Bagottage réseau temporaire: récupération rapide (1s, 1s, 2s)
- Backend down prolongé: monte progressivement sans exploser
- 5min cap évite attente infinie
- Soulage backend lors recovery (pas de stampede avec exponential)
```

**Rationale Result Pattern > try/catch:**
- Code appelant peut interpréter enum facilement
- Switch exhaustif (TypeScript vérifie tous les cas)
- Retry logic centralisée et claire
- Pas de exceptions non catchées qui crashent l'app

---

## 9.6 - Schema Versioning

**Decision:** Simple versioning (pas de Schema Registry pour MVP)

**Approche:**

```typescript
// Mobile envoie version de schema dans headers
POST /sync/push
Headers:
  X-Schema-Version: 1.0.0

// Backend valide compatibilité
if (requestSchemaVersion !== serverSchemaVersion) {
  if (isCompatible(requestSchemaVersion, serverSchemaVersion)) {
    // OK - backward compatible
    applyMigration(requestSchemaVersion);
  } else {
    // KO - force app update
    return { error: 'APP_UPDATE_REQUIRED' };
  }
}
```

**Migration strategy:**

```typescript
// Changement de schéma (ex: ajout colonne)
// V1.0.0 → V1.1.0

// Backend supporte les 2 versions:
function handlePush(data, schemaVersion) {
  switch (schemaVersion) {
    case '1.0.0':
      // Ancienne version - remplir nouvelle colonne avec default
      return {
        ...data,
        newColumn: 'default_value'
      };

    case '1.1.0':
      // Nouvelle version - utiliser directement
      return data;
  }
}
```

**Rationale:**
- MVP mono-user: pas besoin Schema Registry complexe
- Header version suffit pour validation
- Backend peut supporter N-1 versions facilement
- Force update si breaking change critique
- Post-MVP: migrer vers Schema Registry si multi-versions critiques

**Conséquences:**
- Simple à implémenter
- Backend doit gérer migrations manuellement
- Documentation versions obligatoire
- Si scaling: ajouter Schema Registry (Kafka Schema Registry, etc.)

---

## Conséquences Globales ADR-009

**Bénéfices:**
- Sync robuste et performante
- 0 perte de données garantie (NFR6)
- Multi-clients supporté nativement
- Backoffice compatible
- Code maintenable avec Result Pattern

**Trade-offs acceptés:**
- Complexité accrue (2 systèmes: sync metadata + upload files)
- Backend doit gérer conflits et migrations
- Testing complexe (scénarios multi-clients, offline)

**Impact:**
- **Epic 6 (Sync)** : Implémentation complète de cette stratégie
- **Story 2.1-2.6** : Offline-first validé
- **Infrastructure** : Queue système séparé pour uploads

---

## Implementation Status

- ⏳ **Epic 6 (Q1 2026)** : Implémentation sync protocol complet
- ✅ **Story 2.1** : Offline-first storage validé
- ⏳ **Story 6.2-6.3** : Sync bidirectionnel
- ⏳ **Post-MVP** : Schema Registry si nécessaire

---

## References

- WatermelonDB Sync Protocol: https://watermelondb.dev/docs/Sync/Intro
- ⚠️ **UPDATE:** OP-SQLite remplace WatermelonDB - voir ADR-018
- Conflict-free Replicated Data Types (CRDTs): https://crdt.tech/
- Result Pattern TypeScript: https://www.matthewgerstman.com/tech/typescript-result-pattern/

---

## Validation Criteria

ADR considéré succès SI :
- ⏳ Sync bidirectionnel fonctionne (Epic 6)
- ⏳ 0 perte de données en production (1 mois monitoring)
- ⏳ Conflits résolus correctement (tests multi-clients)
- ⏳ Performance < 10s sync complète (NFR)

**Review Date :** 2026-03 (après Epic 6)

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

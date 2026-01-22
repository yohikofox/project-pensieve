---
adr: ADR-011
title: "Performance Optimization Strategy"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-011: Performance Optimization Strategy

**Status:** ✅ ACCEPTÉ

**Context:** Définir les stratégies d'optimisation pour respecter les NFRs de performance critiques (capture < 500ms, transcription < 2x durée audio, digestion < 30s).

**Decision:** 3 décisions validées pour optimisation globale.

---

## 11.1 - Redis Caching Strategy

**Decision:** Cache stratégique ciblé, pas de cache agressif

**Usage de Redis :**

```typescript
// 1. Session cache (JWT payload décodé)
const sessionKey = `session:${userId}`;
await redis.setex(sessionKey, 900, JSON.stringify(sessionData));  // 15min TTL

// 2. User profile cache (fréquemment accédé)
const userKey = `user:${userId}`;
await redis.setex(userKey, 3600, JSON.stringify(userProfile));  // 1h TTL

// 3. Digestion results cache (temps réel récent)
const digestKey = `digest:${captureId}`;
await redis.setex(digestKey, 1800, JSON.stringify(digestResult));  // 30min TTL

// 4. Concordance scores (calculs coûteux)
const concordanceKey = `concordance:${ideaId}`;
await redis.setex(concordanceKey, 3600, JSON.stringify(scores));  // 1h TTL
```

**Ce qui N'est PAS caché :**
- ❌ Captures (changent fréquemment, mobile offline-first a déjà le cache)
- ❌ Todos (état utilisateur volatile)
- ❌ Ideas/Projects (concordance dynamique)

**Invalidation :**

```typescript
// Invalidation ciblée (pas de flush global)
class CacheService {
  async invalidateUser(userId: string) {
    await redis.del(`user:${userId}`);
    await redis.del(`session:${userId}`);
  }

  async invalidateDigest(captureId: string) {
    await redis.del(`digest:${captureId}`);
  }

  // Pattern-based invalidation (attention performance)
  async invalidateConcordanceForIdea(ideaId: string) {
    const keys = await redis.keys(`concordance:*${ideaId}*`);
    if (keys.length > 0) {
      await redis.del(...keys);
    }
  }
}
```

**Rationale :**
- Cache UNIQUEMENT les hot paths (session, profile)
- TTL courts pour éviter stale data
- Mobile a déjà cache complet (~~WatermelonDB~~ OP-SQLite local)
- ⚠️ **UPDATE (2026-01-22):** OP-SQLite remplace WatermelonDB. Voir [ADR-018](./ADR-018-migration-watermelondb-opsqlite.md)
- Redis = cache serveur uniquement
- MVP mono-user = moins de pression cache

**Metrics à monitorer :**
- Cache hit rate (objectif > 80% pour session/user)
- Latence avec/sans cache
- Memory usage Redis

---

## 11.2 - Lazy Loading Strategies

**Decision:** Lazy loading ciblé UI + data, pagination backend

**Mobile (UI Lazy Loading) :**

```typescript
// 1. Liste captures avec virtualisation
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={captures}
  renderItem={({ item }) => <CaptureCard capture={item} />}
  estimatedItemSize={120}
  // Virtualization automatique (ne render que visible)
/>

// 2. Images lazy avec blurhash
<Image
  source={{ uri: capture.thumbnailUrl }}
  placeholder={{ blurhash: capture.blurhash }}
  transition={200}
/>

// 3. Audio player lazy (chargé au tap)
const AudioPlayer = lazy(() => import('./AudioPlayer'));

<Suspense fallback={<AudioPlayerSkeleton />}>
  {isPlaying && <AudioPlayer uri={audioUri} />}
</Suspense>

// 4. Tabs lazy (pas de pré-chargement)
// Tab Actions chargé seulement quand user tape dessus
<Tab.Navigator lazy={true}>
  <Tab.Screen name="Feed" component={FeedScreen} />
  <Tab.Screen name="Actions" component={ActionsScreen} />  {/* Lazy */}
  <Tab.Screen name="Projects" component={ProjectsScreen} />  {/* Lazy */}
</Tab.Navigator>
```

**Backend (Data Pagination) :**

```typescript
// API avec cursor-based pagination
GET /captures?cursor=c123&limit=20

// Query optimisé avec index
const captures = await db.captures
  .where('user_id', userId)
  .where('created_at', '<', cursorDate)
  .orderBy('created_at', 'desc')
  .limit(20);

Response:
{
  data: [...],
  nextCursor: 'c143',
  hasMore: true
}

// Mobile incremental load
class CaptureList {
  async loadMore() {
    if (this.isLoading || !this.hasMore) return;

    const response = await api.getCaptures({
      cursor: this.nextCursor,
      limit: 20
    });

    this.captures.push(...response.data);
    this.nextCursor = response.nextCursor;
    this.hasMore = response.hasMore;
  }
}
```

**~~WatermelonDB~~ OP-SQLite Lazy Queries :**

⚠️ **UPDATE (2026-01-22):** OP-SQLite remplace WatermelonDB. Pattern LIMIT/OFFSET reste valide - observables manuels nécessaires. Voir [ADR-018](./ADR-018-migration-watermelondb-opsqlite.md)

```typescript
// OP-SQLite: Ne charge QUE ce qui est affiché (LIMIT/OFFSET SQL)
const visibleCaptures = await db.execute(
  `SELECT * FROM captures
   WHERE created_at >= ?
   ORDER BY created_at DESC
   LIMIT 50`,
  [last30Days]
);

// Observer pattern manuel (React state + useEffect)
// Expand si user scroll (OFFSET)
```

**Rationale :**
- Capture < 500ms : pas de chargement lourd au lancement
- FlashList : performance lists longues (1000+ items)
- Pagination backend : évite OOM si 10k+ captures
- Tabs lazy : économie mémoire
- Blurhash : perception vitesse (placeholder immédiat)

---

## 11.3 - Audio Compression & Formats

**Decision:** AAC-LC compression, qualité adaptative, nettoyage automatique

**Format Audio :**

```typescript
// Recording config (Expo AV)
const recordingOptions = {
  android: {
    extension: '.m4a',
    outputFormat: Audio.AndroidOutputFormat.MPEG_4,
    audioEncoder: Audio.AndroidAudioEncoder.AAC,
    sampleRate: 44100,
    numberOfChannels: 1,  // Mono (voix)
    bitRate: 64000,       // 64 kbps (balance qualité/taille)
  },
  ios: {
    extension: '.m4a',
    outputFormat: Audio.IOSOutputFormat.MPEG4AAC,
    audioQuality: Audio.IOSAudioQuality.MEDIUM,
    sampleRate: 44100,
    numberOfChannels: 1,
    bitRate: 64000,
    linearPCMBitDepth: 16,
    linearPCMIsBigEndian: false,
    linearPCMIsFloat: false,
  },
  web: {
    mimeType: 'audio/webm;codecs=opus',
    bitsPerSecond: 64000,
  }
};
```

**Tailles Attendues :**

```
Durée    | Taille (64 kbps mono)
---------|----------------------
30s      | ~240 KB
1 min    | ~480 KB
2 min    | ~960 KB
5 min    | ~2.4 MB

5 captures/jour × 1 min × 30 jours = ~72 MB/mois
```

**Compression Serveur (Post-Upload) :**

```typescript
// Optionnel : re-compression serveur pour stockage long terme
class AudioProcessor {
  async compressForStorage(audioFile: Buffer): Promise<Buffer> {
    // FFmpeg : AAC 32 kbps pour stockage (qualité suffisante)
    return ffmpeg(audioFile)
      .audioCodec('aac')
      .audioBitrate('32k')
      .audioChannels(1)
      .toBuffer();
  }
}

// Réduction: 64 kbps → 32 kbps = 50% économie stockage
```

**Nettoyage Local (Mobile) :**

```typescript
// Cleanup audio local après upload réussi + X jours
class StorageManager {
  async cleanupOldAudio() {
    const threshold = Date.now() - (30 * 24 * 60 * 60 * 1000);  // 30 jours

    const oldCaptures = await db.execute(
      `SELECT * FROM captures
       WHERE upload_state = 'completed'
       AND captured_at < ?`,
      [threshold]
    );

    for (const capture of oldCaptures) {
      // Supprimer fichier local
      await FileSystem.deleteAsync(capture.audioLocalUri, { idempotent: true });

      // Garder metadata + URL remote
      await db.execute(
        'UPDATE captures SET audio_local_uri = NULL WHERE id = ?',
        [capture.id]
      );
    }
  }
}

// Cron: tous les jours à 3h du matin
```

**Streaming (Post-MVP) :**

```typescript
// Si fichier pas en local, stream depuis serveur
async playAudio(capture: Capture) {
  const uri = capture.audioLocalUri
    ? capture.audioLocalUri           // Local (rapide)
    : capture.audioRemoteUrl;         // Stream serveur (si cleanup)

  await Audio.Sound.createAsync(
    { uri },
    { shouldPlay: true }
  );
}
```

**Rationale :**
- AAC-LC : meilleur ratio qualité/taille pour voix
- 64 kbps mono : qualité suffisante pour transcription Whisper
- Cleanup 30j : économie stockage mobile (128 GB = limité)
- Format M4A : compatible iOS/Android/Web
- Mono channel : voix uniquement (pas besoin stéréo)

**Whisper Performance :**
- Qualité audio n'impacte pas beaucoup précision transcription
- 64 kbps AAC > suffisant pour Whisper
- Compression réduit aussi temps upload

---

## Conséquences Globales ADR-011

**Bénéfices:**
- NFRs performance respectées (capture < 500ms, chargement < 1s)
- Économie bande passante : ~50% (compression audio)
- Économie stockage mobile : cleanup automatique
- UX fluide : lazy loading + virtualisation
- Cache ciblé : pas de over-caching

**Trade-offs acceptés:**
- Complexité : cleanup jobs + cache invalidation
- Latence streaming : si fichier supprimé localement (acceptable)
- Monitoring : cache hit rate, performance queries

**Impact:**
- **Story 2.1** : Compression audio AAC 64 kbps
- **Epic 3** : FlashList + lazy loading feed
- **Infrastructure** : Redis cache backend
- **Post-MVP** : Cleanup automatique stockage

---

## Implementation Status

- ✅ **Story 2.1** : Compression audio AAC 64 kbps implémenté
- ⏳ **Epic 3** : FlashList + lazy loading
- ✅ **Infrastructure** : Redis configuré
- ⏳ **Post-MVP** : Cleanup automatique audio

---

## References

- FlashList: https://shopify.github.io/flash-list/
- AAC Codec: https://en.wikipedia.org/wiki/Advanced_Audio_Coding
- Redis Best Practices: https://redis.io/docs/manual/patterns/
- Expo Audio: https://docs.expo.dev/versions/latest/sdk/audio/
- Blurhash: https://blurha.sh/

---

## Validation Criteria

ADR considéré succès SI :
- ✅ Capture < 500ms latency (Story 2.1)
- ⏳ Cache hit rate > 80% session/user
- ⏳ Liste 1000+ captures sans lag (Epic 3)
- ⏳ Stockage mobile < 500 MB après 6 mois usage

**Review Date :** 2026-03 (après Epic 3)

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

---
adr: ADR-014
title: "Storage Management"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-014: Storage Management

**Status:** ✅ ACCEPTÉ

**Context:** Gérer le stockage audio (mobile + cloud) avec retention policy et cleanup automatique.

**Decision:** 3 décisions pour optimisation stockage

---

## 14.1 - Audio Retention Policy

**Decision:** Mobile 30 jours, Cloud permanent (avec compression)

**Lifecycle Audio :**

```
Capture → [Mobile 64kbps] → Upload → [Cloud 32kbps compressed]
   ↓                              ↓
Cleanup après 30j          Permanent (tant que compte actif)
```

**Mobile Cleanup :**

```typescript
class StorageManager {
  // Cron quotidien à 3h du matin
  @Cron('0 3 * * *')
  async cleanupOldAudio() {
    const threshold = Date.now() - (30 * 24 * 60 * 60 * 1000);  // 30 jours

    const toCleanup = await db.execute(
      `SELECT * FROM captures
       WHERE upload_state = 'completed'
       AND captured_at < ?
       AND audio_local_uri IS NOT NULL`,
      [threshold]
    );

    let freedSpace = 0;

    for (const capture of toCleanup) {
      try {
        // Récupérer taille fichier avant suppression
        const fileInfo = await FileSystem.getInfoAsync(capture.audioLocalUri);
        if (fileInfo.exists) {
          freedSpace += fileInfo.size;

          // Supprimer fichier
          await FileSystem.deleteAsync(capture.audioLocalUri, { idempotent: true });

          // Mettre à jour record (garder metadata)
          await db.execute(
            'UPDATE captures SET audio_local_uri = NULL WHERE id = ?',
            [capture.id]
          );
        }
      } catch (error) {
        this.logger.error('Cleanup failed', { captureId: capture.id, error });
      }
    }

    this.logger.info(`Cleanup freed ${(freedSpace / 1024 / 1024).toFixed(2)} MB`);
  }

  // User peut déclencher cleanup manuel
  async cleanupManual() {
    // Même logique mais avec confirmation user
  }
}
```

**Cloud Permanent avec Compression :**

```typescript
// Backend : compression post-upload
class AudioStorageService {
  async handleAudioUpload(file: Express.Multer.File, captureId: string) {
    // 1. Upload original temporaire
    const originalKey = `audio/temp/${captureId}.m4a`;
    await this.s3.upload(originalKey, file.buffer);

    // 2. Compression FFmpeg (64kbps → 32kbps)
    const compressed = await this.compressAudio(file.buffer);

    // 3. Upload compressed permanent
    const permanentKey = `audio/${captureId}.m4a`;
    await this.s3.upload(permanentKey, compressed);

    // 4. Supprimer original
    await this.s3.delete(originalKey);

    // 5. Retourner URL permanent
    return this.s3.getSignedUrl(permanentKey);
  }

  private async compressAudio(buffer: Buffer): Promise<Buffer> {
    return new Promise((resolve, reject) => {
      ffmpeg(buffer)
        .audioCodec('aac')
        .audioBitrate('32k')
        .audioChannels(1)
        .on('end', () => resolve(outputBuffer))
        .on('error', reject)
        .run();
    });
  }
}
```

**Rationale :**
- 30 jours mobile : balance stockage/re-écoute (user peut re-écouter récent)
- Cloud permanent : aucune perte de données, compliance RGPD
- Compression cloud : économie 50% stockage (64kbps → 32kbps)

---

## 14.2 - Quotas Utilisateur

**Decision:** Pas de limite MVP, monitoring usage, quotas Post-MVP si abuse

**Monitoring Usage :**

```typescript
// Backend : tracker usage par user
interface UserStorageStats {
  userId: string;
  capturesCount: number;
  totalAudioSizeMB: number;
  averageCaptureSizeMB: number;
  oldestCapture: Date;
  newestCapture: Date;
}

class StorageAnalytics {
  @Cron('0 0 * * 0')  // Hebdomadaire (dimanche minuit)
  async computeStorageStats() {
    const users = await this.userService.findAll();

    for (const user of users) {
      const stats = await this.computeUserStats(user.id);

      // Alerter si usage anormal
      if (stats.totalAudioSizeMB > 5000) {  // > 5 GB
        await this.alertService.send({
          severity: 'info',
          message: `User ${user.id} has ${stats.totalAudioSizeMB} MB audio`,
          userId: user.id,
        });
      }

      // Stocker stats pour analytics
      await this.analyticsService.save(stats);
    }
  }
}
```

**Quotas Post-MVP (si nécessaire) :**

```typescript
const QUOTAS = {
  free: {
    maxCaptures: null,        // Illimité
    maxAudioSizeMB: 1000,     // 1 GB
    maxAudioDurationMin: null, // Illimité
  },

  premium: {
    maxCaptures: null,
    maxAudioSizeMB: 10000,    // 10 GB
    maxAudioDurationMin: null,
  },
};

class QuotaService {
  async checkQuota(userId: string, audioSizeMB: number): Promise<boolean> {
    const user = await this.userService.findById(userId);
    const quota = QUOTAS[user.plan];

    const currentUsage = await this.getCurrentUsage(userId);

    if (currentUsage.totalAudioSizeMB + audioSizeMB > quota.maxAudioSizeMB) {
      // Quota dépassé
      await this.notificationService.send(userId, {
        type: 'quota_exceeded',
        message: 'Quota stockage atteint',
      });
      return false;
    }

    return true;
  }
}
```

**Rationale MVP sans quotas :**
- Mono-user = usage limité naturellement
- 5 captures/jour × 1min × 365j × 32kbps = ~350 MB/an (acceptable)
- Monitoring suffit pour détecter abuse
- Quotas ajoutés seulement si problème identifié

---

## 14.3 - Media Optimization

**Decision:** Thumbnails blurhash, lazy loading, adaptive quality

**Blurhash pour Perception Vitesse :**

```typescript
// Backend : générer blurhash lors digestion
class DigestionService {
  async process(capture: Capture) {
    // Si capture contient image
    if (capture.type === 'image') {
      const blurhash = await this.generateBlurhash(capture.imageFile);

      return {
        ...digest,
        blurhash,  // String compact (20-30 chars)
      };
    }
  }

  private async generateBlurhash(imageBuffer: Buffer): Promise<string> {
    const { data, info } = await sharp(imageBuffer)
      .resize(32, 32, { fit: 'inside' })
      .ensureAlpha()
      .raw()
      .toBuffer({ resolveWithObject: true });

    return encode(
      new Uint8ClampedArray(data),
      info.width,
      info.height,
      4,  // Components X
      3   // Components Y
    );
  }
}

// Mobile : afficher blurhash pendant chargement
<Image
  source={{ uri: capture.imageUrl }}
  placeholder={{ blurhash: capture.blurhash }}
  transition={200}
  style={styles.image}
/>
```

**Adaptive Quality (Post-MVP) :**

```typescript
// Servir audio qualité adaptée à connexion
class AdaptiveAudioService {
  async getAudioUrl(captureId: string, quality: 'low' | 'medium' | 'high') {
    const qualityMap = {
      low: '16kbps',     // 2G/slow 3G
      medium: '32kbps',  // 3G/4G
      high: '64kbps',    // WiFi/5G
    };

    const key = `audio/${captureId}_${qualityMap[quality]}.m4a`;
    return this.s3.getSignedUrl(key);
  }
}

// Mobile détecte connexion
const quality = netInfo.type === 'wifi' ? 'high' : 'medium';
const audioUrl = await api.getAudioUrl(captureId, quality);
```

**Rationale :**
- Blurhash : placeholder immédiat (perception vitesse)
- Lazy loading : économie bande passante
- Adaptive quality : optimisation connexion lente

---

## Conséquences Globales ADR-014

**Bénéfices:**
- Stockage mobile optimisé : cleanup 30j automatique
- Cloud économique : compression 50%
- Quotas : monitoring sans limite artificielle MVP
- UX : blurhash + lazy loading = fluidité

**Trade-offs acceptés:**
- Complexité : cleanup jobs + compression pipeline
- Latence re-écoute : si fichier supprimé localement, stream depuis cloud
- Post-MVP : adaptive quality nécessite multiple encodings

**Impact:**
- **Epic 2** : Audio compression 64kbps → 32kbps
- **Epic 3** : Lazy loading images avec blurhash
- **Infrastructure** : Cleanup cron job quotidien
- **Post-MVP** : Quotas utilisateur si abuse détecté

---

## Implementation Status

- ✅ **Epic 2** : Compression audio 64kbps mobile
- ⏳ **Infrastructure** : Cleanup cron job
- ⏳ **Post-MVP** : Compression cloud 32kbps
- ⏳ **Post-MVP** : Quotas utilisateur
- ⏳ **Post-MVP** : Adaptive quality

---

## References

- FFmpeg Audio Compression: https://trac.ffmpeg.org/wiki/Encode/AAC
- Blurhash: https://blurha.sh/
- Expo FileSystem: https://docs.expo.dev/versions/latest/sdk/filesystem/
- Sharp Image Processing: https://sharp.pixelplumbing.com/

---

## Validation Criteria

ADR considéré succès SI :
- ⏳ Cleanup mobile fonctionne (30j automatique)
- ⏳ Compression cloud 50% économie stockage
- ⏳ Monitoring usage détecte > 5 GB par user
- ⏳ Blurhash affichage < 100ms

**Review Date :** 2026-03 (après Epic 3)

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

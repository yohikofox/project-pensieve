---
adr: ADR-016
title: "Hybrid Architecture - Cloud Auth + Homelab Storage"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-016: Hybrid Architecture - Cloud Auth + Homelab Storage

**Status:** ✅ ACCEPTÉ

**Date:** 2026-01-19

---

## Context

Choix entre trois architectures pour l'authentification et le storage :
1. **Full Cloud** (Supabase Cloud pour auth + storage)
2. **Full Self-Hosted** (GoTrue + MinIO homelab pour auth + storage)
3. **Hybrid** (Supabase Cloud pour auth + MinIO homelab pour storage)

**Contraintes :**
- MVP doit être livrable rapidement (time-to-market critique)
- Social login requis (Google + Apple Sign-In)
- MFA requis en Post-MVP
- Homelab existant avec ressources disponibles
- Storage audio = bottleneck coût (75 GB/mois pour 1000 users)
- Budget infrastructure MVP = €0-10/mois maximum

---

## Decision

Architecture **Hybrid** (Option 3)

**Composants :**

```yaml
Authentication & User Management:
  Provider: Supabase Cloud (Free tier)
  Services:
    - GoTrue (Auth service managed)
    - Email/Password authentication
    - Google OAuth 2.0
    - Apple Sign-In
    - Password reset emails (SMTP managed)
    - Future: MFA (TOTP, SMS)
  Coût: €0/mois (50,000 MAU gratuits)
  SLA: 99.9% (Pro tier si upgrade)

Backend API:
  Location: Homelab (self-hosted)
  Stack: NestJS + TypeScript
  Database: PostgreSQL (homelab)
  Queue: RabbitMQ (homelab)
  Validation JWT: Local (Supabase public key)
  Coût: €0/mois (infrastructure existante)

Storage (Audio Files):
  Location: Homelab (self-hosted)
  Technology: MinIO (S3-compatible)
  Capacity: Illimitée (limité par disques NAS)
  Access Method: Presigned URLs
  Backup: NAS backup strategy existante
  Coût: €0/mois (disques déjà amortis)

Accès Internet:
  Solution: Cloudflare Tunnel (Zero Trust)
  Domaines:
    - auth.pensine.example.local → Supabase Cloud
    - api.pensine.example.local → Homelab NestJS (via tunnel)
    - storage.pensine.example.local → Homelab MinIO (via tunnel)
  SSL: Automatique (Cloudflare managed)
  Coût: €0/mois
```

---

## Rationale

### 1. Pourquoi Supabase Cloud pour Auth (vs self-hosted GoTrue)

```
✅ Social Login Simplifié :
   - Google OAuth : 10 min setup (dashboard config)
   - Apple Sign-In : 10 min setup (dashboard config)
   - vs Custom OAuth2 : 10-15 jours développement

✅ Email & Password Reset :
   - SMTP managed (pas de config Gmail/SendGrid)
   - Email templates customisables
   - Rate limiting built-in

✅ MFA Ready (Future) :
   - TOTP : feature flag dashboard
   - SMS : intégration Twilio built-in
   - vs Custom : 1-2 semaines dev

✅ Maintenance Zero :
   - Updates automatiques
   - Security patches
   - 99.9% SLA (Pro tier)

✅ Coût Optimal :
   - Free tier : 50,000 MAU (suffisant 1-2 ans)
   - Pas de storage utilisé (audios ailleurs)
   - Upgrade Pro : $25/mois seulement si >50K users
```

**Temps économisé :** 10-15 jours développement OAuth2 + MFA + Email

**Risque vendor lock-in :** Mitigé (JWT standard, migration possible, cf. Exit Strategy)

---

### 2. Pourquoi MinIO Homelab pour Storage (vs Supabase Cloud)

```
✅ Coût Storage Illimité :
   Supabase Cloud Free : 1 GB storage
   → Limite atteinte avec ~70 users (10 captures × 1.5 MB)

   Supabase Cloud Pro : 100 GB storage + $0.021/GB additionnel
   → 1000 users = 150 GB/mois stockage cumulatif
   → Année 1 : 1.8 TB = $37/mois storage add-on
   → Coût annuel : ~$750

   MinIO Homelab :
   → NAS 4 TB existant = €0 marginal
   → 1000 users = 1.8 TB stockage = €0
   → Économie : ~$750/an

✅ Scalabilité Storage :
   - 10,000 users = 18 TB stockage
   - Supabase : $378/mois ($4,536/an)
   - Homelab : €200 upgrade disques (one-time)

✅ Control & Privacy :
   - Audios = données potentiellement sensibles (RGPD)
   - 100% contrôle homelab
   - Backup strategy existante
   - Pas de transfert données hors UE

✅ Performance Backend → Storage :
   - MinIO sur LAN : ~1-5ms latency
   - Supabase Cloud : ~50-100ms latency
   - Impact : Digestion IA + transcription plus rapides
```

**Économie annuelle :** ~€700-1000 vs full Supabase Cloud

**Trade-off accepté :** Setup MinIO (4h) + Cloudflare Tunnel (2h) = 1 jour one-time

---

### 3. Pourquoi Hybrid (vs Full Cloud ou Full Self-Hosted)

| Critère | Full Cloud | Hybrid (Choisi) | Full Self-Hosted |
|---------|------------|-----------------|------------------|
| **Setup Time** | 1h | 1 jour | 2-3 jours |
| **Social Login** | ✅ Trivial | ✅ Trivial | ❌ 10j dev |
| **MFA Future** | ✅ Built-in | ✅ Built-in | ❌ 2 semaines dev |
| **Storage Cost** | 💰 $750/an | ✅ €0 | ✅ €0 |
| **Maintenance** | ✅ Zero | ⚠️ MinIO only | ❌ Auth + Storage |
| **Vendor Lock-in** | ⚠️ Moderate | ⚠️ Auth only | ✅ None |
| **Migration Path** | ⚠️ Complexe | ✅ Progressive | N/A |
| **Total Cost/An** | ~$750 | **~€0** | ~€0 + 30h/an |

**Winner :** Hybrid = meilleur compromis time-to-market + coût + simplicité

---

## Architecture Flows

### Flow 1 : User Authentication

```typescript
// Mobile App
const { data, error } = await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: { redirectTo: 'pensine://auth/callback' }
})

// JWT token obtenu
const jwt = supabase.auth.session().access_token

// Envoyé à backend homelab
axios.defaults.headers.common['Authorization'] = `Bearer ${jwt}`

// Backend valide JWT localement (pas de call Supabase)
// → Vérifie signature avec Supabase public key (cached)
```

### Flow 2 : Audio Upload (Presigned URL)

```typescript
// 1. Mobile demande upload URL au backend
POST https://api.pensine.example.local/api/captures/upload-url
Authorization: Bearer {jwt}

// 2. Backend génère presigned URL MinIO
const uploadUrl = await minioService.presignedPutObject(
  `${userId}/${captureId}.m4a`,
  3600 // expire 1 heure
)

// 3. Mobile upload DIRECT vers MinIO homelab
PUT https://storage.pensine.example.local/{presigned-path}
Body: audio blob

// 4. Mobile confirme au backend
POST https://api.pensine.example.local/api/captures/{id}/confirm
```

**Avantages Presigned URL :**
- Backend ne voit jamais l'audio (économise bandwidth)
- Upload parallélisable (plusieurs audios simultanés)
- Pas de limite taille fichier backend

---

## Security Considerations

### JWT Validation (Backend)

```typescript
// Supabase émet JWT standard
// Backend valide localement (offline-compatible)

@Injectable()
export class SupabaseAuthGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const token = this.extractToken(request);

    // Valide JWT avec Supabase public key (cached locally)
    const { data: { user }, error } = await this.supabase.auth.getUser(token);

    if (error || !user) {
      throw new UnauthorizedException('Invalid token');
    }

    request.user = user;
    return true;
  }
}
```

### Storage Ownership (MinIO)

```typescript
// Object paths incluent userId (isolation)
const objectName = `${userId}/${captureId}.m4a`

// Presigned URLs temporaires (1h upload, 24h download)
// Backend vérifie ownership avant génération URL

const capture = await captureRepo.findOne({
  where: { id: captureId, userId: req.user.id }
});

if (!capture) throw new NotFoundException(); // User ne possède pas

const downloadUrl = await minioService.presignedGetObject(
  capture.storageKey,
  86400 // 24 hours
);
```

### RGPD Compliance

```typescript
// Data Export (Article 15)
GET /api/gdpr/export
→ ZIP with:
  - User profile (from Supabase via admin API)
  - Captures metadata (PostgreSQL)
  - Audio files (MinIO download all user objects)

// Data Deletion (Article 17)
POST /api/gdpr/delete-account
1. Delete Supabase user (admin API)
2. Delete PostgreSQL records (cascade)
3. Delete MinIO objects (removeObjects prefix: userId/*)
```

---

## Migration Path (Exit Strategy)

**Si nécessaire migrer de Supabase Cloud → Self-Hosted GoTrue :**

```bash
# 1. Export users DB
pg_dump --schema=auth --host=db.xxxxx.supabase.co > auth-export.sql

# 2. Setup GoTrue homelab
docker-compose up -d gotrue

# 3. Import users
psql -h localhost -d auth < auth-export.sql

# 4. Update OAuth redirect URLs (Google/Apple)
# 5. Deploy backend avec nouvelle SUPABASE_URL

# Downtime : 2-6 heures
# Coût migration : 1-2 jours travail
```

**Triggers de migration :**
- Supabase pricing change défavorable
- Besoin control total (compliance stricte)
- >50,000 MAU (Free tier dépassé)

**Alternative :** Upgrade Supabase Pro ($25/mois) si <100K MAU

---

## Consequences

### Positives

- ✅ **Time-to-market optimal** : MVP en 2 jours (vs 2 semaines full self-hosted)
- ✅ **Coût €0/mois** pendant 1-2 ans (Free tier + homelab existant)
- ✅ **Social login trivial** : Google + Apple en 20 min setup
- ✅ **MFA ready** : Feature flag quand nécessaire
- ✅ **Storage illimité** : MinIO homelab évolutif (upgrade disques simple)
- ✅ **Offline-first compatible** : JWT validation locale (pas de call cloud)
- ✅ **Scalabilité** : 50K users sans coût additionnel

### Négatives

- ⚠️ **Vendor lock-in partiel** : Auth sur Supabase (mitigé : JWT standard, migration possible)
- ⚠️ **Dépendance externe** : Supabase downtime = nouveaux logins impossibles (users existants OK)
- ⚠️ **Complexité Cloudflare Tunnel** : Exposition homelab sur Internet via tunnel
- ⚠️ **Maintenance MinIO** : Backups, monitoring (intégré dans routine ops existante)

### Mitigation risques

- Backup auth DB Supabase : export hebdomadaire automatique
- Monitoring uptime : Uptime Kuma sur Supabase endpoints
- Exit strategy documentée : migration vers GoTrue self-hosted possible en 1-2 jours

---

## Alternatives Rejected

**Alternative A : Full Supabase Cloud (Auth + Storage)**
- ❌ Rejeté : Coût storage prohibitif ($750+/an dès 1000 users)
- ❌ Storage 1 GB Free tier = limite atteinte rapidement
- ❌ Vendor lock-in plus fort (auth + storage)

**Alternative B : Full Self-Hosted (GoTrue + MinIO)**
- ❌ Rejeté : Setup complexe (2-3 jours vs 1 jour hybrid)
- ❌ OAuth2 custom implementation (10-15 jours dev)
- ❌ MFA custom (1-2 semaines dev)
- ❌ Maintenance auth complexe (security patches, updates)
- ❌ Time-to-market trop lent pour MVP

**Alternative C : Custom JWT + OAuth2 from Scratch**
- ❌ Rejeté : 10-15 jours développement OAuth2 flows
- ❌ Sécurité = notre responsabilité (risque bugs critiques)
- ❌ Maintenance continue (Google/Apple API changes)
- ❌ Viole ADR-007 exception justifiée (auth = solved problem)

---

## Implementation

### Stack Homelab Ajusté

```yaml
services:
  # ❌ REMOVED : GoTrue (Auth service)
  # ❌ REMOVED : Redis (JWT blacklist inutile)

  # ✅ KEPT : MinIO (Storage)
  minio:
    image: minio/minio:latest
    volumes:
      - /nas/pensine-audios:/data
    ports:
      - "9000:9000"

  # ✅ KEPT : Backend (API)
  backend:
    environment:
      SUPABASE_URL: https://xxxxx.supabase.co
      SUPABASE_ANON_KEY: ${SUPABASE_ANON_KEY}
      JWT_SECRET: ${JWT_SECRET}  # Same as Supabase
      MINIO_ENDPOINT: minio

  # ✅ KEPT : PostgreSQL (App DB only, pas Auth DB)
  db:
    environment:
      POSTGRES_DB: pensine  # Une seule DB (pas auth DB)

  # ✅ KEPT : RabbitMQ (Queue digestion)
  rabbitmq:
```

### Cloudflare Tunnel Configuration

```bash
# Install cloudflared
cloudflared tunnel create pensine

# Configure routes
cloudflared tunnel route dns pensine api.pensine.example.local
cloudflared tunnel route dns pensine storage.pensine.example.local

# Run tunnel
cloudflared tunnel --config config.yml run pensine
```

**config.yml :**
```yaml
tunnel: <tunnel-id>
credentials-file: /path/to/credentials.json

ingress:
  - hostname: api.pensine.example.local
    service: http://localhost:3000
  - hostname: storage.pensine.example.local
    service: http://localhost:9000
  - service: http_status:404
```

### Dependencies Updated

**Backend (NestJS) :**
```json
{
  "dependencies": {
    "@supabase/supabase-js": "^2.39.0",  // Auth client
    "minio": "^7.1.3"                    // Storage client
  }
}
```

**Mobile (React Native) :**
```json
{
  "dependencies": {
    "@supabase/supabase-js": "^2.39.0",  // Auth client
    "@react-native-async-storage/async-storage": "^1.21.0"  // Session persistence
  }
}
```

---

## Monitoring & Alerting

```yaml
Uptime Checks:
  - https://xxxxx.supabase.co/auth/v1/health (Supabase Auth)
  - https://api.pensine.example.local/health (Backend homelab)
  - https://storage.pensine.example.local/minio/health/live (MinIO homelab)

Alertes:
  - Supabase Auth down > 5 min → Email
  - Backend homelab down > 2 min → Email
  - MinIO down > 2 min → Email
  - Cloudflare Tunnel down > 2 min → Email

Metrics:
  - Supabase MAU usage (dashboard)
  - MinIO storage used (mc du)
  - Backend API latency (Prometheus)
```

---

## Cost Breakdown (Projected)

```
Year 1 (0-1000 users):
- Supabase Cloud Free: €0/mois
- MinIO homelab: €0/mois (existing)
- Cloudflare Tunnel: €0/mois
- Total: €0/mois 🎉

Year 2 (1000-10000 users):
- Supabase Cloud Free: €0/mois (still <50K MAU)
- MinIO homelab: €200 one-time (disk upgrade 4→8 TB)
- Cloudflare Tunnel: €0/mois
- Total: ~€17/mois amortized

Year 3 (10000-50000 users):
- Supabase Cloud Free: €0/mois (at MAU limit)
- MinIO homelab: €0/mois (8 TB sufficient)
- Cloudflare Tunnel: €0/mois
- Total: €0/mois

Year 4+ (>50000 users):
Decision point:
  Option A: Upgrade Supabase Pro ($25/mois)
  Option B: Migrate to self-hosted GoTrue (€0 but 2 jours migration)
```

---

## Validation Criteria

ADR considéré succès SI :
- ✅ MVP shipped en <2 semaines (auth working)
- ✅ Social login (Google + Apple) fonctionnel
- ✅ Storage cost <€10/mois pendant Year 1
- ✅ Zero perte de données (audios + users)
- ✅ Uptime >99% (auth + storage)
- ✅ Migration vers self-hosted possible si nécessaire

**Review Date :** 2026-06 (6 mois post-MVP) - Réévaluer si besoin self-hosted auth

---

## References

- Supabase Documentation: https://supabase.com/docs
- MinIO Documentation: https://min.io/docs
- Cloudflare Tunnel: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps
- OAuth2 Best Practices: https://tools.ietf.org/html/rfc6749
- RGPD Compliance: https://gdpr.eu

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

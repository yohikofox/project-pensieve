---
adr: ADR-010
title: "Security & Encryption Strategy"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-010: Security & Encryption Strategy

**Status:** ✅ ACCEPTÉ

**Context:** Définir la stratégie de sécurité et de chiffrement pour protéger les données sensibles des utilisateurs (audio, transcriptions, idées business).

**Decision:** 5 décisions validées pour la sécurité complète du système.

---

## 10.1 - Stratégie d'Authentification

**Decision:** JWT Access Token (courte durée) + Refresh Token (longue durée)

**Architecture:**

```typescript
// 1. Login
POST /auth/login
{
  email: "user@example.com",
  password: "***"
}

Response:
{
  accessToken: "eyJhbGc...",      // JWT 15min
  refreshToken: "rt_abc123...",   // 30 jours
  expiresIn: 900                  // 15min en secondes
}

// 2. Access Token stocké en mémoire (React state)
// 3. Refresh Token stocké en Keychain/Keystore (sécurisé)

// 4. Refresh automatique avant expiration
GET /auth/refresh
Headers:
  Authorization: Bearer rt_abc123...

Response:
{
  accessToken: "eyJhbGc...",      // Nouveau JWT 15min
  expiresIn: 900
}

// 5. Révocation (logout, compromission)
POST /auth/revoke
Headers:
  Authorization: Bearer rt_abc123...

→ Refresh token supprimé de DB (blacklist)
```

**Caractéristiques:**

**Access Token (JWT):**
- Durée: **15 minutes**
- Stockage: **Mémoire** (React state, pas AsyncStorage)
- Format: JWT signé (HS256 ou RS256)
- Claims: `{ sub: userId, email, iat, exp }`
- Stateless: backend ne stocke pas (vérifie signature)

**Refresh Token:**
- Durée: **30 jours**
- Stockage: **Keychain (iOS) / Keystore (Android)** (chiffré OS)
- Format: Random string opaque (UUID v4)
- Stateful: stocké en DB backend avec metadata
- Révocable: peut être invalidé côté serveur

**Rationale:**
- Access token court (15min) limite window d'exploitation si volé
- Refresh token long (30 jours) évite re-login fréquent
- Stockage séparé: access en mémoire (volatil), refresh sécurisé OS
- Révocation: logout ou compromission invalide refresh token
- Balance sécurité/UX optimale pour mobile

**Rotation Refresh Token (Post-MVP):**
```typescript
// Chaque refresh génère nouveau refresh token (rotation)
POST /auth/refresh
→ Ancien refresh token invalidé
→ Nouveau refresh token émis
→ Protection contre vol de refresh token
```

---

## 10.2 - Chiffrement au Repos (Mobile)

**Decision:** MVP = Keychain/Keystore pour tokens, Post-MVP = SQLCipher pour DB complète

**MVP (Minimal Viable Security):**

```typescript
// Tokens sensibles chiffrés par OS
import * as SecureStore from 'expo-secure-store';

// Stockage refresh token
await SecureStore.setItemAsync('refresh_token', refreshToken);

// Récupération
const refreshToken = await SecureStore.getItemAsync('refresh_token');

// Suppression (logout)
await SecureStore.deleteItemAsync('refresh_token');
```

**Données stockées en Keychain/Keystore (MVP):**
- ✅ Refresh Token (critique)
- ✅ User credentials si "Remember me" (optionnel)
- ❌ DB ~~WatermelonDB~~ OP-SQLite: **non chiffrée** (SQLite standard)
- ⚠️ **UPDATE (2026-01-22):** OP-SQLite remplace WatermelonDB. Voir [ADR-018](./ADR-018-migration-watermelondb-opsqlite.md)

**Rationale MVP:**
- Keychain/Keystore = chiffrement hardware-backed (Secure Enclave iOS, TEE Android)
- Protection tokens critique sans overhead performance
- DB non chiffrée acceptable pour MVP mono-user
- Transcriptions/ideas pas réglementées (pas HIPAA/données santé)

**Post-MVP (Production Hardening):**

```typescript
// SQLCipher pour chiffrement DB complète
import { open } from 'op-sqlite';

const db = open({
  name: 'pensine.db',
  encryptionKey: await getEncryptionKey()  // Dérivée du Keychain
});
```

**Clé chiffrement DB:**
- Générée au premier lancement (crypto.randomBytes(32))
- Stockée dans Keychain/Keystore
- DB inaccessible sans déverrouillage device
- Performance impact: ~15% overhead (acceptable)

**Rationale Post-MVP:**
- Protection complète données sensibles (transcriptions, idées business)
- Conformité certifications (si B2B futur)
- Pas d'impact UX (transparent)

---

## 10.3 - Chiffrement au Repos (Backend)

**Decision:** MVP = Disk/Volume Encryption (Infrastructure), Post-MVP = TDE ou Column-level si régulé

**MVP (Infrastructure-level Encryption):**

```bash
# Docker Volume chiffré (LUKS)
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/encrypted/data \
  --opt o=bind \
  pensine-data

# PostgreSQL stocké sur volume chiffré
services:
  postgres:
    volumes:
      - pensine-data:/var/lib/postgresql/data

# Scaleway/AWS: Volumes chiffrés par défaut
# - AWS EBS: encryption at rest (AES-256)
# - Scaleway Block Storage: LUKS encryption
```

**Rationale MVP:**
- Chiffrement volume = protection vol physique disques
- Transparent pour application (pas de code)
- Gratuit sur cloud providers modernes
- Conforme exigences basiques sécurité

**Post-MVP (Database-level Encryption):**

**Option A: Transparent Data Encryption (TDE)** (PostgreSQL 15+)
```sql
-- Chiffrement automatique toutes les tables
ALTER DATABASE pensine SET encryption = 'on';
```

**Option B: Column-level Encryption** (données ultra-sensibles)
```typescript
// Chiffrement sélectif colonnes sensibles
class Capture {
  id: UUID;

  @Encrypted()  // Chiffré avec clé backend
  normalizedText: string;

  @Encrypted()
  rawContent: string;

  state: string;  // Non chiffré (ok)
}
```

**Quand utiliser Post-MVP:**
- Régulations spécifiques (HIPAA, PCI-DSS)
- Clients B2B exigeants (compliance)
- Multi-tenancy avec isolation forte
- Audit trail chiffré (traçabilité)

**Rationale:**
- MVP: infrastructure encryption suffit
- Post-MVP: database encryption si besoin métier
- Éviter sur-engineering prématuré
- Performance impact TDE: ~10%, Column: ~20%

---

## 10.4 - Communication Sécurisée (TLS/HTTPS)

**Decision:** TLS 1.3 obligatoire, certificats Let's Encrypt, HSTS activé

**Configuration Backend:**

```typescript
// NestJS HTTPS configuration
import * as https from 'https';
import * as fs from 'fs';

const httpsOptions = {
  key: fs.readFileSync('/certs/privkey.pem'),
  cert: fs.readFileSync('/certs/fullchain.pem'),
  // TLS 1.3 minimum
  minVersion: 'TLSv1.3',
  // Ciphers modernes uniquement
  ciphers: [
    'TLS_AES_128_GCM_SHA256',
    'TLS_AES_256_GCM_SHA384',
    'TLS_CHACHA20_POLY1305_SHA256'
  ].join(':')
};

const app = await NestFactory.create(AppModule, {
  httpsOptions
});
```

**Headers Sécurité:**

```typescript
// Helmet.js (security headers)
app.use(helmet({
  hsts: {
    maxAge: 31536000,        // 1 an
    includeSubDomains: true,
    preload: true
  },
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.pensine.example.local"]
    }
  }
}));
```

**Mobile (Certificate Pinning - Post-MVP):**

```typescript
// Expo config.json
{
  "expo": {
    "ios": {
      "infoPlist": {
        "NSAppTransportSecurity": {
          "NSExceptionDomains": {
            "api.pensine.example.local": {
              "NSIncludesSubdomains": true,
              "NSExceptionRequiresForwardSecrecy": true,
              "NSExceptionMinimumTLSVersion": "TLSv1.3",
              "NSPinnedDomains": [
                "api.pensine.example.local"
              ]
            }
          }
        }
      }
    }
  }
}
```

**Rationale:**
- TLS 1.3: protection MITM, forward secrecy
- Let's Encrypt: gratuit, auto-renew, trusted
- HSTS: force HTTPS, protection downgrade attacks
- Certificate pinning (post-MVP): protection CA compromise

**Conséquences:**
- Aucune communication en clair (HTTP interdit)
- Dev local: certificats self-signed ok (pas prod)
- Monitoring expiration certificats (alerte 30j avant)

---

## 10.5 - Conformité RGPD

**Decision:** Data export, account deletion (soft delete 30j), opt-in consent analytics

**Droits Utilisateur (RGPD Articles 15, 17, 20):**

**1. Droit d'accès (Article 15):**
```typescript
// Export complet données utilisateur
GET /user/export

Response: ZIP file containing:
  - user_data.json (profile, metadata)
  - captures.json (toutes les captures)
  - audio_files/ (tous les fichiers audio)
  - thoughts.json (digestions, résumés)
  - todos.json (actions)
  - ideas.json (idées)
  - projects.json (projets)
```

**2. Droit à l'effacement (Article 17):**
```typescript
// Suppression compte
DELETE /user/account

Process:
1. Soft delete (30 jours de grâce)
   → account.state = 'pending_deletion'
   → account.deletion_date = NOW() + 30 days
   → User peut annuler dans les 30j

2. Hard delete (après 30 jours, cron job)
   → Suppression DB: CASCADE sur toutes les tables
   → Suppression S3: audio files
   → Suppression Redis: cache
   → Anonymisation logs: remplace userId par 'deleted_user'

3. Exceptions:
   → Logs applicatifs: 90 jours (debugging)
   → Backups: 7 jours (restauration)
   → Legal hold: si procédure judiciaire en cours
```

**3. Consentement (Article 6):**
```typescript
// Onboarding - Opt-in explicite
const consents = {
  required: [
    {
      type: 'terms_of_service',
      mandatory: true,
      description: "Conditions d'utilisation Pensine"
    },
    {
      type: 'data_processing',
      mandatory: true,
      description: "Traitement données pour fonctionnement app"
    }
  ],
  optional: [
    {
      type: 'analytics',
      mandatory: false,
      description: "Statistiques usage anonymisées (amélioration produit)",
      default: false  // Opt-in, pas opt-out
    },
    {
      type: 'crash_reports',
      mandatory: false,
      description: "Rapports crash automatiques (Sentry)",
      default: true   // Recommandé mais optionnel
    }
  ]
};

// User peut modifier consentements après onboarding
PUT /user/consents
{
  analytics: false,
  crash_reports: true
}
```

**4. Portabilité (Article 20):**
```json
// Format export JSON structuré
{
  "version": "1.0.0",
  "exported_at": "2026-01-13T10:00:00Z",
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "created_at": "2026-01-01T00:00:00Z"
  },
  "captures": [
    {
      "id": "uuid",
      "type": "audio",
      "captured_at": "2026-01-12T15:30:00Z",
      "audio_file": "audio_files/capture_123.m4a",
      "transcription": "...",
      "summary": "...",
      "tags": ["business", "idea"]
    }
  ],
  "todos": [...],
  "ideas": [...]
}
```

**Rationale:**
- Conformité légale UE obligatoire
- Export JSON + audio = portabilité complète
- Soft delete 30j = protection erreur utilisateur
- Opt-in analytics = respect vie privée
- Anonymisation logs = balance sécurité/privacy

**Documentation Utilisateur:**
- Privacy Policy (français + anglais)
- Data Processing Agreement (si B2B)
- FAQ RGPD (droits, procédures)

---

## Conséquences Globales ADR-010

**Bénéfices:**
- Protection données sensibles multi-niveaux
- Conformité RGPD complète
- Balance sécurité / UX / performance

**Trade-offs acceptés:**
- Coût infrastructure: +10% (chiffrement, certificats)
- Maintenance: monitoring certificats, audit logs, RGPD requests
- Complexité accrue: gestion tokens, refresh, révocation

**Impact:**
- **Story 1.3** : Implémentation RGPD (export, deletion)
- **Epic 1** : Auth JWT + Refresh Token
- **Post-MVP** : SQLCipher DB encryption, Certificate Pinning

---

## Implementation Status

- ✅ **Story 1.2** : Auth JWT + Refresh Token implémenté
- ✅ **Story 1.3** : RGPD export/deletion implémenté
- ✅ **Infrastructure** : TLS 1.3 + HSTS configuré
- ⏳ **Post-MVP** : SQLCipher encryption DB
- ⏳ **Post-MVP** : Certificate Pinning mobile

---

## References

- OWASP Mobile Security: https://owasp.org/www-project-mobile-security/
- RGPD (GDPR): https://gdpr.eu/
- JWT Best Practices: https://tools.ietf.org/html/rfc8725
- Expo SecureStore: https://docs.expo.dev/versions/latest/sdk/securestore/
- Let's Encrypt: https://letsencrypt.org/

---

## Validation Criteria

ADR considéré succès SI :
- ✅ Auth JWT fonctionne (Epic 1)
- ✅ RGPD export/deletion fonctionnels (Story 1.3)
- ✅ TLS 1.3 + HSTS activés (Infrastructure)
- ⏳ Aucune faille sécurité critique (1 mois monitoring)
- ⏳ Audit sécurité post-MVP PASS

**Review Date :** 2026-03 (après Epic 2)

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

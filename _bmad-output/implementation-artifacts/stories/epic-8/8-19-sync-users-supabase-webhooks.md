# Story 8.19: Sync Utilisateurs Supabase → Backend via Webhooks

Status: ready-for-dev

## Story

En tant que système,
je veux recevoir les événements Supabase Auth (création, modification, suppression d'utilisateur) via webhook et les répercuter en base PostgreSQL backend,
afin que Supabase soit la source de vérité et que le backend ait toujours une vue cohérente des utilisateurs sans que les admins puissent modifier les données auth directement.

## Contexte

**Problème actuel** : Les users n'existent dans la table `users` du backend PostgreSQL que s'ils ont effectué un export RGPD (`upsertUser()` dans `RgpdService`). Les nouveaux utilisateurs Supabase ne sont pas automatiquement créés côté backend.

**Architecture cible** :
- Supabase Auth = source de vérité (email, password, metadata)
- Backend PostgreSQL = mirror des données non-auth (push token, feature flags, statut)
- Webhook Supabase Auth → Backend NestJS → upsert/soft-delete en DB

**Événements Supabase Auth Webhooks** : Supabase envoie des webhooks sur `user.created`, `user.updated`, `user.deleted` avec signature HMAC-SHA256 vérifiable.

## Acceptance Criteria

1. **AC1 — Endpoint webhook** : `POST /webhooks/supabase/auth` reçoit les événements Supabase Auth. L'endpoint est **public** (pas de `SupabaseAuthGuard`) mais protégé par vérification de signature HMAC-SHA256.

2. **AC2 — Vérification signature** : Le backend vérifie le header `X-Supabase-Signature` (HMAC-SHA256 du body avec la clé secrète webhook). Retourne 401 si signature invalide.

3. **AC3 — Événement `INSERT` (user.created)** : Upsert en DB (`users` table) avec `id` Supabase, `email`, `status: 'active'`, `created_at`. Pattern existant : `RgpdService.upsertUser()`.

4. **AC4 — Événement `UPDATE` (user.updated)** : Met à jour `email` si changé. Ne touche PAS aux champs gérés côté backend (pushToken, debug_mode_access, etc.).

5. **AC5 — Événement `DELETE` (user.deleted)** : Soft delete (`status: 'deleted'`, `deletion_requested_at: now()`). Ne supprime pas la ligne (cohérence RGPD).

6. **AC6 — Idempotence** : Le handler est idempotent — un même événement rejoué plusieurs fois ne crée pas de doublons ni de données corrompues (upsert, pas insert).

7. **AC7 — Variable d'environnement** : `SUPABASE_WEBHOOK_SECRET` dans `.env.example`. La signature est vérifiée uniquement si cette variable est définie (fallback : log warning + accept en dev).

8. **AC8 — Audit log** : Les événements webhook sont loggués (`user_id`, `event_type`, `timestamp`) dans `audit_logs` avec `action: 'WEBHOOK_USER_SYNC'`.

9. **AC9 — Tests unitaires** : Tests couvrent signature valide, signature invalide (401), chaque type d'événement (INSERT/UPDATE/DELETE), idempotence.

10. **AC10 — Configuration Supabase** : La story documente les étapes de configuration du webhook dans le Dashboard Supabase (URL, secret, events à activer).

## Tasks / Subtasks

- [ ] T1 — Créer `WebhookModule` NestJS (AC1)
  - [ ] T1.1 — Créer `backend/src/modules/webhook/` avec structure DDD
  - [ ] T1.2 — `WebhookController` : `POST /webhooks/supabase/auth` (raw body pour signature)
  - [ ] T1.3 — Enregistrer `WebhookModule` dans `AppModule`
  - [ ] T1.4 — Configurer `rawBody: true` dans `NestFactory.create()` (nécessaire pour vérification HMAC)

- [ ] T2 — Service de vérification signature (AC2, AC7)
  - [ ] T2.1 — `SupabaseWebhookGuard` : extrait `X-Supabase-Signature`, calcule HMAC-SHA256 du raw body avec `SUPABASE_WEBHOOK_SECRET`
  - [ ] T2.2 — Comparaison timing-safe (`crypto.timingSafeEqual`)
  - [ ] T2.3 — Si `SUPABASE_WEBHOOK_SECRET` absent : log warning, accepter (mode dev)

- [ ] T3 — Handler événements (AC3, AC4, AC5, AC6)
  - [ ] T3.1 — `UserSyncService` : `handleWebhookEvent(payload: SupabaseAuthWebhookPayload)`
  - [ ] T3.2 — Switch sur `payload.type` : `INSERT` → `upsertUser()`, `UPDATE` → `updateUserEmail()`, `DELETE` → `softDeleteUser()`
  - [ ] T3.3 — Réutiliser/déplacer `upsertUser()` depuis `RgpdService` vers service partagé (ou dupliquer dans `UserSyncService` — voir Dev Notes)
  - [ ] T3.4 — `softDeleteUser()` : `status = 'deleted'`, `deletion_requested_at = new Date()`
  - [ ] T3.5 — Audit log `WEBHOOK_USER_SYNC` après chaque événement traité

- [ ] T4 — Variable d'environnement (AC7)
  - [ ] T4.1 — Ajouter `SUPABASE_WEBHOOK_SECRET=your_webhook_secret` dans `backend/.env.example`

- [ ] T5 — Tests unitaires (AC9)
  - [ ] T5.1 — Test signature valide → 200
  - [ ] T5.2 — Test signature invalide → 401
  - [ ] T5.3 — Test INSERT : user créé en DB
  - [ ] T5.4 — Test INSERT idempotent : deuxième appel ne crée pas de doublon
  - [ ] T5.5 — Test UPDATE : email mis à jour, pushToken inchangé
  - [ ] T5.6 — Test DELETE : soft delete appliqué

- [ ] T6 — Documentation configuration Supabase (AC10)
  - [ ] T6.1 — Ajouter section webhook dans `supabase-setup-instructions.md`

## Dev Notes

### Payload Supabase Auth Webhook

```typescript
interface SupabaseAuthWebhookPayload {
  type: 'INSERT' | 'UPDATE' | 'DELETE';
  table: 'users';
  schema: 'auth';
  record: {
    id: string;
    email: string;
    created_at: string;
    updated_at: string;
    role: string;
    [key: string]: any;
  } | null;
  old_record: { ... } | null;
}
```

### Raw Body pour HMAC (CRITIQUE)

NestJS parse le body JSON par défaut et détruit le raw body. Pour la vérification HMAC, il faut le raw body :

```typescript
// main.ts — modification requise
const app = await NestFactory.create(AppModule, { rawBody: true });
```

```typescript
// Dans le controller
@Post('supabase/auth')
async handleSupabaseWebhook(@Req() req: RawBodyRequest<Request>) {
  const rawBody = req.rawBody; // Buffer pour HMAC
  const parsed = JSON.parse(rawBody.toString());
  ...
}
```

### Vérification signature HMAC-SHA256

```typescript
import * as crypto from 'crypto';

function verifySupabaseSignature(rawBody: Buffer, signature: string, secret: string): boolean {
  const hmac = crypto.createHmac('sha256', secret);
  hmac.update(rawBody);
  const digest = hmac.digest('hex');
  const expected = Buffer.from(`sha256=${digest}`);
  const received = Buffer.from(signature);
  if (expected.length !== received.length) return false;
  return crypto.timingSafeEqual(expected, received);
}
```

### Réutilisation de `upsertUser()`

`RgpdService.upsertUser()` fait exactement ce qu'on veut. Options :
1. **Recommandé** : Déplacer `upsertUser()` dans un `UserRepository` partagé ou `IdentityService`
2. **Acceptable pour MVP** : Dupliquer le logic dans `UserSyncService` (dette technique à documenter)

### Structure `WebhookModule`

```
backend/src/modules/webhook/
├── webhook.module.ts
├── infrastructure/
│   ├── controllers/
│   │   └── supabase-webhook.controller.ts
│   └── guards/
│       └── supabase-webhook.guard.ts
└── application/
    └── services/
        └── user-sync.service.ts
        └── user-sync.service.spec.ts
```

### Configuration Dashboard Supabase

```
Dashboard → Database → Webhooks → Create new webhook
  Name: auth-user-sync
  Table: auth.users
  Events: INSERT, UPDATE, DELETE
  URL: https://YOUR_BACKEND_URL/webhooks/supabase/auth
  HTTP Headers: (aucun requis, signature dans body)
  Secret: (généré par Supabase, copier dans SUPABASE_WEBHOOK_SECRET)
```

⚠️ L'URL doit être **accessible publiquement** (pas localhost). En dev, utiliser ngrok ou Cloudflare Tunnel.

### Conformité ADR

- ADR-016 : Supabase = source de vérité auth. Le webhook consomme les events Supabase, ne modifie pas Supabase.
- Les champs auth (email, password) dans la table `users` backend sont **en lecture seule** pour les autres modules — seul `UserSyncService` les met à jour.

### Commandes test

```bash
cd pensieve/backend && npx jest src/modules/webhook/
```

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

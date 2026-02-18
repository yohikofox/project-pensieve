# Story 8.17: Configuration SMTP et Service Email Transactionnel Backend

Status: ready-for-dev

## Story

En tant qu'administrateur système,
je veux configurer un fournisseur SMTP personnalisé pour Supabase et créer un module email dans le backend NestJS,
afin que les emails d'authentification (reset password) soient envoyés sans limitation de débit et que le backend puisse envoyer des emails transactionnels futurs.

## Contexte

Supabase free tier limite l'envoi d'emails à **2 emails/heure** (rate limit non modifiable sans SMTP custom). Cela bloque les tests du flow forgot password et limitera les utilisateurs en production.

La solution est double :
1. Configurer un SMTP custom dans Supabase (Resend recommandé — 3 000 emails/mois gratuit)
2. Créer un `EmailModule` NestJS avec Nodemailer pour les futurs emails transactionnels backend

## Acceptance Criteria

1. **AC1 — SMTP Supabase configuré** : Supabase utilise un SMTP custom (Resend ou SendGrid). Le rate limit de 2 emails/heure est levé. Un email de test est envoyé avec succès depuis le dashboard Supabase.

2. **AC2 — Flow forgot password fonctionnel end-to-end** : L'email de reset password est reçu en moins de 30 secondes. Le lien ouvre l'app mobile sur l'écran ResetPassword (deep link `pensine://reset-password#...`).

3. **AC3 — EmailModule NestJS créé** : Un module `EmailModule` existe dans le backend avec un service injectable `EmailService`. Il utilise Nodemailer avec la config SMTP des variables d'environnement.

4. **AC4 — Variables d'environnement documentées** : Le fichier `.env.example` du backend est mis à jour avec les variables SMTP (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM`). Le `.env` local est configuré (non commité).

5. **AC5 — Email template reset password** : `EmailService` expose une méthode `sendPasswordResetEmail(email, resetLink)` avec un template HTML minimal et un fallback texte. Cette méthode est appelable mais n'est pas forcément utilisée en production (Supabase gère l'envoi via SMTP).

6. **AC6 — Tests unitaires EmailService** : Au moins 3 tests unitaires couvrent `sendPasswordResetEmail` (succès, échec SMTP, email invalide).

## Tasks / Subtasks

- [ ] T1 — Configurer le fournisseur SMTP dans Supabase (AC1, AC2)
  - [ ] T1.1 — Créer un compte Resend sur resend.com et vérifier le domaine (ou utiliser Gmail App Password pour dev)
  - [ ] T1.2 — Dans Dashboard Supabase → Project Settings → Auth → SMTP Settings : renseigner Host, Port, Username, Password, Sender email
  - [ ] T1.3 — Envoyer un email de test depuis le dashboard Supabase et vérifier réception
  - [ ] T1.4 — Tester le flow complet forgot password depuis l'app mobile

- [ ] T2 — Créer le `EmailModule` NestJS (AC3, AC4, AC5)
  - [ ] T2.1 — Créer `backend/src/modules/email/` avec structure DDD : `domain/`, `application/services/`, `infrastructure/`
  - [ ] T2.2 — Installer `nodemailer` et `@types/nodemailer` dans `backend/`
  - [ ] T2.3 — Créer `EmailService` injectable avec Nodemailer (config via `ConfigService`)
  - [ ] T2.4 — Implémenter `sendPasswordResetEmail(email: string, resetLink: string): Promise<void>`
  - [ ] T2.5 — Enregistrer `EmailModule` dans `AppModule`
  - [ ] T2.6 — Ajouter les variables SMTP dans `backend/.env.example`

- [ ] T3 — Tests unitaires (AC6)
  - [ ] T3.1 — Créer `email.service.spec.ts` avec mock Nodemailer
  - [ ] T3.2 — Test succès envoi (mock transporter.sendMail)
  - [ ] T3.3 — Test échec SMTP (mock rejette la promesse)
  - [ ] T3.4 — Test validation format email

## Dev Notes

### Fournisseurs SMTP recommandés

| Fournisseur | Free tier | Setup |
|-------------|-----------|-------|
| **Resend** (recommandé) | 3 000 emails/mois | `smtp.resend.com:587`, username: `resend` |
| **SendGrid** | 100 emails/jour | `smtp.sendgrid.net:587`, username: `apikey` |
| **Gmail App Password** | Usage personnel | `smtp.gmail.com:587` — pour dev/test seulement |

### Configuration Supabase SMTP

Dashboard → Authentication → SMTP Settings :
```
Host:           smtp.resend.com
Port:           587
Username:       resend
Password:       re_XXXXX (clé API Resend)
Sender email:   noreply@pensine.app
Sender name:    Pensine
```

Référence : [Source: _bmad-output/implementation-artifacts/docs/supabase-setup-instructions.md#SMTP-Configuration]

### Structure EmailModule backend

```
backend/src/modules/email/
├── email.module.ts
└── application/
    └── services/
        └── email.service.ts
        └── email.service.spec.ts
```

Pattern à suivre : `backend/src/modules/notification/` pour la structure modulaire NestJS.

### Variables d'environnement (à ajouter dans `backend/.env.example`)

```bash
# Email (SMTP)
SMTP_HOST=smtp.resend.com
SMTP_PORT=587
SMTP_USER=resend
SMTP_PASS=re_YOUR_API_KEY
SMTP_FROM="Pensine <noreply@pensine.app>"
```

### Deep link Supabase → Mobile

Template email Supabase doit contenir le redirect vers `pensine://reset-password`.
Configurer dans Dashboard → Authentication → Email Templates → Reset Password :
```
{{ .ConfirmationURL }}
```
doit pointer vers `pensine://reset-password` via le redirect configuré dans ADR-016.

Référence : [Source: _bmad-output/planning-artifacts/adrs/ADR-016-hybrid-architecture.md]

### SUPABASE_SERVICE_KEY dans .env

La variable `SUPABASE_SERVICE_KEY` est déjà présente dans le `.env` backend. Elle peut être utilisée pour appeler l'API admin Supabase (`/auth/v1/admin/users/{id}/recovery`) pour générer des liens de recovery en tests sans passer par l'email.

### Conformité ADR

- **ADR-016** : SMTP géré via Supabase Cloud, configuration custom autorisée [Source: ADR-016-hybrid-architecture.md#L48]
- **Pas de nouvel ADR requis** : l'email transactionnel est une déclinaison de l'infrastructure existante
- Module NestJS suit le pattern hexagonal standard du projet

### Tests

Framework : Jest + `@nestjs/testing`. Mocker Nodemailer avec `jest.mock('nodemailer')`.

```typescript
// Pattern mock Nodemailer
const sendMailMock = jest.fn().mockResolvedValue({ messageId: 'test-id' });
jest.mock('nodemailer', () => ({
  createTransport: jest.fn(() => ({ sendMail: sendMailMock })),
}));
```

Commande : `cd pensieve/backend && npx jest src/modules/email/`

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

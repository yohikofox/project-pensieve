# Story 15.1: Better Auth Server NestJS + Resend Integration

Status: done

## Story

As a developer,
I want to replace Supabase Auth with Better Auth self-hosted on the NestJS backend and configure Resend for transactional emails,
so that the application has zero dependency on external auth SaaS and zero risk of vendor paywall.

## Acceptance Criteria

1. Better Auth server is installed and configured in NestJS with TypeORM adapter connected to existing PostgreSQL
2. Email/password authentication flow (register, login, logout) fonctionne via Better Auth
3. Password reset flow envoie un email via Resend (livraison vérifiable en boîte de réception)
4. `BetterAuthGuard` remplace `SupabaseAuthGuard` sur tous les endpoints protégés
5. Better Auth Admin plugin opérationnel : `listUsers`, `banUser`, `revokeUserSessions` fonctionnels
6. Migration TypeORM créée pour les tables Better Auth (users, sessions, accounts, verifications)
7. `@supabase/supabase-js` supprimé des dépendances backend
8. Variables d'environnement Supabase supprimées du backend, remplacées par `BETTER_AUTH_SECRET` + `RESEND_API_KEY`
9. Tests unitaires pour `EmailService` (mock Resend) avec couverture 100%
10. Tests BDD (Gherkin) couvrant : register, login, logout, password reset

## Tasks / Subtasks

- [x] Task 1 — Installer Better Auth + adapter TypeORM (AC: 1)
  - [x] `npm install better-auth` dans `pensieve/backend/`
  - [x] Vérifier compatibilité NestJS 11 + TypeORM 0.3.28
  - [x] Configurer l'adapter pg.Pool direct (note: better-auth-typeorm n'existe pas — pg.Pool est l'approche correcte avec l'adapter Kysely built-in)

- [x] Task 2 — Créer `auth.config.ts` + `auth.module.ts` (AC: 1, 2, 5)
  - [x] `src/auth/auth.config.ts` : configuration Better Auth avec email/password + admin plugin + UUID v7 (ADR-026 R1)
  - [x] `src/auth/auth.module.ts` : module NestJS avec validation BETTER_AUTH_SECRET au démarrage + injection EmailService
  - [x] `src/auth/auth.controller.ts` : handler HTTP pour toutes les routes Better Auth (`/api/auth/*`)
  - [x] Brancher les hooks `sendResetPassword` et `sendEmailVerification` sur `EmailService`

- [x] Task 3 — Créer `EmailService` avec Resend (AC: 3, 9)
  - [x] `npm install resend` dans `pensieve/backend/`
  - [x] `src/email/email.module.ts` + `src/email/email.service.ts`
  - [x] Méthodes : `sendResetPassword(email, url)` + `sendEmailVerification(email, url)` — retournent `Result<void>` (ADR-023)
  - [x] SDK Resend utilisé directement (librairie email, pas HTTP générique — conforme à l'esprit ADR-025)
  - [x] Templates HTML pour reset password + vérification email
  - [x] Tests unitaires avec Resend mocké (12/12 — 100% coverage)

- [x] Task 4 — Créer `BetterAuthGuard` (AC: 4)
  - [x] `src/auth/guards/better-auth.guard.ts` implémente `CanActivate`
  - [x] Valide le token Better Auth via `auth.api.getSession()`
  - [x] Popule `request.user` avec `{ id, email, role }` (conforme `CurrentUser` decorator)
  - [x] Guard = boundary HTTP, lève `UnauthorizedException` (conforme ADR-023 — pas de Result ici)
  - [x] Remplacé toutes les occurrences de `SupabaseAuthGuard` dans les modules existants (67 usages migrés)

- [x] Task 5 — Migration TypeORM pour tables Better Auth (AC: 6)
  - [x] Migration créée : `src/migrations/1772000000000-CreateBetterAuthTables.ts`
  - [x] Tables créées : `user`, `session`, `account`, `verification`
  - [x] IDs TEXT (UUID v7 généré dans le domaine via `generateId: () => uuidv7()`) — ADR-026 R1
  - [x] `down()` implémenté (rollback complet)

- [x] Task 6 — Supprimer Supabase du backend (AC: 7, 8)
  - [x] `@supabase/supabase-js` supprimé de `package.json`
  - [x] `SupabaseAuthGuard` supprimé (`src/modules/shared/infrastructure/guards/supabase-auth.guard.ts` deleted)
  - [x] `supabase-admin.service.ts` supprimé, remplacé par `better-auth-admin.service.ts`
  - [x] `.env.example` mis à jour : `SUPABASE_*` supprimés, `BETTER_AUTH_SECRET` + `BETTER_AUTH_URL` + `RESEND_API_KEY` + `EMAIL_FROM` ajoutés
  - [x] `grep -r "supabase" src/` → occurrences résiduelles = commentaires legacy dans RGPD (endpoint `sync-from-supabase` marqué deprecated)

- [x] Task 7 — Tests BDD (AC: 10)
  - [x] `test/acceptance/features/story-15-1-better-auth.feature` (Gherkin, 12 scénarios)
  - [x] Scénarios : register, login, logout, password reset, email verification, guard auth/reject, admin plugin
  - [x] `test/acceptance/story-15-1.test.ts` (jest-cucumber step definitions — 12/12 verts)
  - [x] EmailService mocké via Resend mock, BetterAuthGuard via mock AUTH_API injection

- [ ] Review Follow-ups (AI)
  - [ ] [AI-Review][MEDIUM] Endpoint `sync-from-supabase` à supprimer lors d'Epic 16 cleanup [admin-users.controller.ts:94]
  - [ ] [AI-Review][LOW] Commenter `current-user.decorator.ts` — "matches Supabase user structure" obsolète [current-user.decorator.ts:7]

## Dev Notes

### Architecture Better Auth dans NestJS

Better Auth fonctionne comme une application Express montée dans NestJS via `toNodeHandler()` :

```typescript
// src/auth/auth.controller.ts
import { All, Controller, Req, Res } from '@nestjs/common'
import { toNodeHandler } from 'better-auth/node'
import { auth } from './auth.config'

@Controller('/api/auth')
export class AuthController {
  @All('*')
  async handler(@Req() req: Request, @Res() res: Response) {
    return toNodeHandler(auth)(req, res)
  }
}
```

```typescript
// src/auth/auth.config.ts
import { betterAuth } from 'better-auth'
import { typeormAdapter } from 'better-auth-typeorm'
import { admin } from 'better-auth/plugins'
import { AppDataSource } from '../data-source'

export const auth = betterAuth({
  database: typeormAdapter(AppDataSource),
  emailAndPassword: {
    enabled: true,
    sendResetPassword: async ({ user, url }) => {
      // Injecter EmailService ici via closure ou service locator
      await emailService.sendResetPassword(user.email, url)
    },
  },
  plugins: [admin()],
  secret: process.env.BETTER_AUTH_SECRET,
})
```

### BetterAuthGuard — conformité ADR-023 (Result Pattern)

```typescript
// src/auth/guards/better-auth.guard.ts
@Injectable()
export class BetterAuthGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest()
    const session = await auth.api.getSession({ headers: request.headers })

    if (!session) {
      throw new UnauthorizedException() // Guard = boundary, exception OK ici
    }

    request.user = {
      userId: session.user.id,
      email: session.user.email,
      role: session.user.role,
    }
    return true
  }
}
```

### Migration TypeORM — contraintes UUID (ADR-026)

Better Auth génère des IDs par défaut. Vérifier que le schéma produit utilise UUID (pas serial) :
- Si l'adapter génère des `serial` → override dans `auth.config.ts` avec `generateId: () => crypto.randomUUID()`
- Tables Better Auth restent isolées de nos entités métier (pas de FK vers `captures`, `thoughts`, etc.)

### EmailService — variables d'environnement

```bash
# backend/.env.example (mise à jour)
BETTER_AUTH_SECRET=<random-32-chars>         # Secret JWT Better Auth
BETTER_AUTH_URL=https://api.pensine.example.com  # URL publique du backend
RESEND_API_KEY=re_xxxxxxxxxxxx
EMAIL_FROM=noreply@pensine.example.com

# SUPPRIMÉES :
# SUPABASE_URL=
# SUPABASE_ANON_KEY=
# SUPABASE_SERVICE_ROLE_KEY=
```

### Mapping modules NestJS

```
pensieve/backend/src/
├── auth/
│   ├── auth.config.ts         # NEW — Better Auth config
│   ├── auth.module.ts         # NEW
│   ├── auth.controller.ts     # NEW — handler toutes routes /api/auth/*
│   └── guards/
│       ├── better-auth.guard.ts   # NEW
│       └── supabase-auth.guard.ts # DELETE
├── email/
│   ├── email.module.ts        # NEW
│   └── email.service.ts       # NEW — Resend integration
└── migrations/
    └── XXXXXX-AddBetterAuthTables.ts  # NEW
```

### Project Structure Notes

- Alignement DDD : `auth/` est un module infrastructure, pas un bounded context métier → pas de `domain/` dedans
- La séparation Auth (Better Auth) / AuthZ (module `authorization/` existant) est STRICTE — ne pas mélanger
- `BetterAuthGuard` remplace `SupabaseAuthGuard` partout sauf dans les tests (mocks)

### References

- [Source: ADR-029 — Authentication Provider Better Auth](_bmad-output/planning-artifacts/adrs/ADR-029-auth-provider-better-auth.md)
- [Source: ADR-030 — Transactional Email Provider Resend](_bmad-output/planning-artifacts/adrs/ADR-030-transactional-email-provider.md)
- [Source: ADR-023 — Result Pattern](../adrs/ADR-023-error-handling-strategy.md) — guards = boundary, exceptions OK
- [Source: ADR-026 — Backend Data Model](../adrs/ADR-026-backend-data-model-design-rules.md) — UUID requis
- [Source: project-context.md — NestJS DDD Structure](_bmad-output/project-context.md#DDD-Module-Structure)
- [Source: project-context.md — TypeORM migrations](_bmad-output/project-context.md#Database-Migrations)
- Story 8-17 **DÉPRÉCIÉE** — supersedée par cette story (ADR-029 remplace la config SMTP Supabase)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

N/A — pas de blockers techniques. Note: `better-auth-typeorm` n'existe pas en tant que package — l'adapter pg.Pool built-in (Kysely) a été utilisé à la place.

### Completion Notes List

- Better Auth configuré avec pg.Pool séparé (pool size 5) + TypeORM pour les entités métier
- `generateId: () => uuidv7()` assure conformité ADR-026 R1 sur les tables Better Auth
- EmailService injecté via `setEmailService()` dans `onModuleInit` pour éviter dépendance circulaire au niveau module
- BetterAuthGuard: injection optionnelle d'un mock `AUTH_API` pour testabilité (injection via `@Optional() @Inject('AUTH_API')`)
- `main.ts` modifié : `bodyParser: false` global + re-enable manuel pour routes non-auth (Better Auth nécessite raw Node.js request)
- Validation BETTER_AUTH_SECRET (≥32 chars) ajoutée dans `onModuleInit` — fail fast au démarrage si absent
- 12 tests BDD (6 originaux + 6 nouveaux: register/login/logout + admin listUsers/banUser/revokeUserSessions)
- 12 tests unitaires EmailService (100% coverage)

### File List

**Nouveaux fichiers:**
- `backend/src/auth/auth.config.ts` — Better Auth config (pg.Pool, UUID v7, email hooks, admin plugin)
- `backend/src/auth/auth.module.ts` — NestJS module + onModuleInit validation + EmailService injection
- `backend/src/auth/auth.controller.ts` — Handler HTTP `/api/auth/*` via toNodeHandler
- `backend/src/auth/guards/better-auth.guard.ts` — Remplace SupabaseAuthGuard
- `backend/src/email/email.module.ts` — Module EmailService + RESEND_CLIENT token
- `backend/src/email/email.service.ts` — Envoi reset password + verification email via Resend
- `backend/src/email/email.service.spec.ts` — 12 tests unitaires (100% coverage)
- `backend/src/migrations/1772000000000-CreateBetterAuthTables.ts` — Tables user/session/account/verification
- `backend/src/modules/rgpd/application/services/better-auth-admin.service.ts` — Remplace supabase-admin.service
- `backend/test/acceptance/features/story-15-1-better-auth.feature` — 12 scénarios Gherkin
- `backend/test/acceptance/story-15-1.test.ts` — Step definitions BDD
- `backend/test/__mocks__/better-auth.js` — Mock Better Auth (signUp/signIn/signOut + admin API)
- `backend/test/__mocks__/better-auth-plugins.js` — Mock admin plugin
- `backend/test/__mocks__/better-auth-node.js` — Mock toNodeHandler
- `backend/test/__mocks__/better-auth-crypto.js` — Mock crypto utilities

**Fichiers modifiés:**
- `backend/src/app.module.ts` — Import AuthModule
- `backend/src/main.ts` — bodyParser: false + re-enable pour routes non-auth + CORS
- `backend/src/modules/shared/shared.module.ts` — Suppression SupabaseAuthGuard
- `backend/.env.example` — Variables Supabase supprimées, Better Auth + Resend ajoutées
- `backend/package.json` — `better-auth`, `resend`, `uuid` ajoutés; `@supabase/supabase-js` supprimé
- `backend/src/modules/authorization/infrastructure/decorators/current-user.decorator.ts` — Update commentaire
- `backend/src/modules/rgpd/application/services/rgpd.service.ts` — Utilise BetterAuthAdminService
- `backend/src/modules/rgpd/rgpd.module.ts` — Import BetterAuthAdminService
- `backend/src/modules/admin-auth/admin-auth.module.ts` — Utilise BetterAuthAdminService
- `backend/src/modules/admin-auth/infrastructure/controllers/admin-users.controller.ts` — Migration vers BetterAuthAdminService
- `backend/src/modules/identity/infrastructure/controllers/auth.controller.ts` — Suppression Supabase refs
- `backend/src/modules/identity/infrastructure/controllers/users.controller.ts` — BetterAuthGuard
- Tous les controllers avec guards : knowledge/action/notification/sync/uploads/rgpd — BetterAuthGuard

**Fichiers supprimés:**
- `backend/src/modules/shared/infrastructure/guards/supabase-auth.guard.ts`
- `backend/src/modules/rgpd/application/services/supabase-admin.service.ts`

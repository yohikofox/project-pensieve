# Story 15.1: Better Auth Server NestJS + Resend Integration

Status: ready-for-dev

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

- [ ] Task 1 — Installer Better Auth + adapter TypeORM (AC: 1)
  - [ ] `npm install better-auth` dans `pensieve/backend/`
  - [ ] Vérifier compatibilité NestJS 11 + TypeORM 0.3.28
  - [ ] Configurer l'adapter TypeORM : `betterAuth({ database: typeormAdapter(dataSource) })`

- [ ] Task 2 — Créer `auth.config.ts` + `auth.module.ts` (AC: 1, 2, 5)
  - [ ] `src/auth/auth.config.ts` : configuration Better Auth avec email/password + admin plugin
  - [ ] `src/auth/auth.module.ts` : module NestJS qui expose Better Auth
  - [ ] `src/auth/auth.controller.ts` : handler HTTP pour toutes les routes Better Auth (`/api/auth/[...all]`)
  - [ ] Brancher le hook `sendResetPassword` sur `EmailService`

- [ ] Task 3 — Créer `EmailService` avec Resend (AC: 3, 9)
  - [ ] `npm install resend` dans `pensieve/backend/`
  - [ ] `src/email/email.module.ts` + `src/email/email.service.ts`
  - [ ] Méthodes : `sendResetPassword(email, url)` + `sendEmailVerification(email, url)`
  - [ ] Utiliser `fetch` natif si Resend SDK n'est pas un wrapper fetch (vérifier — sinon SDK OK car c'est une librairie email, pas HTTP générique)
  - [ ] Templates HTML pour reset password + vérification email
  - [ ] Tests unitaires avec Resend mocké (100% coverage)

- [ ] Task 4 — Créer `BetterAuthGuard` (AC: 4)
  - [ ] `src/auth/guards/better-auth.guard.ts` implémente `CanActivate`
  - [ ] Valide le token Better Auth via `auth.api.getSession()`
  - [ ] Popule `request.user` avec `userId` + `email` + `role`
  - [ ] Retourne `Result<User>` en cas d'erreur (conforme ADR-023)
  - [ ] Remplacer toutes les occurrences de `SupabaseAuthGuard` dans les modules existants

- [ ] Task 5 — Migration TypeORM pour tables Better Auth (AC: 6)
  - [ ] Générer migration : `npx typeorm migration:generate src/migrations/AddBetterAuthTables -d src/data-source.ts`
  - [ ] Vérifier tables créées : `users`, `sessions`, `accounts`, `verifications`
  - [ ] S'assurer que `users.id` est UUID conforme ADR-026 (pas serial/int)
  - [ ] Tester migration up + down

- [ ] Task 6 — Supprimer Supabase du backend (AC: 7, 8)
  - [ ] `npm uninstall @supabase/supabase-js` dans `pensieve/backend/`
  - [ ] Supprimer `SupabaseAuthGuard` + tout fichier Supabase backend
  - [ ] Mettre à jour `.env.example` : supprimer `SUPABASE_*`, ajouter `BETTER_AUTH_SECRET` + `BETTER_AUTH_URL` + `RESEND_API_KEY` + `EMAIL_FROM`
  - [ ] Vérifier `grep -r "supabase" src/` → zéro occurrences

- [ ] Task 7 — Tests BDD (AC: 10)
  - [ ] Créer `test/acceptance/features/story-15-1.feature` (Gherkin)
  - [ ] Scénarios : register, login, logout, password reset (email envoyé)
  - [ ] Créer `test/acceptance/story-15-1.test.ts` (jest-cucumber step definitions)
  - [ ] Mock EmailService pour éviter vrais envois en test

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

### Completion Notes List

### File List

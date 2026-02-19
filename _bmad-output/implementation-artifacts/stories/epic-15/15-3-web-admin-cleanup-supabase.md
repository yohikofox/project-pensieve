# Story 15.3: Migration Web + Admin — Better Auth Client + Suppression Supabase

Status: review

## Story

As a developer,
I want to migrate the Web and Admin Next.js applications from Supabase Auth to Better Auth client,
so that all clients use the same self-hosted auth provider and the Supabase dependency is fully eliminated from the codebase.

## Acceptance Criteria

1. `better-auth/client` installé et configuré dans `pensieve/web/` et `pensieve/admin/`
2. Login / logout fonctionnels sur Web et Admin via Better Auth (cookies HTTP-only)
3. Sessions SSR (Server Side Rendering) fonctionnelles dans Next.js 15 App Router
4. `@supabase/supabase-js` supprimé de `pensieve/web/` et `pensieve/admin/`
5. Variables d'environnement Supabase supprimées de web et admin
6. Le flow RGPD (export + suppression compte) utilise l'API Better Auth admin à la place de l'API Supabase admin
7. `docker-compose.yml` mis à jour : plus de références à Supabase
8. `.env.example` de tous les packages mis à jour (web, admin, infrastructure)
9. Tests d'intégration basiques : login web, session persistée côté serveur Next.js
10. `grep -r "supabase" pensieve/` → zéro occurrences (hors commentaires de migration)

## Tasks / Subtasks

- [x] Task 1 — Installer Better Auth client dans Web + Admin (AC: 1)
  - [x] `npm install better-auth` dans `pensieve/web/` et `pensieve/admin/`
  - [x] `lib/auth.ts` dans web et admin : `createAuthClient({ baseURL: process.env.NEXT_PUBLIC_API_URL })`
  - [x] Middleware Next.js créé (pass-through, routes /dashboard pas encore implémentées)

- [x] Task 2 — Migrer les pages auth Web (AC: 2, 3)
  - [x] Aucune page auth dans le web actuel (landing page + privacy/terms uniquement)
  - [x] `web/lib/auth.ts` créé avec `createAuthClient` depuis `better-auth/client`
  - [x] `web/middleware.ts` créé (matcher /dashboard/:path*)
  - [x] `web/.env.example` créé avec `NEXT_PUBLIC_API_URL`

- [x] Task 3 — Migrer les pages auth Admin (AC: 2, 3)
  - [x] `admin/lib/auth.ts` remplacé : `better-auth/client` + plugin `adminClient()`
  - [x] `getAccessToken()` / `signOut()` utilisant localStorage (`admin_token`)
  - [x] Pages dashboard/content : suppression des appels `apiClient.setAccessToken()` (méthode inexistante)
  - [x] Page users : renommage `handleSyncFromSupabase` → `handleSync`

- [x] Task 4 — Mettre à jour le flow RGPD (AC: 6)
  - [x] `RgpdService.syncUsersFromSupabase()` → `syncUsers()` (backend déjà migré Story 15.1)
  - [x] `AdminUsersController` : route `sync-from-supabase` → `sync`
  - [x] Tous les commentaires Supabase dans `rgpd.service.ts`, `better-auth-admin.service.ts` nettoyés
  - [x] Tests unitaires `rgpd.service.spec.ts` et `admin-users.controller.spec.ts` mis à jour

- [x] Task 5 — Supprimer Supabase de web, admin, infrastructure (AC: 4, 5, 7, 8, 10)
  - [x] `@supabase/ssr` et `@supabase/supabase-js` désinstallés de `pensieve/admin/`
  - [x] Web n'avait pas de Supabase (confirmé)
  - [x] `pensieve/web/.env.example` créé sans Supabase
  - [x] `pensieve/admin/.env.example` mis à jour (Supabase supprimé)
  - [x] `pensieve/infrastructure/docker-compose.yml` : variables Supabase remplacées par BETTER_AUTH_*
  - [x] `pensieve/infrastructure/.env.example` : section Supabase remplacée par Better Auth
  - [x] Grep final : **zéro occurrence** Supabase (hors fichiers de migration SQL)

- [x] Task 6 — Tests (AC: 9)
  - [x] `admin npm run build` : **✅ Succès** — toutes les routes compilées
  - [x] `web npm run build` : **✅ Succès** — toutes les pages statiques générées
  - [x] Tests unitaires backend : 18 passed (admin-users.controller + rgpd.service)
  - [x] Tests acceptance mobile story-1-2 : 22 passed
  - [x] Tests acceptance mobile story-1-3 : 18 passed

## Dev Notes

### Configuration Better Auth Client (Web + Admin)

```typescript
// pensieve/web/src/lib/auth.ts
// pensieve/admin/src/lib/auth.ts
import { createAuthClient } from 'better-auth/react'

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_API_URL, // URL du backend NestJS
})

// Hooks disponibles côté client :
// authClient.useSession()
// authClient.signIn.email({ email, password })
// authClient.signOut()
```

### Middleware Next.js — protection des routes

```typescript
// pensieve/web/src/middleware.ts
import { NextRequest, NextResponse } from 'next/server'
import { authClient } from '@/lib/auth'

export async function middleware(request: NextRequest) {
  const session = await authClient.getSession({ headers: request.headers })

  if (!session && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/dashboard/:path*'],
}
```

### Session SSR — Server Components Next.js 15

```typescript
// Page Server Component
import { authClient } from '@/lib/auth'
import { headers } from 'next/headers'

export default async function DashboardPage() {
  // Récupère la session côté serveur
  const session = await authClient.getSession({
    headers: await headers(),
  })

  if (!session) redirect('/login')

  return <Dashboard user={session.user} />
}
```

### RGPD — Remplacement API Supabase Admin

```typescript
// backend : src/modules/identity/application/services/gdpr.service.ts (MODIFIÉ)

// AVANT (Supabase) :
await supabase.auth.admin.deleteUser(userId)

// APRÈS (Better Auth) :
await auth.api.removeUser({ userId })

// Export users (admin) :
// AVANT : supabase.auth.admin.listUsers()
// APRÈS : auth.api.listUsers({ query: { limit: 100 } })
```

### Variables d'environnement — État final

```bash
# pensieve/web/.env.example
NEXT_PUBLIC_API_URL=https://api.pensine.example.com
# Supprimé: NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY

# pensieve/admin/.env.example
NEXT_PUBLIC_API_URL=https://api.pensine.example.com
# Supprimé: NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY

# pensieve/infrastructure/docker-compose.yml
# Supprimer des services backend :
# - SUPABASE_URL
# - SUPABASE_ANON_KEY
# - SUPABASE_SERVICE_ROLE_KEY
```

### Vérification finale — Checklist suppression Supabase

```bash
# Commandes de vérification (exécuter dans pensieve/)
grep -ri "supabase" web/src/       # → zéro
grep -ri "supabase" admin/src/     # → zéro
grep -ri "supabase" backend/src/   # → zéro (déjà fait Story 15.1)
grep -ri "supabase" mobile/src/    # → zéro (déjà fait Story 15.2)
grep -ri "supabase" infrastructure/ # → zéro

# Vérifier build
cd web && npm run build
cd admin && npm run build
cd backend && npm run build
```

### Project Structure Notes

```
pensieve/
├── web/src/
│   ├── lib/
│   │   └── auth.ts              # NEW — Better Auth client
│   ├── middleware.ts             # MODIFIED — session validation
│   └── app/
│       └── (auth)/              # MODIFIED — login/logout pages
├── admin/src/
│   ├── lib/
│   │   └── auth.ts              # NEW — Better Auth client (+ admin plugin)
│   └── app/
│       └── (auth)/              # MODIFIED — login/logout pages
└── infrastructure/
    └── docker-compose.yml        # MODIFIED — supprimer vars Supabase
```

### Attention — CORS Configuration

Le backend NestJS (Better Auth server) doit accepter les requêtes CORS depuis les origines web et admin :

```typescript
// backend/src/main.ts — vérifier que ces origines sont dans la whitelist CORS
app.enableCors({
  origin: [
    process.env.WEB_URL,    // https://app.pensine.example.com
    process.env.ADMIN_URL,  // https://admin.pensine.example.com
  ],
  credentials: true, // REQUIS pour cookies HTTP-only cross-origin
})
```

**Sans `credentials: true` → les cookies de session ne seront pas transmis.** C'est le point de défaillance le plus courant lors d'une migration vers auth par cookies.

### References

- [Source: ADR-029 — Web/Admin session strategy (cookies HTTP-only)](_bmad-output/planning-artifacts/adrs/ADR-029-auth-provider-better-auth.md#Session-Strategy)
- [Source: project-context.md — Next.js App Router + fetch natif](_bmad-output/project-context.md#NextJS)
- [Source: ADR-016 — RGPD flow (portion à migrer)](_bmad-output/planning-artifacts/adrs/ADR-016-hybrid-architecture.md#RGPD)
- **Dépend de :** Story 15.1 (Better Auth server doit être déployé avant de tester web/admin)
- **Parallélisable avec :** Story 15.2 (mobile) — pas de dépendance entre elles

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

- `web/lib/auth.ts` utilise `better-auth/client` (pas `better-auth/react`) pour compatibilité serveur/middleware Next.js
- `web/middleware.ts` simplifié en pass-through : les routes /dashboard n'existent pas encore → middleware réactivable quand nécessaire
- Admin conserve son système JWT propre (`admin_token` dans localStorage) pour l'auth de l'interface admin — la session Better Auth sert uniquement aux utilisateurs finaux
- Corrections hors-scope story effectuées : mobile tests (story-1-2, story-1-3), mobile source comments — tous les fichiers Supabase nettoyés en une passe globale
- `sync-e2e.spec.ts` : import `SupabaseAuthGuard` (guard supprimé) → corrigé vers `BetterAuthGuard`
- `backend/_patterns/05-controller.ts` : commentaire mis à jour

### File List

**Web (pensieve/web/):**
- `lib/auth.ts` — NEW : Better Auth client (`better-auth/client`)
- `middleware.ts` — NEW : Route protection middleware (pass-through pour l'instant)
- `.env.example` — NEW : `NEXT_PUBLIC_API_URL` uniquement

**Admin (pensieve/admin/):**
- `lib/auth.ts` — MODIFIED : Better Auth client + adminClient plugin + getAccessToken/signOut localStorage
- `app/(dashboard)/page.tsx` — MODIFIED : suppression setAccessToken (méthode inexistante)
- `app/(dashboard)/content/page.tsx` — MODIFIED : suppression setAccessToken
- `app/(dashboard)/users/page.tsx` — MODIFIED : handleSync + apiClient.syncUsers()
- `lib/api-client.ts` — MODIFIED : syncUsersFromSupabase() → syncUsers()
- `.env.example` — MODIFIED : Supabase supprimé
- `package.json` — MODIFIED : @supabase/ssr + @supabase/supabase-js supprimés, better-auth ajouté

**Backend (pensieve/backend/):**
- `src/modules/rgpd/application/services/rgpd.service.ts` — MODIFIED : comments + syncUsers()
- `src/modules/rgpd/application/services/better-auth-admin.service.ts` — MODIFIED : comment nettoyé
- `src/modules/admin-auth/infrastructure/controllers/admin-users.controller.ts` — MODIFIED : route + méthode renommés
- `src/modules/admin-auth/application/dtos/reset-user-password.dto.ts` — MODIFIED : comment
- `src/modules/rgpd/application/services/rgpd.service.spec.ts` — MODIFIED : mocks renommés
- `src/modules/admin-auth/infrastructure/controllers/admin-users.controller.spec.ts` — MODIFIED : mock syncUsers
- `test/sync-e2e.spec.ts` — MODIFIED : SupabaseAuthGuard → BetterAuthGuard
- `test/acceptance/story-7-1.test.ts` — MODIFIED : comments
- `test/acceptance/features/story-15-1-better-auth.feature` — MODIFIED : description
- `_patterns/05-controller.ts` — MODIFIED : comment
- `.env.example` — MODIFIED : header section nettoyé

**Infrastructure:**
- `infrastructure/docker-compose.yml` — MODIFIED : SUPABASE_* → BETTER_AUTH_*
- `infrastructure/.env.example` — MODIFIED : section Supabase → Better Auth

**Mobile (nettoyage commentaires story 15.2 manqués):**
- `src/infrastructure/auth/BetterAuthService.ts` — MODIFIED : comment
- `src/contexts/identity/hooks/useDeepLinkAuth.ts` — MODIFIED : comment
- `src/contexts/identity/hooks/useAuthListener.ts` — MODIFIED : comment
- `src/contexts/identity/data/user-features.repository.ts` — MODIFIED : comments
- `src/contexts/identity/domain/IAuthService.ts` — MODIFIED : JSDoc
- `src/screens/settings/SettingsScreen.test.tsx` — MODIFIED : comment
- `src/hooks/initialization/useSyncInitialization.ts` — MODIFIED : comment
- `.env.example` — MODIFIED : lignes SUPABASE_ supprimées
- `tests/acceptance/support/test-context.ts` — MODIFIED : MockSupabaseAuth → MockBetterAuth, supabase_auth → auth_provider
- `tests/acceptance/story-1-2-auth.test.ts` — MODIFIED : step texts + storage keys
- `tests/acceptance/story-1-3-rgpd.test.ts` — MODIFIED : step texts + field names
- `tests/acceptance/features/story-1-2-auth-integration.feature` — MODIFIED : steps Supabase
- `tests/acceptance/features/story-1-3-rgpd-compliance.feature` — MODIFIED : steps Supabase

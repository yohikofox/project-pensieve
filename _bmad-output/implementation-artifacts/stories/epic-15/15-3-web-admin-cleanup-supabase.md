# Story 15.3: Migration Web + Admin — Better Auth Client + Suppression Supabase

Status: ready-for-dev

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

- [ ] Task 1 — Installer Better Auth client dans Web + Admin (AC: 1)
  - [ ] `npm install better-auth` dans `pensieve/web/` et `pensieve/admin/`
  - [ ] `src/lib/auth.ts` dans web et admin : `createAuthClient({ baseURL: process.env.NEXT_PUBLIC_API_URL })`
  - [ ] Configurer le cookie handler Better Auth pour Next.js (SSR-compatible)

- [ ] Task 2 — Migrer les pages auth Web (AC: 2, 3)
  - [ ] Identifier les pages/composants auth existants dans `pensieve/web/`
  - [ ] Remplacer les appels Supabase par les hooks/calls Better Auth
  - [ ] `signIn()` → `authClient.signIn.email({ email, password })`
  - [ ] `signOut()` → `authClient.signOut()`
  - [ ] Vérifier que les cookies HTTP-only sont bien posés par le server Better Auth (NestJS)
  - [ ] Middleware Next.js : valider la session via Better Auth pour les routes protégées

- [ ] Task 3 — Migrer les pages auth Admin (AC: 2, 3)
  - [ ] Même démarche que Task 2 pour `pensieve/admin/`
  - [ ] Plugin admin Better Auth : configurer `authClient` avec les méthodes admin (`listUsers`, `banUser`, etc.)
  - [ ] Vérifier que l'interface admin de gestion des users utilise le plugin admin Better Auth

- [ ] Task 4 — Mettre à jour le flow RGPD (AC: 6)
  - [ ] Localiser le flow RGPD (Story 1-3 v2 : export + suppression compte)
  - [ ] Backend : remplacer `supabase.auth.admin.deleteUser(userId)` par `auth.api.removeUser({ userId })`
  - [ ] Backend : remplacer `supabase.auth.admin.listUsers()` par `auth.api.listUsers()`
  - [ ] Tester export RGPD + suppression compte de bout en bout

- [ ] Task 5 — Supprimer Supabase de web, admin, infrastructure (AC: 4, 5, 7, 8, 10)
  - [ ] `npm uninstall @supabase/supabase-js` dans `pensieve/web/` et `pensieve/admin/`
  - [ ] Supprimer tous fichiers Supabase dans web et admin
  - [ ] Mettre à jour `pensieve/web/.env.example` : supprimer `NEXT_PUBLIC_SUPABASE_*`, ajouter `NEXT_PUBLIC_API_URL`
  - [ ] Mettre à jour `pensieve/admin/.env.example` : même chose
  - [ ] Mettre à jour `pensieve/infrastructure/docker-compose.yml` : supprimer variables Supabase du service backend
  - [ ] Vérification finale : `grep -ri "supabase" pensieve/` → zéro occurrences actives

- [ ] Task 6 — Tests (AC: 9)
  - [ ] Tests d'intégration Next.js : middleware auth, session persistée, redirect si non authentifié
  - [ ] Vérifier que le build Next.js passe sans erreur (`npm run build`)
  - [ ] Vérifier que TypeScript compile sans erreur dans web et admin

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

### File List

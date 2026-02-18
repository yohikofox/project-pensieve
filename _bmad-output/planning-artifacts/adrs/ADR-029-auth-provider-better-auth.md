---
adr: ADR-029
title: "Authentication Provider — Better Auth Self-Hosted (Révision Auth de ADR-016)"
date: 2026-02-18
status: "✅ Accepted"
supersedes: "ADR-016 (portion Auth uniquement — MinIO Storage reste valide)"
context: "Phase 4 - Implementation - Remplacement Supabase Auth"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-029: Authentication Provider — Better Auth Self-Hosted

**Status:** ✅ ACCEPTÉ
**Date:** 2026-02-18
**Supersède :** ADR-016 (portion Auth uniquement — la partie MinIO/Storage reste inchangée)

---

## Context & Problem

**Problème identifié :**

ADR-016 a choisi Supabase Cloud pour l'authentification avec comme justification principale le time-to-market MVP et le social login. Plusieurs frictions sont apparues à l'usage :

1. **Rate limiting SMTP** : Supabase Free tier limite à 2-3 emails/heure — insuffisant pour les flows d'auth (password reset) même à faible volume.
2. **Risque paywall** : Le free tier Supabase peut changer. Dépendre d'un service externe pour une brique critique (auth) sans en contrôler le pricing crée un risque d'rug-pull sur une app qui ne génère pas de revenus.
3. **Usage réel** : Supabase n'est utilisé que pour l'auth email/password — aucune feature OAuth, aucun social login actif. Le ratio "valeur reçue / risque vendor lock-in" est défavorable.
4. **Complexité architecturale** : La validation JWT Supabase dans NestJS crée une dépendance au service cloud pour chaque request (ou nécessite un cache de la clé publique).

**Contraintes confirmées :**
- Budget prod : €0 pour ≥500 users
- Pas de migration users (aucun utilisateur prod actif hors compte dev)
- Social login : non requis aujourd'hui, mais prévu en Post-MVP
- Architecture multi-client : Mobile (React Native), Web (Next.js), Admin (Next.js), Backend (NestJS)
- Mobile offline-first : token expiry strategy adaptée (cf. Decision)
- RBAC complexe : rôles + tiers + permissions individuelles gérés par le module AuthZ NestJS existant

---

## Decision

**Better Auth self-hosted, server dans NestJS backend.**

Better Auth est une librairie TypeScript open-source qui tourne sur notre infrastructure existante. Zéro dépendance à un service externe pour l'authentification.

### Topologie

```
                    ┌─────────────────────────────┐
                    │   Better Auth Server        │
                    │   (NestJS — Express adapter)│
                    │   api.pensine.example.local  │
                    └──────────────┬──────────────┘
                                   │
           ┌───────────────────────┼────────────────────────┐
           │                       │                        │
    ┌──────▼───────┐     ┌─────────▼──────┐     ┌──────────▼─────┐
    │    Mobile    │     │      Web       │     │     Admin      │
    │ React Native │     │  Next.js 15   │     │   Next.js      │
    │  Expo SDK 54 │     │  Cookies SSR  │     │  Cookies SSR   │
    │  JWT Tokens  │     │               │     │                │
    │  SecureStore │     └───────────────┘     └────────────────┘
    └──────────────┘
```

### Configuration Better Auth (NestJS)

```typescript
// src/auth/auth.config.ts
import { betterAuth } from 'better-auth'
import { admin } from 'better-auth/plugins'
import { typeormAdapter } from 'better-auth-typeorm' // adapter TypeORM

export const auth = betterAuth({
  database: typeormAdapter(dataSource),

  emailAndPassword: {
    enabled: true,
    sendResetPassword: async ({ user, url }) => {
      await emailService.sendResetPassword(user.email, url)
      // emailService utilise Resend (ADR-030)
    }
  },

  plugins: [
    admin()  // User lifecycle : ban, list, impersonate, sessions
  ],

  // OAuth futur : ajouter ici sans refacto architectural
  // socialProviders: {
  //   google: { clientId: '...', clientSecret: '...' },
  //   apple:  { clientId: '...', clientSecret: '...' }
  // }
})
```

### Session Strategy par Client

| Client | Mécanisme | Stockage | Sécurité |
|--------|-----------|----------|----------|
| **Web** (Next.js) | HTTP-only cookies | Navigateur | CSRF protection built-in |
| **Admin** (Next.js) | HTTP-only cookies | Navigateur | CSRF protection built-in |
| **Mobile** (React Native) | JWT access token + refresh token | Expo SecureStore | Conforme ADR-022 (pas AsyncStorage) |
| **Backend** (NestJS) | Validation locale du token | Mémoire | Pas d'appel cloud |

### Stratégie Token Offline (Mobile)

```typescript
// mobile/src/infrastructure/auth/AuthTokenManager.ts

class AuthTokenManager {
  async getValidToken(): Promise<string> {
    const token = await SecureStore.getItemAsync('access_token')
    const expiresAt = await SecureStore.getItemAsync('token_expires_at')

    if (!isExpired(expiresAt)) {
      return token  // Token valide → utiliser directement
    }

    // Token expiré → tenter refresh
    try {
      return await this.refreshToken()
    } catch (error) {
      if (isNetworkError(error)) {
        // Hors réseau : token reste valide jusqu'à 23:59 du jour courant
        const endOfDay = getEndOfCurrentDay() // 23:59:59 local time
        if (Date.now() < endOfDay) {
          return token  // Pas de réseau mais toujours aujourd'hui → OK
        }
        throw new AuthExpiredError('Session expirée — reconnexion requise')
      }
      throw error  // Erreur auth réelle (token révoqué, etc.) → logout
    }
  }

  private getEndOfCurrentDay(): number {
    const now = new Date()
    return new Date(
      now.getFullYear(), now.getMonth(), now.getDate(),
      23, 59, 59, 999
    ).getTime()
  }
}
```

**Règle offline :** Si le refresh échoue à cause du réseau ET que nous sommes encore le même jour → l'app reste utilisable (opérations locales). La sync reprend au retour réseau.

### Séparation Auth / AuthZ

```
Better Auth                         NestJS AuthZ Module
──────────────────────────          ──────────────────────────────────
"Qui es-tu ?" (Authentication)      "Que peux-tu faire ?" (Authorization)

- Identity (email, password)        - Roles (admin, moderator, user...)
- Sessions / tokens / refresh       - Tiers d'abonnement (free, pro...)
- Password reset via email          - Permissions individuelles (PBAC)
- Admin plugin : user lifecycle     - ACL (accès ressource spécifique)
  (ban, list, impersonate, roles)   - Guards NestJS (CanActivate)
```

Le JWT Better Auth contient uniquement : `userId`, `email`, `role` (tag basique). Les permissions complètes sont chargées depuis la DB par le module AuthZ NestJS.

### Better Auth Admin Plugin

Le plugin `admin` gère le **lifecycle utilisateur**, complémentaire (non concurrent) au RBAC NestJS :

| Feature | Utilité |
|---------|---------|
| `listUsers` | Dashboard admin — filtrer/chercher users |
| `createUser` | Créer user sans onboarding |
| `banUser` / `unbanUser` | Bloquer un compte abusif |
| `impersonateUser` | Debug en tant qu'un user |
| `revokeUserSessions` | Forcer déconnexion (sécurité) |
| `setRole` | Tag basique `user` / `admin` |

### Extensibilité OAuth (Post-MVP)

Better Auth est plugin-based. Ajouter Google/Apple Sign-In = 5 lignes dans `auth.config.ts`. Zéro refacto architectural, zéro changement de schéma de données.

---

## Options Considérées

| Critère | Supabase Auth (actuel) | **Better Auth** (choisi) | Clerk | Auth.js |
|---------|------------------------|--------------------------|-------|---------|
| **Coût 500 users** | €0 (risque paywall) | **€0 garanti** | €0 (paywall <10K) | €0 |
| **Self-hosted** | ❌ | **✅** | ❌ | ✅ partiel |
| **TypeScript natif** | ✅ | **✅** | ✅ | ✅ |
| **React Native** | ✅ | **✅** | ✅ | ⚠️ limité |
| **NestJS adapter** | ✅ | **✅** | ❌ | ⚠️ |
| **OAuth extensible** | ✅ | **✅ plugin** | ✅ | ✅ |
| **Risque vendor lock-in** | ⚠️ élevé | **✅ aucun** | ⚠️ élevé | ✅ faible |
| **SMTP custom** | ⚠️ rate limited | **✅ ADR-030** | ✅ | ✅ |
| **Score** | 6/10 | **9/10** | 6/10 | 7/10 |

---

## Consequences

### ✅ Bénéfices

1. **Zéro risque paywall** : Better Auth tourne sur notre homelab, coût = €0 à vie quelle que soit la croissance
2. **Contrôle total** : sessions, tokens, expiry — tout est configurable
3. **Extensibilité OAuth** : plugin Google/Apple en 5 lignes quand besoin
4. **Conformité ADR-022** : mobile utilise SecureStore (pas AsyncStorage)
5. **Séparation propre** : Auth (Better Auth) vs AuthZ (NestJS) — pas de conflit avec le système RBAC/PBAC/ACL existant
6. **Offline-first compatible** : stratégie token end-of-day cohérente avec l'usage mobile

### ⚠️ Trade-offs acceptés

1. **Maintenance auth** : On prend en charge les updates Better Auth (hebdomadaire `npm update`, pas de patches sécurité à gérer manuellement — la librairie est stateless)
2. **Setup initial** : +1 jour vs "juste configurer Supabase" — justifié par élimination du risque long terme
3. **TypeORM adapter** : Better Auth a un adapter TypeORM — à vérifier la maturité au moment de l'implémentation

### 🔄 Impact sur architecture existante

- **ADR-016 Storage (MinIO)** : Inchangé — continue de fonctionner
- **ADR-016 Auth (Supabase)** : Superseded par ce document
- **Module `authorization` NestJS** : Inchangé — Better Auth fournit juste le `userId`
- **Guard NestJS** : Remplacer `SupabaseAuthGuard` par `BetterAuthGuard`
- **Mobile** : Remplacer `@supabase/supabase-js` par `@better-auth/react-native`
- **RGPD** : Même flow de suppression — `auth.api.deleteUser(userId)` remplace l'appel Supabase admin API

---

## Implementation

### Étapes de mise en œuvre

1. Installer Better Auth dans NestJS backend
2. Configurer adapter TypeORM (users table)
3. Implémenter `BetterAuthGuard` (remplace `SupabaseAuthGuard`)
4. Implémenter `AuthTokenManager` mobile (stratégie offline)
5. Intégrer client `@better-auth/react-native` (mobile)
6. Intégrer client `better-auth/client` (web + admin)
7. Configurer plugin `admin`
8. Connecter emailService avec Resend (ADR-030)
9. Supprimer dépendance `@supabase/supabase-js`
10. Tests d'intégration auth flows (login, logout, reset password, offline token)

### Fichiers impactés

```
pensieve/backend/src/auth/
├── auth.config.ts          # NEW — configuration Better Auth
├── auth.module.ts          # MODIFIED
├── auth.controller.ts      # MODIFIED — routes Better Auth handler
└── guards/
    └── better-auth.guard.ts # NEW (remplace supabase-auth.guard.ts)

pensieve/mobile/src/infrastructure/auth/
├── AuthTokenManager.ts     # NEW — stratégie offline token
└── auth.service.ts         # MODIFIED — better-auth client

pensieve/web/src/lib/
└── auth.ts                 # NEW — better-auth client config

pensieve/admin/src/lib/
└── auth.ts                 # NEW — better-auth client config
```

---

## Validation Criteria

ADR considéré succès SI :
- ✅ Login / logout fonctionnel sur Mobile, Web, Admin
- ✅ Password reset email envoyé via Resend (ADR-030)
- ✅ Token offline mobile valide jusqu'à 23:59 si refresh réseau KO
- ✅ Module AuthZ NestJS inchangé (RBAC/tiers/permissions fonctionnels)
- ✅ Plugin admin : ban/list/impersonate opérationnels
- ⏳ OAuth Google/Apple : non requis MVP — architecture prête

**Review Date :** 2026-05 (3 mois post-implémentation)

---

## References

- [Better Auth Documentation](https://better-auth.com)
- [Better Auth React Native](https://better-auth.com/docs/integrations/react-native)
- [Better Auth NestJS](https://better-auth.com/docs/integrations/nestjs)
- [Better Auth Admin Plugin](https://better-auth.com/docs/plugins/admin)
- ADR-016 — Hybrid Architecture (superseded partiellement)
- ADR-022 — State Persistence OP-SQLite (SecureStore pour tokens)
- ADR-030 — Transactional Email Provider (Resend)

---

## Decision Log

**2026-02-18** — Discussion yohikofox / Winston

→ Friction identifiée : Supabase SMTP rate limit 2-3/h + risque paywall
→ Besoin : solution sans risque vendor lock-in, €0 garanti à 500 users
→ Multi-client validé : Mobile + Web + Admin + Backend
→ Social login : non requis maintenant, extensibilité plugin confirmée
→ RBAC : Better Auth Auth uniquement, AuthZ reste dans NestJS — pas de conflit
→ Offline token : valide jusqu'à 23:59 si refresh impossible (pas de réseau)
→ Email transactionnel : délégué à ADR-030 (Resend)
→ Décision finale : Better Auth self-hosted ✅

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

---

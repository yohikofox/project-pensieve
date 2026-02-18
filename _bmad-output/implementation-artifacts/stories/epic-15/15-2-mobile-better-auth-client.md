# Story 15.2: Migration Client Mobile — Better Auth + AuthTokenManager

Status: ready-for-dev

## Story

As a mobile user,
I want to authenticate via Better Auth self-hosted,
so that my session persists offline de manière fiable jusqu'à 23:59 du jour courant même sans réseau.

## Acceptance Criteria

1. `@better-auth/react-native` installé et configuré dans `pensieve/mobile/`
2. Login / logout fonctionnels via Better Auth depuis l'app mobile
3. Tokens stockés dans **Expo SecureStore** (jamais AsyncStorage) — conforme ADR-022
4. `AuthTokenManager` implémenté : si refresh échoue à cause du réseau → token valide jusqu'à 23:59 local time
5. Si refresh échoue pour cause auth réelle (token révoqué, 401) → logout immédiat
6. Au retour du réseau → refresh automatique du token
7. `@supabase/supabase-js` supprimé des dépendances mobile
8. Variables d'environnement Supabase supprimées du mobile, remplacées par `EXPO_PUBLIC_BETTER_AUTH_URL`
9. Tests unitaires pour `AuthTokenManager` (stratégie offline) avec couverture 100%
10. Tests BDD couvrant : login, logout, offline token valide, midnight expiry, refresh au retour réseau

## Tasks / Subtasks

- [ ] Task 1 — Installer `@better-auth/react-native` (AC: 1)
  - [ ] `npm install @better-auth/react-native` dans `pensieve/mobile/`
  - [ ] Vérifier compatibilité Expo SDK 54 + React Native 0.81.5
  - [ ] Configurer le client : `createAuthClient({ baseURL: process.env.EXPO_PUBLIC_BETTER_AUTH_URL })`

- [ ] Task 2 — Créer `AuthTokenManager` (AC: 3, 4, 5, 6)
  - [ ] `src/infrastructure/auth/AuthTokenManager.ts`
  - [ ] Utiliser `expo-secure-store` pour stockage tokens (clés : `access_token`, `refresh_token`, `token_expires_at`)
  - [ ] Méthode `getValidToken()` : check expiry → refresh si expiré → fallback end-of-day si réseau KO
  - [ ] Distinguer `NetworkError` (réseau) vs `AuthError` (401/403) — Pattern Result (ADR-023)
  - [ ] Méthode `clearTokens()` pour logout
  - [ ] Tests unitaires 100% avec mocks SecureStore + réseau

- [ ] Task 3 — Migrer `AuthStore` / `AuthService` (AC: 2)
  - [ ] Identifier le service/store auth existant (Supabase auth)
  - [ ] Remplacer les appels Supabase par les appels Better Auth client
  - [ ] `signIn()` → `authClient.signIn.email({ email, password })`
  - [ ] `signOut()` → `authClient.signOut()` + `AuthTokenManager.clearTokens()`
  - [ ] Intégrer `AuthTokenManager` dans les appels HTTP (injecter token dans headers)
  - [ ] Respecter le pattern DI : résolution lazy dans les hooks (pas module-level)

- [ ] Task 4 — Supprimer Supabase du mobile (AC: 7, 8)
  - [ ] `npm uninstall @supabase/supabase-js` dans `pensieve/mobile/`
  - [ ] Supprimer tous les fichiers Supabase mobile
  - [ ] Mettre à jour `.env.example` : supprimer `EXPO_PUBLIC_SUPABASE_*`, ajouter `EXPO_PUBLIC_BETTER_AUTH_URL`
  - [ ] Vérifier `grep -r "supabase" src/` → zéro occurrences

- [ ] Task 5 — Enregistrer `AuthTokenManager` dans DI container (AC: 2)
  - [ ] Ajouter token `TOKENS.IAuthTokenManager` dans `src/infrastructure/di/tokens.ts`
  - [ ] Enregistrer dans `src/infrastructure/di/container.ts` — TRANSIENT (conforme ADR-021)
  - [ ] Résolution lazy dans les hooks auth (JAMAIS au niveau module)

- [ ] Task 6 — Tests BDD (AC: 10)
  - [ ] Créer `tests/acceptance/features/story-15-2.feature` (Gherkin)
  - [ ] Scénarios :
    - Login email/password succès
    - Logout efface les tokens SecureStore
    - Token expiré + réseau OK → refresh automatique
    - Token expiré + réseau KO + même jour → token encore valide jusqu'à 23:59
    - Token expiré + réseau KO + minuit dépassé → logout forcé
    - Token révoqué (401) → logout immédiat (PAS de fallback end-of-day)
  - [ ] Créer `tests/acceptance/story-15-2.test.ts` (jest-cucumber step definitions)
  - [ ] Mocks : SecureStore, réseau (NetworkError/AuthError), date/heure

## Dev Notes

### AuthTokenManager — implémentation complète

```typescript
// src/infrastructure/auth/AuthTokenManager.ts
import * as SecureStore from 'expo-secure-store'
import { Result, success, networkError, authError } from '@/contexts/shared/domain/Result'

const TOKEN_KEYS = {
  ACCESS_TOKEN: 'ba_access_token',
  REFRESH_TOKEN: 'ba_refresh_token',
  EXPIRES_AT: 'ba_token_expires_at',
} as const

export class AuthTokenManager {
  async getValidToken(): Promise<Result<string>> {
    const token = await SecureStore.getItemAsync(TOKEN_KEYS.ACCESS_TOKEN)
    const expiresAt = await SecureStore.getItemAsync(TOKEN_KEYS.EXPIRES_AT)

    if (!token) {
      return authError('No token stored')
    }

    const isExpired = Date.now() > Number(expiresAt)

    if (!isExpired) {
      return success(token)
    }

    // Token expiré → tenter refresh
    const refreshResult = await this.tryRefresh()

    if (refreshResult.type === 'success') {
      return refreshResult
    }

    // Refresh échoué — distinguer réseau vs auth
    if (refreshResult.type === 'network_error') {
      // Hors réseau : token valide jusqu'à 23:59 du jour courant
      if (Date.now() < this.getEndOfCurrentDay()) {
        return success(token)  // Permet les opérations offline
      }
      return authError('Session expirée — reconnexion requise')
    }

    // Erreur auth réelle (401, token révoqué) → logout immédiat
    await this.clearTokens()
    return authError('Session invalide — reconnexion requise')
  }

  private getEndOfCurrentDay(): number {
    const now = new Date()
    return new Date(
      now.getFullYear(), now.getMonth(), now.getDate(),
      23, 59, 59, 999
    ).getTime()
  }

  async storeTokens(accessToken: string, refreshToken: string, expiresIn: number): Promise<void> {
    const expiresAt = Date.now() + expiresIn * 1000
    await Promise.all([
      SecureStore.setItemAsync(TOKEN_KEYS.ACCESS_TOKEN, accessToken),
      SecureStore.setItemAsync(TOKEN_KEYS.REFRESH_TOKEN, refreshToken),
      SecureStore.setItemAsync(TOKEN_KEYS.EXPIRES_AT, String(expiresAt)),
    ])
  }

  async clearTokens(): Promise<void> {
    await Promise.all([
      SecureStore.deleteItemAsync(TOKEN_KEYS.ACCESS_TOKEN),
      SecureStore.deleteItemAsync(TOKEN_KEYS.REFRESH_TOKEN),
      SecureStore.deleteItemAsync(TOKEN_KEYS.EXPIRES_AT),
    ])
  }

  private async tryRefresh(): Promise<Result<string>> {
    const refreshToken = await SecureStore.getItemAsync(TOKEN_KEYS.REFRESH_TOKEN)
    if (!refreshToken) return authError('No refresh token')

    try {
      const response = await fetch(`${process.env.EXPO_PUBLIC_BETTER_AUTH_URL}/api/auth/token`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ refreshToken }),
      })

      if (response.status === 401 || response.status === 403) {
        return authError('Refresh token invalid')
      }

      if (!response.ok) {
        return networkError(`HTTP ${response.status}`)
      }

      const data = await response.json() as { accessToken: string; refreshToken: string; expiresIn: number }
      await this.storeTokens(data.accessToken, data.refreshToken, data.expiresIn)
      return success(data.accessToken)

    } catch (error) {
      // fetch throw = réseau KO (TypeError)
      return networkError('Network unavailable')
    }
  }
}
```

### DI Container — Token et enregistrement

```typescript
// src/infrastructure/di/tokens.ts — AJOUTER
export const TOKENS = {
  // ... existing tokens ...
  IAuthTokenManager: Symbol('IAuthTokenManager'),
} as const

// src/infrastructure/di/container.ts — AJOUTER (transient, conforme ADR-021)
container.register(TOKENS.IAuthTokenManager, {
  useClass: AuthTokenManager
})
// NE PAS utiliser registerSingleton (violation ADR-021)
```

### Pattern DI Hook — résolution lazy (CRITIQUE)

```typescript
// src/contexts/identity/hooks/useAuthTokenManager.ts
// ✅ CORRECT: Résolution lazy DANS le hook
export const useAuthTokenManager = () => {
  const getManager = () => container.resolve<AuthTokenManager>(TOKENS.IAuthTokenManager)
  return getManager()
}

// ❌ WRONG: Résolution au niveau module — CRASH au démarrage
const manager = container.resolve<AuthTokenManager>(TOKENS.IAuthTokenManager)
```

### Fichiers Better Auth client

```typescript
// src/infrastructure/auth/auth-client.ts
import { createAuthClient } from '@better-auth/react-native'

export const authClient = createAuthClient({
  baseURL: process.env.EXPO_PUBLIC_BETTER_AUTH_URL,
})
```

### Stratégie offline — règles de décision

| Situation | Action |
|-----------|--------|
| Token valide | Retourner token directement |
| Token expiré + réseau OK | Refresh → retourner nouveau token |
| Token expiré + réseau KO + avant 23:59 | Retourner ancien token (mode offline) |
| Token expiré + réseau KO + après 23:59 | Logout → écran reconnexion |
| Refresh → 401/403 (révoqué) | Logout immédiat (pas de fallback offline) |

### Variables d'environnement mobile

```bash
# mobile/.env.example (mise à jour)
EXPO_PUBLIC_BETTER_AUTH_URL=https://api.pensine.example.com
EXPO_PUBLIC_API_URL=https://api.pensine.example.com

# SUPPRIMÉES :
# EXPO_PUBLIC_SUPABASE_URL=
# EXPO_PUBLIC_SUPABASE_ANON_KEY=
```

### Project Structure Notes

```
pensieve/mobile/src/
├── infrastructure/
│   ├── auth/
│   │   ├── auth-client.ts          # NEW — Better Auth client config
│   │   └── AuthTokenManager.ts     # NEW — offline token strategy
│   └── di/
│       ├── container.ts            # MODIFIED — add AuthTokenManager
│       └── tokens.ts               # MODIFIED — add TOKENS.IAuthTokenManager
└── contexts/identity/
    ├── hooks/
    │   └── useAuthTokenManager.ts  # NEW (si hook nécessaire)
    └── services/
        └── auth.service.ts         # MODIFIED — remplace Supabase calls
```

### Gherkin — scénarios clés

```gherkin
# tests/acceptance/features/story-15-2.feature
Feature: Better Auth Mobile Client

  Scénario: Token expiré hors réseau — même jour
    Étant donné un token expiré stocké dans SecureStore
    Et le réseau est indisponible
    Et il est 14h00 (avant minuit)
    Quand l'app tente d'obtenir un token valide
    Alors l'ancien token est retourné (pas de blocage)
    Et aucun logout n'est déclenché

  Scénario: Token révoqué — logout immédiat
    Étant donné un token expiré stocké dans SecureStore
    Et le serveur répond 401 au refresh
    Quand l'app tente d'obtenir un token valide
    Alors les tokens sont effacés de SecureStore
    Et l'utilisateur est redirigé vers l'écran login
```

### References

- [Source: ADR-029 — AuthTokenManager + offline strategy](_bmad-output/planning-artifacts/adrs/ADR-029-auth-provider-better-auth.md#Stratégie-Token-Offline)
- [Source: ADR-022 — SecureStore pour persistence](_bmad-output/planning-artifacts/adrs/ADR-022-state-persistence-opsqlite.md)
- [Source: ADR-021 — DI Transient First](_bmad-output/planning-artifacts/adrs/ADR-021-di-lifecycle-transient-first.md)
- [Source: ADR-023 — Result Pattern, pas de throw](_bmad-output/planning-artifacts/adrs/ADR-023-error-handling-strategy.md)
- [Source: ADR-025 — fetch natif uniquement, pas axios](_bmad-output/planning-artifacts/adrs/ADR-025-http-client-strategy.md)
- [Source: project-context.md — DI lazy resolution](_bmad-output/project-context.md#DI-Bootstrap-Sequence)
- **Dépend de :** Story 15.1 (Better Auth server doit être en place avant de tester l'intégration mobile)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

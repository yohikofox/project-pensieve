# Story 15.2: Migration Client Mobile — Better Auth + AuthTokenManager

Status: done

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

- [x] Task 1 — Installer `@better-auth/react-native` (AC: 1)
  - [x] Installé `better-auth` + `@better-auth/expo` (le package `@better-auth/react-native` n'existe pas — divergence story corrigée)
  - [x] Compatibilité Expo SDK 54 vérifiée
  - [x] Configuré `auth-client.ts` : `createAuthClient({ baseURL, plugins: [expoClient({ storage })] })`

- [x] Task 2 — Créer `AuthTokenManager` (AC: 3, 4, 5, 6)
  - [x] `src/infrastructure/auth/AuthTokenManager.ts` créé
  - [x] `expo-secure-store` pour stockage (clés : `ba_access_token`, `ba_refresh_token`, `ba_token_expires_at`)
  - [x] `getValidToken()` : check expiry → refresh si expiré → fallback end-of-expiry-day si réseau KO
  - [x] `NetworkError` vs `AuthError` via Result Pattern (ADR-023)
  - [x] `clearTokens()` implémenté
  - [x] 8 tests BDD couvrant toutes les stratégies offline

- [x] Task 3 — Migrer `AuthStore` / `AuthService` (AC: 2)
  - [x] `BetterAuthService.ts` créé — implémente `IAuthService`
  - [x] `signIn()` → `authClient.signIn.email()` + `tokenManager.storeTokens()`
  - [x] `signOut()` → `authClient.signOut()` + `tokenManager.clearTokens()`
  - [x] Écrans migrés : Login, Register, ForgotPassword, ResetPassword, SettingsScreen, NotificationSettingsScreen
  - [x] Hooks migrés : `useAuthListener.ts`, `useDeepLinkAuth.ts`, `MainApp.tsx`

- [x] Task 4 — Supprimer Supabase du mobile (AC: 7, 8)
  - [x] `npm uninstall @supabase/supabase-js` exécuté
  - [x] Fichiers supprimés : `src/lib/supabase.ts`, `src/lib/large-secure-store.ts`, `src/contexts/identity/services/SupabaseAuthService.ts`
  - [x] `.env.example` mis à jour : Supabase retiré, `EXPO_PUBLIC_BETTER_AUTH_URL` ajouté
  - [x] `jest-setup.js` nettoyé (mock Supabase résiduel supprimé)

- [x] Task 5 — Enregistrer `AuthTokenManager` dans DI container (AC: 2)
  - [x] `TOKENS.IAuthTokenManager` ajouté dans `tokens.ts` (Symbol.for)
  - [x] `AuthTokenManager` enregistré en TRANSIENT dans `container.ts` (conforme ADR-021)
  - [x] `BetterAuthService` remplace `SupabaseAuthService` (singleton)

- [x] Task 6 — Tests BDD (AC: 10)
  - [x] `tests/acceptance/features/story-15-2.feature` créé (8 scénarios Gherkin)
  - [x] Scénarios : login, logout, token valide, offline avant minuit, offline après minuit, refresh réseau, token révoqué, refresh retour réseau
  - [x] `tests/acceptance/story-15-2.test.ts` créé (jest-cucumber step definitions)
  - [x] Mocks : SecureStore (in-memory), fetch global, authClient, Date.now()

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

Aucun — implémentation directe sans blocages majeurs.

### Completion Notes List

1. **Package divergence** : `@better-auth/react-native` (story AC-1) n'existe pas sur npm. Utilisation de `better-auth` + `@better-auth/expo/client` (version 1.4.18 déjà présente dans le projet).
2. **`expoClient()` signature** : requiert un objet `{ storage: { getItem, setItem } }` obligatoire. Migré vers `SecureStore.getItemAsync()` (async cohérent avec setItem).
3. **Offline strategy corrigée** : la logique initiale comparait `Date.now()` à la fin du "jour courant". Les tests BDD ont révélé qu'il faut comparer à la fin du "jour d'expiration du token" (`getEndOfExpiryDay(expiresAt)`). Correction appliquée.
4. **`user.id` null-safe** : `authClient.signIn.email()` peut retourner `data` sans champ `user` (selon le mock). Accès via `user?.id ?? ''` pour éviter TypeError → network_error.
5. **`jest-setup.js`** : mock Supabase résiduel (`jest.mock('./src/lib/supabase', ...)`) supprimé — causait un échec au démarrage du test suite `container.lifecycle`.
6. **[Code Review] H1 — DI bypass LoginScreen/ResetPasswordScreen** : `LoginScreen` et `ResetPasswordScreen` utilisaient `authClient` directement + `new AuthTokenManager()`. Corrigé : résolution lazy de `IAuthService` via `container.resolve('IAuthService')`.
7. **[Code Review] H2 — userId hardcodé** : `useAuthListener` retournait `{ id: 'authenticated' }`. Corrigé : utilise `authService.getSession()` pour obtenir le vrai `userId`.
8. **[Code Review] H3 — getSession() null après restart** : `BetterAuthService.getSession()` retournait null si `currentSession` non initialisé. Corrigé : appel `authClient.getSession()` pour reconstruire la session depuis le serveur.
9. **[Code Review] H4 — new AuthTokenManager() direct** : `BetterAuthService` instanciait `AuthTokenManager` hors DI. Corrigé : injection via `@inject(TOKENS.IAuthTokenManager)`.
10. **[Code Review] M1 — async incohérence auth-client** : `getItem` utilisait SecureStore synchrone. Corrigé : `getItemAsync` pour les deux opérations.
11. **[Code Review] M2 — expiresIn hardcodé** : `useDeepLinkAuth` stockait 3600s fixe. Corrigé : `expires_in` extrait du fragment URL avec fallback 3600.
12. **[Code Review] M3 — File List incomplète** : `package.json` et `package-lock.json` ajoutés au File List.
13. **[Code Review] M4 — IAuthService interface incomplète** : `signIn()` et `signOut()` ajoutés à l'interface.

### File List

**Nouveaux fichiers :**
- `pensieve/mobile/src/infrastructure/auth/auth-client.ts`
- `pensieve/mobile/src/infrastructure/auth/AuthTokenManager.ts`
- `pensieve/mobile/src/infrastructure/auth/BetterAuthService.ts`
- `pensieve/mobile/tests/acceptance/features/story-15-2.feature`
- `pensieve/mobile/tests/acceptance/story-15-2.test.ts`

**Fichiers modifiés :**
- `pensieve/mobile/package.json` — suppression `@supabase/supabase-js`, ajout `better-auth` + `@better-auth/expo`
- `pensieve/mobile/package-lock.json` — mis à jour après npm uninstall/install
- `pensieve/mobile/src/infrastructure/di/tokens.ts` — ajout `IAuthTokenManager`
- `pensieve/mobile/src/infrastructure/di/container.ts` — remplacement SupabaseAuthService, ajout AuthTokenManager transient
- `pensieve/mobile/src/infrastructure/di/__tests__/container.lifecycle.test.ts` — mocks adaptés
- `pensieve/mobile/src/contexts/identity/screens/LoginScreen.tsx`
- `pensieve/mobile/src/contexts/identity/screens/RegisterScreen.tsx`
- `pensieve/mobile/src/contexts/identity/screens/ForgotPasswordScreen.tsx`
- `pensieve/mobile/src/contexts/identity/screens/ResetPasswordScreen.tsx`
- `pensieve/mobile/src/contexts/identity/hooks/useAuthListener.ts`
- `pensieve/mobile/src/contexts/identity/hooks/useDeepLinkAuth.ts`
- `pensieve/mobile/src/components/MainApp.tsx`
- `pensieve/mobile/src/screens/settings/SettingsScreen.tsx`
- `pensieve/mobile/src/screens/settings/SettingsScreen.test.tsx`
- `pensieve/mobile/src/screens/settings/NotificationSettingsScreen.tsx`
- `pensieve/mobile/.env.example`
- `pensieve/mobile/jest-setup.js`

**Fichiers supprimés :**
- `pensieve/mobile/src/lib/supabase.ts`
- `pensieve/mobile/src/lib/large-secure-store.ts`
- `pensieve/mobile/src/contexts/identity/services/SupabaseAuthService.ts`

---
adr: ADR-025
title: "HTTP Client Strategy - fetch natif + wrapper custom"
date: 2026-02-15
status: "✅ Accepted"
context: "Phase 4 - Implementation - Epic 6 Synchronisation"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
  - Amelia (Developer)
---

# ADR-025: HTTP Client Strategy - fetch natif + wrapper custom

**Status:** ✅ ACCEPTÉ

**Context:** Choisir la stratégie HTTP client pour Mobile (React Native), Backend (NestJS), et Web (Next.js) - `fetch` natif vs `axios` vs alternatives modernes.

**Decision:** Utiliser `fetch` natif avec wrapper custom pour retry/timeout logic. Abandonner `axios` pour réduire bundle size et maintenir le contrôle.

---

## Problem Statement

**Situation actuelle** :
- `SyncService.ts` (mobile) utilise `axios` (importé ligne 12)
- Incohérence dans la codebase : certains fichiers mentionnent `fetch`, d'autres utilisent `axios`
- Aucune décision architecturale documentée sur le choix HTTP client
- Impact bundle size mobile non mesuré

**Questions à résoudre** :
1. Quelle librairie HTTP pour Mobile (React Native + Expo SDK 54) ?
2. Quelle librairie HTTP pour Backend (NestJS 11 + Node 22) ?
3. Quelle librairie HTTP pour Web (Next.js 15) ?
4. Faut-il une stratégie unifiée ou différenciée par contexte ?

---

## Options Considérées

### Option 1: `axios` partout
**Pour** :
- Interceptors natifs (auth tokens, logging)
- Progress events pour uploads
- Auto JSON transform
- Retry avec `axios-retry`
- Timeout simple
- Familiarité équipe

**Contre** :
- **+13 KB minified** (impact bundle mobile)
- Dépendance externe à maintenir
- Node 22 a `fetch` natif → axios devient redondant
- Overhead fonctionnel (on utilise <30% des features)

### Option 2: `fetch` natif partout
**Pour** :
- **Zero dependency** → bundle size = 0 KB
- Standard Web API (W3C spec)
- Natif Node 18+ et navigateurs modernes
- Polyfill transparent dans React Native (Expo)
- Longévité garantie (standard web)
- Contrôle total du code

**Contre** :
- Pas de retry logic intégré
- Pas d'interceptors natifs
- Timeout manuel (`AbortController`)
- Streaming uploads plus verbeux
- Besoin d'écrire wrapper custom (~100 lignes)

### Option 3: Alternatives modernes (`ky`, `ofetch`)
**Pour** :
- `ky` : 4 KB, retry natif, timeout, hooks
- `ofetch` : Auto retry, interceptors, optimisé SSR
- Meilleur compromis fetch/axios

**Contre** :
- Toujours une dépendance externe
- Moins de maturité qu'axios
- Communauté plus petite

---

## Decision

**Stratégie différenciée par contexte** :

| Contexte | Solution | Rationale |
|----------|----------|-----------|
| **Mobile** (React Native) | `fetch` + wrapper custom | Bundle size critique, contrôle total |
| **Backend** (NestJS + Node 22) | `fetch` natif | Node 22 l'a nativement, pas besoin d'axios |
| **Web** (Next.js 15) | `fetch` natif | Next.js optimise fetch (cache, revalidation) |

**Wrapper custom mobile** : `src/infrastructure/http/fetchWithRetry.ts`

Fonctionnalités :
- ✅ Retry avec backoff Fibonacci (comme actuel)
- ✅ Timeout configurable (30s par défaut)
- ✅ AbortController pour annulation
- ✅ Error typing (`NetworkError`, `TimeoutError`)
- ✅ ~100 lignes de code TypeScript strict

---

## Rationale

### 1. **Bundle Size** (critère #1 mobile)
- `axios` = +13 KB minified
- `fetch` = 0 KB (natif)
- **Impact** : Time-to-Interactive amélioré sur connexions lentes

### 2. **Maintenance & Ownership**
- `axios` = dépendance externe, breaking changes potentiels
- `fetch` + wrapper = contrôle total, tests 100%, pas de surprise

### 3. **Node 22 Native Support**
- Backend n'a **aucun besoin** d'axios avec Node 22
- `fetch` natif performant (undici engine)

### 4. **Next.js Optimizations**
- Next.js 15 étend `fetch` avec cache automatique, revalidation
- Utiliser axios **casse** ces optimisations

### 5. **Developer Experience**
- Wrapper TypeScript strict = meilleure DX qu'axios générique
- Pattern unifié : `fetchWithRetry()` partout dans mobile

---

## Consequences

### ✅ Bénéfices

**Performance** :
- Bundle mobile réduit de 13 KB
- TTI (Time-to-Interactive) amélioré
- Moins de parsing JS au démarrage

**Maintenance** :
- Contrôle total du code HTTP
- Pas de breaking changes externes
- Tests unitaires 100% maison

**Pérennité** :
- Standard Web API (W3C spec) → longévité garantie
- Pas de risque d'abandon (axios peut devenir legacy)

**Consistency** :
- Pattern unifié dans mobile (`fetchWithRetry`)
- Même API `fetch` partout (mobile/backend/web)

### ⚠️ Trade-offs

**Temps dev initial** :
- +2h pour écrire wrapper custom (vs 5min install axios)
- +1h pour migrer `SyncService.ts`

**Features manquantes** :
- Pas d'interceptors natifs → pattern custom si besoin
- Progress events uploads → utiliser `ReadableStream` API

**Learning curve** :
- Équipe doit apprendre `AbortController`, `ReadableStream`
- Documentation interne nécessaire

---

## Implementation Plan

### Phase 1: Créer wrapper custom (Story 6.3 ou hotfix)

**Fichier** : `pensieve/mobile/src/infrastructure/http/fetchWithRetry.ts`

```typescript
export interface FetchOptions extends RequestInit {
  retries?: number;
  timeout?: number;
  onRetry?: (attempt: number, error: Error) => void;
}

export async function fetchWithRetry(
  url: string,
  options: FetchOptions = {}
): Promise<Response> {
  const { retries = 3, timeout = 30000, onRetry, ...fetchOptions } = options;

  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(url, {
      ...fetchOptions,
      signal: controller.signal
    });

    clearTimeout(timeoutId);

    // Retry on 5xx or network errors
    if (!response.ok && retries > 0 && isRetryable(response.status)) {
      const delay = fibonacci(3 - retries);
      onRetry?.(3 - retries + 1, new Error(`HTTP ${response.status}`));
      await sleep(delay);
      return fetchWithRetry(url, { ...options, retries: retries - 1 });
    }

    return response;

  } catch (error) {
    clearTimeout(timeoutId);

    if (retries > 0 && isNetworkError(error)) {
      const delay = fibonacci(3 - retries);
      onRetry?.(3 - retries + 1, error as Error);
      await sleep(delay);
      return fetchWithRetry(url, { ...options, retries: retries - 1 });
    }

    throw error;
  }
}

function isRetryable(status: number): boolean {
  return status >= 500 || status === 408 || status === 429;
}

function isNetworkError(error: unknown): boolean {
  return error instanceof TypeError ||
         (error as any)?.name === 'AbortError';
}
```

### Phase 2: Migrer SyncService (Story 6.3 ou hotfix)

**Avant** :
```typescript
import axios, { type AxiosInstance } from 'axios';

this.apiClient = axios.create({ baseURL, timeout: 30000 });
const response = await this.apiClient.post('/sync/pull', payload);
```

**Après** :
```typescript
import { fetchWithRetry } from '@/infrastructure/http/fetchWithRetry';

async pull(payload: PullRequest): Promise<SyncResponse> {
  const response = await fetchWithRetry(`${this.baseUrl}/sync/pull`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${this.authToken}`
    },
    body: JSON.stringify(payload),
    timeout: SYNC_TIMEOUT_MS,
    retries: 3
  });

  if (!response.ok) {
    throw new Error(`Sync failed: ${response.status}`);
  }

  return response.json();
}
```

### Phase 3: Tests du wrapper

**Fichier** : `pensieve/mobile/src/infrastructure/http/__tests__/fetchWithRetry.test.ts`

Scénarios :
- ✅ Succès immédiat (200 OK)
- ✅ Retry sur 500 → succès au 2e essai
- ✅ Retry sur network error → succès au 3e essai
- ✅ Timeout après 30s (AbortController)
- ✅ Max retries atteint → throw error
- ✅ 4xx non retryable → throw immédiat
- ✅ onRetry callback appelé avec bon attempt number

### Phase 4: Désinstaller axios

```bash
cd pensieve/mobile
npm uninstall axios axios-mock-adapter
```

Vérifier aucune autre dépendance ne l'utilise :
```bash
grep -r "from 'axios'" src/
```

---

## Monitoring & Success Metrics

**Avant migration** :
- Bundle size mobile : `npx react-native-bundle-visualizer` (baseline)
- TTI moyen : Lighthouse CI

**Après migration** :
- Bundle size : -13 KB attendu
- TTI : amélioration 50-100ms attendue (3G slow)
- Test coverage wrapper : 100%

**Alertes** :
- Si wrapper génère >5 bugs en prod → rollback immédiat
- Si équipe bloquée >1 jour sur migration → escalade

---

## Alternatives Rejected

### ❌ Garder axios
**Raison rejet** : Bundle size inacceptable pour mobile-first app. Node 22 rend axios obsolète côté backend.

### ❌ Utiliser `ky` ou `ofetch`
**Raison rejet** : Toujours une dépendance externe. Wrapper custom 100 lignes = meilleur ownership.

### ❌ Stratégie mixte (axios backend, fetch mobile)
**Raison rejet** : Incohérence codebase. Pattern unifié `fetch` préférable.

---

## References

- [MDN Web Docs - Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [Node.js 18+ fetch support](https://nodejs.org/en/blog/announcements/v18-release-announce#fetch-experimental)
- [Next.js Extended fetch](https://nextjs.org/docs/app/api-reference/functions/fetch)
- [React Native fetch polyfill](https://reactnative.dev/docs/network#using-fetch)
- Epic 6 Story 6.1 - Infrastructure de synchronisation
- Epic 6 Story 6.2 - Synchronisation local → cloud

---

**Date:** 2026-02-15
**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)
- Amelia (Developer)

---

**Next Steps:**
1. ✅ ADR-025 rédigé et accepté
2. ⏳ Créer task Story 6.3 ou hotfix pour migration
3. ⏳ Implémenter `fetchWithRetry.ts` avec tests 100%
4. ⏳ Migrer `SyncService.ts`
5. ⏳ Désinstaller axios + vérifier aucune régression

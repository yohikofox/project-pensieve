# Story 22.1 — Spike : Valider "Sign in with ChatGPT" OAuth pour Pensine

Status: backlog

## Story (Spike)

As a **product owner**,
I want **a technical validation of "Sign in with ChatGPT" OAuth integration for Pensine**,
So that **Epic 22 implementation can proceed on confirmed foundations and avoid ToS violations**.

## Context

L'ADR-036 définit deux mécanismes BYOK pour Epic 22 :
1. Clé API OpenAI manuelle
2. Connexion du compte ChatGPT via OAuth

Ce spike valide la faisabilité et la légitimité de l'option 2 **avant** toute implémentation.

### Ce qu'on sait déjà (2026-02-26)

**Feature officielle confirmée :** OpenAI a lancé officiellement "Sign in with ChatGPT" en juin 2025 pour les apps tierces — modèle identique à "Sign in with Google/Apple".

| Point validé | Détail |
|--------------|--------|
| Mécanisme technique | OAuth2 Authorization Code Flow + PKCE (S256) |
| Données partagées | Nom, email, photo de profil |
| Utilisateurs éligibles | Free, Plus, Pro (pas Enterprise/Edu/Team) |
| Accès développeur | Formulaire d'intérêt officiel sur openai.com |
| Modèle incitation actuel | API credits offerts à l'utilisateur ($5 Plus / $50 Pro) |
| Endpoint observé (Codex CLI) | `chatgpt.com/backend-api/codex/responses` |
| `client_id` Codex CLI | `app_EMoamEEZ73f0CkXaXp7hrann` (tests seulement) |
| SDK officiel | `developers.openai.com/apps-sdk/` |

**Client TypeScript de référence (prototype non-production) :**
Un client TypeScript implémentant le flow OAuth2 PKCE complet existe comme référence d'implémentation :
- `TokenStorage` : persistance locale des tokens
- `TokenManager` : refresh automatique
- `OAuthClient` : flow PKCE complet
- `SseStreamParser` : parsing Server-Sent Events
- `ChatMockClient` : client principal

Ce client utilise des modules Node.js (`fs`, `http`, `os`) qui nécessitent une adaptation React Native.

### Questions ouvertes à valider

1. **Enregistrement officiel** : Soumettre le formulaire d'intérêt OpenAI → obtenir un `client_id` propre à Pensine
2. **Endpoint de production** : L'endpoint `codex/responses` est-il disponible pour apps non-Codex ? Quelle est l'API officielle ?
3. **Utilisation des tokens** : Les tokens OAuth donnent accès à l'API OpenAI ou uniquement à l'identité ?
4. **Revenue sharing** : OpenAI propose-t-il un modèle d'acquisition (Pensine → nouveau subscriber ChatGPT) ?
5. **React Native compatibility** : Adaptation `expo-auth-session` pour le flow PKCE

## Acceptance Criteria

### AC1 — Enregistrement développeur réalisé
**Given** Pensine comme app tierce
**When** le formulaire d'intérêt OpenAI est soumis
**Then** un `client_id` officiel est obtenu (ou un délai de réponse est documenté)

### AC2 — Flow OAuth2 PKCE validé en sandbox
**Given** le `client_id` de test ou officiel
**When** le flow OAuth est initié depuis React Native (via `expo-auth-session`)
**Then**:
- L'utilisateur est redirigé vers `chatgpt.com/auth`
- L'autorisation est accordée
- Un `access_token` et `refresh_token` sont reçus
- Les tokens sont stockés dans `expo-secure-store`

### AC3 — Portée des tokens documentée
**Given** un `access_token` ChatGPT OAuth valide
**When** un appel API est effectué
**Then** la portée exacte est documentée :
- ✅ Accès API OpenAI (modèles, completions) → impacte l'architecture BYOK
- ❓ Accès uniquement aux endpoints ChatGPT web
- ❓ Limites de rate, quota, model access

### AC4 — Adaptation React Native prototypée
**Given** le client TypeScript de référence (Node.js)
**When** le spike est terminé
**Then** un mapping complet est produit :
- `fs` → `expo-secure-store`
- `http` / `https` → `fetch` natif
- `EventSource (SSE)` → `expo-sse` ou fetch streaming
- Gestion `redirect_uri` avec Expo deep links

### AC5 — Décision documentée
**Given** les résultats du spike
**When** toutes les AC précédentes sont complétées
**Then** un document de décision est produit avec :
- Go / No-Go pour l'implémentation en Epic 22
- Architecture proposée (si Go)
- Risques et mitigations
- ADR-036 mis à jour avec les findings définitifs

## Dev Notes

### Références clés

- **ADR-036** — LLM Local-First + BYOK : stratégie globale
- **Formulaire développeur OpenAI** : https://openai.com/form/sign-in-with-chatgpt-interest (à vérifier URL exacte)
- **Apps SDK officiel** : https://developers.openai.com/apps-sdk/
- **Articles de référence** :
  - MediaNama (juin 2025) — Launch announcement
  - TechCrunch (mai 2025) — Developer program details

### Pattern d'implémentation cible (React Native)

```typescript
// expo-auth-session pour PKCE
import * as AuthSession from 'expo-auth-session';
import * as SecureStore from 'expo-secure-store';

const discovery = {
  authorizationEndpoint: 'https://chatgpt.com/authorize',  // À confirmer
  tokenEndpoint: 'https://chatgpt.com/token',              // À confirmer
};

const request = new AuthSession.AuthRequest({
  clientId: 'PENSINE_CLIENT_ID',  // À obtenir via formulaire OpenAI
  scopes: ['openid', 'email', 'profile'],
  redirectUri: AuthSession.makeRedirectUri({ scheme: 'pensine' }),
  usePKCE: true,
});
```

### Stockage des tokens

```typescript
// Conforme ADR-022 (OP-SQLite pour état) + expo-secure-store pour secrets
await SecureStore.setItemAsync('chatgpt_access_token', accessToken);
await SecureStore.setItemAsync('chatgpt_refresh_token', refreshToken);
```

## Tasks

- [ ] Soumettre le formulaire d'intérêt développeur OpenAI "Sign in with ChatGPT"
- [ ] Documenter les endpoints officiels du flow OAuth (discovery document)
- [ ] Valider la portée des tokens obtenus (API OpenAI ou uniquement identité)
- [ ] Prototyper le flow avec `expo-auth-session` (PKCE)
- [ ] Tester le stockage/refresh token avec `expo-secure-store`
- [ ] Documenter le mapping Node.js → React Native complet
- [ ] Produire le document de décision Go/No-Go
- [ ] Mettre à jour ADR-036 avec findings définitifs

## Story Size

**Type :** Spike (time-boxed 2-3 jours)
**Priorité :** Basse — à réaliser avant Epic 22 (post-monétisation)
**Dépend de :** Epic 18 (infrastructure LLM config), Epic 22 (BYOK power users)
**Bloque :** Stories 22.2+ (implémentation BYOK OAuth)

## References

- ADR-036 — LLM Local-First + BYOK Optionnel
- Epic 22 — LLM Power User (BYOK OpenAI + OAuth ChatGPT)
- NFR3 (performance cloud <15s), NFR5 (feedback visuel)

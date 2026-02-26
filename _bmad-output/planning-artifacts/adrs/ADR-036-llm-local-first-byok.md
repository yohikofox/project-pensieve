# ADR-036 — LLM Local-First Strategy + BYOK Optionnel

**Date:** 2026-02-26
**Statut:** Accepté
**Auteurs:** yohikofox, Winston (Architect)
**Session:** Architecture Mode Délégation

---

## Contexte

La contrainte économique est explicite : **zéro coût LLM avant monétisation**. L'application doit prouver sa valeur uniquement via des modèles locaux et gratuits, sans dépendance à des APIs cloud payantes (OpenAI, Anthropic, etc.).

Cette contrainte a été validée empiriquement : le modèle `gemma3n-2b` via `litert-lm` produit déjà des résultats exploitables pour l'extraction d'intents (date résolue, contact extrait, type d'action identifié) en ~14 secondes.

**Logs de référence (2026-02-25) :**
```
[CaptureAnalysisService] Initialized with model: gemma3n-2b backend: litert-lm
[CaptureAnalysisService] LLM result: {"items":[{
  "title":"Prendre rendez-vous avec Paul jeudi à 14h pour la révision du camion",
  "deadline_date":"26-02-2026, 14:00",
  "target":"Paul"
}]}
modelId: "file:///data/.../gemma3n-2b-tpu.litertlm"
processingDurationMs: 13844
```

## Décision

### 1. Local-First : toujours, pas seulement en fallback

Le modèle local est le chemin **primaire**, pas le fallback. Aucun appel LLM cloud n'est effectué sans consentement explicite de l'utilisateur.

**Modèle actuel :** `gemma3n-2b` via `litert-lm` (LiteRT — successeur TensorFlow Lite)
**Géré par :** Epic 18 (Remote LLM Config Service — configuration administrable sans mise à jour app)

### 2. Dual SLA de performance (révision NFR3)

Le SLA unique "< 30s" est remplacé par deux SLAs selon le chemin d'exécution :

| Chemin | SLA Digestion IA | NFR |
|--------|-----------------|-----|
| **Local (défaut)** | < 60s | NFR3-local |
| **Cloud (opt-in BYOK)** | < 15s | NFR3-cloud |

**Impératif :** NFR5 (feedback visuel) devient encore plus critique pour le chemin local. Un spinner avec progression temps réel est obligatoire pour toute opération > 5s.

### 3. BYOK (Bring Your Own Key) — Post-monétisation

Deux mécanismes permettront aux utilisateurs exigeants d'utiliser leurs propres crédits LLM :

| Option | Mécanisme | Friction | Epic |
|--------|-----------|---------|------|
| **Clé API OpenAI** | L'utilisateur entre sa clé → stockée dans expo-secure-store | Élevée (setup manuel) | Epic 22 |
| **OAuth ChatGPT** | L'utilisateur se connecte avec son compte ChatGPT | Faible (1 tap) | Epic 22 |

**"Sign in with ChatGPT" — Feature officielle OpenAI (validée 2026-02-26) :**

OpenAI a lancé officiellement "Sign in with ChatGPT" en juin 2025 pour les apps tierces, exactement sur le modèle "Sign in with Google/Apple". Ce n'est PAS un détournement de Codex — c'est un produit officiel.

| Point | Détail |
|-------|--------|
| **Mécanisme** | OAuth2 PKCE (Authorization Code Flow) — standard |
| **Données partagées** | Nom, email, photo de profil |
| **Utilisateurs éligibles** | Free, Plus, Pro (pas Enterprise/Edu/Team) |
| **Accès développeur** | Formulaire d'intérêt sur openai.com/apps — toutes tailles d'apps |
| **Modèle actuel** | Utilisateurs reçoivent des API credits en s'authentifiant ($5 Plus, $50 Pro) |
| **client_id Codex CLI** | `app_EMoamEEZ73f0CkXaXp7hrann` — pour tests uniquement ; production → enregistrement officiel |
| **Endpoint observé** | `chatgpt.com/backend-api/codex/responses` — non documenté officiellement |
| **Apps SDK officiel** | `developers.openai.com/apps-sdk/` — voie de production recommandée |

**Statut légal/ToS :** ✅ Légitimement supporté. La voie de production passe par l'enregistrement via le formulaire développeur OpenAI pour obtenir un `client_id` propre à Pensine.

**Adaptation React Native requise (spike 22.1) :**
- `fs` → `expo-secure-store` (stockage tokens)
- `http` → `expo-auth-session` (PKCE flow natif)
- L'endpoint `codex/responses` doit être remplacé par l'API officielle une fois enregistré

**Modèle acquisition :** Le scénario "l'utilisateur connecte son compte ChatGPT existant et Pensine utilise ses tokens" est exactement le cas d'usage supporté. La monétisation directe (Pensine génère des abonnements pour OpenAI avec revenue sharing) n'est pas encore officiellement disponible — à surveiller pour Epic 22 post-lancement.

### 4. Routing local → cloud

Lorsqu'une clé BYOK est configurée :

```
Request LLM
  → clé BYOK présente ? → cloud LLM (OpenAI/Anthropic)
  → sinon              → local LLM (gemma3n-2b via litert-lm)
```

Epic 18 (Remote LLM Config) fournit l'infrastructure de configuration. Epic 22 ajoute la couche BYOK utilisateur.

## Conséquences

**Positives :**
- Zéro coût avant monétisation — contrainte respectée dès le MVP
- Valeur prouvée localement — pas de dépendance à une API tierce
- Architecture préparée pour le cloud sans y être liée
- Modèles locaux s'améliorent : Gemma 3n, Phi-4-mini, SmolLM2 — l'ADR s'adapte automatiquement via Epic 18

**Négatives :**
- Performance dégradée vs cloud (~14s vs ~2s pour action_items)
- Compensé par NFR5 (feedback visuel) et le Dual SLA

## Références

- Epic 18 (Remote LLM Config Service — configuration des modèles locaux)
- Epic 22 (LLM Power User — BYOK OpenAI + OAuth ChatGPT)
- NFR3 (performance digestion — révisé), NFR5 (feedback visuel)
- ADR-004 (Digestion IA — Single LLM call strategy, toujours valide)

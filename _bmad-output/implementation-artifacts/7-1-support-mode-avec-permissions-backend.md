# Story 7.1: Support Mode avec Permissions Backend

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **user ayant besoin de support technique**,
I want **un système de permissions backend qui contrôle l'accès au mode debug de l'app**,
So that **je puisse activer/désactiver le mode debug sans republier l'app, permettant un support rapide et flexible pour les power users et early adopters**.

**Context:** Cette story permet de gérer l'accès au mode debug depuis l'interface admin backend, avec un système à double niveau (permission d'accès + activation locale). Cas d'usage principal : déboguer des problèmes utilisateurs en production (ex: logs locaux, retry transcription, rapport d'erreur) sans attendre de publication App Store/Play Store.

## Acceptance Criteria

### AC1: API Backend - Endpoint de Récupération des Permissions Utilisateur
**Given** je suis un utilisateur authentifié
**When** l'application mobile requête `GET /api/users/:userId/features`
**Then** l'API retourne un objet JSON avec les permissions de l'utilisateur
**And** la réponse contient au minimum `{ "debug_mode_access": true|false }`
**And** la réponse utilise un format extensible pour futures permissions
**And** l'endpoint est protégé par authentification (JWT/session)
**And** un utilisateur ne peut accéder qu'à ses propres permissions

### AC2: Interface Admin - Toggle Permission Debug Mode
**Given** je suis connecté à l'interface admin backend
**When** je consulte le profil d'un utilisateur
**Then** je vois un toggle "Activer Support Mode" (ou "Debug Mode Access")
**And** je peux activer/désactiver cette permission
**And** le changement est immédiatement persisté en base de données
**And** la nouvelle valeur sera récupérée au prochain fetch mobile

### AC3: Mobile - Fetch des Permissions au Démarrage
**Given** l'application mobile démarre
**When** l'utilisateur est authentifié
**Then** l'app requête automatiquement `/api/users/:userId/features`
**And** la réponse est stockée en cache mémoire (store/context)
**And** si la requête échoue (réseau, serveur), on utilise le cache précédent
**And** le cache précédent expire à minuit (00:00) du jour courant

### AC4: Mobile - Gestion du Cache Offline jusqu'à Minuit
**Given** l'app a précédemment récupéré les permissions (ex: `debug_mode_access: true`)
**When** l'app redémarre en mode offline (pas de réseau)
**Then** l'app utilise les permissions en cache
**And** si la date actuelle < minuit du jour de cache, les permissions sont valides
**And** si la date actuelle >= minuit du jour de cache, les permissions expirent
**And** si expirées ET offline, on considère `debug_mode_access: false` par défaut (sécurité)
**And** le timestamp de cache est persisté (AsyncStorage/MMKV)

### AC5: Mobile - Refresh des Permissions (Sans Reconnexion)
**Given** je suis dans l'application mobile
**When** j'accède à mon profil utilisateur ou aux settings
**Then** un refetch automatique des permissions est déclenché (optionnel : pull-to-refresh)
**And** les nouvelles permissions sont appliquées immédiatement
**And** si `debug_mode_access` change de `false` → `true`, le switch debug apparaît dans settings
**And** si `debug_mode_access` change de `true` → `false`, le switch debug disparaît (et debug mode est désactivé)

### AC6: Settings UI - Affichage Conditionnel du Switch Debug Mode
**Given** mes permissions backend incluent `debug_mode_access: true`
**When** j'ouvre les Settings de l'app
**Then** je vois un switch "Mode Debug" (ou "Support Mode")
**And** le switch est interactif (je peux l'activer/désactiver localement)
**And** l'état ON/OFF est persisté localement (AsyncStorage/MMKV)
**And** si `debug_mode_access: false` (backend), le switch n'apparaît PAS

### AC7: Settings UI - Masquage du Switch si Permission Refusée
**Given** mes permissions backend incluent `debug_mode_access: false`
**When** j'ouvre les Settings de l'app
**Then** le switch "Mode Debug" n'est pas visible
**And** même si j'avais activé le debug mode précédemment, il est désormais désactivé
**And** toutes les features debug sont inaccessibles

### AC8: SettingsStore - Vérification Permission Avant Activation Debug
**Given** je tente d'activer le mode debug via le switch dans settings
**When** je toggle le switch à ON
**Then** le settingsStore vérifie `debugModeAccess` (permission backend)
**And** si `debugModeAccess === true`, le debug mode s'active (état local = ON)
**And** si `debugModeAccess === false`, le toggle est ignoré (reste OFF, ou affiche erreur)
**And** l'état local du debug mode est persisté uniquement si permission valide

### AC9: Intégration avec Features Debug Existantes
**Given** le mode debug est activé (permission backend + toggle local ON)
**When** j'utilise les features debug existantes dans l'app
**Then** toutes les fonctionnalités debug continuent de fonctionner normalement
**And** le settingsStore expose `isDebugModeEnabled` qui combine :
  - `debugModeAccess` (permission backend) AND
  - `debugModeLocalToggle` (état local du switch)
**And** `isDebugModeEnabled = debugModeAccess && debugModeLocalToggle`

### AC10: Persistance et Synchronisation de l'État
**Given** j'ai activé le mode debug (permission backend + toggle local)
**When** je ferme et rouvre l'app
**Then** le settingsStore restaure :
  - Les permissions backend depuis le cache (valide jusqu'à minuit)
  - L'état local du toggle debug (AsyncStorage/MMKV)
**And** le mode debug reste activé si les 2 conditions sont toujours vraies
**And** si les permissions backend ont expiré (après minuit), un refetch est tenté

### AC11: Scénario de Support - Activation Rapide pour Debugging
**Given** un utilisateur (ex: mon frère) rencontre un bug critique
**When** j'active `debug_mode_access: true` depuis l'interface admin
**And** l'utilisateur rafraîchit son profil mobile (pull-to-refresh ou redémarrage)
**Then** le switch "Mode Debug" apparaît dans ses settings
**And** il peut l'activer pour accéder aux logs locaux
**And** il peut utiliser les features de retry/rapport d'erreur
**And** après résolution, je désactive `debug_mode_access: false` depuis l'admin
**And** au prochain refresh mobile, le mode debug disparaît automatiquement

## Tasks / Subtasks

### Backend Tasks

- [ ] **Task 1: Créer l'Endpoint API de Récupération des Permissions** (AC: 1)
  - [ ] Subtask 1.1: Définir le modèle de données `UserFeatures`
    - Créer table `user_features` (si non existante) ou ajouter colonnes à `users`
    - Colonnes : `user_id` (FK), `debug_mode_access` (boolean, default false)
    - Indexer sur `user_id` pour performance
  - [ ] Subtask 1.2: Implémenter `GET /api/users/:userId/features`
    - Controller : `UsersController.getFeatures()`
    - Service : `UsersService.getUserFeatures(userId: string)`
    - DTO de réponse : `UserFeaturesDto { debug_mode_access: boolean }`
    - Protection : Guard JWT + validation ownership (userId === req.user.id)
  - [ ] Subtask 1.3: Écrire tests unitaires et E2E
    - Test : utilisateur authentifié récupère ses permissions
    - Test : utilisateur ne peut pas récupérer permissions d'un autre user (403)
    - Test : format de réponse JSON valide

- [ ] **Task 2: Interface Admin - Gestion de la Permission Debug** (AC: 2)
  - [ ] Subtask 2.1: Ajouter un toggle dans l'interface admin utilisateur
    - Composant : `UserFeaturesToggle` (React/Vue selon stack admin)
    - Afficher switch "Activer Support Mode"
    - État initial basé sur valeur DB (`debug_mode_access`)
  - [ ] Subtask 2.2: Implémenter l'endpoint de mise à jour
    - Endpoint : `PATCH /api/admin/users/:userId/features`
    - Body : `{ debug_mode_access: boolean }`
    - Protection : Guard Admin (rôle admin requis)
    - Validation : userId existe, valeur boolean
  - [ ] Subtask 2.3: Connecter le toggle UI à l'API
    - On toggle → `PATCH /api/admin/users/:userId/features`
    - Feedback utilisateur (loading, success, erreur)
    - Rafraîchir l'état après succès

### Mobile Tasks

- [ ] **Task 3: Service de Gestion des Permissions** (AC: 3, 4, 5, 10)
  - [ ] Subtask 3.1: Créer `UserFeaturesService`
    - Méthode : `fetchUserFeatures(): Promise<UserFeaturesDto>`
    - Méthode : `getCachedFeatures(): UserFeaturesDto | null`
    - Méthode : `isCacheValid(): boolean` (vérifie expiration minuit)
    - Méthode : `clearExpiredCache(): void`
  - [ ] Subtask 3.2: Implémenter le cache avec expiration à minuit
    - Stocker : `{ features: UserFeaturesDto, cachedAt: timestamp }`
    - Utiliser AsyncStorage ou MMKV pour persistance
    - Logique expiration : `if (now >= startOfTomorrow(cachedAt)) { expired }`
    - Si expiré ET offline → `{ debug_mode_access: false }` par défaut
  - [ ] Subtask 3.3: Fetch au démarrage de l'app
    - Hook : `useEffect` dans App.tsx ou context provider
    - Si online → fetch + update cache
    - Si offline → utiliser cache si valide, sinon défaut sécurisé
  - [ ] Subtask 3.4: Méthode de refresh manuel
    - Exposer `refreshUserFeatures()` dans le service
    - Utilisable depuis profil/settings (pull-to-refresh)

- [ ] **Task 4: Intégration dans SettingsStore** (AC: 6, 7, 8, 9)
  - [ ] Subtask 4.1: Ajouter état `debugModeAccess` (permission backend)
    - Store Zustand/Redux : `debugModeAccess: boolean`
    - Initialisé depuis `UserFeaturesService.getCachedFeatures()`
  - [ ] Subtask 4.2: Ajouter état `debugModeLocalToggle` (état local)
    - Store : `debugModeLocalToggle: boolean`
    - Persisté via AsyncStorage/MMKV
  - [ ] Subtask 4.3: Computed property `isDebugModeEnabled`
    - `isDebugModeEnabled = debugModeAccess && debugModeLocalToggle`
    - Utilisé par toutes les features debug existantes
  - [ ] Subtask 4.4: Action `toggleDebugMode()`
    - Vérifier `debugModeAccess === true` avant toggle
    - Si `false`, bloquer avec message d'erreur (ou ignorer silencieusement)
    - Si `true`, mettre à jour `debugModeLocalToggle` et persister

- [ ] **Task 5: UI Settings - Affichage Conditionnel du Switch** (AC: 6, 7)
  - [ ] Subtask 5.1: Créer composant `DebugModeToggle`
    - Affichage conditionnel : `{debugModeAccess && <DebugModeToggle />}`
    - Switch contrôlé : `value={debugModeLocalToggle}` `onValueChange={toggleDebugMode}`
  - [ ] Subtask 5.2: Intégrer dans l'écran Settings
    - Section "Développeur" ou "Support"
    - Label : "Mode Debug" ou "Support Mode"
    - Description optionnelle : "Permet d'accéder aux logs et outils de diagnostic"

### Testing & Validation Tasks

- [ ] **Task 6: Tests Backend** (AC: 1, 2)
  - [ ] Subtask 6.1: Tests unitaires `UsersService.getUserFeatures()`
    - Test : retourne `debug_mode_access: false` par défaut
    - Test : retourne `debug_mode_access: true` si activé
  - [ ] Subtask 6.2: Tests E2E endpoint `/api/users/:userId/features`
    - Test : utilisateur récupère ses propres permissions
    - Test : utilisateur ne peut pas accéder aux permissions d'un autre (403)
    - Test : unauthenticated → 401
  - [ ] Subtask 6.3: Tests admin endpoint `PATCH /api/admin/users/:userId/features`
    - Test : admin peut modifier les permissions
    - Test : non-admin → 403
    - Test : valeur persiste en DB

- [ ] **Task 7: Tests Mobile** (AC: 3, 4, 5, 6, 7, 8, 9, 10)
  - [ ] Subtask 7.1: Tests unitaires `UserFeaturesService`
    - Test : fetch online met à jour le cache
    - Test : cache valide jusqu'à minuit
    - Test : cache expiré après minuit
    - Test : offline + cache expiré → défaut `debug_mode_access: false`
  - [ ] Subtask 7.2: Tests unitaires `SettingsStore`
    - Test : `isDebugModeEnabled = debugModeAccess && debugModeLocalToggle`
    - Test : `toggleDebugMode()` bloqué si `debugModeAccess === false`
    - Test : `toggleDebugMode()` fonctionne si `debugModeAccess === true`
  - [ ] Subtask 7.3: Tests d'intégration UI Settings
    - Test : switch visible si `debugModeAccess === true`
    - Test : switch invisible si `debugModeAccess === false`
    - Test : toggle switch met à jour l'état local

- [ ] **Task 8: Documentation & Validation Scénario de Support** (AC: 11)
  - [ ] Subtask 8.1: Documenter le processus de support
    - Guide : "Comment activer le mode debug pour un utilisateur"
    - Étapes : Admin UI → Toggle → User refresh → Accès logs
  - [ ] Subtask 8.2: Tester le scénario end-to-end en staging
    - Créer utilisateur de test
    - Activer `debug_mode_access` via admin
    - Vérifier apparition du switch mobile
    - Activer debug mode et vérifier logs/features
    - Désactiver `debug_mode_access` et vérifier disparition

## Technical Notes

### API Response Format (Extensible)

```json
{
  "debug_mode_access": true,
  "error_reporting_enabled": true,  // future feature flag
  "transcription_retry_enabled": true  // future feature flag
}
```

### Mobile Cache Structure

```typescript
interface UserFeaturesCache {
  features: {
    debug_mode_access: boolean;
    // ... autres permissions futures
  };
  cachedAt: number; // Unix timestamp
}
```

### Expiration Logic (Minuit)

```typescript
function isCacheValid(cachedAt: number): boolean {
  const now = new Date();
  const cacheDate = new Date(cachedAt);
  const midnight = new Date(cacheDate);
  midnight.setHours(24, 0, 0, 0); // Minuit du jour suivant

  return now < midnight;
}
```

### SettingsStore Integration

```typescript
// Pseudo-code
interface SettingsStore {
  debugModeAccess: boolean; // from backend
  debugModeLocalToggle: boolean; // from AsyncStorage

  get isDebugModeEnabled(): boolean {
    return this.debugModeAccess && this.debugModeLocalToggle;
  }

  toggleDebugMode(): void {
    if (!this.debugModeAccess) {
      console.warn('Debug mode access denied by backend');
      return;
    }
    this.debugModeLocalToggle = !this.debugModeLocalToggle;
    // Persister dans AsyncStorage
  }
}
```

## Dependencies

- **Backend:** Nécessite authentification JWT/session fonctionnelle (Epic 1)
- **Mobile:** Nécessite settingsStore existant (supposé déjà implémenté)
- **Admin:** Nécessite interface admin backend (à créer ou étendre)

## Risks & Mitigations

**Risque 1:** Cache expiré + offline → utilisateur perd accès debug mode de manière inattendue
**Mitigation:** Documenter le comportement, expiration à minuit est prévisible, utilisateur peut se reconnecter au réseau

**Risque 2:** Sécurité - utilisateur malveillant tente de forcer `debug_mode_access: true` en local
**Mitigation:** La permission est vérifiée côté backend à chaque fetch, impossible de forger localement

**Risque 3:** UX - utilisateur ne comprend pas pourquoi le switch debug disparaît soudainement
**Mitigation:** Message explicatif si permission révoquée (ex: "Mode debug désactivé par l'administrateur")

**Risque 4:** Performance - fetch permissions à chaque démarrage app
**Mitigation:** Cache valide jusqu'à minuit réduit la fréquence, requête légère (< 1KB JSON)

## Definition of Done

- [ ] Tous les AC (AC1-AC11) sont implémentés et testés
- [ ] Tests backend (unitaires + E2E) passent à 100%
- [ ] Tests mobile (unitaires + intégration) passent à 100%
- [ ] Interface admin permet d'activer/désactiver la permission
- [ ] Scénario de support end-to-end validé en staging
- [ ] Documentation utilisateur admin créée
- [ ] Code review complété et approuvé
- [ ] Merge dans main branch

## Notes Additionnelles

Cette story pose les fondations d'un système de **feature flags** extensible. Futures stories pourront réutiliser cette infrastructure pour d'autres permissions :
- `error_reporting_enabled` (Story 7.2 ?)
- `transcription_retry_enabled` (si besoin de gate cette feature)
- `beta_features_access` (pour tester de nouvelles features avec des power users)

**Approche Progressive:**
- **MVP (cette story):** Permission debug mode uniquement
- **V2:** Ajouter durée d'activation temporaire (ex: "Activer pour 24h")
- **V3:** Granularité par feature (plusieurs toggles : logs, retry, etc.)

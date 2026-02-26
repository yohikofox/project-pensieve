# Story 24.3: Feature Flag System — Adaptation Mobile & UI Gating

Status: ready-for-dev

## Story

As a **utilisateur de l'application mobile**,
I want **que l'application affiche uniquement les fonctionnalités auxquelles j'ai accès selon mes feature flags**,
So that **l'expérience soit propre et cohérente, sans onglets ou boutons inaccessibles qui créent de la confusion**.

**Context:** Story 24.1 a refactoré le système backend vers un `Record<string, boolean>`. Cette story adapte le mobile pour consommer ce nouveau format dynamique, met à jour le `settingsStore` et gate les éléments UI : tabs Actualités / Projets et boutons de capture (Photo/Video, URL, Document, Presse-papiers).

## Acceptance Criteria

### AC1: Interface UserFeatures Dynamique
**Given** l'API `/api/users/:userId/features` retourne `Record<string, boolean>`
**When** le mobile parse la réponse
**Then** le type `UserFeatures` est `Record<string, boolean>` (plus de champs statiques)
**And** les clés de features sont définies comme constantes dans `FEATURE_KEYS`

```typescript
export const FEATURE_KEYS = {
  DEBUG_MODE: 'debug_mode',
  DATA_MINING: 'data_mining',
  NEWS_TAB: 'news_tab',
  PROJECTS_TAB: 'projects_tab',
  CAPTURE_MEDIA_BUTTONS: 'capture_media_buttons',
} as const;
```

### AC2: SettingsStore — Features Dynamiques
**Given** les features sont récupérées depuis le backend
**When** `useSyncUserFeatures` synchronise les features dans le store
**Then** le `settingsStore` stocke `features: Record<string, boolean>` (non persisté — volatile)
**And** `debugModeAccess: boolean` et `dataMiningEnabled: boolean` sont supprimés du store
**And** `getFeature(key: string): boolean` retourne la valeur ou `false` si absent
**And** la logique `isDebugModeEnabled()` utilise `getFeature(FEATURE_KEYS.DEBUG_MODE)`
**And** la logique `isDataMiningEnabled()` utilise `getFeature(FEATURE_KEYS.DATA_MINING)`
**And** le double-gate du debug mode est préservé : `getFeature('debug_mode') && debugMode`

### AC3: Masquage du Tab Actualités
**Given** l'utilisateur authentifié a `features['news_tab'] = false`
**When** l'application affiche la navigation par tabs
**Then** le tab "Actualités" n'est pas rendu (absent du DOM/composant)
**And** si `features['news_tab'] = true`, le tab "Actualités" est visible et navigable normalement

### AC4: Masquage du Tab Projets
**Given** l'utilisateur authentifié a `features['projects_tab'] = false`
**When** l'application affiche la navigation par tabs
**Then** le tab "Projets" n'est pas rendu (absent)
**And** si `features['projects_tab'] = true`, le tab "Projets" est visible normalement

### AC5: Masquage des Boutons de Capture Média
**Given** l'utilisateur authentifié a `features['capture_media_buttons'] = false`
**When** l'utilisateur ouvre l'écran "Capturer"
**Then** les boutons suivants ne sont pas rendus : "Photo/Vidéo", "URL", "Document", "Presse-papiers"
**And** seul le bouton d'enregistrement audio reste visible (non gatée)
**And** si `features['capture_media_buttons'] = true`, tous les boutons sont visibles

### AC6: Comportement Offline
**Given** l'utilisateur est offline
**When** les features sont chargées depuis le cache (valid jusqu'à minuit)
**Then** les règles de gating (AC3-AC5) s'appliquent identiquement avec les valeurs en cache
**And** si le cache est expiré et offline, toutes les features valent `false` (comportement sécuritaire)

### AC7: Pas d'impact sur les features existantes
**Given** le settingsStore est mis à jour
**When** les features debug sont consultées
**Then** `CalibrationGridWrapper`, `DevPanel`, `SqlConsole` et `LottieGallery` fonctionnent identiquement
**And** la section "Development" dans Settings n'est visible que si `getFeature('debug_mode') = true`
**And** la section "DataMining" n'est visible que si `getFeature('data_mining') = true`

## Tasks / Subtasks

### Mobile — Domain & Service

- [ ] **Task 1: Feature Keys Constants** (AC1)
  - [ ] Subtask 1.1: Créer `mobile/src/contexts/identity/domain/feature-keys.ts` avec `FEATURE_KEYS` const
  - [ ] Subtask 1.2: Mettre à jour `user-features.model.ts` — type `UserFeatures = Record<string, boolean>`

- [ ] **Task 2: Adapter UserFeaturesRepository** (AC1)
  - [ ] Subtask 2.1: Mettre à jour le parsing de la réponse API (de champs statiques vers `Record<string, boolean>`)
  - [ ] Subtask 2.2: Adapter `UserFeaturesCache` (stocker le `Record` complet)

- [ ] **Task 3: Adapter UserFeaturesService** (AC1)
  - [ ] Subtask 3.1: Mettre à jour les types de retour
  - [ ] Subtask 3.2: Mettre à jour le fallback offline (`{}` vide → toutes features = false)

### Mobile — SettingsStore

- [ ] **Task 4: Refactoring SettingsStore** (AC2)
  - [ ] Subtask 4.1: Remplacer `debugModeAccess: boolean` et `dataMiningEnabled: boolean` par `features: Record<string, boolean>` (non persisté)
  - [ ] Subtask 4.2: Ajouter `setFeatures(features: Record<string, boolean>)` action
  - [ ] Subtask 4.3: Ajouter `getFeature(key: string): boolean` selector
  - [ ] Subtask 4.4: Mettre à jour `isDebugModeEnabled()` → `getFeature('debug_mode') && debugMode`
  - [ ] Subtask 4.5: Mettre à jour `setDebugMode()` et `toggleDebugMode()` → utiliser `getFeature('debug_mode')`
  - [ ] Subtask 4.6: Mettre à jour `setDebugModeAccess()` → `setFeatures(...)` (compatibilité supprimée)
  - [ ] Subtask 4.7: Mettre à jour `setDataMiningEnabled()` → supprimé (géré via `setFeatures`)

- [ ] **Task 5: Adapter useSyncUserFeatures** (AC2)
  - [ ] Subtask 5.1: Utiliser `store.setFeatures(features)` au lieu des setters individuels

### Mobile — UI Gating

- [ ] **Task 6: Gate Tab Actualités** (AC3)
  - [ ] Subtask 6.1: Identifier le composant/navigateur gérant les tabs
  - [ ] Subtask 6.2: Conditionner le rendu du tab Actualités sur `getFeature(FEATURE_KEYS.NEWS_TAB)`
  - [ ] Subtask 6.3: S'assurer que la navigation ne plante pas si le tab est absent (deep links, état initial)

- [ ] **Task 7: Gate Tab Projets** (AC4)
  - [ ] Subtask 7.1: Conditionner le rendu du tab Projets sur `getFeature(FEATURE_KEYS.PROJECTS_TAB)`
  - [ ] Subtask 7.2: Même précaution navigation que Task 6

- [ ] **Task 8: Gate Boutons Capture Média** (AC5)
  - [ ] Subtask 8.1: Identifier les boutons dans l'écran Capturer (Photo/Vidéo, URL, Document, Presse-papiers)
  - [ ] Subtask 8.2: Wrapper dans `{getFeature(FEATURE_KEYS.CAPTURE_MEDIA_BUTTONS) && <BoutonsMedia />}`
  - [ ] Subtask 8.3: Valider que l'écran reste fonctionnel (enregistrement audio) sans ces boutons

### Tests

- [ ] **Task 9: Tests** (AC1-AC7)
  - [ ] Subtask 9.1: Mettre à jour tests `settingsStore.debugMode.test.ts` (nouveaux getters)
  - [ ] Subtask 9.2: Tests `getFeature()` — clé existante, clé absente, offline fallback
  - [ ] Subtask 9.3: Tests rendu conditionnel tabs (jest + React Native Testing Library)
  - [ ] Subtask 9.4: Tests rendu conditionnel boutons capture
  - [ ] Subtask 9.5: Tests BDD `story-24-3.feature`
    - Scénario: news_tab=false → tab absent
    - Scénario: capture_media_buttons=false → boutons absents
    - Scénario: offline → features toutes false → gating appliqué

## Technical Notes

### Sélecteur Zustand Recommandé

```typescript
// Dans les composants
const showNewsTab = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.NEWS_TAB));
const showProjectsTab = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.PROJECTS_TAB));
const showMediaButtons = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.CAPTURE_MEDIA_BUTTONS));

// Dans le navigateur
{showNewsTab && <Tab.Screen name="Actualités" ... />}
{showProjectsTab && <Tab.Screen name="Projets" ... />}
```

### Attention Navigation
Si un utilisateur a un deep link vers un tab masqué, le navigateur doit fallback sur le tab par défaut (Capturer ou Feed). À valider lors des tests d'intégration.

### Migration des Tests Existants
Les tests `settingsStore.debugMode.test.ts` doivent être mis à jour :
- `setDebugModeAccess(true)` → `setFeatures({ debug_mode: true })`
- `state.debugModeAccess` → `state.getFeature('debug_mode')`

## Dependencies

- Story 24.1 (endpoint `/api/users/:userId/features` retourne `Record<string, boolean>`) doit être `done`
- Story 24.2 n'est pas requise pour cette story (peut se développer en parallèle)

## Definition of Done

- [ ] `UserFeatures = Record<string, boolean>` — plus de champs statiques
- [ ] `FEATURE_KEYS` constants créées
- [ ] `settingsStore` refactoré sans `debugModeAccess` / `dataMiningEnabled`
- [ ] Tabs Actualités et Projets masqués quand features = false
- [ ] Boutons capture média masqués quand feature = false
- [ ] Tous les tests existants (debug mode) mis à jour et passent
- [ ] Tests BDD passent

## Dev Notes

### Architecture Compliance

- **ADR-022 (AsyncStorage)** : Ne JAMAIS persister `features: Record<string, boolean>` directement dans AsyncStorage via Zustand `persist`. Le cache features est géré par `UserFeaturesRepository` (clé `@pensieve:userFeatures` avec `cachedAt` pour expiry midnight). Le champ `features` dans `settingsStore` doit être dans `partialize` comme volatile (exclu de la persistance).
- **ADR-023 (Result Pattern)** : `UserFeaturesRepository.fetchUserFeatures()` retourne `RepositoryResult<Record<string, boolean>>`. Conserver ce pattern lors de la mise à jour du type de retour.
- **ADR-021 (Transient First)** : Pas de changement de lifecycle requis — `UserFeaturesRepository` et `UserFeaturesService` restent transient.
- **Double gate debug mode** : Conserver impérativement `isDebugModeEnabled() = getFeature('debug_mode') && debugMode` — le toggle manuel utilisateur ET l'accès feature sont tous les deux requis. Voir `settingsStore.ts` pour la logique actuelle.
- **Fallback offline sécuritaire** : Si cache expiré ET offline → `{}` vide → tous les `getFeature()` retournent `false` → comportement conservateur (aucune feature accessible). C'est intentionnel (security by default).

### Project Structure Notes

- **Nouveau fichier** : `mobile/src/contexts/identity/domain/feature-keys.ts` — constantes `FEATURE_KEYS`
- **Fichiers à modifier** :
  - `mobile/src/stores/settingsStore.ts` — remplacer `debugModeAccess`/`dataMiningEnabled` par `features: Record<string, boolean>` + action `setFeatures()` + selector `getFeature()`
  - `mobile/src/contexts/identity/data/user-features.repository.ts` — parsing `Record<string, boolean>`
  - `mobile/src/contexts/identity/services/user-features.service.ts` — types de retour + fallback
  - `mobile/src/contexts/identity/hooks/useSyncUserFeatures.ts` — utiliser `setFeatures()` au lieu des setters individuels
  - Navigation tabs : localiser dans `mobile/src/navigation/` (chercher `Tab.Navigator` ou `BottomTabNavigator`)
  - CaptureScreen : localiser dans `mobile/src/screens/` ou `mobile/src/contexts/capture/` pour les boutons média
- **Tests à mettre à jour** : `mobile/tests/` — chercher tous les tests utilisant `setDebugModeAccess`, `dataMiningEnabled`, `debugModeAccess` → migrer vers `setFeatures({...})`

### Testing Standards

- **Pattern BDD mobile** : Voir `mobile/tests/acceptance/story-3-1-captures-list.test.ts`
- **Zustand testing** : `useSettingsStore.getState()` pour assertions directes sur le store
- **React Native Testing Library** : Pour tests de rendu conditionnel (tabs, boutons) — `queryByTestId` ou `queryByRole` pour vérifier l'absence d'un élément
- **Tests unitaires `getFeature()`** : Couvrir — clé présente/absente, offline fallback (`{}` → false), double gate debug mode
- **Scénarios BDD minimum** :
  1. `news_tab = false` → tab Actualités absent du rendu
  2. `capture_media_buttons = false` → boutons Photo/URL/Document/Presse-papiers absents
  3. Offline avec cache expiré → features toutes `false` → gating appliqué

### References

- [Source: mobile/src/stores/settingsStore.ts] — Store à refactorer (debugModeAccess, dataMiningEnabled, partialize)
- [Source: mobile/src/contexts/identity/data/user-features.repository.ts] — Repository + cache expiry logic (isCacheValid)
- [Source: mobile/src/contexts/identity/services/user-features.service.ts] — Fallback chain server → cache → defaults
- [Source: mobile/src/contexts/identity/hooks/useSyncUserFeatures.ts] — Hook d'intégration settingsStore
- [Source: mobile/src/infrastructure/di/tokens.ts] — Tokens DI existants
- [Source: mobile/src/infrastructure/di/container.ts] — Enregistrements DI (transient pattern)
- [Source: _bmad-output/planning-artifacts/architecture.md#ADR-022] — AsyncStorage restrictions
- [Source: _bmad-output/planning-artifacts/architecture.md#ADR-023] — Result Pattern
- [Source: _bmad-output/implementation-artifacts/stories/epic-24/24-1-feature-flag-system-backend.md] — Story prérequis (format API + FEATURE_KEYS)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

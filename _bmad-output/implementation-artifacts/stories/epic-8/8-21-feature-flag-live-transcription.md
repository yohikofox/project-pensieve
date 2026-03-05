# Story 8.21 : Feature Flag — Transcription Live (OFF par défaut)

Status: done

## Story

En tant qu'**administrateur Pensine**,
je veux **contrôler l'activation du bouton "Live" dans CaptureScreen par utilisateur/rôle/permission**,
afin de **déployer progressivement la feature de transcription live et la désactiver si nécessaire**.

## Contexte

### Problème

Le bouton **"Live"** (transcription temps réel — Story 8.6, issue #11) a été ajouté directement dans `CAPTURE_TOOLS_ALWAYS` sans intégration au système de feature flags (Epic 24, déjà complet). La feature est donc **visible pour tous les utilisateurs sans contrôle d'activation**.

La décision produit : **OFF par défaut** (nécessite activation explicite par l'admin).

### Système de feature flags existant (Epic 24 — tout done)

L'infrastructure est entièrement opérationnelle :

**Backend** — `FeatureResolutionService` (deny-wins) :
- Catalogue de features en base (table `features`)
- Assignations par `user` / `role` / `permission`
- Endpoint : `GET /api/users/:userId/features` → `Record<string, boolean>`

**Mobile** — chaîne `UserFeaturesRepository` → `UserFeaturesService` → `useSyncUserFeatures` → `settingsStore.getFeature(key)` :
- `FEATURE_KEYS` centralisé dans `mobile/src/contexts/identity/domain/feature-keys.ts`
- `getFeature(key)` retourne `false` si clé absente (secure-by-default)
- Pattern de gating en place pour `capture_media_buttons`, `news_tab`, `projects_tab`

**Admin** — CRUD features + assignations déjà disponible.

### Ce qui manque

1. Le feature flag `live_transcription` n'est pas enregistré dans le catalogue backend
2. `FEATURE_KEYS` mobile ne contient pas `LIVE_TRANSCRIPTION`
3. Le bouton "live" est dans `CAPTURE_TOOLS_ALWAYS` au lieu d'être conditionnel

### Pattern de référence — `CAPTURE_MEDIA_BUTTONS`

```typescript
// CaptureScreen.tsx — pattern existant à reproduire
const showMediaButtons = useSettingsStore((s) =>
  s.getFeature(FEATURE_KEYS.CAPTURE_MEDIA_BUTTONS)
);

const captureTools = useMemo(
  () => [
    ...CAPTURE_TOOLS_ALWAYS,
    ...(showMediaButtons ? CAPTURE_TOOLS_MEDIA : []),
  ],
  [showMediaButtons]
);
```

## Acceptance Criteria

### AC1 : Feature flag enregistré dans le catalogue backend (OFF par défaut)

**Given** l'administrateur accède à la liste des features dans l'interface admin
**When** la migration de seed est appliquée
**Then** une entrée `live_transcription` est présente dans le catalogue features
**And** sa valeur par défaut est `false` (OFF)
**And** aucune assignation user/role/permission n'est créée par défaut

### AC2 : Bouton Live masqué si feature désactivée

**Given** un utilisateur dont la feature `live_transcription` est `false` (ou absente)
**When** il ouvre `CaptureScreen`
**Then** seuls les boutons **Voice** et **Text** sont visibles
**And** le bouton **Live** n'apparaît pas

### AC3 : Bouton Live visible si feature activée

**Given** un utilisateur dont la feature `live_transcription` est `true` (assignée via admin)
**When** il ouvre `CaptureScreen`
**Then** les boutons **Voice**, **Text** et **Live** sont visibles
**And** le bouton Live fonctionne comme avant (comportement Story 8.6 inchangé)

### AC4 : Secure-by-default — comportement offline

**Given** l'utilisateur est hors ligne avec un cache expiré
**When** il ouvre `CaptureScreen`
**Then** le bouton Live est masqué (getFeature retourne `false` → secure-by-default)
**And** aucune erreur n'est affichée

### AC5 : Tests unitaires et BDD

**Given** l'implémentation est complète
**When** les tests sont exécutés
**Then** les scénarios BDD dans `story-8-21-feature-flag-live-transcription.feature` couvrent :
  - AC2 : bouton Live absent si feature `false`
  - AC3 : bouton Live présent si feature `true`
  - AC4 : comportement offline (feature absente → `false`)
**And** les tests unitaires couvrent `FEATURE_KEYS.LIVE_TRANSCRIPTION` et le calcul de `captureTools`
**And** `npm run test:unit` passe sans régression
**And** `npm run test:acceptance` passe sans régression

## Tasks / Subtasks

### Task 1 : Backend — enregistrer le feature flag (AC1)

- [x] Subtask 1.1 : Créer une migration (ou seed) dans `backend/src/database/migrations/` (ou `seeds/`) pour insérer la feature dans le catalogue :
  ```sql
  INSERT INTO features (id, key, description, default_value)
  VALUES (
    'fe000006-0000-7000-8000-000000000006',
    'live_transcription',
    'Active le bouton de transcription en temps réel dans CaptureScreen',
    false
  )
  ON CONFLICT (key) DO NOTHING;
  ```
  **Note** : La table `features` (migration `1780100000000-CreateFeatureFlagTables.ts`) n'a pas de colonne `name`. Colonnes réelles : `id`, `key`, `description`, `default_value`, `created_at`, `updated_at`, `deleted_at`.

- [x] Subtask 1.2 : Vérifier que `FeatureResolutionService` retourne `false` pour `live_transcription` si aucune assignation — tester manuellement via `GET /api/users/:id/features`.

### Task 2 : Mobile — ajouter `LIVE_TRANSCRIPTION` à `FEATURE_KEYS` (AC2, AC3)

- [x] Subtask 2.1 : Dans `mobile/src/contexts/identity/domain/feature-keys.ts`, ajouter :
  ```typescript
  export const FEATURE_KEYS = {
    // ... existants ...
    LIVE_TRANSCRIPTION: 'live_transcription',
  } as const;
  ```

### Task 3 : Mobile — gater le bouton Live dans `CaptureScreen` (AC2, AC3, AC4)

- [x] Subtask 3.1 : Dans `CaptureScreen.tsx`, **extraire** le tool "live" de `CAPTURE_TOOLS_ALWAYS` vers un nouveau tableau :
  ```typescript
  // AVANT (à modifier) :
  const CAPTURE_TOOLS_ALWAYS: CaptureTool[] = [
    { id: "voice", ... },
    { id: "text", ... },
    { id: "live", ... },  // ← supprimer d'ici
  ];

  // APRÈS :
  const CAPTURE_TOOLS_ALWAYS: CaptureTool[] = [
    { id: "voice", ... },
    { id: "text", ... },
  ];

  const CAPTURE_TOOLS_LIVE: CaptureTool[] = [
    {
      id: "live",
      labelKey: "capture.tools.live",
      iconName: "zap",
      color: colors.warning[500],
      available: true,
    },
  ];
  ```

- [x] Subtask 3.2 : Ajouter le sélecteur feature flag dans `CaptureScreenContent` (même pattern que `showMediaButtons`, ligne 285).
  ⚠️ **IMPORTANT** : `showLiveTranscription` est **déjà utilisé** comme state variable (ligne 291 — état de l'overlay). Utiliser le nom `isLiveTranscriptionEnabled` pour le sélecteur feature flag :
  ```typescript
  const isLiveTranscriptionEnabled = useSettingsStore((s) =>
    s.getFeature(FEATURE_KEYS.LIVE_TRANSCRIPTION)
  );
  ```

- [x] Subtask 3.3 : Modifier le calcul de `captureTools` (lignes 286-288). Le pattern existant est un simple spread **sans** `useMemo` — le reproduire à l'identique :
  ```typescript
  // AVANT (lignes 284-288 actuelles) :
  const showMediaButtons = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.CAPTURE_MEDIA_BUTTONS));
  const captureTools = showMediaButtons
    ? [...CAPTURE_TOOLS_ALWAYS, ...CAPTURE_TOOLS_MEDIA]
    : CAPTURE_TOOLS_ALWAYS;

  // APRÈS :
  const showMediaButtons = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.CAPTURE_MEDIA_BUTTONS));
  const isLiveTranscriptionEnabled = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.LIVE_TRANSCRIPTION));
  const captureTools = [
    ...CAPTURE_TOOLS_ALWAYS,
    ...(isLiveTranscriptionEnabled ? CAPTURE_TOOLS_LIVE : []),
    ...(showMediaButtons ? CAPTURE_TOOLS_MEDIA : []),
  ];
  ```

### Task 4 : Tests BDD (AC5)

- [x] Subtask 4.1 : Créer `mobile/tests/acceptance/features/story-8-21-feature-flag-live-transcription.feature`
- [x] Subtask 4.2 : Écrire les scénarios (voir "Scénarios BDD" ci-dessous)
- [x] Subtask 4.3 : Créer `mobile/tests/acceptance/story-8-21-feature-flag-live-transcription.test.ts` avec step definitions
- [x] Subtask 4.4 : Mocker `settingsStore.getFeature` pour simuler feature ON/OFF

### Task 5 : Tests unitaires (AC5)

- [x] Subtask 5.1 : Vérifier que `FEATURE_KEYS.LIVE_TRANSCRIPTION === 'live_transcription'`
- [x] Subtask 5.2 : Tester la composition de `captureTools` :
  - feature OFF → tableau sans `{ id: "live" }`
  - feature ON → tableau avec `{ id: "live" }` à la bonne position
  - feature absente (`undefined`) → même comportement que OFF

### Task 6 : Validation finale (AC5)

- [x] Subtask 6.1 : `npm run test:unit` — zéro régression
- [x] Subtask 6.2 : `npm run test:acceptance` — zéro régression (15 failures préexistantes, aucune régression)
- [ ] Subtask 6.3 : Test manuel — vérifier bouton absent par défaut, puis assigner `live_transcription=true` via admin, vérifier apparition

## Scénarios BDD

```gherkin
# language: fr
Fonctionnalité: Feature flag — Transcription Live (OFF par défaut)
  En tant qu'administrateur
  Je veux contrôler l'activation du bouton Live par utilisateur
  Afin de déployer la feature progressivement

  Scénario: Bouton Live masqué si feature désactivée
    Étant donné que la feature "live_transcription" est désactivée pour l'utilisateur
    Quand l'utilisateur ouvre CaptureScreen
    Alors seuls les boutons "Voice" et "Text" sont affichés
    Et le bouton "Live" n'est pas visible

  Scénario: Bouton Live visible si feature activée
    Étant donné que la feature "live_transcription" est activée pour l'utilisateur
    Quand l'utilisateur ouvre CaptureScreen
    Alors les boutons "Voice", "Text" et "Live" sont tous affichés
    Et le bouton "Live" est fonctionnel (peut déclencher l'overlay)

  Scénario: Comportement offline — feature absente → bouton masqué
    Étant donné que l'utilisateur est hors ligne avec un cache expiré
    Et que getFeature("live_transcription") retourne false
    Quand l'utilisateur ouvre CaptureScreen
    Alors le bouton "Live" n'est pas affiché
    Et aucune erreur n'est propagée à l'UI
```

## Dev Notes

### 🔑 Fichiers à modifier

```
pensieve/mobile/src/contexts/identity/domain/feature-keys.ts
  ← Task 2: Ajouter LIVE_TRANSCRIPTION: 'live_transcription'

pensieve/mobile/src/screens/capture/CaptureScreen.tsx
  ← Task 3: Extraire "live" de CAPTURE_TOOLS_ALWAYS → CAPTURE_TOOLS_LIVE
  ← Task 3: Ajouter sélecteur showLiveTranscription
  ← Task 3: Modifier useMemo captureTools

pensieve/backend/ (migration ou seed)
  ← Task 1: Insérer 'live_transcription' dans table features (default: false)
```

### 📁 Fichiers à créer

```
pensieve/mobile/tests/acceptance/features/story-8-21-feature-flag-live-transcription.feature
pensieve/mobile/tests/acceptance/story-8-21-feature-flag-live-transcription.test.ts
```

### ⛔ Fichiers à NE PAS modifier

```
pensieve/mobile/src/hooks/useLiveTranscription.ts
  ← Le hook est inchangé — le gating se fait au niveau du bouton dans CaptureScreen

pensieve/mobile/src/components/capture/LiveTranscriptionOverlay.tsx
  ← Composant inchangé — il ne sera simplement plus atteignable si le bouton est masqué

pensieve/backend/src/modules/features/ (code applicatif)
  ← L'infra FeatureResolutionService ne change pas — juste une donnée de catalogue
```

### 🏗️ Architecture

```
Admin UI
  └─ Assigne live_transcription=true à user/role/permission

Backend — FeatureResolutionService (deny-wins)
  └─ GET /api/users/:id/features → { ..., live_transcription: true|false }

Mobile — useSyncUserFeatures → settingsStore.setFeatures()
  └─ settingsStore.getFeature('live_transcription') → true | false

CaptureScreen
  └─ showLiveTranscription = getFeature(FEATURE_KEYS.LIVE_TRANSCRIPTION)
       ├─ false → captureTools = [voice, text]           (OFF par défaut)
       └─ true  → captureTools = [voice, text, live]
```

### 📋 Conformité ADR

| ADR | Impact | Décision |
|-----|--------|----------|
| **ADR-024** (Clean Code / SRP) | Le gating est dans `CaptureScreen` (consommateur), pas dans `useLiveTranscription` (logique métier) — bonne séparation | ✅ |
| **ADR-022** (Persistence) | `features` dans `settingsStore` est non-persisté (`partialize`) — conforme au pattern Epic 24 | ✅ |
| **ADR-023** (Result Pattern) | Aucune nouvelle méthode service — `getFeature()` est synchrone et retourne `boolean` (existant) | ✅ |

### ⚠️ Point de vigilance — Backend

Vérifier dans le code Epic 24 (story 24-1) :
- Le nom exact de la table (`features` ? `feature_catalog` ?)
- Les colonnes (`key`, `name`, `description`, `default_value` ? ou autre ?)
- Si seed ou migration est le mécanisme retenu pour ajouter des features

Consulter `backend/src/database/migrations/` ou `seeds/` pour le pattern.

### References

- [Source: `mobile/src/screens/capture/CaptureScreen.tsx` lignes 71-93] — `CAPTURE_TOOLS_ALWAYS` + tool "live" à déplacer
- [Source: `mobile/src/screens/capture/CaptureScreen.tsx` lignes ~284-288] — pattern `showMediaButtons` à reproduire
- [Source: `mobile/src/contexts/identity/domain/feature-keys.ts`] — constantes FEATURE_KEYS existantes
- [Source: `mobile/src/stores/settingsStore.ts`] — `getFeature()` sélecteur Zustand
- [Source: Story 24-3] — pattern complet gating feature flags mobile
- [Source: Story 24-1] — structure catalogue backend + migrations/seeds
- [Source: Story 8.6 — `8-6-transcription-live-avec-waveform.md`] — comportement Live à préserver

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

**Créés :**
- `pensieve/mobile/tests/acceptance/features/story-8-21-feature-flag-live-transcription.feature`
- `pensieve/mobile/tests/acceptance/story-8-21-feature-flag-live-transcription.test.ts`
- `pensieve/backend/src/migrations/1780700000000-SeedLiveTranscriptionFeature.ts`
- `pensieve/mobile/src/screens/capture/capture-tools.ts` — module pur (constantes + `computeCaptureTools`)

**Modifiés :**
- `pensieve/mobile/src/contexts/identity/domain/feature-keys.ts` — ajout `LIVE_TRANSCRIPTION`
- `pensieve/mobile/src/screens/capture/CaptureScreen.tsx` — import depuis `capture-tools.ts`, sélecteur `isLiveTranscriptionEnabled`, appel `computeCaptureTools()`

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-02-28 | Story créée — gap identifié : bouton Live (Story 8.6) ajouté sans feature flag. Pattern Epic 24 (`CAPTURE_MEDIA_BUTTONS`) reproduit pour `live_transcription`. OFF par défaut. Périmètre minimal : 1 seed backend + 1 constante FEATURE_KEYS + 3 lignes CaptureScreen. | yohikofox |
| 2026-03-05 | Implémentation complète — TDD RED→GREEN : 9 tests (6 unit + 3 BDD), 0 régression. LIVE_TRANSCRIPTION dans FEATURE_KEYS, CAPTURE_TOOLS_LIVE extrait de CAPTURE_TOOLS_ALWAYS, sélecteur `isLiveTranscriptionEnabled` dans CaptureScreenContent, migration backend créée. Status → review. | yohikofox |
| 2026-03-05 | Code review adversarial — 4 issues fixées : (H1) commit pensieve cb83ed3, (H2) extraction `capture-tools.ts` (module pur) + tests importent vraies sources, (M1) assertion AC3 enrichie (`iconName`/`color`/`labelKey`), (M2) SQL Task 1.1 corrigé (suppression colonne `name` inexistante). Status → done. | yohikofox |

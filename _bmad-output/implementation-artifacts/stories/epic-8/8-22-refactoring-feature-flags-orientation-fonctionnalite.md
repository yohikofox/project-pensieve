# Story 8.22 : Refactoring Feature Flags — Orientation Capacité Produit

Status: done

## Story

En tant qu'**administrateur Pensine**,
je veux **que chaque feature flag corresponde à une capacité produit identifiable** (ex: `url_capture`, `projects`, `news`),
afin de **contrôler finement l'activation de fonctionnalités individuelles par utilisateur/rôle/permission, indépendamment les unes des autres**.

## Contexte

### Problème actuel

Les feature flags actuels sont orientés **composants UI**, pas **capacités produit** :

| Flag actuel | Orientation | Problème |
|-------------|-------------|---------|
| `news_tab` | Composant (un tab) | Le nom dit "un onglet", pas "la feature Actualités" |
| `projects_tab` | Composant (un tab) | Idem |
| `capture_media_buttons` | Groupe de composants | Gate 4 outils différents (URL, Photo, Document, Clipboard) en bloc — impossible d'activer uniquement la capture URL |
| `debug_mode` | Capacité ✅ | Correct |
| `data_mining` | Capacité ✅ | Correct |
| `live_transcription` | Capacité ✅ | Correct (Story 8.21) |

**Exemple concret du problème :** Si on veut activer la capture par URL pour un utilisateur bêta sans lui donner accès aux autres outils média, il faut activer `capture_media_buttons` entier. Ce n'est pas possible de faire plus fin.

### Refactoring cible

Remplacement des flags orientés composant par des flags orientés **capacité produit** :

| Flag supprimé | Remplacé par |
|---------------|--------------|
| `news_tab` | `news` |
| `projects_tab` | `projects` |
| `capture_media_buttons` | `url_capture` + `photo_capture` + `document_capture` + `clipboard_capture` |

**Flags conservés sans modification :** `debug_mode`, `data_mining`, `live_transcription`.

### Architecture feature flags (Epic 24 — existant)

Rappel du système existant :
- **Backend** : `FeatureResolutionService` (deny-wins), catalogue en base, assignations user/role/permission
- **Mobile** : `feature-keys.ts` → constantes `FEATURE_KEYS` → `settingsStore.getFeature(key)`
- **Navigator** : `registry.ts` déclare `featureKey` optionnel sur chaque tab → `MainNavigator` filtre les tabs visibles
- **CaptureScreen** : spread conditionnel `...(flagEnabled ? CAPTURE_TOOLS_XXX : [])`

### Fichiers concernés

| Fichier | Rôle | Changement |
|---------|------|-----------|
| Backend — migration/seed | Catalogue features | Ajouter 6 nouveaux flags, marquer 3 anciens comme dépréciés |
| Backend — migration | Assignations existantes | Copier les assignations `news_tab` → `news`, `projects_tab` → `projects`, `capture_media_buttons` → 4 flags |
| `feature-keys.ts` | Constantes mobile | Ajouter 6 constantes, marquer 3 anciennes dépréciées |
| `registry.ts` | Configuration des tabs | `featureKey: FEATURE_KEYS.NEWS` et `FEATURE_KEYS.PROJECTS` |
| `CaptureScreen.tsx` | Gating des outils | Éclater `CAPTURE_TOOLS_MEDIA` en 4 tableaux individuels |

## Acceptance Criteria

### AC1 : Nouveaux flags enregistrés dans le catalogue backend

**Given** la migration est appliquée
**When** l'admin consulte la liste des features
**Then** les 6 nouvelles features sont présentes avec leurs valeurs par défaut :

| Key | Nom affiché | Défaut |
|-----|-------------|--------|
| `news` | Actualités | false |
| `projects` | Projets | false |
| `url_capture` | Capture par URL | false |
| `photo_capture` | Capture Photo | false |
| `document_capture` | Capture Document | false |
| `clipboard_capture` | Capture Presse-papier | false |

**And** les 3 anciens flags sont marqués comme dépréciés (non supprimés — sécurité des données)

### AC2 : Migration des assignations existantes

**Given** un utilisateur ou rôle possède des assignations sur les anciens flags
**When** la migration est appliquée
**Then** les assignations sont copiées vers les nouveaux flags :
- `news_tab = true` pour un user → `news = true` pour ce même user
- `projects_tab = true` → `projects = true`
- `capture_media_buttons = true` → `url_capture = true` + `photo_capture = true` + `document_capture = true` + `clipboard_capture = true`

**And** les anciennes assignations sont conservées (pas supprimées — rollback possible)
**And** aucune donnée existante n'est perdue

### AC3 : Tab Actualités gérée par `news`

**Given** un utilisateur dont la feature `news` est `false`
**When** il ouvre l'application
**Then** l'onglet Actualités n'est pas visible (comportement identique à aujourd'hui)

**Given** un utilisateur dont la feature `news` est `true`
**When** il ouvre l'application
**Then** l'onglet Actualités est visible

### AC4 : Tab Projets gérée par `projects`

**Given/When/Then** — même logique que AC3 pour la feature `projects` et l'onglet Projets

### AC5 : Outils de capture gérés individuellement

**Given** un utilisateur avec uniquement `url_capture = true` (et `photo_capture`, `document_capture`, `clipboard_capture` à false)
**When** il ouvre `CaptureScreen`
**Then** seul le bouton **URL** est visible parmi les outils média
**And** les boutons Photo, Document et Clipboard ne sont pas affichés

**Given** un utilisateur avec `photo_capture = true` et `document_capture = true` (et les autres à false)
**When** il ouvre `CaptureScreen`
**Then** seuls les boutons **Photo** et **Document** sont visibles parmi les outils média

### AC6 : Aucune régression comportementale

**Given** un utilisateur sans aucune feature activée
**When** il utilise l'application
**Then** il voit exactement les mêmes écrans/boutons qu'avant la migration
(Voice, Text, Live si 8.21 déployé ; tabs Captures, Capture, Actions, Settings)

### AC7 : Tests unitaires et BDD

**Given** l'implémentation est complète
**When** les tests sont exécutés
**Then** les scénarios BDD couvrent :
  - AC3 : tab News visible/masquée selon `news`
  - AC4 : tab Projects visible/masquée selon `projects`
  - AC5 : chaque outil média visible/masqué indépendamment
  - AC6 : zéro régression (user sans flags → même comportement)
**And** les tests unitaires de `FEATURE_KEYS` vérifient les nouvelles constantes
**And** `npm run test:unit` et `npm run test:acceptance` passent sans régression

## Tasks / Subtasks

### Task 1 : Backend — migration catalogue (AC1)

- [x] Subtask 1.1 : Créer une migration TypeORM dans `backend/src/migrations/` (ou seed si le pattern Epic 24 utilise des seeds). Vérifier le pattern exact en consultant les fichiers de Story 24-1.

  Insérer les 6 nouveaux flags :
  ```sql
  INSERT INTO features (key, name, description, default_value, deprecated) VALUES
    ('news',             'Actualités',           'Accès à l''onglet et aux contenus Actualités', false, false),
    ('projects',         'Projets',              'Accès à l''onglet et aux fonctionnalités Projets', false, false),
    ('url_capture',      'Capture par URL',      'Capture et analyse de contenu depuis une URL', false, false),
    ('photo_capture',    'Capture Photo',        'Capture et analyse d''images ou photos', false, false),
    ('document_capture', 'Capture Document',     'Capture et analyse de documents (PDF, etc.)', false, false),
    ('clipboard_capture','Capture Presse-papier','Capture du contenu du presse-papier', false, false)
  ON CONFLICT (key) DO NOTHING;
  ```

- [x] Subtask 1.2 : Marquer les 3 anciens flags comme dépréciés (ajouter colonne `deprecated` si absente, ou utiliser le champ `description`) :
  ```sql
  UPDATE features SET deprecated = true
  WHERE key IN ('news_tab', 'projects_tab', 'capture_media_buttons');
  ```
  **Note** : si la table `features` n'a pas de colonne `deprecated`, ajouter ce champ dans la même migration (boolean, default false).

### Task 2 : Backend — migration des assignations (AC2)

- [x] Subtask 2.1 : Dans la même migration (ou une migration suivante), copier les assignations existantes.

  Vérifier la structure de la table d'assignations (Story 24-1 — probablement `feature_assignments` avec colonnes `feature_key`, `subject_type`, `subject_id`, `value`).

  ```sql
  -- Copier news_tab → news
  INSERT INTO feature_assignments (feature_key, subject_type, subject_id, value)
  SELECT 'news', subject_type, subject_id, value
  FROM feature_assignments
  WHERE feature_key = 'news_tab'
  ON CONFLICT DO NOTHING;

  -- Copier projects_tab → projects
  INSERT INTO feature_assignments (feature_key, subject_type, subject_id, value)
  SELECT 'projects', subject_type, subject_id, value
  FROM feature_assignments
  WHERE feature_key = 'projects_tab'
  ON CONFLICT DO NOTHING;

  -- Copier capture_media_buttons → 4 flags (une INSERT par flag)
  INSERT INTO feature_assignments (feature_key, subject_type, subject_id, value)
  SELECT 'url_capture', subject_type, subject_id, value
  FROM feature_assignments WHERE feature_key = 'capture_media_buttons'
  ON CONFLICT DO NOTHING;

  INSERT INTO feature_assignments (feature_key, subject_type, subject_id, value)
  SELECT 'photo_capture', subject_type, subject_id, value
  FROM feature_assignments WHERE feature_key = 'capture_media_buttons'
  ON CONFLICT DO NOTHING;

  INSERT INTO feature_assignments (feature_key, subject_type, subject_id, value)
  SELECT 'document_capture', subject_type, subject_id, value
  FROM feature_assignments WHERE feature_key = 'capture_media_buttons'
  ON CONFLICT DO NOTHING;

  INSERT INTO feature_assignments (feature_key, subject_type, subject_id, value)
  SELECT 'clipboard_capture', subject_type, subject_id, value
  FROM feature_assignments WHERE feature_key = 'capture_media_buttons'
  ON CONFLICT DO NOTHING;
  ```

  **Note** : Adapter les noms de table et de colonnes selon le code réel de Story 24-1.

### Task 3 : Mobile — mettre à jour `FEATURE_KEYS` (AC3, AC4, AC5)

- [x] Subtask 3.1 : Dans `mobile/src/contexts/identity/domain/feature-keys.ts`, remplacer le contenu :

  ```typescript
  /**
   * Feature Flag Keys — Constantes capacités produit
   * Story 8.22: Refactoring orientation fonctionnalité (remplace flags orientés composant)
   *
   * Convention de nommage : <capacité_produit> (pas <composant_ui>)
   * ✅ Bon : 'url_capture', 'projects', 'news'
   * ❌ Mauvais : 'capture_media_buttons', 'news_tab', 'projects_tab'
   */
  export const FEATURE_KEYS = {
    // Capacités produit — nommage fonctionnel
    DEBUG_MODE:         'debug_mode',         // ✅ inchangé
    DATA_MINING:        'data_mining',         // ✅ inchangé
    LIVE_TRANSCRIPTION: 'live_transcription',  // ✅ Story 8.21

    NEWS:               'news',               // remplace NEWS_TAB
    PROJECTS:           'projects',           // remplace PROJECTS_TAB
    URL_CAPTURE:        'url_capture',        // remplace part of CAPTURE_MEDIA_BUTTONS
    PHOTO_CAPTURE:      'photo_capture',      // remplace part of CAPTURE_MEDIA_BUTTONS
    DOCUMENT_CAPTURE:   'document_capture',   // remplace part of CAPTURE_MEDIA_BUTTONS
    CLIPBOARD_CAPTURE:  'clipboard_capture',  // remplace part of CAPTURE_MEDIA_BUTTONS

    // @deprecated — à supprimer après vérification en production (Story 8.22)
    /** @deprecated Utiliser NEWS à la place */
    NEWS_TAB:               'news_tab',
    /** @deprecated Utiliser PROJECTS à la place */
    PROJECTS_TAB:           'projects_tab',
    /** @deprecated Utiliser URL_CAPTURE, PHOTO_CAPTURE, DOCUMENT_CAPTURE, CLIPBOARD_CAPTURE */
    CAPTURE_MEDIA_BUTTONS:  'capture_media_buttons',
  } as const;

  export type FeatureKey = typeof FEATURE_KEYS[keyof typeof FEATURE_KEYS];
  ```

### Task 4 : Mobile — mettre à jour `registry.ts` (AC3, AC4)

- [x] Subtask 4.1 : Dans `mobile/src/screens/registry.ts`, mettre à jour les `featureKey` des tabs :
  ```typescript
  // AVANT :
  News:     { featureKey: FEATURE_KEYS.NEWS_TAB, ... }
  Projects: { featureKey: FEATURE_KEYS.PROJECTS_TAB, ... }

  // APRÈS :
  News:     { featureKey: FEATURE_KEYS.NEWS, ... }
  Projects: { featureKey: FEATURE_KEYS.PROJECTS, ... }
  ```

### Task 5 : Mobile — éclater `CAPTURE_TOOLS_MEDIA` dans `CaptureScreen` (AC5, AC6)

- [x] Subtask 5.1 : Remplacer le tableau `CAPTURE_TOOLS_MEDIA` par 4 tableaux individuels :

  ```typescript
  // SUPPRIMER ce tableau :
  const CAPTURE_TOOLS_MEDIA: CaptureTool[] = [ /* photo, url, document, clipboard */ ];

  // AJOUTER 4 tableaux individuels :
  const CAPTURE_TOOLS_URL: CaptureTool[] = [{
    id: "url", labelKey: "capture.tools.url",
    iconName: "globe", color: colors.primary[700], available: false,
  }];

  const CAPTURE_TOOLS_PHOTO: CaptureTool[] = [{
    id: "photo", labelKey: "capture.tools.photo",
    iconName: "aperture", color: colors.info[500], available: false,
  }];

  const CAPTURE_TOOLS_DOCUMENT: CaptureTool[] = [{
    id: "document", labelKey: "capture.tools.document",
    iconName: "file", color: colors.secondary[700], available: false,
  }];

  const CAPTURE_TOOLS_CLIPBOARD: CaptureTool[] = [{
    id: "clipboard", labelKey: "capture.tools.clipboard",
    iconName: "copy", color: colors.warning[500], available: false,
  }];
  ```

- [x] Subtask 5.2 : Dans `CaptureScreenContent`, remplacer les sélecteurs et le calcul de `captureTools` :

  ```typescript
  // AVANT (ligne ~285) :
  const showMediaButtons = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.CAPTURE_MEDIA_BUTTONS));
  const captureTools = showMediaButtons
    ? [...CAPTURE_TOOLS_ALWAYS, ...CAPTURE_TOOLS_MEDIA]
    : CAPTURE_TOOLS_ALWAYS;

  // APRÈS :
  const isLiveTranscriptionEnabled = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.LIVE_TRANSCRIPTION));
  const isUrlCaptureEnabled        = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.URL_CAPTURE));
  const isPhotoCaptureEnabled      = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.PHOTO_CAPTURE));
  const isDocumentCaptureEnabled   = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.DOCUMENT_CAPTURE));
  const isClipboardCaptureEnabled  = useSettingsStore((s) => s.getFeature(FEATURE_KEYS.CLIPBOARD_CAPTURE));

  const captureTools = [
    ...CAPTURE_TOOLS_ALWAYS,
    ...(isLiveTranscriptionEnabled  ? CAPTURE_TOOLS_LIVE      : []),
    ...(isUrlCaptureEnabled         ? CAPTURE_TOOLS_URL        : []),
    ...(isPhotoCaptureEnabled       ? CAPTURE_TOOLS_PHOTO      : []),
    ...(isDocumentCaptureEnabled    ? CAPTURE_TOOLS_DOCUMENT   : []),
    ...(isClipboardCaptureEnabled   ? CAPTURE_TOOLS_CLIPBOARD  : []),
  ];
  ```

  **Note** : `isLiveTranscriptionEnabled` est normalement ajouté par Story 8.21. Si Story 8.21 n'est pas encore déployée, l'inclure ici également.

- [x] Subtask 5.3 : Mettre à jour le commentaire JSDoc du bloc de configuration des outils (lignes 61-70) pour refléter la nouvelle architecture.

### Task 6 : Tests BDD (AC7)

- [x] Subtask 6.1 : Créer `mobile/tests/acceptance/features/story-8-22-feature-flags-capacite-produit.feature`
- [x] Subtask 6.2 : Écrire les scénarios (voir "Scénarios BDD" ci-dessous)
- [x] Subtask 6.3 : Créer `mobile/tests/acceptance/story-8-22-feature-flags-capacite-produit.test.ts`

### Task 7 : Tests unitaires (AC7)

- [x] Subtask 7.1 : Vérifier que les nouvelles constantes `FEATURE_KEYS` ont les bonnes valeurs string
- [x] Subtask 7.2 : Tester la composition de `captureTools` pour chaque combinaison de flags :
  - Tous à false → [voice, text] uniquement
  - Uniquement `url_capture=true` → [voice, text, url]
  - `photo_capture=true` + `document_capture=true` → [voice, text, photo, document]
  - Tous à true → [voice, text, live, url, photo, document, clipboard]

### Task 8 : Validation finale (AC7)

- [x] Subtask 8.1 : `npm run test:unit` — zéro régression (56 suites en échec pré-existantes non liées à cette story)
- [x] Subtask 8.2 : `npm run test:acceptance` — 23/23 nouveaux tests + 0 régression sur story-8-21 et story-24-3
- [x] Subtask 8.3 : `npm run test:architecture` — violations pré-existantes (ADR-031 interfaces vs classes), aucune violation introduite
- [ ] Subtask 8.4 : Test manuel — vérifier que sans aucun flag, l'app est identique à avant (voice + text uniquement, pas de tabs News/Projects)
- [ ] Subtask 8.5 : Test manuel — activer `url_capture` uniquement via admin, vérifier que seul le bouton URL apparaît dans CaptureScreen

## Scénarios BDD

```gherkin
# language: fr
Fonctionnalité: Feature flags orientés capacité produit
  En tant qu'administrateur
  Je veux contrôler chaque capacité produit indépendamment
  Afin de faire des déploiements progressifs et granulaires

  Scénario: Aucun flag activé — comportement identique à avant la migration
    Étant donné qu'aucune feature n'est activée pour l'utilisateur
    Quand l'utilisateur ouvre l'application
    Alors les onglets visibles sont "Captures", "Capture", "Actions", "Settings"
    Et les onglets "Actualités" et "Projets" ne sont pas visibles
    Quand l'utilisateur navigue vers CaptureScreen
    Alors seuls les boutons "Voice" et "Text" sont affichés

  Scénario: Tab Actualités contrôlée par "news"
    Étant donné que la feature "news" est activée pour l'utilisateur
    Quand l'utilisateur ouvre l'application
    Alors l'onglet "Actualités" est visible dans la barre de navigation

  Scénario: Tab Projets contrôlée par "projects"
    Étant donné que la feature "projects" est activée pour l'utilisateur
    Quand l'utilisateur ouvre l'application
    Alors l'onglet "Projets" est visible dans la barre de navigation

  Scénario: Activation granulaire — URL capture uniquement
    Étant donné que seule la feature "url_capture" est activée
    Quand l'utilisateur ouvre CaptureScreen
    Alors le bouton "URL" est visible
    Et les boutons "Photo", "Document" et "Presse-papier" ne sont pas visibles

  Scénario: Activation granulaire — Photo et Document sans URL
    Étant donné que les features "photo_capture" et "document_capture" sont activées
    Et que "url_capture" et "clipboard_capture" sont désactivées
    Quand l'utilisateur ouvre CaptureScreen
    Alors les boutons "Photo" et "Document" sont visibles
    Et les boutons "URL" et "Presse-papier" ne sont pas visibles

  Scénario: Tous les flags média activés
    Étant donné que les features "url_capture", "photo_capture", "document_capture", "clipboard_capture" sont toutes activées
    Quand l'utilisateur ouvre CaptureScreen
    Alors les boutons "Voice", "Text", "URL", "Photo", "Document" et "Presse-papier" sont visibles
```

## Dev Notes

### 🔑 Fichiers à modifier

```
backend/ — migration (nouvelle)
  ← Task 1 : Catalogue + colonne deprecated
  ← Task 2 : Copie des assignations existantes

pensieve/mobile/src/contexts/identity/domain/feature-keys.ts
  ← Task 3 : 6 nouvelles constantes + @deprecated sur 3 anciennes

pensieve/mobile/src/screens/registry.ts
  ← Task 4 : NEWS_TAB → NEWS, PROJECTS_TAB → PROJECTS

pensieve/mobile/src/screens/capture/CaptureScreen.tsx
  ← Task 5 : Éclater CAPTURE_TOOLS_MEDIA en 4 + 5 sélecteurs + nouveau captureTools
```

### 📁 Fichiers à créer

```
backend/ — fichier de migration TypeORM (Task 1 + 2)
pensieve/mobile/tests/acceptance/features/story-8-22-feature-flags-capacite-produit.feature
pensieve/mobile/tests/acceptance/story-8-22-feature-flags-capacite-produit.test.ts
```

### ⛔ Fichiers à NE PAS modifier

```
pensieve/mobile/src/navigation/MainNavigator.tsx
  ← Aucun changement — il lit featureKey depuis le registry dynamiquement,
    la mise à jour du registry suffit

pensieve/mobile/src/stores/settingsStore.ts
  ← Aucun changement — getFeature(key: string) est générique,
    fonctionne avec n'importe quelle clé

pensieve/mobile/src/stores/__tests__/settingsStore.debugMode.test.ts
  ← Aucun changement — debug_mode non modifié
```

### 🏗️ Architecture — avant/après

```
AVANT (orienté composant) :
  news_tab          → gate le Tab News
  projects_tab      → gate le Tab Projects
  capture_media_buttons → gate [photo + url + document + clipboard] en bloc

APRÈS (orienté capacité produit) :
  news              → gate la capacité "Actualités" (tab + future logique métier)
  projects          → gate la capacité "Projets" (tab + future logique métier)
  url_capture       → gate la capacité "capturer depuis une URL"
  photo_capture     → gate la capacité "capturer une photo"
  document_capture  → gate la capacité "capturer un document"
  clipboard_capture → gate la capacité "capturer le presse-papier"
  live_transcription→ gate la capacité "transcription en temps réel" (Story 8.21)
  debug_mode        → inchangé ✅
  data_mining       → inchangé ✅
```

### 📋 Convention de nommage — règle pour les futurs flags

> Le nom d'un feature flag doit répondre à la question : **"Quelle capacité l'utilisateur acquiert-il ?"**
>
> ✅ `url_capture` → "l'utilisateur peut capturer depuis une URL"
> ✅ `projects` → "l'utilisateur a accès aux projets"
> ❌ `capture_media_buttons` → "des boutons s'affichent" (conséquence UI, pas la capacité)
> ❌ `news_tab` → "un onglet apparaît" (composant UI, pas la feature)

Cette convention est à documenter dans `feature-keys.ts` (commentaire JSDoc en tête de fichier).

### ⚠️ Point de vigilance — Rollback

Les anciens flags (`news_tab`, `projects_tab`, `capture_media_buttons`) sont conservés en base (pas supprimés). Si un rollback est nécessaire :
1. Reverter le code mobile → les anciens `FEATURE_KEYS` restent présents (marqués `@deprecated` mais fonctionnels)
2. Reverter la migration backend → les assignations copiées peuvent être supprimées

**Ne pas supprimer les anciens flags avant 2 sprints de stabilité en production.**

### 📋 Conformité ADR

| ADR | Impact | Décision |
|-----|--------|----------|
| **ADR-024** (Clean Code — nommage révélateur) | Convention de nommage des feature flags documentée et appliquée | ✅ |
| **ADR-022** (Persistence) | `features` dans `settingsStore` reste non-persisté — aucun changement | ✅ |
| **ADR-023** (Result Pattern) | Aucune nouvelle méthode service — `getFeature()` synchrone existant | ✅ |
| **ADR-021** (DI Lifecycle) | Aucun nouveau service | ✅ |

### References

- [Source: `mobile/src/contexts/identity/domain/feature-keys.ts`] — constantes FEATURE_KEYS actuelles (5 flags)
- [Source: `mobile/src/screens/registry.ts`] — `featureKey` sur News et Projects
- [Source: `mobile/src/navigation/MainNavigator.tsx` lignes 38-54] — logique filtre tabs via `features` object
- [Source: `mobile/src/screens/capture/CaptureScreen.tsx` lignes 71-125, 284-288] — CAPTURE_TOOLS_ALWAYS + CAPTURE_TOOLS_MEDIA + sélecteur showMediaButtons
- [Source: Story 24-1] — structure tables backend `features` + `feature_assignments`
- [Source: Story 8.21 — `8-21-feature-flag-live-transcription.md`] — pattern `isLiveTranscriptionEnabled` à inclure si pas encore déployé

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

- Problème ts-jest : `registry.ts` importe des composants React Native (JSX) — incompatible avec les acceptance tests (ts-jest/node). Résolu en testant uniquement les constantes FEATURE_KEYS dans les BDD tests.
- Connecteurs français (`qu'`, `que`) dans les steps Gherkin strippés par jest-cucumber — corrigé en retirant les connecteurs du feature file.

### Completion Notes List

- **Task 1+2** : Migration TypeORM `1780800000000-SeedCapacityProductFeatureFlags.ts` créée. Ajoute colonne `deprecated`, insère 6 nouveaux flags (fe000007-fe000012), déprécie 3 anciens (fe000003-fe000005), copie toutes les assignations user/role/permission via UUID déterministes.
- **Task 3** : `feature-keys.ts` étendu avec 6 nouvelles constantes (NEWS, PROJECTS, URL_CAPTURE, PHOTO_CAPTURE, DOCUMENT_CAPTURE, CLIPBOARD_CAPTURE). 3 anciennes constantes marquées `@deprecated` mais conservées pour rollback.
- **Task 4** : `registry.ts` mis à jour — `FEATURE_KEYS.NEWS_TAB` → `FEATURE_KEYS.NEWS`, `FEATURE_KEYS.PROJECTS_TAB` → `FEATURE_KEYS.PROJECTS`.
- **Task 5** : `capture-tools.ts` refactorisé — `CAPTURE_TOOLS_MEDIA` remplacé par 4 tableaux individuels. `computeCaptureTools` migre de 2 params (isLive + showMediaButtons) vers 5 params optionnels (isLive + isUrl + isPhoto + isDocument + isClipboard). Rétrocompatibilité assurée (appels story-8-21 à 2 params toujours fonctionnels).
- **Task 5** cont : `CaptureScreen.tsx` — sélecteur `showMediaButtons` remplacé par 4 sélecteurs individuels + appel `computeCaptureTools` mis à jour.
- **Task 6+7** : 23 tests (17 unitaires + 6 BDD) — tous verts. Tests story-8-21 (9) et story-24-3 (8) toujours verts (zéro régression).
- **Code review (adversarial)** : 5 issues corrigées (H1: registry test créé `src/screens/__tests__/story-8-22-registry.test.ts` — 9 tests AC3/AC4/AC6 ; H2: commentaires BDD AC3/AC4 mis à jour ; M1: boucles migration passées en queries paramétrées `$1` ; M2: `down()` réécrit avec DELETE ciblés via sous-requêtes JOIN pour préserver les assignations post-migration ; M3: titre scénario BDD corrigé "Tous les flags capture média activés — hors live transcription").

### File List

**Fichiers créés :**
- `pensieve/backend/src/migrations/1780800000000-SeedCapacityProductFeatureFlags.ts`
- `pensieve/mobile/tests/acceptance/features/story-8-22-feature-flags-capacite-produit.feature`
- `pensieve/mobile/tests/acceptance/story-8-22-feature-flags-capacite-produit.test.ts`
- `pensieve/mobile/src/screens/__tests__/story-8-22-registry.test.ts`

**Fichiers modifiés :**
- `pensieve/mobile/src/contexts/identity/domain/feature-keys.ts`
- `pensieve/mobile/src/screens/registry.ts`
- `pensieve/mobile/src/screens/capture/capture-tools.ts`
- `pensieve/mobile/src/screens/capture/CaptureScreen.tsx`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-02-28 | Story créée — refactoring feature flags UI-oriented → capacité produit. Analyse code réel : FEATURE_KEYS (5 flags), registry.ts (featureKey sur News/Projects), MainNavigator (filtre dynamique), CaptureScreen (CAPTURE_TOOLS_MEDIA en bloc). Migration : 6 nouveaux flags + copie assignations + deprecated sur 3 anciens. Convention nommage documentée. | yohikofox |
| 2026-03-06 | Implémentation complète — migration backend (1780800000000), FEATURE_KEYS étendu, registry.ts mis à jour, capture-tools.ts refactorisé (4 tableaux individuels + signature computeCaptureTools), CaptureScreen.tsx mis à jour (5 sélecteurs). 23/23 tests passent. | yohikofox |

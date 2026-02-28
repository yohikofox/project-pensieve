# Story 8.4: Transcription Native par Défaut

Status: ready-for-dev

<!-- Validation optionnelle : run validate-create-story avant dev-story -->

## Story

En tant qu'**utilisateur qui installe Pensine pour la première fois**,
je veux **pouvoir créer une capture vocale immédiatement sans télécharger de modèle**,
afin que **l'onboarding soit fluide et sans friction au premier lancement**.

**GitHub Issue:** [#12 - feat: Changer le mode de transcription par défaut de Whisper vers Natif](https://github.com/yohikofox/pensieve/issues/12)

## Contexte

### État Actuel

`TranscriptionEngineService.getSelectedEngineType()` retourne `'whisper'` par défaut (ligne 53) quand aucune préférence n'est persistée en AsyncStorage. Cela signifie que :
- Les nouveaux utilisateurs voient le guide de téléchargement de modèle Whisper dès le premier lancement
- Aucune capture vocale n'est possible tant que le modèle n'est pas téléchargé (~75 MB minimum)
- **Friction inutile** : la transcription native (iOS Speech Framework / Android SpeechRecognizer) est disponible immédiatement, sans téléchargement

### Interaction avec Story 24.4 (FirstLaunchInitializer)

`FirstLaunchInitializer.run()` (story 24.4) configure 'native' **uniquement pour Pixel 9+** (détection NPU Tensor G4+). Pour tous les autres appareils, aucun moteur n'est configuré explicitement → défaut 'whisper' implicite.

**Story 8-4 généralise ce comportement** : 'native' par défaut pour TOUS les nouveaux utilisateurs.

### Contrainte Migration

Les utilisateurs existants (ceux qui ont déjà lancé l'app, donc `FIRST_LAUNCH_KEY = 'true'` en AsyncStorage) utilisaient 'whisper' implicitement. **Ne pas modifier leur expérience.** Une migration one-time est nécessaire pour leur persister explicitement 'whisper'.

## Acceptance Criteria

### AC1: Transcription native par défaut pour les nouveaux utilisateurs

**Given** un utilisateur installe Pensine pour la première fois (non-Pixel 9+)
**When** il tente de transcrire un audio
**Then** la transcription native (iOS/Android natif) est utilisée
**And** aucun téléchargement de modèle n'est requis avant la première capture
**And** `TranscriptionEngineService.getSelectedEngineType()` retourne `'native'` par défaut quand aucune préférence n'est persistée

### AC2: Migration silencieuse des utilisateurs existants

**Given** un utilisateur existant (déjà lancé l'app avant story 8.4, donc `FIRST_LAUNCH_KEY = 'true'`)
**When** l'app est mise à jour et `getSelectedEngineType()` est appelé
**Then** sa préférence est migrée une seule fois vers 'whisper' explicitement (persisté en AsyncStorage)
**And** il continue à utiliser Whisper comme auparavant
**And** la migration ne s'exécute qu'une seule fois (clé de marquage `@pensieve/transcription_default_migrated`)
**And** aucun message d'alerte ou popup ne s'affiche à l'utilisateur lors de la migration

### AC3: FirstLaunchInitializer — 'native' pour tous les nouveaux appareils

**Given** un nouveau utilisateur lance l'app pour la première fois (non-Pixel 9+)
**When** `FirstLaunchInitializer.run()` s'exécute
**Then** `engineService.setSelectedEngineType('native')` est appelé pour **tous les appareils** (pas seulement Pixel 9+)
**And** les Pixel 9+ continuent à recevoir 'native' + Gemma 3 1B (comportement story 24.4 inchangé)
**And** `FIRST_LAUNCH_KEY = 'true'` est persisté après initialisation (comportement existant préservé)

### AC4: Utilisateurs avec préférence explicite inchangés

**Given** un utilisateur a explicitement choisi 'whisper' ou 'native' dans les settings
**When** l'app est mise à jour (clé `@pensieve/transcription_engine` non-null en AsyncStorage)
**Then** sa préférence explicite est conservée
**And** aucune migration ne modifie sa valeur
**And** le comportement est identique avant et après la story 8.4

### AC5: UI Settings inchangée

**Given** `TranscriptionEngineSettingsScreen.tsx` existe déjà
**When** un utilisateur accède aux paramètres de transcription
**Then** il peut toujours choisir entre 'native' et 'whisper' manuellement
**And** l'état par défaut affiché reflète 'native' pour les nouveaux utilisateurs
**And** aucune modification UI n'est nécessaire (écran existant fonctionnel)

### AC6: Tests unitaires et BDD

**Given** les changements sont implémentés
**When** je lance les tests
**Then** les tests unitaires de `TranscriptionEngineService` couvrent les 3 cas : (1) préférence explicite, (2) migration utilisateur existant, (3) nouvel utilisateur → 'native'
**And** les tests de `FirstLaunchInitializer` couvrent le cas non-Pixel 9+ → 'native' défaut
**And** `npm run test:unit` passe sans régression
**And** `npm run test:acceptance` passe sans régression

## Tasks / Subtasks

### Task 1: Modifier `TranscriptionEngineService.ts` — Nouveau défaut + Migration (AC1, AC2, AC4)

- [ ] Subtask 1.1 : Ouvrir `mobile/src/contexts/Normalization/services/TranscriptionEngineService.ts`
- [ ] Subtask 1.2 : Ajouter les constantes de migration après `STORAGE_KEY` :
  ```typescript
  const STORAGE_KEY = '@pensieve/transcription_engine';
  const FIRST_LAUNCH_KEY = '@pensieve/first_launch_completed'; // Même valeur que FirstLaunchInitializer
  const MIGRATION_KEY = '@pensieve/transcription_default_migrated'; // Marquage one-time
  ```
- [ ] Subtask 1.3 : Remplacer `getSelectedEngineType()` par la logique avec migration :
  ```typescript
  async getSelectedEngineType(): Promise<TranscriptionEngineType> {
    try {
      // Cas 1 : Préférence explicite déjà définie → retourner telle quelle
      const stored = await AsyncStorage.getItem(STORAGE_KEY);
      if (stored === 'native' || stored === 'whisper') {
        this.currentEngineType = stored;
        return stored;
      }

      // Cas 2 : Aucune préférence — vérifier si migration est nécessaire
      const migrated = await AsyncStorage.getItem(MIGRATION_KEY);
      if (!migrated) {
        // Première fois après story 8.4 — détecter si utilisateur existant
        const firstLaunchDone = await AsyncStorage.getItem(FIRST_LAUNCH_KEY);
        if (firstLaunchDone === 'true') {
          // Utilisateur existant sans préférence → préserver 'whisper' implicite
          await AsyncStorage.setItem(STORAGE_KEY, 'whisper');
          await AsyncStorage.setItem(MIGRATION_KEY, 'done');
          this.currentEngineType = 'whisper';
          return 'whisper';
        }
        // Nouvel utilisateur → marquer migration done (FirstLaunchInitializer gère le reste)
        await AsyncStorage.setItem(MIGRATION_KEY, 'done');
      }
    } catch (error) {
      console.error('[TranscriptionEngineService] Failed to get preference:', error);
    }
    return 'native'; // Nouveau défaut : natif pour les nouveaux utilisateurs
  }
  ```
- [ ] Subtask 1.4 : Mettre à jour le commentaire de la ligne `return 'native'` (était `return 'whisper'`)
- [ ] Subtask 1.5 : Mettre à jour le commentaire JSDoc de `getSelectedEngineType()` pour refléter le nouveau comportement

### Task 2: Modifier `FirstLaunchInitializer.ts` — Étendre à tous les appareils (AC3)

- [ ] Subtask 2.1 : Ouvrir `mobile/src/contexts/identity/services/FirstLaunchInitializer.ts`
- [ ] Subtask 2.2 : Modifier le bloc `run()` pour appeler `setSelectedEngineType('native')` pour tous les appareils :
  ```typescript
  const npuInfo = await this.npuService.detectNPU();
  if (this.isPixel9Plus(npuInfo)) {
    // Pixel 9+ : native + Gemma 3 1B (comportement story 24.4 inchangé)
    await this.engineService.setSelectedEngineType('native');
    useSettingsStore.getState().setAutoTranscription(true);
    await this.downloadGemmaWithFallback(onProgress);
  } else {
    // Tous les autres appareils : native par défaut (story 8.4)
    await this.engineService.setSelectedEngineType('native');
  }
  ```
- [ ] Subtask 2.3 : Vérifier que `finally` persiste encore `FIRST_LAUNCH_KEY = 'true'` (comportement existant préservé)

### Task 3: Tests unitaires `TranscriptionEngineService` (AC6)

- [ ] Subtask 3.1 : Ouvrir ou créer `mobile/src/contexts/Normalization/services/__tests__/TranscriptionEngineService.test.ts`
- [ ] Subtask 3.2 : Ajouter/mettre à jour les tests pour couvrir les 3 cas de `getSelectedEngineType()` :
  ```
  Cas 1 — Préférence explicite 'native' → retourne 'native'
  Cas 2 — Préférence explicite 'whisper' → retourne 'whisper'
  Cas 3 — Aucune préférence, FIRST_LAUNCH_KEY = 'true', migration non faite → retourne 'whisper' (migration)
  Cas 4 — Aucune préférence, FIRST_LAUNCH_KEY = null, migration non faite → retourne 'native'
  Cas 5 — Aucune préférence, migration déjà faite → retourne 'native'
  ```
- [ ] Subtask 3.3 : Mocker `AsyncStorage` (pattern jest.mock existant dans le projet)

### Task 4: Tests unitaires `FirstLaunchInitializer` — Cas non-Pixel 9+ (AC3, AC6)

- [ ] Subtask 4.1 : Ouvrir `mobile/src/contexts/identity/services/__tests__/FirstLaunchInitializer.test.ts`
- [ ] Subtask 4.2 : Ajouter le test : "Non-Pixel 9+ device → setSelectedEngineType('native') called"
- [ ] Subtask 4.3 : Vérifier que le test existant "Pixel 9+ → native + Gemma" passe toujours

### Task 5: Tests d'intégration (AC6)

- [ ] Subtask 5.1 : Exécuter `npm run test:unit` dans `pensieve/mobile/` — zéro régression
- [ ] Subtask 5.2 : Exécuter `npm run test:acceptance` dans `pensieve/mobile/` — zéro régression
- [ ] Subtask 5.3 : Exécuter `npm run test:architecture` — conformité ADR maintenue

### Task 6: Fermeture issue (post-merge)

- [ ] Subtask 6.1 : Commenter l'issue #12 avec le commit reference et fermer

## Dev Notes

### 🔑 Fichier Principal : `TranscriptionEngineService.ts`

**Emplacement :** `pensieve/mobile/src/contexts/Normalization/services/TranscriptionEngineService.ts`

**Changement critique (ligne 53) :**
```typescript
// AVANT (comportement actuel)
return 'whisper'; // Default to Whisper

// APRÈS (story 8.4)
return 'native'; // Nouveau défaut pour les nouveaux utilisateurs
```

**Clés AsyncStorage impliquées :**
```typescript
const STORAGE_KEY = '@pensieve/transcription_engine';          // Préférence utilisateur
const FIRST_LAUNCH_KEY = '@pensieve/first_launch_completed';   // Posé par FirstLaunchInitializer
const MIGRATION_KEY = '@pensieve/transcription_default_migrated'; // Nouveau — marquage one-time
```

**⚠️ NOTE : `FIRST_LAUNCH_KEY` est définie à la fois dans `FirstLaunchInitializer.ts` et `TranscriptionEngineService.ts`** (même string `'@pensieve/first_launch_completed'`). Ne pas l'importer depuis `FirstLaunchInitializer.ts` pour éviter une dépendance circulaire (Normalization → identity). Dupliquer la constante est acceptable dans ce cas.

### 🏗️ Interaction Story 24.4 — Architecture

`FirstLaunchInitializer` injecte `TranscriptionEngineService` via DI (ligne 47). L'ordre d'exécution au premier lancement est :

```
bootstrap() → container.resolve(FirstLaunchInitializer) → run()
  → engineService.setSelectedEngineType('native')  [pour tous ou Pixel 9+]
  → AsyncStorage.setItem(FIRST_LAUNCH_KEY, 'true')

Plus tard : composant demande getSelectedEngineType()
  → STORAGE_KEY = 'native' (persisté par FirstLaunchInitializer)  ← Résultat propre
```

Pour les utilisateurs existants (app update) :
```
bootstrap() → FirstLaunchInitializer.run()
  → FIRST_LAUNCH_KEY = 'true' déjà là → return early (no-op)

Plus tard : getSelectedEngineType()
  → STORAGE_KEY = null (jamais explicitement défini)
  → MIGRATION_KEY = null (first time after update)
  → FIRST_LAUNCH_KEY = 'true' → existing user
  → Set STORAGE_KEY = 'whisper', MIGRATION_KEY = 'done'
  → return 'whisper' ← Migration transparente
```

### 📋 Pattern AsyncStorage

**Conforme ADR-022** : `STORAGE_KEY` contient un réglage UI non-critique → `AsyncStorage` OK.

Le pattern de read/write AsyncStorage dans ce service est déjà établi et conforme. Ne pas changer le mécanisme de stockage.

### 🧪 Tests Pattern

**Tests unitaires** (`babel-jest`) dans `src/contexts/Normalization/services/__tests__/`:

Mock pattern AsyncStorage (conforme au projet) :
```typescript
jest.mock('@react-native-async-storage/async-storage', () => ({
  getItem: jest.fn(),
  setItem: jest.fn(),
}));

// Dans chaque test :
(AsyncStorage.getItem as jest.Mock).mockImplementation((key: string) => {
  if (key === '@pensieve/transcription_engine') return Promise.resolve(null);
  if (key === '@pensieve/first_launch_completed') return Promise.resolve('true');
  if (key === '@pensieve/transcription_default_migrated') return Promise.resolve(null);
  return Promise.resolve(null);
});
```

### ⚠️ Garde-Fous Critiques

1. **Ne pas modifier `TranscriptionEngineSettingsScreen.tsx`** — l'UI existante fonctionne déjà correctement sans changement
2. **Ne pas modifier `settingsStore.ts`** — pas de nouveau state à ajouter pour cette story
3. **`FIRST_LAUNCH_KEY` constante dupliquée intentionnellement** — évite dépendance circulaire Normalization → identity. Acceptable car c'est une string littérale stable.
4. **La logique de migration est asynchrone** — ne pas introduire de `await` dans des chemins synchrones
5. **Tester sur device Android (non-Pixel 9+) et iOS** — vérifier que le défaut 'native' fonctionne sur les deux plateformes

### 🏗️ Architecture Compliance

- **ADR-022 (AsyncStorage)** : Clé de préférence UI non-critique → OK
- **ADR-021 (DI Lifecycle)** : `TranscriptionEngineService` et `FirstLaunchInitializer` sont déjà singletons — ne pas modifier leur lifecycle
- **ADR-023 (Result Pattern)** : `getSelectedEngineType()` retourne `TranscriptionEngineType` (pas de Result<T> nécessaire — erreur fallbackée silencieusement via catch)
- **ADR-024 (Clean Code)** : Constantes nommées, commentaires explicatifs sur les cas de migration

### Project Structure Notes

**Bounded Context Normalization :**
```
pensieve/mobile/src/contexts/Normalization/services/
├── TranscriptionEngineService.ts                           ← MODIFIER (Task 1)
└── __tests__/
    └── TranscriptionEngineService.test.ts                  ← CRÉER ou MODIFIER (Task 3)

pensieve/mobile/src/contexts/identity/services/
├── FirstLaunchInitializer.ts                               ← MODIFIER (Task 2)
└── __tests__/
    └── FirstLaunchInitializer.test.ts                      ← MODIFIER (Task 4)
```

**Fichiers à ne PAS modifier :**
```
pensieve/mobile/src/screens/settings/TranscriptionEngineSettingsScreen.tsx  ← AUCUNE MODIFICATION
pensieve/mobile/src/stores/settingsStore.ts                                  ← AUCUNE MODIFICATION
```

### References

- [Source: GitHub Issue #12](https://github.com/yohikofox/pensieve/issues/12) — Spécification originale
- [Source: `pensieve/mobile/src/contexts/Normalization/services/TranscriptionEngineService.ts#53`] — Défaut actuel 'whisper'
- [Source: `pensieve/mobile/src/contexts/identity/services/FirstLaunchInitializer.ts#62-67`] — Logique Pixel 9+ actuelle (story 24.4)
- [Source: `_bmad-output/implementation-artifacts/stories/epic-8/8-20-fix-native-transcription-resultat-tronque.md`] — Story liée : bug accumulation segments natif (DONE)
- [Source: `_bmad-output/implementation-artifacts/stories/epic-8/8-3-fix-audio-trim-truncating-transcriptions.md`] — Story liée : trim audio (ready-for-dev)
- [Source: ADR-022] — AsyncStorage pour préférences UI non-critiques

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-02-28 | Story créée depuis issue GitHub #12 — analyse complète du code existant (TranscriptionEngineService, FirstLaunchInitializer), stratégie migration one-time définie | yohikofox |

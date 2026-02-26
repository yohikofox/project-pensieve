# Story 25.3: Tour Démo — Re-déclenchement depuis les Settings

Status: ready-for-dev

## Story

En tant qu'**utilisateur ayant déjà vu le tour démo**,
je veux **pouvoir le relancer manuellement depuis les paramètres**,
afin de **revoir les fonctionnalités clés à tout moment**.

Dépendances :
- **Story 25.1** doit être complète (TourProvider, TourService)
- **Story 25.2** doit être complète (TourPreferenceRepository, logique shouldShowTour)

## Acceptance Criteria

### AC1 : Option dans les Settings
**Given** l'utilisateur ouvre l'écran Settings
**When** il navigue dans la section "À propos" ou "Aide"
**Then** il voit l'option "Revoir la visite guidée" (ou équivalent i18n)
**And** l'option est visible même si le tour a déjà été vu

### AC2 : Re-déclenchement forcé
**Given** l'utilisateur appuie sur "Revoir la visite guidée"
**When** la confirmation est donnée (optionnelle — voir AC3)
**Then** le tour démarre immédiatement, peu importe le statut local ou backend
**And** `shouldShowTour(forceFromSettings: true)` retourne `true` sans consulter local/backend

### AC3 : Pas de confirmation requise
**Given** l'utilisateur appuie sur "Revoir la visite guidée"
**When** il appuie sur le bouton
**Then** le tour démarre directement (pas de dialog de confirmation)
**And** si l'utilisateur était sur un autre écran (Settings), il est navigué vers CaptureScreen d'abord

### AC4 : Navigation vers CaptureScreen avant le tour
**Given** l'utilisateur est sur l'écran Settings
**When** il déclenche le tour
**Then** l'application navigue vers l'onglet Captures / CaptureScreen
**And** le tour démarre depuis l'étape 1

### AC5 : Le tour Settings n'écrase pas l'historique de complétion
**Given** l'utilisateur re-déclenche le tour depuis Settings
**When** il complète ou passe ce second tour
**Then** `tourCompleted` reste `true` en local et backend (pas de régression)
**And** la synchro backend n'est PAS re-déclenchée (déjà synchronisé)

### AC6 : Libellé i18n
**Given** la langue de l'app est définie
**When** l'option Settings est affichée
**Then** le libellé est traduit (clé i18n : `settings.replayTour`)
**And** les langues supportées : `fr` ("Revoir la visite guidée"), `en` ("Replay the tour")

## Tasks / Subtasks

### Mobile — Store & Service (modification story 25.1/25.2)

- [ ] **Task 1 : Ajouter `startTourFromSettings()` dans TourService** (AC2)
  - [ ] Nouvelle méthode publique dans `ITourService` : `startTourFromSettings(): void`
  - [ ] Implémentation : appelle `startTour(steps)` directement avec `forceFromSettings: true`
  - [ ] Ne ré-écrit PAS la persistance locale (AC5)
  - [ ] N'appelle PAS le sync backend (AC5)

### Mobile — Composant Settings

- [ ] **Task 2 : Option "Revoir la visite guidée" dans SettingsScreen** (AC1, AC3, AC4, AC6)
  - [ ] Localiser le fichier `SettingsScreen` existant (probablement dans `src/screens/settings/`)
  - [ ] Ajouter une entrée dans la section "Aide" ou "À propos"
  - [ ] Composant : `<SettingsRow label={t('settings.replayTour')} onPress={handleReplayTour} icon="play-circle" />`
  - [ ] Handler `handleReplayTour` :
    - [ ] Appeler `tourService.startTourFromSettings()`
    - [ ] Naviguer vers tab Captures (`navigation.navigate('Captures')`) — puis TourProvider démarre sur CaptureScreen (AC4)

- [ ] **Task 3 : Clés i18n** (AC6)
  - [ ] Ajouter `settings.replayTour` dans les fichiers de traduction français et anglais
  - [ ] `fr`: "Revoir la visite guidée"
  - [ ] `en`: "Replay the tour"

### Tests

- [ ] **Task 4 : Unit tests startTourFromSettings** (AC2, AC5)
  - [ ] Dans `TourService.test.ts` (story 25.1)
  - [ ] Vérifier que `startTourFromSettings()` déclenche le tour sans modifier la persistance
  - [ ] Vérifier que `tourCompleted` reste inchangé après re-déclenchement settings

- [ ] **Task 5 : BDD/Gherkin** (AC1, AC2, AC4, AC5)
  - [ ] `tests/acceptance/features/story-25-3.feature`
  - [ ] `tests/acceptance/story-25-3.test.ts`
  - [ ] Scenarios : option visible dans settings, re-déclenchement depuis settings, navigation vers CaptureScreen, pas d'écrasement historique

## Dev Notes

### Localisation du SettingsScreen

```
src/screens/settings/SettingsScreen.tsx  (ou SettingsContent.tsx selon le pattern Wrapper/Content)
src/navigation/SettingsStackNavigator.tsx
```

### Pattern navigation inter-tabs depuis Settings

```typescript
// Dans SettingsScreen — navigation vers Captures tab
const handleReplayTour = () => {
  const tourService = container.resolve<ITourService>(TOKENS.ITourService);
  tourService.startTourFromSettings();
  // Naviguer vers l'onglet Captures (root navigator)
  navigation.getParent()?.navigate('Captures');
};
```

⚠️ `navigation.getParent()` remonte au BottomTabNavigator. Vérifier le nom exact de l'onglet dans `registry.ts`.

### Logique AC5 — pas de synchro backend

```typescript
startTourFromSettings(): void {
  // Charge les steps depuis le registre et démarre
  // NOTE: force=true signifie qu'on NE vérifie PAS la persistance
  // NOTE: on ne modifie PAS tourCompleted (déjà true ou non)
  this.forceFromSettings = true;
  this.startTour(this.buildSteps());
}
```

### i18n — fichiers de traduction

Chercher les fichiers i18n dans : `src/i18n/` ou `src/locales/`
Pattern probable : `src/i18n/fr.json` et `src/i18n/en.json`

### Section Settings appropriée

Chercher dans le SettingsScreen existant la section "Aide", "Support" ou "À propos".
Si aucune section appropriée : créer une section "Aide" avec cette entrée.

### Project Structure Notes

- Modification mineure du `TourService` (ajout d'une méthode publique)
- Modification de `SettingsScreen` (ajout d'une ligne)
- Ajout de clés i18n dans les fichiers de traduction existants
- Tests dans `tests/acceptance/`

### References

- [Source: story 25.1] TourService, ITourService interface, TOKENS.ITourService
- [Source: story 25.2] TourPreferenceRepository, shouldShowTour(forceFromSettings)
- [Source: src/navigation/SettingsStackNavigator.tsx] Navigation settings
- [Source: src/screens/registry.ts] Noms des tabs (pour navigation inter-tab)
- [Source: architecture.md — i18n] Pattern i18next

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

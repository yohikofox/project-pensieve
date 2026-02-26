# Story 25.1: Tour Démo — Overlay Spotlight & Navigation Multi-Écrans

Status: ready-for-dev

## Story

En tant qu'**utilisateur découvrant l'application pour la première fois**,
je veux **être guidé par un tour interactif overlay** qui met en surbrillance les éléments clés de l'écran Capture et du détail d'une capture,
afin de **comprendre les fonctionnalités principales sans lire de documentation**.

## Acceptance Criteria

### AC1 : Déclenchement automatique au premier lancement
**Given** l'application vient d'être installée et l'utilisateur est authentifié
**When** l'écran principal (`CaptureScreen`) est rendu pour la première fois
**Then** le tour démarre automatiquement sans action de l'utilisateur
**And** un overlay semi-transparent couvre l'interface

### AC2 : Spotlight sur l'élément actif
**Given** le tour est actif
**When** une étape est affichée
**Then** l'élément cible est mis en surbrillance via un trou/spotlight dans l'overlay
**And** les autres éléments sont assombris (opacity < 0.6)
**And** le spotlight suit la forme (rectangulaire ou circulaire) de l'élément cible

### AC3 : Tooltip avec contenu contextuel
**Given** une étape du tour est active
**When** l'overlay est rendu
**Then** un tooltip s'affiche avec : titre, description, numéro d'étape (ex: "2 / 6")
**And** les boutons "Suivant", "Précédent" (si step > 1), "Passer"

### AC4 : Navigation entre étapes intra-écran
**Given** plusieurs étapes sont définies sur le même écran
**When** l'utilisateur appuie sur "Suivant"
**Then** l'overlay se déplace vers le prochain élément
**And** la transition est animée (durée < 300ms)

### AC5 : Navigation inter-écrans (Capture → CaptureDetail)
**Given** l'étape 3 est sur `CaptureScreen` et l'étape 4 est sur `CaptureDetailScreen`
**When** l'utilisateur appuie sur "Suivant" à l'étape 3
**Then** l'application navigue automatiquement vers `CaptureDetailScreen` avec une capture de démonstration
**And** l'étape 4 s'active automatiquement sur le nouvel écran

### AC6 : Abandon du tour ("Passer")
**Given** le tour est actif
**When** l'utilisateur appuie sur "Passer"
**Then** le tour se ferme immédiatement
**And** un événement `TourSkipped` est émis sur l'EventBus
**And** l'interface reprend son état normal sans perturbation

### AC7 : Fin du tour (dernière étape)
**Given** l'utilisateur est à la dernière étape
**When** il appuie sur "Terminer"
**Then** le tour se ferme avec une animation de fin
**And** un événement `TourCompleted` est émis sur l'EventBus

### AC8 : Enregistrement des steps par les écrans
**Given** `CaptureScreen` et `CaptureDetailScreen` sont montés
**When** ils s'enregistrent via `useTourStep()`
**Then** le `TourProvider` connaît les coordonnées de chaque élément référencé
**And** les coordonnées se recalculent si l'orientation change

### AC9 : Pas de tour si déjà vu (guard)
**Given** l'état local indique que le tour a déjà été complété ou passé
**When** l'application démarre
**Then** le tour ne se déclenche PAS

## Étapes du Tour (Séquence Fixe)

| Step | Écran | Élément | Titre | Description |
|------|-------|---------|-------|-------------|
| 1 | CaptureScreen | Bouton d'enregistrement (grand bouton central) | Capturez en 1 tap | Appuyez ici pour démarrer un enregistrement audio instantané |
| 2 | CaptureScreen | Feed des captures (liste) | Vos captures | Toutes vos pensées enregistrées apparaissent ici, dans l'ordre |
| 3 | CaptureScreen | Badge d'état de transcription | Traitement automatique | Vos enregistrements sont transcrits automatiquement en arrière-plan |
| 4 | CaptureDetailScreen | Zone de transcription | Le texte de votre pensée | La transcription complète de votre enregistrement audio |
| 5 | CaptureDetailScreen | Section insights / idées | Ce que l'IA a compris | L'IA extrait les idées clés et résume vos pensées |
| 6 | CaptureDetailScreen | Section todos | Actions détectées | Les actions à faire sont extraites automatiquement |

## Tasks / Subtasks

### Mobile — Domaine & Tokens DI

- [ ] **Task 1 : Interfaces domaine** (AC1-AC9)
  - [ ] Créer `src/contexts/shared/tour/domain/ITourService.ts` avec interface : `startTour()`, `nextStep()`, `prevStep()`, `skipTour()`, `completeTour()`, `getCurrentStep()`, `isActive()`
  - [ ] Créer `src/contexts/shared/tour/domain/TourStep.ts` (type: `id`, `screen`, `order`, `title`, `description`, `ref`, `shape: 'rect' | 'circle'`)
  - [ ] Créer `src/contexts/shared/tour/domain/TourEvents.ts` : `TourCompleted`, `TourSkipped` (étendre `DomainEvent`)
  - [ ] Ajouter `ITourService` dans `src/infrastructure/di/tokens.ts`

### Mobile — Service TourService

- [ ] **Task 2 : TourService** (AC1-AC4, AC6, AC7, AC9)
  - [ ] Créer `src/contexts/shared/tour/services/TourService.ts`
  - [ ] Implémenter `startTour(steps: TourStep[])` — trie par `order`, active step 0
  - [ ] Implémenter `nextStep()` — avance, émet `TourCompleted` si dernière step
  - [ ] Implémenter `prevStep()` — recule si `currentIndex > 0`
  - [ ] Implémenter `skipTour()` — émet `TourSkipped`
  - [ ] Implémenter `isActive(): boolean`
  - [ ] Implémenter `shouldShowTour(localCompleted: boolean, forceFromSettings: boolean, currentVersion: string): boolean`
  - [ ] **Lifecycle DI** : `registerSingleton` (état partagé entre écrans — justification requise dans commentaire)

### Mobile — DI Container

- [ ] **Task 3 : Enregistrement DI** (AC1)
  - [ ] Ajouter `TOKENS.ITourService` dans `tokens.ts`
  - [ ] Enregistrer `TourService` comme singleton dans `container.ts` avec commentaire de justification ADR-021

### Mobile — Zustand Store

- [ ] **Task 4 : useTourStore** (AC2, AC3, AC4, AC8)
  - [ ] Créer `src/stores/tourStore.ts` avec Zustand
  - [ ] State : `isActive`, `currentStep: TourStep | null`, `totalSteps`, `stepIndex`
  - [ ] Actions : `setStep()`, `setActive()`, `reset()`
  - [ ] Alimenté par les événements EventBus (`TourCompleted`, `TourSkipped`)

### Mobile — Hook useTourStep

- [ ] **Task 5 : useTourStep hook** (AC8)
  - [ ] Créer `src/hooks/useTourStep.ts`
  - [ ] Paramètres : `{ id, screen, order, title, description, shape? }`
  - [ ] Retourne un `ref` React à attacher à l'élément UI
  - [ ] Enregistre les coordonnées via `UIManager.measure()` après mount
  - [ ] Re-mesure au `onLayout` (AC8 orientation change)

### Mobile — Composants UI

- [ ] **Task 6 : SpotlightOverlay** (AC2)
  - [ ] Créer `src/components/tour/SpotlightOverlay.tsx`
  - [ ] Fond noir semi-transparent (Modal `transparent`, z-index maximal)
  - [ ] Découpe de la zone spotlight via `<MaskedView>` ou clip path Reanimated (Reanimated 4.1.1)
  - [ ] Transition animée position/taille (Reanimated `withSpring` — durée ≤ 300ms, AC4)
  - [ ] Props : `spotlightRect: { x, y, width, height }`, `shape: 'rect' | 'circle'`

- [ ] **Task 7 : TourTooltip** (AC3)
  - [ ] Créer `src/components/tour/TourTooltip.tsx`
  - [ ] Props : `title`, `description`, `currentIndex`, `totalSteps`, `onNext`, `onPrev`, `onSkip`
  - [ ] Bouton "Précédent" absent si `currentIndex === 0`
  - [ ] Bouton "Suivant" devient "Terminer" si `currentIndex === totalSteps - 1`
  - [ ] Position adaptive (au-dessus ou en-dessous selon l'espace disponible)
  - [ ] Stylé avec NativeWind (tailwind classes)

- [ ] **Task 8 : TourProvider** (AC1, AC5, AC9)
  - [ ] Créer `src/components/tour/TourProvider.tsx`
  - [ ] Vit **au-dessus du Navigator** dans `MainApp.tsx`
  - [ ] Subscribe à `useTourStore` pour réactivité
  - [ ] Gère la navigation inter-écrans via `useNavigation()` (AC5)
  - [ ] Injecte `TourService` via `container.resolve(TOKENS.ITourService)` (lazy, pas module level)
  - [ ] Rend `<SpotlightOverlay>` + `<TourTooltip>` seulement si `isActive`

### Mobile — Intégration Écrans

- [ ] **Task 9 : Intégration CaptureScreen** (AC1-AC4, AC8)
  - [ ] Ajouter `useTourStep` pour les 3 éléments (bouton enregistrement, feed, badge état)
  - [ ] Passer les refs aux éléments UI existants (ne pas modifier la logique)

- [ ] **Task 10 : Intégration CaptureDetailScreen** (AC5, AC8)
  - [ ] Ajouter `useTourStep` pour les 3 éléments (transcription, insights, todos)
  - [ ] La navigation inter-écrans utilise une capture de demo (ou la première capture disponible)

- [ ] **Task 11 : Intégration MainApp.tsx** (AC1, AC9)
  - [ ] Envelopper le NavigationContainer avec `<TourProvider>`
  - [ ] `TourProvider` vérifie au démarrage si le tour doit se déclencher

### Tests

- [ ] **Task 12 : Unit tests TourService** (AC1-AC9)
  - [ ] `src/contexts/shared/tour/services/TourService.test.ts`
  - [ ] Couvrir : startTour, nextStep (dernière → TourCompleted), prevStep, skipTour, shouldShowTour
  - [ ] Vérifier émission EventBus `TourCompleted` et `TourSkipped`

- [ ] **Task 13 : BDD/Gherkin** (AC1, AC5, AC6, AC7, AC9)
  - [ ] Créer `tests/acceptance/features/story-25-1.feature`
  - [ ] Créer `tests/acceptance/story-25-1.test.ts`
  - [ ] Scenarios : déclenchement auto, navigation inter-écrans, skip, complétion, guard si déjà vu

## Dev Notes

### Architecture TourProvider (position dans l'arbre React)

```
<SafeAreaProvider>
  <TourProvider>          ← NOUVEAU — vit ici pour survivre aux navigations
    <NavigationContainer>
      <MainNavigator>
        <CaptureScreen>   ← useTourStep() steps 1-3
        <CaptureDetailScreen> ← useTourStep() steps 4-6
      </MainNavigator>
    </NavigationContainer>
  </TourProvider>
</SafeAreaProvider>
```

### Pattern useTourStep (implémentation)

```typescript
// Hook utilisé dans chaque écran participant
const recordButtonRef = useTourStep({
  id: 'capture-record-button',
  screen: 'Capture',
  order: 1,
  title: 'Capturez en 1 tap',
  description: 'Appuyez ici pour démarrer un enregistrement audio.',
  shape: 'circle',
});
// Attaché à l'élément : <RecordButton ref={recordButtonRef} ... />
```

### Mesure des éléments UI

```typescript
// Dans useTourStep — après mount via useLayoutEffect
UIManager.measure(findNodeHandle(ref.current), (x, y, width, height, pageX, pageY) => {
  // pageX/pageY sont les coordonnées absolues dans la fenêtre
  tourService.registerStepPosition(id, { x: pageX, y: pageY, width, height });
});
```

### SpotlightOverlay — approche technique

Utiliser `react-native-reanimated` (version 4.1.1 déjà installée) pour les animations.
L'effet spotlight est obtenu via un `<MaskedView>` (expo-image est disponible, mais MaskedView de `@react-native-masked-view/masked-view` doit être vérifié) **ou** via une approche SVG/Canvas.

**Alternative plus simple** (recommandée pour cette implémentation) :
- Un `<Modal transparent>` avec 4 rectangles noirs positionnés autour de la zone spotlight (top, bottom, left, right)
- Animés avec `Animated.Value` / Reanimated `useAnimatedStyle`
- Plus compatible et plus simple à déboguer

### Navigation inter-écrans (AC5)

```typescript
// Dans TourProvider — gestion transition étape 3 → 4
const navigation = useNavigation<NativeStackNavigationProp<...>>();

const handleNextStep = () => {
  const nextStep = tourService.nextStep();
  if (nextStep.screen !== currentScreen) {
    // Naviguer vers le nouvel écran avant d'afficher le step
    navigation.navigate(nextStep.screen, { tourMode: true, captureId: demoCapture?.id });
    // Le nouvel écran monte ses steps, TourProvider réagit
  }
};
```

### TourService — Lifecycle singleton (ADR-021)

```typescript
// container.ts — justification singleton obligatoire (ADR-021)
// SINGLETON: TourService maintient l'état du tour partagé entre tous les écrans
// (stepRegistry, currentIndex, isActive). Doit survivre aux navigations.
container.registerSingleton<ITourService>(TOKENS.ITourService, TourService);
```

### Constante version du tour

```typescript
// src/contexts/shared/tour/tourConfig.ts
export const CURRENT_TOUR_VERSION = '1.0';
export const TOUR_SCREENS = ['Capture', 'CaptureDetail'] as const;
```

### Project Structure Notes

- Nouveaux fichiers dans `src/contexts/shared/tour/` (pattern contexte DDD)
- Composants dans `src/components/tour/`
- Hook dans `src/hooks/useTourStep.ts`
- Store dans `src/stores/tourStore.ts`
- Tests BDD dans `tests/acceptance/`

### References

- [Source: architecture.md — DI TSyringe ADR-021] Transient first, singletons justifiés
- [Source: architecture.md — ADR-025] fetch natif uniquement (pas de lib externe HTTP)
- [Source: pensieve/mobile/package.json] react-native-reanimated@4.1.1, @react-navigation/native@^7
- [Source: container.ts — FirstLaunchInitializer] Pattern singleton service "à usage unique"
- [Source: architecture.md — EventBus] RxJS Subject pour communication inter-contextes
- [Source: pensieve/CLAUDE.md] babel-jest pour unit tests, ts-jest pour BDD

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

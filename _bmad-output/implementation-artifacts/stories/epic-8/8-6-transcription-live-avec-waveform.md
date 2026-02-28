# Story 8.6 : Transcription Live avec Waveform

Status: ready-for-dev

<!-- Validation optionnelle : run validate-create-story avant dev-story -->

## Story

En tant qu'**utilisateur souhaitant capturer une pensée rapidement sans étape post-transcription**,
je veux **voir ma parole transcrite en temps réel avec une visualisation audio pendant que je parle**,
afin de **vérifier que le micro m'écoute, suivre le texte produit instantanément et sauvegarder la transcription finale d'un geste**.

**GitHub Issue :** [#11 - feat: Transcription en temps réel avec visualisation audio et affichage du texte live](https://github.com/yohikofox/pensieve/issues/11)

## Contexte

### Ce qui est déjà implémenté

#### NativeTranscriptionEngine — `startRealTime()` complet

`NativeTranscriptionEngine.ts` (463 lignes) possède **déjà** une implémentation fonctionnelle de la transcription temps réel via `expo-speech-recognition` :

```typescript
async startRealTime(
  config: TranscriptionEngineConfig,
  onPartialResult: (result: TranscriptionEngineResult) => void,
  onFinalResult: (result: TranscriptionEngineResult) => void,
): Promise<void>
```

- Événements : `result` (partiel + final), `error`, `end`
- Accumulation : `this.accumulatedText` (segments `isFinal` seulement)
- Accesseur : `getAccumulatedText(): string`
- Arrêt : `stopRealTime()` → `ExpoSpeechRecognitionModule.stop()`
- Abandon : `cancel()` → `ExpoSpeechRecognitionModule.abort()`

#### expo-speech-recognition — volumechange event

La bibliothèque expose un événement `volumechange` avec `value: number` (float -2 à 10) :

```typescript
// Déclaré dans ExpoSpeechRecognitionModule.types.ts:170
volumechange: {
  value: number; // float -2 à 10 — considérer <0 comme inaudible
};
```

Doit être activé via `volumeChangeEventOptions: { enabled: true, interval: 100 }` dans les options de `start()`.

#### CaptureScreen — outils de capture

`CAPTURE_TOOLS_ALWAYS` (lignes 69-84) contient `voice` et `text`. Un nouveau bouton `live` sera ajouté.

#### TextCaptureService — sauvegarde de texte

Déjà utilisé pour les captures texte rapide (Story 2.2). La transcription finale sera sauvegardée en tant que capture de type `text`.

### Ce qui manque (Story 8.6 — à implémenter)

- Aucune entrée "Live Transcription" dans CaptureScreen
- `NativeTranscriptionEngine.startRealTime()` n'active pas les événements de volume
- Pas d'overlay `LiveTranscriptionOverlay.tsx`
- Pas de hook `useLiveTranscription.ts`

### Scénario utilisateur actuel (manquant)

```
Utilisateur :
  1. Ouvre CaptureScreen
  2. Voit deux boutons : Voice (enregistrement + transcription différée) et Text
  3. 🔴 Aucun mode "Live" où voir le texte apparaître en temps réel
  4. Pour avoir la transcription rapide, doit passer par Voice → attendre l'enregistrement → attendre la transcription
```

### Décision architecturale

1. **Nouveau bouton "Live"** dans `CAPTURE_TOOLS_ALWAYS` → déclenche `useLiveTranscription`
2. **Pas d'audio sauvegardé** — `expo-speech-recognition` gère le micro directement, sans fichier audio. Résultat = capture de type `text`.
3. **Événements volume** — activer `volumeChangeEventOptions` dans `NativeTranscriptionEngine.startRealTime()` via nouveau flag optionnel dans `TranscriptionEngineConfig`.
4. **Garde moteur** — si l'utilisateur a sélectionné Whisper (pas de real-time), afficher un toast d'information et proposer de basculer vers Native.
5. **`LiveTranscriptionOverlay`** — plein écran (comme `RecordingOverlay`), dans une `Modal` React Native.

## Acceptance Criteria

### AC1 : Bouton "Live" dans CaptureScreen

**Given** l'utilisateur ouvre CaptureScreen
**When** l'écran s'affiche
**Then** un bouton "Live" est visible dans la grille de capture (aux côtés de Voice et Text)
**And** l'icône est `mic-off` ou `zap` (symbolique — pas littérale)
**And** le bouton est désactivé si une capture est en cours (`state !== 'idle'`)

**Given** le moteur de transcription sélectionné est `whisper` (pas de real-time)
**When** l'utilisateur appuie sur le bouton "Live"
**Then** un toast informatif s'affiche : "La transcription live requiert le moteur natif. Activez-le dans Paramètres > Transcription."
**And** l'overlay de transcription live N'est PAS ouvert

**Given** le moteur de transcription est `native`
**When** l'utilisateur appuie sur le bouton "Live"
**Then** l'overlay `LiveTranscriptionOverlay` s'ouvre immédiatement

### AC2 : Visualisation audio en temps réel (VU meter)

**Given** l'overlay de transcription live est ouvert
**When** le moteur démarre (`startRealTime()`)
**Then** la reconnaissance vocale démarre automatiquement sans autre interaction utilisateur
**And** un VU meter (barres animées) est visible
**And** les barres réagissent aux événements `volumechange` de `expo-speech-recognition`
**And** une valeur `value >= 0` fait grandir les barres proportionnellement (échelle 0-10)
**And** une valeur `value < 0` (inaudible) maintient les barres à hauteur minimale
**And** l'animation est fluide (interpolation ou Animated.Value)

### AC3 : Affichage texte progressif avec auto-scroll

**Given** le moteur détecte de la parole
**When** un résultat partiel (`isPartial: true`) arrive
**Then** le texte partiel s'affiche dans une zone de texte dédiée
**And** la zone de texte auto-scroll vers le bas pour montrer le nouveau texte

**When** un résultat final (`isPartial: false`) arrive
**Then** le texte est ajouté au texte confirmé
**And** la zone partielle est effacée (le moteur commencera le segment suivant)

### AC4 : Différenciation visuelle partiel / confirmé

**Given** du texte est en cours de transcription
**Then** le texte **confirmé** (segments `isFinal`) s'affiche en couleur primaire (`text-text-primary`)
**And** le texte **partiel** (segment en cours, `isPartial`) s'affiche en couleur secondaire avec style italique (`text-text-secondary italic`)
**And** les deux sont dans la même zone de scroll, le partiel apparaissant après le confirmé

### AC5 : Contrôles Stop (sauvegarde) et Cancel (abandon)

**Given** l'overlay de transcription live est ouvert
**When** l'utilisateur appuie sur "Arrêter"
**Then** `NativeTranscriptionEngine.stopRealTime()` est appelé
**And** le texte accumulé (`getAccumulatedText()`) est récupéré
**And** si du texte est présent → la capture est sauvegardée (AC6)
**And** si aucun texte n'a été capturé → toast "Aucun texte capturé" et fermeture sans sauvegarde
**And** l'overlay se ferme

**When** l'utilisateur appuie sur "Annuler"
**Then** `NativeTranscriptionEngine.cancel()` est appelé
**And** aucune capture n'est sauvegardée
**And** l'overlay se ferme sans toast

**Given** l'overlay est en cours de fermeture (état `isSaving`)
**Then** les boutons Stop et Cancel sont désactivés pour éviter les double-appels

### AC6 : Sauvegarde de la transcription finale comme capture texte

**Given** l'utilisateur appuie sur "Arrêter"
**And** du texte accumulé est disponible
**When** la sauvegarde se déclenche
**Then** `TextCaptureService.createTextCapture(text)` est appelé avec le texte accumulé complet
**And** la capture est créée avec `type: 'text'`, `state: 'ready'`, `rawContent: textAccumulé`
**And** un toast de confirmation s'affiche : "Transcription sauvegardée"
**And** l'EventBus émet l'événement de nouvelle capture (pour rafraîchissement du feed)

**Note :** Pas d'audio sauvegardé — la transcription live produit uniquement du texte.

### AC7 : Tests unitaires et BDD

**Given** l'implémentation est complète
**When** les tests sont exécutés
**Then** les scénarios BDD dans `story-8-6-transcription-live-avec-waveform.feature` couvrent AC1, AC2, AC3, AC5, AC6
**And** le hook `useLiveTranscription` a des tests unitaires couvrant :
  - Démarrage avec moteur natif → `startRealTime()` appelé
  - Démarrage avec moteur whisper → toast affiché, pas de `startRealTime()`
  - Stop avec texte → `TextCaptureService.createTextCapture()` appelé
  - Stop sans texte → toast "Aucun texte", pas de sauvegarde
  - Cancel → `cancel()` appelé, pas de sauvegarde
**And** `npm run test:unit` passe sans régression
**And** `npm run test:acceptance` passe sans régression

## Tasks / Subtasks

### Task 1 : Activer les événements volume dans `NativeTranscriptionEngine` (AC2)

- [ ] Subtask 1.1 : Ouvrir `mobile/src/contexts/Normalization/services/ITranscriptionEngine.ts`
- [ ] Subtask 1.2 : Ajouter le champ optionnel `enableVolumeEvents?: boolean` à `TranscriptionEngineConfig` :
  ```typescript
  export interface TranscriptionEngineConfig {
    language: string;
    vocabulary?: string[];
    enableVolumeEvents?: boolean; // Story 8.6: Active volumechange events
  }
  ```
- [ ] Subtask 1.3 : Ouvrir `mobile/src/contexts/Normalization/services/NativeTranscriptionEngine.ts`
- [ ] Subtask 1.4 : Ajouter la propriété privée `private volumeSubscription?: { remove: () => void }` dans la classe
- [ ] Subtask 1.5 : Ajouter le callback `private volumeCallback?: (value: number) => void`
- [ ] Subtask 1.6 : Modifier `startRealTime()` pour accepter un 4e paramètre optionnel :
  ```typescript
  async startRealTime(
    config: TranscriptionEngineConfig,
    onPartialResult: (result: TranscriptionEngineResult) => void,
    onFinalResult: (result: TranscriptionEngineResult) => void,
    onVolumeChange?: (value: number) => void, // Story 8.6
  ): Promise<void>
  ```
- [ ] Subtask 1.7 : Dans `startRealTime()`, stocker le callback et souscrire à `volumechange` si fourni :
  ```typescript
  if (onVolumeChange) {
    this.volumeCallback = onVolumeChange;
    this.volumeSubscription = ExpoSpeechRecognitionModule.addListener(
      'volumechange',
      (event) => {
        if (this.volumeCallback) {
          this.volumeCallback(event.value);
        }
      },
    );
  }
  ```
- [ ] Subtask 1.8 : Activer `volumeChangeEventOptions` dans les options `start()` quand `config.enableVolumeEvents` :
  ```typescript
  await ExpoSpeechRecognitionModule.start({
    lang: locale,
    interimResults: true,
    continuous: true,
    contextualStrings: config.vocabulary,
    requiresOnDeviceRecognition: false,
    addsPunctuation: true,
    ...(config.enableVolumeEvents ? {
      volumeChangeEventOptions: { enabled: true, interval: 100 },
    } : {}),
  });
  ```
- [ ] Subtask 1.9 : Mettre à jour `cleanup()` pour supprimer `this.volumeSubscription?.remove()` et réinitialiser `volumeCallback`
- [ ] Subtask 1.10 : Mettre à jour `ITranscriptionEngine.startRealTime?()` dans l'interface avec le 4e paramètre optionnel

### Task 2 : Créer le hook `useLiveTranscription.ts` (AC1, AC2, AC3, AC4, AC5, AC6)

- [ ] Subtask 2.1 : Créer `mobile/src/hooks/useLiveTranscription.ts`
- [ ] Subtask 2.2 : Implémenter le hook :
  ```typescript
  interface LiveTranscriptionState {
    isListening: boolean;
    isSaving: boolean;
    confirmedText: string;
    partialText: string;
    volumeLevel: number; // -2 à 10
  }

  interface UseLiveTranscriptionReturn {
    state: LiveTranscriptionState;
    startListening: () => Promise<void>;
    stopAndSave: () => Promise<void>;
    cancel: () => Promise<void>;
  }

  export function useLiveTranscription(): UseLiveTranscriptionReturn
  ```
- [ ] Subtask 2.3 : Dans `startListening()` :
  - Résoudre `TranscriptionEngineService` via container (lazy)
  - Récupérer l'engine préféré
  - Si engine != 'native' → `toast.info(t('liveTranscription.nativeRequired'))` et `return`
  - Résoudre `NativeTranscriptionEngine` via container
  - Appeler `engine.startRealTime(config, onPartial, onFinal, onVolumeChange)`
  - `onPartial(result)` → `setState({ partialText: result.text })`
  - `onFinal(result)` → `setState(s => ({ confirmedText: s.confirmedText + (s.confirmedText ? ' ' : '') + result.text, partialText: '' }))`
  - `onVolumeChange(value)` → `setState({ volumeLevel: value })`
  - Gestion d'erreur : `toast.error(t('liveTranscription.startError'))`
- [ ] Subtask 2.4 : Dans `stopAndSave()` :
  - Passer `isSaving: true`
  - Appeler `engine.stopRealTime()`
  - Récupérer `finalText = engine.getAccumulatedText()` + `state.partialText` (au cas où un partiel est en cours)
  - Si `finalText.trim() === ''` → `toast.info(t('liveTranscription.noText'))`, fermer
  - Sinon → `TextCaptureService.createTextCapture(finalText.trim())`
  - `toast.success(t('liveTranscription.saved'))`
  - Passer `isSaving: false`, fermer l'overlay
- [ ] Subtask 2.5 : Dans `cancel()` :
  - Appeler `engine.cancel()`
  - Pas de toast, fermeture directe
- [ ] Subtask 2.6 : Cleanup `useEffect` : si composant démonté pendant écoute → appeler `cancel()`
- [ ] Subtask 2.7 : Config de transcription :
  ```typescript
  const config: TranscriptionEngineConfig = {
    language: settingsStore.transcription.language ?? 'fr-FR',
    enableVolumeEvents: true,
  };
  ```

### Task 3 : Créer `LiveTranscriptionOverlay.tsx` (AC2, AC3, AC4, AC5)

- [ ] Subtask 3.1 : Créer `mobile/src/components/capture/LiveTranscriptionOverlay.tsx`
- [ ] Subtask 3.2 : Interface :
  ```typescript
  interface LiveTranscriptionOverlayProps {
    confirmedText: string;
    partialText: string;
    volumeLevel: number;     // -2 à 10
    isSaving: boolean;
    onStop: () => void;
    onCancel: () => void;
  }
  ```
- [ ] Subtask 3.3 : VU meter — composant `LiveVUMeter` (inline dans le fichier) :
  - 12 barres animées via `Animated.Value`
  - Hauteur de chaque barre : interpolée depuis `volumeLevel` (0 = hauteur min 4px, 10 = hauteur max 48px)
  - Barres en couleur primaire (`colors.primary[500]`)
  - Utiliser un `useEffect` qui anime toutes les barres avec un léger décalage (stagger) pour un effet visuel dynamique
  - Si `volumeLevel < 0` → barres à hauteur minimale
- [ ] Subtask 3.4 : Zone de texte live :
  ```tsx
  <ScrollView ref={scrollRef} onContentSizeChange={() => scrollRef.current?.scrollToEnd()}>
    <Text style={styles.confirmedText}>{confirmedText}</Text>
    {partialText ? (
      <Text style={styles.partialText}>{partialText}</Text>
    ) : null}
  </ScrollView>
  ```
- [ ] Subtask 3.5 : Styles :
  - `confirmedText` : `color: themeColors.textPrimary`, `fontSize: 16`, `lineHeight: 24`
  - `partialText` : `color: themeColors.textSecondary`, `fontStyle: 'italic'`, `fontSize: 16`
  - Zone de texte : hauteur fixe (ex: 180px) avec scroll
- [ ] Subtask 3.6 : Contrôles :
  ```tsx
  <TouchableOpacity onPress={onStop} disabled={isSaving}>
    <Text>{isSaving ? t('liveTranscription.saving') : t('liveTranscription.stop')}</Text>
  </TouchableOpacity>
  <TouchableOpacity onPress={onCancel} disabled={isSaving}>
    <Text>{t('liveTranscription.cancel')}</Text>
  </TouchableOpacity>
  ```
- [ ] Subtask 3.7 : Placeholder si `confirmedText === ''` et `partialText === ''` :
  ```tsx
  <Text style={styles.placeholder}>{t('liveTranscription.placeholder', 'Parlez maintenant...')}</Text>
  ```
- [ ] Subtask 3.8 : Overlay plein écran — `backgroundColor: 'rgba(0,0,0,0.9)'` comme `RecordingOverlay`

### Task 4 : Intégrer dans `CaptureScreen.tsx` (AC1)

- [ ] Subtask 4.1 : Ouvrir `mobile/src/screens/capture/CaptureScreen.tsx`
- [ ] Subtask 4.2 : Ajouter le bouton "Live" dans `CAPTURE_TOOLS_ALWAYS` :
  ```typescript
  {
    id: "live",
    labelKey: "capture.tools.live",
    iconName: "zap", // Symbole instantané / temps réel
    color: colors.warning[500], // Couleur distincte de Voice (bleu) et Text (secondaire)
    available: true,
  },
  ```
- [ ] Subtask 4.3 : Ajouter les états et la ref du hook dans `CaptureScreenContent` :
  ```typescript
  const [showLiveTranscription, setShowLiveTranscription] = useState(false);
  const { startListening, stopAndSave, cancel: cancelLive, state: liveState } = useLiveTranscription({
    onClose: () => setShowLiveTranscription(false),
  });
  ```
- [ ] Subtask 4.4 : Mettre à jour `handleToolPress` pour gérer `'live'` :
  ```typescript
  case 'live':
    setShowLiveTranscription(true);
    // startListening sera appelé dans l'overlay via useEffect
    break;
  ```
- [ ] Subtask 4.5 : Ajouter la Modal pour `LiveTranscriptionOverlay` (après `RecordingOverlay`) :
  ```tsx
  <Modal
    visible={showLiveTranscription}
    animationType="fade"
    transparent={true}
    onRequestClose={cancelLive}
  >
    <LiveTranscriptionOverlay
      confirmedText={liveState.confirmedText}
      partialText={liveState.partialText}
      volumeLevel={liveState.volumeLevel}
      isSaving={liveState.isSaving}
      onStop={stopAndSave}
      onCancel={cancelLive}
    />
  </Modal>
  ```
- [ ] Subtask 4.6 : Dans `useLiveTranscription`, ajouter `onClose: () => void` dans les options pour fermer le modal après stop/cancel

### Task 5 : Clés de traduction i18n

- [ ] Subtask 5.1 : Ajouter les clés manquantes dans les fichiers de traduction (`fr.json`, `en.json`) :
  ```json
  "capture": {
    "tools": {
      "live": "Live"
    }
  },
  "liveTranscription": {
    "nativeRequired": "La transcription live requiert le moteur natif. Activez-le dans Paramètres > Transcription.",
    "startError": "Impossible de démarrer la transcription live.",
    "noText": "Aucun texte n'a été capturé.",
    "saved": "Transcription sauvegardée.",
    "placeholder": "Parlez maintenant...",
    "stop": "Arrêter",
    "cancel": "Annuler",
    "saving": "Sauvegarde...",
    "title": "Transcription en direct"
  }
  ```
- [ ] Subtask 5.2 : Vérifier que le chemin i18n est cohérent avec `i18n/fr.json` existant

### Task 6 : Tests BDD (AC7)

- [ ] Subtask 6.1 : Créer `mobile/tests/acceptance/features/story-8-6-transcription-live-avec-waveform.feature`
- [ ] Subtask 6.2 : Écrire les scénarios Gherkin (voir section "Scénarios BDD" ci-dessous)
- [ ] Subtask 6.3 : Créer `mobile/tests/acceptance/story-8-6-transcription-live-avec-waveform.test.ts` avec step definitions
- [ ] Subtask 6.4 : Mocker `NativeTranscriptionEngine`, `TranscriptionEngineService`, `TextCaptureService` via TSyringe container

### Task 7 : Tests unitaires — `useLiveTranscription` (AC7)

- [ ] Subtask 7.1 : Créer `mobile/src/hooks/__tests__/useLiveTranscription.test.ts`
- [ ] Subtask 7.2 : Cas testés :
  ```
  Cas 1 — moteur native → startRealTime appelé, isListening=true
  Cas 2 — moteur whisper → toast info, startRealTime NON appelé
  Cas 3 — stop avec texte accumulé → TextCaptureService.createTextCapture appelé
  Cas 4 — stop sans texte → toast "Aucun texte", pas de createTextCapture
  Cas 5 — cancel → cancel() appelé, pas de createTextCapture
  Cas 6 — onFinal reçu → confirmedText mis à jour
  Cas 7 — onPartial reçu → partialText mis à jour, confirmedText inchangé
  Cas 8 — onVolumeChange reçu → volumeLevel mis à jour
  ```
- [ ] Subtask 7.3 : Pattern de mock identique au pattern établi dans `useCaptureDetailInit.test.ts`

### Task 8 : Validation finale (AC7)

- [ ] Subtask 8.1 : `npm run test:unit` dans `pensieve/mobile/` — zéro régression
- [ ] Subtask 8.2 : `npm run test:acceptance` dans `pensieve/mobile/` — zéro régression
- [ ] Subtask 8.3 : `npm run test:architecture` — conformité ADR maintenue
- [ ] Subtask 8.4 : Test manuel sur device : démarrer Live, parler, vérifier waveform + texte + sauvegarde
- [ ] Subtask 8.5 : Fermer l'issue GitHub #11 avec référence au commit

## Scénarios BDD (Feature File)

```gherkin
# language: fr
Fonctionnalité: Transcription Live avec Waveform
  En tant qu'utilisateur souhaitant capturer une pensée rapidement
  Je veux voir ma parole transcrite en temps réel avec une visualisation audio
  Afin de confirmer que le micro m'écoute et sauvegarder le texte instantanément

  Contexte:
    Étant donné que je suis un utilisateur authentifié
    Et que l'application est lancée sur CaptureScreen
    Et que le moteur de transcription est configuré sur "native"

  Scénario: Bouton Live visible dans CaptureScreen
    Quand j'observe la grille de capture
    Alors je vois le bouton "Live" en plus de "Voice" et "Text"
    Et le bouton est actif (state = 'idle')

  Scénario: Ouverture de l'overlay Live Transcription
    Quand j'appuie sur le bouton "Live"
    Alors l'overlay de transcription live s'ouvre
    Et le VU meter s'affiche
    Et la zone de texte affiche "Parlez maintenant..."
    Et le bouton "Arrêter" est visible
    Et le bouton "Annuler" est visible

  Scénario: Moteur Whisper — message informatif
    Étant donné que le moteur de transcription est configuré sur "whisper"
    Quand j'appuie sur le bouton "Live"
    Alors un toast informatif s'affiche : "La transcription live requiert le moteur natif"
    Et l'overlay de transcription live ne s'ouvre PAS

  Scénario: Affichage du texte partiel puis confirmé
    Étant donné que l'overlay de transcription live est ouvert
    Quand un résultat partiel "bonjour" arrive
    Alors je vois "bonjour" en style secondaire (partiel)
    Quand un résultat final "bonjour monde" arrive
    Alors je vois "bonjour monde" en style primaire (confirmé)
    Et la zone partielle est vide

  Scénario: Stop et sauvegarde de la transcription
    Étant donné que l'overlay est ouvert
    Et que le texte accumulé est "Penser à rappeler Marie demain"
    Quand j'appuie sur "Arrêter"
    Alors TextCaptureService.createTextCapture est appelé avec "Penser à rappeler Marie demain"
    Et un toast "Transcription sauvegardée" s'affiche
    Et l'overlay se ferme

  Scénario: Stop sans texte — aucune sauvegarde
    Étant donné que l'overlay est ouvert
    Et qu'aucun texte n'a été capturé
    Quand j'appuie sur "Arrêter"
    Alors TextCaptureService.createTextCapture N'est PAS appelé
    Et un toast "Aucun texte n'a été capturé" s'affiche
    Et l'overlay se ferme

  Scénario: Annuler la transcription live
    Étant donné que l'overlay est ouvert
    Et que du texte est en cours de transcription
    Quand j'appuie sur "Annuler"
    Alors TextCaptureService.createTextCapture N'est PAS appelé
    Et l'overlay se ferme sans toast
```

## Dev Notes

### 🔑 Fichiers à modifier

```
pensieve/mobile/src/contexts/Normalization/services/ITranscriptionEngine.ts
  ← Task 1: Ajouter enableVolumeEvents?: boolean dans TranscriptionEngineConfig
  ← Task 1: Ajouter onVolumeChange? dans signature startRealTime? de l'interface

pensieve/mobile/src/contexts/Normalization/services/NativeTranscriptionEngine.ts
  ← Task 1: Activer volumechange events, 4e paramètre onVolumeChange

pensieve/mobile/src/screens/capture/CaptureScreen.tsx
  ← Task 4: Ajouter bouton "live" dans CAPTURE_TOOLS_ALWAYS + Modal + intégration hook
```

### 📁 Fichiers à créer

```
pensieve/mobile/src/hooks/useLiveTranscription.ts
  ← Task 2: Hook principal (startListening, stopAndSave, cancel, state)

pensieve/mobile/src/components/capture/LiveTranscriptionOverlay.tsx
  ← Task 3: Overlay plein écran avec VU meter + texte live + contrôles

pensieve/mobile/src/hooks/__tests__/useLiveTranscription.test.ts
  ← Task 7: Tests unitaires du hook

pensieve/mobile/tests/acceptance/features/story-8-6-transcription-live-avec-waveform.feature
  ← Task 6: Scénarios Gherkin

pensieve/mobile/tests/acceptance/story-8-6-transcription-live-avec-waveform.test.ts
  ← Task 6: Step definitions BDD
```

### ⛔ Fichiers à NE PAS modifier

```
pensieve/mobile/src/components/audio/Waveform.tsx          ← Spécifique au playback (expo-audio + useAudioSampleListener)
pensieve/mobile/src/components/capture/RecordingOverlay.tsx ← Pattern audio classique, ne pas altérer
pensieve/mobile/src/contexts/capture/services/RecordingService.ts ← Pas d'audio dans le mode Live
pensieve/mobile/modules/expo-waveform-extractor/            ← Uniquement pour extraction de waveform depuis fichier audio
```

### 🏗️ Architecture — Volume Events

Le VU meter utilise les événements natifs de `expo-speech-recognition` :

```
expo-speech-recognition
  └─ volumechange event { value: number }  (-2 to 10)
       │
       ▼
NativeTranscriptionEngine.startRealTime(..., onVolumeChange)
  └─ this.volumeSubscription = addListener('volumechange', ...)
       │
       ▼
useLiveTranscription
  └─ setState({ volumeLevel: value })
       │
       ▼
LiveTranscriptionOverlay
  └─ LiveVUMeter
       └─ Animated bars (height interpolated from volumeLevel)
```

**Pourquoi pas `react-native-audio-api` ?** — Bibliothèque non installée dans le projet. `expo-speech-recognition` fournit déjà le niveau audio nativement. Pas de dépendance supplémentaire (YAGNI).

**Pourquoi pas `expo-audio` metering en parallèle ?** — Conflit d'accès au microphone entre `expo-audio` recording et `expo-speech-recognition`. Une seule source suffit.

### 🏗️ Architecture — Sauvegarde comme capture `text`

Le mode Live Transcription produit directement du texte (pas de fichier audio) :

```
Capture audio classique :
  audio file → RecordingService → transcription queue → normalizedText

Live Transcription (Story 8.6) :
  speech recognition → accumulatedText → TextCaptureService.createTextCapture(text)
                                          type: 'text', state: 'ready'
```

`TextCaptureService` déjà utilisé en Story 2.2 — pattern établi.

### 🏗️ Pattern de référence — Hook `useLiveTranscription`

Suivre le pattern de `useCaptureDetailInit.ts` pour la résolution DI (lazy, dans `useEffect`/`useCallback`) :

```typescript
// ✅ CORRECT — lazy resolution
const nativeEngine = useMemo(() => {
  return container.resolve(NativeTranscriptionEngine);
}, []);
```

### 📋 Conformité ADR

| ADR | Impact | Conformité |
|-----|--------|-----------|
| **ADR-021** (DI Lifecycle — Transient First) | `useLiveTranscription` résout les services lazily via `container.resolve()` dans `useMemo` | ✅ |
| **ADR-022** (AsyncStorage) | Aucune donnée persistée dans AsyncStorage | ✅ |
| **ADR-023** (Result Pattern) | `TextCaptureService.createTextCapture()` retourne `Result<T>` — gérer les cas error/success | ⚠️ Vérifier le type de retour et gérer l'erreur |
| **ADR-024** (Clean Code) | Constantes nommées, SRP respecté (hook = logique, overlay = UI) | ✅ |
| **ADR-018** (OP-SQLite / WatermelonDB deprecated) | Aucun impact — TextCaptureService utilise OP-SQLite internement | ✅ |

### 📋 Vérification `TextCaptureService.createTextCapture()`

Avant d'appeler ce service, vérifier son type de retour exact dans `mobile/src/contexts/capture/services/TextCaptureService.ts` :
- Si `Result<T>` → traiter `result.type !== 'success'` avec toast d'erreur
- Valider le champ `rawContent` attendu (texte brut)

### 🎨 Design VU Meter — implémentation simple

Pour éviter la complexité, utiliser des barres statiques avec animation `useEffect` basée sur `volumeLevel` :

```typescript
const bars = Array.from({ length: 12 }, (_, i) => {
  const animatedHeight = useRef(new Animated.Value(4)).current;

  useEffect(() => {
    const clampedVolume = Math.max(0, Math.min(10, volumeLevel));
    const targetHeight = volumeLevel < 0 ? 4 : 4 + (clampedVolume / 10) * 44;
    // Stagger par index pour effet naturel
    Animated.timing(animatedHeight, {
      toValue: targetHeight * (0.6 + Math.random() * 0.4), // légère variation
      duration: 100,
      useNativeDriver: false, // height n'est pas supporté par native driver
    }).start();
  }, [volumeLevel]);

  return animatedHeight;
});
```

**Note :** `useNativeDriver: false` est requis pour l'animation de `height`. Acceptable pour des barres simples (12 éléments max).

### 🚦 Garde moteur Whisper

Vérifier le moteur préféré AVANT d'ouvrir l'overlay :

```typescript
const engineService = container.resolve(TranscriptionEngineService);
const preferredEngine = await engineService.getPreferredEngine();

if (preferredEngine !== 'native') {
  toast.info(t('liveTranscription.nativeRequired'));
  return; // Ne pas ouvrir l'overlay
}
```

### Project Structure Notes

```
pensieve/mobile/src/
├── contexts/Normalization/services/
│   ├── ITranscriptionEngine.ts          ← MODIFIER : enableVolumeEvents + onVolumeChange signature
│   └── NativeTranscriptionEngine.ts     ← MODIFIER : volumechange support
├── hooks/
│   ├── useLiveTranscription.ts          ← CRÉER : hook principal
│   └── __tests__/
│       └── useLiveTranscription.test.ts ← CRÉER : tests unitaires
├── components/capture/
│   ├── LiveTranscriptionOverlay.tsx     ← CRÉER : UI overlay
│   └── RecordingOverlay.tsx             ← NE PAS MODIFIER
└── screens/capture/
    └── CaptureScreen.tsx                ← MODIFIER : bouton Live + Modal

tests/acceptance/
├── features/
│   └── story-8-6-transcription-live-avec-waveform.feature ← CRÉER
└── story-8-6-transcription-live-avec-waveform.test.ts     ← CRÉER
```

### References

- [Source: GitHub Issue #11](https://github.com/yohikofox/pensieve/issues/11) — Spécification originale
- [Source: `mobile/src/contexts/Normalization/services/NativeTranscriptionEngine.ts#273-372`] — `startRealTime()` existant
- [Source: `mobile/node_modules/expo-speech-recognition/src/ExpoSpeechRecognitionModule.types.ts#170`] — `volumechange` event `{ value: number }`
- [Source: `mobile/src/screens/capture/CaptureScreen.tsx#69-84`] — `CAPTURE_TOOLS_ALWAYS` (Voice + Text)
- [Source: `mobile/src/components/capture/RecordingOverlay.tsx`] — Pattern overlay plein écran à reproduire
- [Source: `mobile/src/contexts/capture/services/TextCaptureService.ts`] — Service de sauvegarde texte
- [Source: `mobile/src/hooks/useCaptureDetailInit.ts#157-184`] — Pattern lazy DI resolution
- [Source: ADR-021] — DI Lifecycle (Transient First)
- [Source: ADR-023] — Result Pattern
- [Source: `_bmad-output/implementation-artifacts/stories/epic-8/8-5-guide-config-modele-si-absent.md`] — Story précédente (contexte pattern store + DI)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-02-28 | Story créée depuis issue GitHub #11 — analyse exhaustive : NativeTranscriptionEngine.startRealTime() existant, expo-speech-recognition volumechange event, CaptureScreen tools, TextCaptureService. Architecture décidée : VU meter via volumechange natif, sauvegarde text capture, guard moteur whisper. | yohikofox |

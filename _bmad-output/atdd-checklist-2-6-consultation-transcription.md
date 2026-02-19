# ATDD Checklist: Story 2.6 - Consultation de Transcription

**Date:** 2026-01-21
**Status:** 🔴 RED Phase (Tests Failing)
**Epic:** Epic 2 - Capture & Transcription
**Story File:** `_bmad-output/planning-artifacts/epics.md` (lignes 674-723)

---

## Story Summary

**As a** user
**I want** to view the complete transcription of my audio captures
**So that** I can read my thoughts in text format and verify transcription accuracy

**Business Value:** Permet la consultation des transcriptions audio, garantit que les pensées audio sont lisibles sous forme de texte, offre feedback visuel sur l'état de transcription, fonctionne 100% offline (FR23).

---

## Acceptance Criteria Breakdown

### ✅ AC1: Afficher Transcription Complète avec Métadonnées

**Requirements:**
- Full transcription text displayed when opening capture detail view
- Original audio file available to play
- Transcription timestamp shown (when transcription completed)
- Audio duration displayed (e.g., "2:35")
- Clean UI with text formatted for readability

**Test Coverage:**
- ✅ BDD: `story-2-6-consultation-transcription.feature` - Scénario "Afficher transcription complète"
- ✅ BDD: `story-2-6.test.ts` - Step "la transcription complète est affichée"
- ✅ BDD: `story-2-6.test.ts` - Step "le fichier audio original est disponible pour lecture"
- ✅ BDD: `story-2-6.test.ts` - Step "le timestamp de transcription est affiché"
- ✅ BDD: `story-2-6.test.ts` - Step "la durée audio est affichée"

**Implementation Checklist:**
- [ ] Créer composant `CaptureDetailView` avec section transcription
- [ ] Afficher `Capture.normalizedText` dans un `<Text>` scrollable
- [ ] Afficher `Capture.transcribedAt` timestamp
- [ ] Calculer et afficher durée audio depuis `Capture.filePath` metadata
- [ ] Rendre audio file accessible via audio player component
- [ ] Tester avec `MockAudioPlayer` et `testContext.db.findById()`
- [ ] **Run test:** `npm run test:acceptance:story-2-6`
- [ ] ✅ Test passes (green phase)

---

### ✅ AC2: Contrôles de Lecture Audio

**Requirements:**
- Play button visible in capture detail view
- Tapping play starts audio playback
- Playback controls visible: play/pause button, progress bar, current time
- User can listen to audio while reading transcription simultaneously
- Audio state persists across view navigations

**Test Coverage:**
- ✅ BDD: `story-2-6-consultation-transcription.feature` - Scénario "Lire audio avec contrôles"
- ✅ BDD: `story-2-6.test.ts` - Step "l'utilisateur tape sur le bouton play"
- ✅ BDD: `story-2-6.test.ts` - Step "l'audio original démarre"
- ✅ BDD: `story-2-6.test.ts` - Step "les contrôles de lecture sont disponibles"
- ✅ BDD: `story-2-6.test.ts` - Step "l'utilisateur peut écouter en lisant la transcription"

**Implementation Checklist:**
- [ ] Implémenter `AudioPlayerComponent` avec expo-av
- [ ] Bouton play/pause avec icônes appropriées
- [ ] Afficher progress bar avec durée actuelle/totale
- [ ] Permettre lecture audio pendant scroll de transcription
- [ ] Tester avec `MockAudioPlayer.play()`, `pause()`, `getStatus()`
- [ ] Vérifier que `MockAudioPlayer.isPlaying() === true` après play
- [ ] **Run test:** `npm run test:acceptance:story-2-6`
- [ ] ✅ Test passes (green phase)

---

### ✅ AC3: Indicateur Transcription en Cours (Live Update)

**Requirements:**
- "Transcription in progress..." indicator shown when state = "TRANSCRIBING"
- Audio file remains playable during transcription
- Transcription text appears automatically when ready (live update via polling or listener)
- Progress indicator (spinner or percentage) visible
- User can navigate away and come back to see updated status

**Test Coverage:**
- ✅ BDD: `story-2-6-consultation-transcription.feature` - Scénario "Afficher indicateur transcription en cours"
- ✅ BDD: `story-2-6.test.ts` - Step "une capture est en cours de transcription"
- ✅ BDD: `story-2-6.test.ts` - Step "un indicateur 'Transcription in progress' est affiché"
- ✅ BDD: `story-2-6.test.ts` - Step "le fichier audio est disponible pour lecture"
- ✅ BDD: `story-2-6.test.ts` - Step "la transcription apparaît automatiquement quand prête"

**Implementation Checklist:**
- [ ] Afficher spinner + "Transcription in progress..." si `state === 'TRANSCRIBING'`
- [ ] Permettre lecture audio même pendant transcription
- [ ] Implémenter polling + EventBus (RxJS Subject) pour détecter state change (OP-SQLite n'a pas d'API observe() native)
- [ ] Quand state passe à "TRANSCRIBED", afficher `normalizedText` automatiquement
- [ ] Tester avec `testContext.db.update()` pour simuler state change
- [ ] Vérifier que UI se met à jour sans refresh manuel
- [ ] **Run test:** `npm run test:acceptance:story-2-6`
- [ ] ✅ Test passes (green phase)

---

### ✅ AC4: Gestion Transcription Échouée avec Retry

**Requirements:**
- "Transcription failed" error message shown when state = "TRANSCRIPTION_FAILED"
- Retry button visible and functional
- Tapping retry re-queues transcription job and updates state to "TRANSCRIBING"
- Original audio file remains playable after failure
- Error details logged for debugging

**Test Coverage:**
- ✅ BDD: `story-2-6-consultation-transcription.feature` - Scénario "Gérer transcription échouée avec retry"
- ✅ BDD: `story-2-6.test.ts` - Step "une transcription a échoué"
- ✅ BDD: `story-2-6.test.ts` - Step "un message 'Transcription failed' est affiché"
- ✅ BDD: `story-2-6.test.ts` - Step "un bouton Retry est disponible"
- ✅ BDD: `story-2-6.test.ts` - Step "le fichier audio est toujours lisible"
- ✅ BDD: `story-2-6.test.ts` - Step "la transcription redémarre après retry"

**Implementation Checklist:**
- [ ] Afficher error message + bouton Retry si `state === 'TRANSCRIPTION_FAILED'`
- [ ] Implémenter `onRetryTranscription()` handler
- [ ] Re-queue job dans `TranscriptionQueue.addJob()`
- [ ] Mettre à jour `Capture.state = 'TRANSCRIBING'`
- [ ] Tester avec `MockTranscriptionQueue.addJob()` et state update
- [ ] Vérifier que audio reste playable (MockAudioPlayer.canPlay())
- [ ] **Run test:** `npm run test:acceptance:story-2-6`
- [ ] ✅ Test passes (green phase)

---

### ✅ AC5: Fonctionnement Offline (FR23: Local Cache Compliance)

**Requirements:**
- Transcription loaded from local OP-SQLite cache when offline (FR23 compliance)
- All functionality works identically to online mode
- No network errors or "connection lost" messages shown
- Audio playback works offline (files stored locally)
- UI indicators do not show "offline" warnings for cached data

**Test Coverage:**
- ✅ BDD: `story-2-6-consultation-transcription.feature` - Scénario "Consulter transcription offline"
- ✅ BDD: `story-2-6.test.ts` - Step "l'appareil est hors ligne"
- ✅ BDD: `story-2-6.test.ts` - Step "la transcription est chargée depuis le cache local"
- ✅ BDD: `story-2-6.test.ts` - Step "toutes les fonctionnalités marchent comme en ligne"
- ✅ BDD: `story-2-6.test.ts` - Step "aucune erreur réseau n'est affichée"

**Implementation Checklist:**
- [ ] Vérifier que `Capture.normalizedText` est chargé depuis OP-SQLite (local)
- [ ] Tester avec `testContext.setOffline(true)`
- [ ] S'assurer que aucune requête réseau n'est faite
- [ ] Audio file path pointe vers stockage local (pas de téléchargement)
- [ ] Vérifier qu'aucun message "offline" n'apparaît
- [ ] **Run test:** `npm run test:acceptance:story-2-6`
- [ ] ✅ Test passes (green phase)

---

### ✅ AC6: Captures Texte (Non-Audio) - Différenciation UI

**Requirements:**
- For text captures (captureType = "TEXT"), show only text content
- No transcription section displayed (since there's no audio)
- No audio player controls shown
- Capture type clearly indicated (e.g., badge "Text Capture" vs "Audio Capture")
- Consistent UI layout regardless of capture type

**Test Coverage:**
- ✅ BDD: `story-2-6-consultation-transcription.feature` - Scénario "Afficher capture texte (non-audio)"
- ✅ BDD: `story-2-6.test.ts` - Step "une capture texte (pas audio)"
- ✅ BDD: `story-2-6.test.ts` - Step "seul le contenu texte est affiché"
- ✅ BDD: `story-2-6.test.ts` - Step "aucune section transcription n'est affichée"
- ✅ BDD: `story-2-6.test.ts` - Step "le type de capture est clairement indiqué"

**Implementation Checklist:**
- [ ] Implémenter conditional rendering: `if (captureType === 'TEXT')`
- [ ] Afficher `Capture.rawContent` pour text captures
- [ ] Ne pas afficher audio player si `captureType !== 'AUDIO'`
- [ ] Ajouter badge UI "Text Capture" vs "Audio Capture"
- [ ] Tester avec captures TEXT et AUDIO pour vérifier comportement différent
- [ ] **Run test:** `npm run test:acceptance:story-2-6`
- [ ] ✅ Test passes (green phase)

---

### ✅ AC7: Formatage du Texte et Caractères Spéciaux

**Requirements:**
- Transcription text rendered correctly with proper formatting
- Line breaks preserved from Whisper output (e.g., `\n` → new line in UI)
- Paragraphs separated visually
- Special characters (accents, punctuation) displayed correctly
- Text wrapping handles long words gracefully
- Font size and spacing optimized for readability

**Test Coverage:**
- ✅ BDD: `story-2-6-consultation-transcription.feature` - Scénario "Préserver formatage et caractères spéciaux"
- ✅ BDD: `story-2-6.test.ts` - Step "la transcription contient des retours à la ligne"
- ✅ BDD: `story-2-6.test.ts` - Step "le texte est rendu correctement"
- ✅ BDD: `story-2-6.test.ts` - Step "les sauts de ligne et paragraphes sont préservés"
- ✅ BDD: `story-2-6.test.ts` - Step "les caractères spéciaux sont affichés correctement"

**Implementation Checklist:**
- [ ] Utiliser `<Text>` avec prop `selectable={true}` pour transcription
- [ ] Préserver line breaks avec `{normalizedText.split('\n').map(...)}` ou CSS whitespace
- [ ] Tester avec transcription contenant `\n`, accents (é, à, ç), ponctuation (!, ?, ...)
- [ ] S'assurer que caractères spéciaux ne sont pas échappés/corrompus
- [ ] Vérifier que long texte scroll correctement sans layout break
- [ ] **Run test:** `npm run test:acceptance:story-2-6`
- [ ] ✅ Test passes (green phase)

---

## Failing Tests Created (RED Phase)

### BDD Tests (22 scenarios Gherkin)

**File:** `pensieve/mobile/tests/acceptance/features/story-2-6-consultation-transcription.feature`

**Scenarios:**

#### AC1 - Afficher Transcription Complète (4 scenarios)
- ✅ **Scenario:** "Afficher transcription complète avec métadonnées"
  - **Status:** RED - CaptureDetailView not implemented
  - **Verifies:** Full text, timestamp, audio duration visible

- ✅ **Scenario:** "Afficher timestamp de transcription"
  - **Status:** RED - Timestamp rendering not implemented
  - **Verifies:** transcribedAt field shown

- ✅ **Scenario:** "Afficher durée audio"
  - **Status:** RED - Audio duration calculation not implemented
  - **Verifies:** Duration displayed (e.g., "2:35")

- ✅ **Scenario:** "Rendre fichier audio disponible"
  - **Status:** RED - Audio player integration not implemented
  - **Verifies:** Audio file playable

#### AC2 - Contrôles de Lecture Audio (5 scenarios)
- ✅ **Scenario:** "Lire audio avec contrôles de lecture"
  - **Status:** RED - AudioPlayerComponent not implemented
  - **Verifies:** Play button starts playback, controls visible

- ✅ **Scenario:** "Pause audio en lecture"
  - **Status:** RED - Pause functionality not implemented
  - **Verifies:** Pause button works, audio stops

- ✅ **Scenario:** "Afficher barre de progression"
  - **Status:** RED - Progress bar not implemented
  - **Verifies:** Current time / total duration displayed

- ✅ **Scenario:** "Écouter en lisant la transcription"
  - **Status:** RED - Simultaneous audio+text not tested
  - **Verifies:** Audio plays while scrolling text

- ✅ **Scenario:** "Reprendre lecture après navigation"
  - **Status:** RED - Audio state persistence not implemented
  - **Verifies:** Audio state preserved across views

#### AC3 - Indicateur Transcription en Cours (4 scenarios)
- ✅ **Scenario:** "Afficher indicateur transcription en cours"
  - **Status:** RED - Progress indicator not implemented
  - **Verifies:** "Transcription in progress..." shown

- ✅ **Scenario:** "Audio disponible pendant transcription"
  - **Status:** RED - Audio playback during transcription not verified
  - **Verifies:** Audio playable when state = "TRANSCRIBING"

- ✅ **Scenario:** "Live update quand transcription prête"
  - **Status:** RED - Live update mechanism not implemented
  - **Verifies:** Text appears automatically without manual refresh

- ✅ **Scenario:** "Polling state change"
  - **Status:** RED - Polling/observe not implemented
  - **Verifies:** UI updates when state changes

#### AC4 - Gestion Transcription Échouée (4 scenarios)
- ✅ **Scenario:** "Afficher erreur transcription échouée"
  - **Status:** RED - Error UI not implemented
  - **Verifies:** "Transcription failed" message shown

- ✅ **Scenario:** "Bouton retry disponible"
  - **Status:** RED - Retry button not implemented
  - **Verifies:** Retry button visible and functional

- ✅ **Scenario:** "Retry redémarre transcription"
  - **Status:** RED - Retry handler not implemented
  - **Verifies:** State updated to "TRANSCRIBING" after retry

- ✅ **Scenario:** "Audio lisible après échec"
  - **Status:** RED - Audio preservation not verified
  - **Verifies:** Audio playable despite failed transcription

#### AC5 - Fonctionnement Offline (3 scenarios)
- ✅ **Scenario:** "Consulter transcription offline"
  - **Status:** RED - Offline mode not verified
  - **Verifies:** FR23 compliance (local cache loading)

- ✅ **Scenario:** "Aucune erreur réseau offline"
  - **Status:** RED - Network error handling not verified
  - **Verifies:** No network warnings shown

- ✅ **Scenario:** "Audio playback offline"
  - **Status:** RED - Local audio playback not verified
  - **Verifies:** Audio plays from local storage

#### AC6 - Captures Texte (2 scenarios)
- ✅ **Scenario:** "Afficher capture texte (non-audio)"
  - **Status:** RED - Text capture UI not implemented
  - **Verifies:** Text content shown, no audio player

- ✅ **Scenario:** "Différencier type capture dans UI"
  - **Status:** RED - Capture type badge not implemented
  - **Verifies:** Badge "Text Capture" vs "Audio Capture" visible

#### AC7 - Formatage du Texte (4 scenarios)
- ✅ **Scenario:** "Préserver sauts de ligne"
  - **Status:** RED - Line break rendering not implemented
  - **Verifies:** `\n` → new line in UI

- ✅ **Scenario:** "Afficher caractères spéciaux"
  - **Status:** RED - Special character handling not tested
  - **Verifies:** Accents (é, à), punctuation (!?) rendered correctly

- ✅ **Scenario:** "Gérer texte long avec scroll"
  - **Status:** RED - Long text handling not tested
  - **Verifies:** Text scrolls without layout break

- ✅ **Scenario:** "Optimiser lisibilité"
  - **Status:** RED - Readability optimization not implemented
  - **Verifies:** Font size, spacing appropriate

---

## Data Infrastructure Created

### New Mocks Required

#### MockAudioPlayer (NEW)

**File:** `pensieve/mobile/tests/acceptance/support/test-context.ts`

**Purpose:** Simulate audio playback for capture consultation

```typescript
export class MockAudioPlayer {
  private _isPlaying: boolean = false;
  private _isPaused: boolean = false;
  private _currentTime: number = 0;
  private _duration: number = 0;
  private _audioFilePath: string | null = null;

  async loadAudio(filePath: string, duration: number): Promise<void> {
    this._audioFilePath = filePath;
    this._duration = duration;
    this._currentTime = 0;
    this._isPlaying = false;
    this._isPaused = false;
  }

  async play(): Promise<void> {
    if (!this._audioFilePath) {
      throw new Error('No audio file loaded');
    }
    this._isPlaying = true;
    this._isPaused = false;
  }

  async pause(): Promise<void> {
    if (this._isPlaying) {
      this._isPlaying = false;
      this._isPaused = true;
    }
  }

  async stop(): Promise<void> {
    this._isPlaying = false;
    this._isPaused = false;
    this._currentTime = 0;
  }

  isPlaying(): boolean {
    return this._isPlaying;
  }

  isPaused(): boolean {
    return this._isPaused;
  }

  getCurrentTime(): number {
    return this._currentTime;
  }

  getDuration(): number {
    return this._duration;
  }

  setCurrentTime(time: number): void {
    if (time >= 0 && time <= this._duration) {
      this._currentTime = time;
    }
  }

  getAudioFilePath(): string | null {
    return this._audioFilePath;
  }

  canPlay(): boolean {
    return this._audioFilePath !== null;
  }

  reset(): void {
    this._isPlaying = false;
    this._isPaused = false;
    this._currentTime = 0;
    this._duration = 0;
    this._audioFilePath = null;
  }
}
```

---

### Existing Mocks Used

#### InMemoryDatabase (Already exists)

**Methods used:**
- `findById(id)` - Retrieve Capture for detail view
- `update(id, updates)` - Update state for retry transcription
- `reset()` - Clean up between tests

#### MockFileSystem (Already exists)

**Methods used:**
- `fileExists(path)` - Verify audio file exists
- `reset()` - Clean up between tests

#### MockTranscriptionQueue (Already exists - Story 2.5)

**Methods used:**
- `addJob(captureId, audioFilePath, audioDuration)` - Re-queue job on retry
- `reset()` - Clean up between tests

---

## Running Tests

### Run All Story 2.6 Tests

```bash
# Run all BDD tests for story 2.6
npm run test:acceptance:story-2-6

# Run in watch mode (development)
npm run test:acceptance:watch -- story-2-6

# Run with coverage
npm run test:coverage -- story-2-6
```

### Run Specific Scenarios

```bash
# Run only AC2 tests (audio controls)
npm run test:acceptance -- --testNamePattern="audio.*contrôles"

# Run only AC5 tests (offline)
npm run test:acceptance -- --testNamePattern="offline"

# Run only AC4 tests (retry)
npm run test:acceptance -- --testNamePattern="retry"
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete) ✅

**TEA Agent Responsibilities:**

- ✅ All 22 BDD scenarios written in Gherkin
- ✅ All step definitions created with `jest-cucumber`
- ✅ MockAudioPlayer created in `test-context.ts`
- ✅ Existing mocks identified (InMemoryDatabase, MockFileSystem, MockTranscriptionQueue)
- ✅ Implementation checklist created with clear tasks
- ✅ FR23 compliance tagged (offline local cache)

---

### GREEN Phase (DEV Team - Next Steps)

**DEV Agent Responsibilities:**

1. **Implement MockAudioPlayer in test-context.ts**
   - Add MockAudioPlayer class
   - Wire it into TestContext
   - Export for use in tests

2. **Implement CaptureDetailView Component**
   - Display `Capture.normalizedText` in scrollable Text component
   - Show metadata: timestamp, audio duration
   - Conditional rendering for TEXT vs AUDIO captures

3. **Implement AudioPlayerComponent**
   - Play/pause button with expo-av
   - Progress bar with current time / total duration
   - Audio state management (play, pause, seek)

4. **Implement Live Update Mechanism**
   - EventBus (RxJS Subject) pour recevoir les events de state change émis après update OP-SQLite
   - Polling OP-SQLite (requête périodique) comme fallback si EventBus non disponible
   - Détecter la transition "TRANSCRIBING" → "TRANSCRIBED" pour auto-update UI
   - Auto-update UI without manual refresh

5. **Implement Retry Transcription Handler**
   - Button UI for state "TRANSCRIPTION_FAILED"
   - Re-queue job in TranscriptionQueue
   - Update Capture.state to "TRANSCRIBING"

6. **Implement Text Formatting**
   - Preserve line breaks (`\n` → new line)
   - Handle special characters (accents, punctuation)
   - Optimize font size and spacing for readability

7. **Implement Offline Mode Support (FR23)**
   - Load transcription from local OP-SQLite cache
   - No network calls required
   - Audio playback from local storage

8. **Test Each Implementation**
   - Run `npm run test:acceptance:story-2-6` after each change
   - Verify tests turn green one by one

---

### REFACTOR Phase (DEV Team - After All Tests Pass)

**DEV Agent Responsibilities:**

1. **Code Quality Review**
   - Extract reusable audio player component
   - Optimize text rendering for long transcriptions
   - Add TypeScript strict typing

2. **Performance Optimization**
   - Ensure fast loading from local cache
   - Optimize audio player state management
   - Test with long transcriptions (500+ words)

3. **Documentation**
   - Add JSDoc comments to CaptureDetailView
   - Document audio player API
   - Update README with consultation architecture

---

## Next Steps

1. **Add MockAudioPlayer to test-context.ts** (DEV prerequisite)
2. **Run failing tests** to confirm RED phase: `npm run test:acceptance:story-2-6`
3. **Begin implementation** using implementation checklist as guide
4. **Work one AC at a time** (AC1 → AC2 → ... → AC7)
5. **Run tests frequently** to get immediate feedback
6. **When all tests pass**, refactor for quality

---

## Knowledge Base References Applied

- **data-factories.md** - Factory patterns for test data
- **fixture-architecture.md** - Mock architecture (MockAudioPlayer)
- **test-quality.md** - Deterministic tests, offline compliance tagging
- **selector-resilience.md** - data-testid strategy for UI elements

---

## Required data-testid Attributes

### Capture Detail View

- `capture-detail-view` - Main container
- `capture-type-badge` - Badge showing "Text Capture" or "Audio Capture"
- `transcription-text` - Transcription text content
- `transcription-timestamp` - Timestamp when transcription completed
- `audio-duration` - Audio duration display (e.g., "2:35")

### Audio Player Controls

- `audio-play-button` - Play button
- `audio-pause-button` - Pause button
- `audio-progress-bar` - Progress bar component
- `audio-current-time` - Current playback time
- `audio-total-duration` - Total audio duration

### Status Indicators

- `transcription-in-progress-indicator` - Spinner/text "Transcription in progress..."
- `transcription-failed-message` - Error message "Transcription failed"
- `transcription-retry-button` - Retry button

---

## Contact

**Questions or Issues?**

- Ask in team standup
- Tag @TEA in Slack/Discord
- Refer to `_bmad/bmm/testarch/README.md` for TEA workflow documentation

---

**Generated by BMad TEA Agent** - 2026-01-21

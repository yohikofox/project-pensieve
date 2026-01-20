# Story 2.6: Consultation de Transcription

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **user**,
I want **to read the full transcription of my audio captures**,
so that **I can review my thoughts in text form without re-listening to the audio** (FR8).

## Acceptance Criteria

### AC1: Display Transcription in Capture Detail View
**Given** I have a capture with completed transcription
**When** I open the capture detail view
**Then** the full transcription text is displayed
**And** the text is readable with proper formatting
**And** the transcription is scrollable if longer than the screen

### AC2: Show Transcription Status for In-Progress Captures
**Given** a capture is currently being transcribed
**When** I open the capture detail view
**Then** I see a progress indicator showing "Transcribing..."
**And** the transcription text appears automatically when complete
**And** I do not need to refresh or navigate away

### AC3: Handle Failed Transcriptions Gracefully
**Given** a capture with failed transcription
**When** I open the capture detail view
**Then** I see an error message explaining the failure
**And** a "Retry Transcription" button is displayed
**And** the original audio is still playable

### AC4: Text Capture Shows Original Text Immediately
**Given** I have a text capture (not audio)
**When** I open the capture detail view
**Then** the original text is displayed immediately (no transcription needed)
**And** the display is identical to audio transcriptions (consistent UI)

### AC5: Copy Transcription to Clipboard
**Given** I am viewing a transcription
**When** I long-press on the transcription text
**Then** a context menu appears with "Copy" option
**And** the full transcription is copied to clipboard

### AC6: Offline Access to Transcriptions
**Given** I am offline
**When** I open a capture with completed transcription
**Then** the transcription is displayed immediately from local cache (NFR7 compliance)
**And** no network error is shown

## Tasks / Subtasks

- [ ] **Task 1: Create CaptureDetailView Component** (AC: 1, 2, 3, 4)
  - [ ] Subtask 1.1: Design CaptureDetailView layout
    - Header with capture timestamp and type icon
    - Audio playback controls (if type='audio')
    - Transcription/text display area (scrollable)
    - Action buttons (copy, share, etc.)
  - [ ] Subtask 1.2: Display transcription based on Capture state
    - If state='ready': Show Capture.normalizedText
    - If state='processing': Show progress indicator
    - If state='failed': Show error + retry button
    - If type='text': Show Capture.rawContent (no transcription)
  - [ ] Subtask 1.3: Implement auto-update for in-progress transcriptions
    - Use WatermelonDB observe() to watch Capture changes
    - Update UI reactively when transcription completes
    - No manual refresh required

- [ ] **Task 2: Implement Transcription Text Display** (AC: 1, 6)
  - [ ] Subtask 2.1: Style transcription text
    - Use readable font size (16-18pt)
    - Proper line height for readability (1.5)
    - Support dark/light mode
    - Left-aligned, justified text
  - [ ] Subtask 2.2: Implement scrollable text area
    - Use ScrollView for long transcriptions
    - Smooth scrolling performance
    - Show scroll indicators
  - [ ] Subtask 2.3: Handle empty or missing transcriptions
    - Show placeholder if normalizedText is null
    - Placeholder: "Transcription will appear here..."
    - Only for audio captures (not text)

- [ ] **Task 3: Implement Transcription Status Indicators** (AC: 2, 3)
  - [ ] Subtask 3.1: Show "Transcribing..." progress
    - Animated spinner
    - Progress text: "Transcribing... (Xm Ys remaining)"
    - Estimate time based on audio duration (2x rule)
  - [ ] Subtask 3.2: Show transcription failure UI
    - Error icon with message
    - User-friendly error text (from Capture.transcriptionError)
    - "Retry Transcription" button
  - [ ] Subtask 3.3: Handle retry action
    - Call TranscriptionService to re-queue capture
    - Update UI to "Transcribing..." state
    - Show success/failure after retry

- [ ] **Task 4: Implement Copy to Clipboard** (AC: 5)
  - [ ] Subtask 4.1: Add long-press gesture recognizer
    - Use React Native's onLongPress on Text component
    - Show context menu with "Copy" option
  - [ ] Subtask 4.2: Implement copy functionality
    - Use Clipboard API (from @react-native-clipboard/clipboard)
    - Copy full Capture.normalizedText
    - Show confirmation toast: "Copied to clipboard"
  - [ ] Subtask 4.3: Add copy button as alternative
    - Toolbar button with copy icon
    - Accessible for users who don't know long-press
    - Same copy functionality

- [ ] **Task 5: Add Audio Playback Controls (Optional Enhancement)** (AC: 1)
  - [ ] Subtask 5.1: Show audio player if type='audio'
    - Play/Pause button
    - Seek bar with current time/duration
    - Playback speed control (0.5x, 1x, 1.5x, 2x)
  - [ ] Subtask 5.2: Sync audio playback with transcription
    - Highlight current word/sentence during playback (future enhancement)
    - For MVP: Just show audio player above transcription

- [ ] **Task 6: Implement Navigation to CaptureDetailView** (AC: All)
  - [ ] Subtask 6.1: Add navigation from feed
    - Tap on capture in feed → navigate to CaptureDetailView
    - Pass Capture ID as route param
    - Use React Navigation
  - [ ] Subtask 6.2: Add back button
    - Header back button to return to feed
    - Support Android hardware back button
  - [ ] Subtask 6.3: Deep link support (optional)
    - Allow notifications to open specific capture detail
    - URL scheme: pensine://capture/{captureId}

- [ ] **Task 7: Offline Mode Support** (AC: 6)
  - [ ] Subtask 7.1: Load transcription from WatermelonDB
    - No network calls required (already local)
    - Read Capture.normalizedText directly
    - Instant display (< 1s per NFR4)
  - [ ] Subtask 7.2: Verify offline functionality
    - Test with network disabled
    - Ensure no errors or loading states
    - Confirm UI is identical offline/online

- [ ] **Task 8: Write Comprehensive Tests** (AC: All)
  - [ ] Subtask 8.1: Component tests for CaptureDetailView
    - Test rendering with completed transcription
    - Test rendering with in-progress transcription
    - Test rendering with failed transcription
    - Test text capture display
  - [ ] Subtask 8.2: Integration tests for navigation
    - Test navigation from feed to detail view
    - Test back button navigation
    - Test deep link navigation (if implemented)
  - [ ] Subtask 8.3: Unit tests for copy functionality
    - Test clipboard copy
    - Test toast confirmation
  - [ ] Subtask 8.4: Offline tests
    - Test detail view loads offline
    - Test transcription displayed from cache
    - Verify no network errors

## Dev Notes

### Architecture Context

**Bounded Context:** Capture Context (UI layer)
**UI Components:**
- CaptureDetailView (main component)
- TranscriptionDisplay (sub-component)
- AudioPlayer (sub-component, optional)

**Data Flow:**
```
WatermelonDB Capture → CaptureDetailView → TranscriptionDisplay
                                          ↓
                                     User sees text
```

### Technical Stack

**Already installed:**
- React Native (Text, ScrollView, TouchableOpacity)
- WatermelonDB (observe() for reactive updates)
- React Navigation (for navigation)
- Clipboard API (`@react-native-clipboard/clipboard`)

**Optional:**
- expo-av (audio playback controls)

### UX Requirements (Liquid Glass Design System)

**Typography:**
- Font size: 16-18pt for body text
- Line height: 1.5 for readability
- Font family: System default (SF Pro on iOS, Roboto on Android)
- Support dark/light modes

**Scrolling:**
- Smooth 60fps scrolling
- Scroll indicators visible
- Bounce effect on over-scroll (iOS native)

**Interactions:**
- Long-press for context menu (copy)
- Toolbar button as alternative (accessibility)
- Haptic feedback on copy action

**Animations:**
- Smooth transitions between states (processing → ready)
- Fade-in animation for transcription text
- Toast animation for "Copied" confirmation

### Performance Requirements

**NFR4:** Chargement < 1s
- Transcription loaded from local WatermelonDB
- No network calls
- Instant display

**Smooth Scrolling:**
- Optimize long text rendering
- Use React Native's optimized Text component
- No janky scrolling even for 1000+ word transcriptions

### Reactive UI Updates

**WatermelonDB observe():**
```typescript
const capture = useMemo(() =>
  database.collections.get('captures').findAndObserve(captureId),
  [captureId]
)

// Automatically updates when transcription completes
```

**State transitions:**
- "processing" → Show spinner
- "ready" → Show transcription
- "failed" → Show error + retry

### Reuse from Stories 2.1-2.5

**Already implemented:**
- Capture WatermelonDB model (with normalizedText)
- TranscriptionService (retry logic from 2.5)
- Offline-first architecture
- React Navigation setup

**New:**
- CaptureDetailView component
- Transcription text display
- Copy to clipboard

### File Structure

```
apps/mobile/
├── src/
│   ├── contexts/
│   │   └── Capture/
│   │       └── ui/
│   │           ├── CaptureDetailView.tsx  # NEW (main screen)
│   │           ├── TranscriptionDisplay.tsx  # NEW (text display)
│   │           └── AudioPlayer.tsx  # NEW (optional)
```

### Navigation Setup

**React Navigation:**
```typescript
// App.tsx or navigation config
<Stack.Screen
  name="CaptureDetail"
  component={CaptureDetailView}
  options={{ title: 'Capture Details' }}
/>

// Navigate from feed:
navigation.navigate('CaptureDetail', { captureId: 'abc-123' })
```

### Clipboard Integration

**@react-native-clipboard/clipboard:**
```typescript
import Clipboard from '@react-native-clipboard/clipboard'

const copyTranscription = () => {
  Clipboard.setString(capture.normalizedText)
  showToast('Copied to clipboard')
}
```

### Testing Standards

- **Component tests:** CaptureDetailView rendering (all states)
- **Integration tests:** Navigation, reactive updates
- **Unit tests:** Copy functionality, error handling
- **Offline tests:** Verify offline display
- **Performance tests:** Verify < 1s load time

### Dependencies

**New:**
- `@react-native-clipboard/clipboard` (clipboard copy)

**Already installed:**
- `@nozbe/watermelondb`
- `react-navigation`
- `expo-av` (if audio playback needed)

### Accessibility Considerations

**WCAG Compliance:**
- Readable text size (16-18pt minimum)
- High contrast for dark/light modes
- Copy button as alternative to long-press (not all users know gestures)
- Screen reader support (accessibilityLabel on components)

### Previous Story Intelligence (Stories 2.1-2.5)

**Learnings:**
- WatermelonDB observe() works great for reactive updates
- Offline-first patterns well-established
- Capture state machine reliable
- Retry logic from 2.5 can be reused

**Code patterns to reuse:**
- WatermelonDB observe() for reactive UI
- State-based rendering (switch on Capture.state)
- Error handling with user-friendly messages
- Offline-first data loading

### References

- [Source: _bmad-output/planning-artifacts/epics.md#Story 2.6]
- [Source: _bmad-output/planning-artifacts/architecture.md#Capture Context]
- [Source: _bmad-output/implementation-artifacts/2-5-transcription-on-device-avec-whisper.md#TranscriptionService]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5

### Debug Log References

<!-- Populate during implementation -->

### Completion Notes List

<!-- Populate during implementation -->

### File List

<!-- Populate during implementation -->

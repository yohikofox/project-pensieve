# Story 3.2 - Next Steps Guide

## Current Status (2026-02-02)

✅ **Story 3.2a COMPLETE** - Core infrastructure delivered

**What's Done**:
- observeById() reactive updates (AC5)
- Offline functionality verified (AC4)
- Test infrastructure in place
- Existing CaptureDetailScreen validated (Story 2.6)

**What's Next**: Two sub-stories for complete AC coverage

---

## Sub-Story 3.2b: Audio Player & Waveform

### Goal
Complete AC2 by implementing full audio playback UI with waveform visualization and transcription sync.

### Acceptance Criteria
From original AC2:
- Audio player with play/pause controls ✅ (logic exists in mocks)
- Seek bar with current time / total duration
- Waveform visualization of audio
- Full transcription text display ✅ (exists)
- Playback position highlighted in transcription (sync)
- Tap on transcription word → seek audio to that position

### Tasks to Implement

**Task 1: Audio Player Component**
- [ ] Create `AudioPlayer.tsx` component
  - Use expo-av's Audio API
  - Play/pause button with state management
  - Seek slider with onSlidingComplete
  - Time display (current / total)
  - Loading state while audio loads

**Task 2: Waveform Visualization**
- [ ] Research and choose implementation:
  - Option A: react-native-audio-waveform (if Expo-compatible)
  - Option B: Custom canvas with audio buffer analysis
- [ ] Integrate waveform with seek functionality
- [ ] Show playback position on waveform

**Task 3: Playback-Transcription Sync**
- [ ] Calculate word timing:
  ```typescript
  const msPerWord = audioDuration / words.length;
  const currentWordIndex = Math.floor(positionMs / msPerWord);
  ```
- [ ] Highlight current word during playback
- [ ] Auto-scroll transcription to keep current word visible
- [ ] Implement tap-on-word-to-seek functionality

**Task 4: Integration into CaptureDetailScreen**
- [ ] Conditionally render AudioPlayer for audio captures
- [ ] Pass capture.rawContent (audio file path) to player
- [ ] Pass capture.normalizedText to sync component
- [ ] Handle audio loading errors gracefully

**Task 5: Tests**
- [ ] Component tests for AudioPlayer
- [ ] Component tests for Waveform
- [ ] Integration test: playback changes highlight
- [ ] Integration test: tap word seeks audio
- [ ] BDD tests with real UI rendering

### Technical Notes

**expo-av Example**:
```typescript
import { Audio } from 'expo-av';

const { sound } = await Audio.Sound.createAsync(
  { uri: audioUri },
  { shouldPlay: false },
  onPlaybackStatusUpdate
);

const onPlaybackStatusUpdate = (status: AVPlaybackStatus) => {
  if (status.isLoaded) {
    setPosition(status.positionMillis);
    setDuration(status.durationMillis || 0);
    setIsPlaying(status.isPlaying);
  }
};
```

**File References**:
- CaptureDetailScreen: `mobile/src/screens/captures/CaptureDetailScreen.tsx`
- Design system: `mobile/src/design-system/components/`
- Test mocks: MockAudioPlayer already exists in test-context.ts

### Estimated Effort
**Medium**: 3-5 subtasks, ~1-2 days with testing

---

## Sub-Story 3.2c: Navigation & Interactions

### Goal
Complete AC6, AC7, AC8 by implementing gesture-based navigation and context menu.

### Acceptance Criteria

**AC6 - Swipe Navigation**:
- Swipe left → next capture (if exists)
- Swipe right → previous capture (if exists)
- Smooth horizontal slide animation
- No navigation at first/last capture

**AC7 - Return to Feed**:
- Back button → return to feed
- Feed scrolls to previously viewed capture
- Swipe down → return to feed (optional gesture)

**AC8 - Context Menu**:
- Long-press → context menu appears
- Haptic feedback on activation
- Actions: Share, Delete, Edit
- Share: Export capture content
- Delete: Confirm dialog → remove capture
- Edit: Navigate to edit screen (if applicable)

### Tasks to Implement

**Task 1: Swipe Gesture Handler**
- [ ] Install react-native-gesture-handler (if not present)
- [ ] Create GestureDetector wrapper in CaptureDetailScreen
- [ ] Implement Fling gesture for left/right swipe
- [ ] Pass captureIds array and currentIndex via navigation params
- [ ] Navigate to next/prev capture on swipe
- [ ] Add boundary checks (first/last)

**Task 2: Navigation Animation**
- [ ] Use LayoutAnimation or react-native-reanimated
- [ ] Horizontal slide transition
- [ ] Update capture content smoothly
- [ ] Maintain scroll position in transcription view

**Task 3: Feed Position Preservation**
- [ ] Pass captureIndex param when navigating to detail
- [ ] Store index in navigation state
- [ ] On back navigation, pass index back to feed
- [ ] Implement scrollToIndex in CapturesListScreen

**Task 4: Context Menu Component**
- [ ] Create `ContextMenu.tsx` component
- [ ] Add LongPress gesture to detail content area
- [ ] Trigger haptic feedback (expo-haptics)
- [ ] Show modal with action buttons
- [ ] Implement Share action (Share API)
- [ ] Implement Delete action (with confirmation)
- [ ] Implement Edit action (navigate to edit screen)

**Task 5: Tests**
- [ ] Gesture tests (swipe, long-press)
- [ ] Navigation state tests
- [ ] Context menu action tests
- [ ] BDD scenarios for AC6, AC7, AC8

### Technical Notes

**Gesture Handler Example**:
```typescript
import { GestureDetector, Gesture } from 'react-native-gesture-handler';

const swipeGesture = Gesture.Fling()
  .direction(Directions.LEFT | Directions.RIGHT)
  .onEnd((event) => {
    if (event.x < 0 && hasNext) navigateToNext();
    if (event.x > 0 && hasPrev) navigateToPrev();
  });
```

**Haptic Feedback**:
```typescript
import * as Haptics from 'expo-haptics';
Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
```

**File References**:
- CaptureDetailScreen: `mobile/src/screens/captures/CaptureDetailScreen.tsx`
- CapturesListScreen: `mobile/src/screens/captures/CapturesListScreen.tsx`
- Navigation: React Navigation stack

### Estimated Effort
**Medium**: 4-6 subtasks, ~1-2 days with testing

---

## Implementation Order Recommendation

1. **Current Sprint**: ✅ Story 3.2a (DONE)
2. **Next Sprint**: Story 3.2b (Audio Player - higher user value)
3. **Following Sprint**: Story 3.2c (Gestures - UX polish)

This order prioritizes:
- ✅ Immediate value (reactive updates, offline)
- 🎵 Core feature completeness (audio playback)
- ✨ UX enhancements (gestures, menus)

---

## Quick Start for Next Developer

**To implement Story 3.2b**:
1. Read this guide's 3.2b section
2. Review existing CaptureDetailScreen.tsx (line 750+ has audio detection)
3. Check MockAudioPlayer in test-context.ts for expected behavior
4. Start with AudioPlayer component, then Waveform, then sync
5. Run tests: `npm test -- story-3-2b` (create tests first)

**To implement Story 3.2c**:
1. Read this guide's 3.2c section
2. Install react-native-gesture-handler if needed
3. Start with swipe navigation, then context menu
4. Test gestures on real device (simulators may behave differently)
5. Ensure haptic feedback works (requires physical device)

---

## Questions?

If anything is unclear, refer to:
- Story 3.2 file: `3-2-vue-detail-dune-capture.md`
- Dev Agent Record section for implementation context
- Story 2.6 (CaptureDetailScreen baseline)
- Story 3.1 (CapturesListScreen for navigation context)

**Architecture docs**: `_bmad-output/planning-artifacts/architecture.md`
**PRD**: `_bmad-output/planning-artifacts/prd.md`

# Story 2.8: Bouton Retry pour Transcriptions en Échec

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **user**,
I want **a visible "Retry" button on capture cards that failed transcription**,
So that **I can manually retry the transcription without navigating to settings or detail views**.

**Context:** Cette story améliore l'UX de Story 2.5 (Transcription Whisper) et Story 2.6 (Consultation) en ajoutant un retry manuel avec rate limiting (3 tentatives / 20 minutes) directement sur les cartes de capture.

**Issue GitHub:** #1 - feat: Bouton de retry pour les transcriptions en échec

## Acceptance Criteria

### AC1: Display Retry Button on Failed Capture Cards
**Given** I have a capture with transcription status 'failed'
**When** I view the capture card in the feed
**Then** a "Retry" button is prominently displayed on the card
**And** the button replaces or complements the "Failed" status badge
**And** the button is clearly actionable (distinct color, icon, text)

### AC2: Retry Button Triggers Manual Transcription Attempt
**Given** I am viewing a failed capture card
**When** I tap the "Retry" button
**Then** the transcription process is manually triggered for this capture
**And** the capture state immediately changes to 'processing'
**And** the capture is queued in TranscriptionQueueService
**And** a progress indicator replaces the "Retry" button

### AC3: Rate Limiting - Maximum 3 Retries per 20-Minute Window
**Given** I have retried a capture transcription X times within the last 20 minutes
**When** I attempt to retry again
**Then** if X < 3, the retry proceeds normally
**And** if X >= 3, the "Retry" button is disabled
**And** a message is shown: "Retry limit reached. Try again in Y minutes"
**And** the countdown updates in real-time until the 20-minute window resets

### AC4: Reset Retry Counter After 20-Minute Window
**Given** a capture has reached the retry limit (3 attempts in 20 minutes)
**When** 20 minutes have elapsed since the first retry attempt
**Then** the retry counter resets to 0
**And** the "Retry" button becomes enabled again
**And** the user can attempt 3 more retries

### AC5: Show Progress Indicator During Retry Transcription
**Given** I have triggered a manual retry
**When** the transcription is in progress
**Then** a progress indicator (spinner or progress bar) is shown on the capture card
**And** the indicator shows "Transcribing..." status text
**And** the capture card state reflects 'processing'
**And** the UI updates reactively when transcription completes (success or failure)

### AC6: Update Card on Retry Success
**Given** a manual retry has been triggered
**When** the transcription completes successfully
**Then** the capture state changes to 'ready'
**And** the "Retry" button is removed
**And** the transcription text is displayed (in detail view)
**And** a success indicator briefly appears (e.g., green checkmark animation)

### AC7: Update Card on Retry Failure (Attempts Remaining)
**Given** a manual retry has failed
**When** the capture state changes back to 'failed'
**And** retry attempts remaining < 3
**Then** the "Retry" button reappears
**And** the retry counter is incremented
**And** the failure timestamp is updated
**And** the error message is updated (if different)

### AC8: Debug Mode - Show Detailed Error Messages
**Given** I have enabled debug mode in app settings
**When** I view a failed capture card
**Then** the detailed error message from transcription is displayed
**And** the error includes technical details (e.g., "Whisper error: audio format not supported")
**And** in normal mode (debug disabled), only a generic message is shown: "Transcription failed"

### AC9: Persist Retry Attempts and Timestamps Across App Restarts
**Given** I have retried a capture 2 times within a 20-minute window
**When** I close and reopen the app
**Then** the retry counter is preserved (stored in OP-SQLite)
**And** the 20-minute window countdown continues correctly
**And** the "Retry" button reflects the current state (enabled/disabled)

## Tasks / Subtasks

- [x] **Task 1: Extend Capture Model with Retry Tracking** (AC: 3, 4, 9)
  - [x] Subtask 1.1: Add retry fields to Capture model
    - `retryCount: number` (default 0)
    - `retryWindowStartAt: Date | null` (timestamp of first retry in current window)
    - `lastRetryAt: Date | null` (timestamp of most recent retry)
    - `transcriptionError: string | null` (error message from last failure)
  - [x] Subtask 1.2: Create OP-SQLite migration
    - Add columns: retry_count, retry_window_start_at, last_retry_at, transcription_error
    - Default values: 0, null, null, null
    - Migration version: v14 (schema version updated)
  - [x] Subtask 1.3: Update Capture TypeScript interface
    - Add new fields with proper types
    - Update tests to include new fields

- [x] **Task 2: Implement Retry Rate Limiting Logic** (AC: 3, 4)
  - [x] Subtask 2.1: Create RetryLimitService
    - Method: `canRetry(capture: Capture): RetryCheckResult`
    - Logic: Check if retryCount < 3 within 20-minute window
    - Logic: If window expired (>= 20 min since retryWindowStartAt), allow retry
  - [x] Subtask 2.2: Implement retry window reset
    - Calculate time elapsed since retryWindowStartAt
    - If >= 20 minutes, reset allowed
    - Return true (can retry)
  - [x] Subtask 2.3: Implement countdown timer for disabled retry
    - Calculate remaining time until window expires
    - Format as "Try again in X minutes"
    - Helper method: getRetryStatusMessage()

- [x] **Task 3: Add Retry Button to Capture Card Component** (AC: 1, 2)
  - [x] Subtask 3.1: Update CaptureCard component
    - Identify where "Failed" status badge is shown
    - Add "Retry" button next to or replacing status badge
    - Button only visible if capture.state === 'failed'
  - [x] Subtask 3.2: Style Retry button (Liquid Glass design)
    - Prominent color (e.g., orange or blue, not red for error)
    - Icon: circular arrow or refresh icon
    - Text: "Retry" or just icon (depending on space)
    - Haptic feedback on press
  - [x] Subtask 3.3: Implement onRetry handler
    - Call TranscriptionQueueService.retryFailedByCaptureId(captureId)
    - Update capture state to 'processing'
    - Increment retryCount
    - Update lastRetryAt to current timestamp
    - If retryCount === 1, set retryWindowStartAt to current timestamp

- [x] **Task 4: Disable Retry Button When Limit Reached** (AC: 3)
  - [x] Subtask 4.1: Check retry limit before rendering button
    - Call RetryLimitService.canRetry(capture)
    - If false, disable button (grey out, no interaction)
  - [x] Subtask 4.2: Show countdown message when disabled
    - Display "Retry limit reached. Try again in X minutes"
    - Calculate X from (retryWindowStartAt + 20 minutes) - now
    - Update every minute
  - [x] Subtask 4.3: Auto-enable button when window resets
    - Use useEffect or interval to check if window expired
    - When expired, re-enable button (reactive update)

- [x] **Task 5: Show Progress Indicator During Retry** (AC: 5)
  - [x] Subtask 5.1: Replace "Retry" button with spinner when processing
    - If capture.state === 'processing', show spinner
    - Spinner animates (rotate)
    - Text: "Transcribing..."
  - [x] Subtask 5.2: Use withObservables for reactive updates
    - Capture state changes trigger UI re-render
    - No manual refresh needed (proven pattern from Story 2.5/2.6)

- [x] **Task 6: Handle Retry Success** (AC: 6)
  - [x] Subtask 6.1: Update capture on successful retry
    - TranscriptionWorker marks capture as 'ready'
    - Capture card automatically updates (withObservables)
    - Show brief success animation (green checkmark, fade-in)
  - [x] Subtask 6.2: Remove retry button on success
    - Retry button only shows for state 'failed'
    - On 'ready', show normal transcription UI

- [x] **Task 7: Handle Retry Failure (Attempts Remaining)** (AC: 7)
  - [x] Subtask 7.1: Update capture on failed retry
    - TranscriptionWorker marks capture as 'failed'
    - Increment retryCount
    - Update lastRetryAt
    - Update transcriptionError with latest error message
  - [x] Subtask 7.2: Show retry button again (if attempts remaining)
    - If retryCount < 3, show "Retry" button
    - If retryCount >= 3, disable button + show countdown

- [x] **Task 8: Implement Debug Mode for Error Messages** (AC: 8)
  - [x] Subtask 8.1: Add debug mode setting
    - Create app setting in AsyncStorage: 'debug_mode' (boolean)
    - Add toggle in Settings screen (or developer menu)
  - [x] Subtask 8.2: Conditionally display error messages
    - If debug_mode === true, show capture.transcriptionError (detailed)
    - If debug_mode === false, show generic: "Transcription failed"
  - [x] Subtask 8.3: Style error message display
    - Small text below "Retry" button
    - Red color for error
    - Scrollable if very long (in detail view)

- [ ] **Task 9: Persist Retry Data Across App Restarts** (AC: 9)
  - [ ] Subtask 9.1: Ensure OP-SQLite persists retry fields
    - Verify migration v7 applied successfully
    - Test that retryCount, timestamps persist after app restart
  - [ ] Subtask 9.2: Verify retry window logic after restart
    - Close app mid-retry window
    - Reopen app
    - Countdown continues correctly
    - Button state (enabled/disabled) is accurate

- [ ] **Task 10: Update TranscriptionQueueService for Manual Retry** (AC: 2)
  - [ ] Subtask 10.1: Ensure retryFailedByCaptureId() exists (from Story 2.5)
    - Verify method signature: `retryFailedByCaptureId(captureId: string): Promise<void>`
    - Method should queue capture for transcription
  - [ ] Subtask 10.2: Add retry metadata update
    - Before queuing, update capture retry fields
    - Increment retryCount
    - Update lastRetryAt
    - Set retryWindowStartAt if first retry

- [ ] **Task 11: Write Comprehensive Tests** (AC: All)
  - [ ] Subtask 11.1: Unit tests for RetryLimitService
    - Test canRetry() logic (various scenarios)
    - Test window reset after 20 minutes
    - Test countdown calculation
  - [ ] Subtask 11.2: Component tests for Retry button
    - Test button visibility (only on failed captures)
    - Test button disabled state (when limit reached)
    - Test onRetry handler called
  - [ ] Subtask 11.3: Integration tests for retry flow
    - Test full retry flow: tap button → transcription → success/failure
    - Test rate limiting (3 retries → disabled → wait 20 min → enabled)
    - Test state persistence across app restart
  - [ ] Subtask 11.4: BDD tests (Gherkin) for all ACs
    - AC1: Display retry button
    - AC2: Trigger manual retry
    - AC3: Rate limiting (3 retries / 20 min)
    - AC4: Reset counter after 20 min
    - AC5: Progress indicator
    - AC6: Success handling
    - AC7: Failure handling (attempts remaining)
    - AC8: Debug mode error messages
    - AC9: Persistence across restarts

## Dev Notes

### Architecture Context

**Bounded Context:** Normalization Context (Domain Service layer) + Capture Context (UI layer)

**New Components:**
- `RetryLimitService` (singleton service, retry rate limiting logic)
- Debug mode setting (AsyncStorage)

**Modified Components:**
- `Capture` model (add retry fields)
- `CaptureCard` component (add Retry button)
- `TranscriptionQueueService` (update retry metadata)
- `TranscriptionWorker` (capture transcriptionError on failure)
- `CaptureDetailScreen` (show detailed error in debug mode)

**Data Flow:**
```
User taps "Retry" button on failed capture card
    ↓
RetryLimitService.canRetry(capture) → check rate limit
    ↓
If allowed → TranscriptionQueueService.retryFailedByCaptureId(captureId)
    ↓
Update capture: retryCount++, lastRetryAt = now, state = 'processing'
    ↓
TranscriptionWorker processes → success or failure
    ↓
Update capture state → UI updates reactively (withObservables)
```

### Technical Stack

**Already installed:**
- React Native (Button, TouchableOpacity, ActivityIndicator)
- OP-SQLite (ADR-018)
- AsyncStorage (for debug mode setting)
- Whisper.rn (from Story 2.5)

**No new dependencies required.**

### Capture Model Extensions

**New Fields:**
```typescript
interface Capture {
  // ... existing fields from Story 2.1-2.6 ...

  // NEW for Story 2.8
  retryCount: number                // Number of retries in current 20-min window
  retryWindowStartAt: Date | null   // Timestamp of first retry in window
  lastRetryAt: Date | null          // Timestamp of most recent retry
  transcriptionError: string | null // Detailed error message from last failure
}
```

**Migration v7:**
```sql
-- Add retry tracking columns
ALTER TABLE captures ADD COLUMN retry_count INTEGER DEFAULT 0;
ALTER TABLE captures ADD COLUMN retry_window_start_at INTEGER; -- Unix timestamp (ms)
ALTER TABLE captures ADD COLUMN last_retry_at INTEGER;         -- Unix timestamp (ms)
ALTER TABLE captures ADD COLUMN transcription_error TEXT;
```

### Retry Rate Limiting Logic

**RetryLimitService:**
```typescript
class RetryLimitService {
  private readonly RETRY_LIMIT = 3
  private readonly WINDOW_DURATION_MS = 20 * 60 * 1000 // 20 minutes

  canRetry(capture: Capture): { allowed: boolean; remainingTime?: number } {
    // No window started yet → allow retry
    if (!capture.retryWindowStartAt) {
      return { allowed: true }
    }

    const now = Date.now()
    const windowStart = capture.retryWindowStartAt.getTime()
    const elapsed = now - windowStart

    // Window expired (> 20 min) → reset and allow
    if (elapsed > this.WINDOW_DURATION_MS) {
      return { allowed: true }
    }

    // Within window → check count
    if (capture.retryCount < this.RETRY_LIMIT) {
      return { allowed: true }
    }

    // Limit reached within window → deny + remaining time
    const remainingMs = this.WINDOW_DURATION_MS - elapsed
    return { allowed: false, remainingTime: Math.ceil(remainingMs / 1000 / 60) } // minutes
  }

  resetRetryWindow(capture: Capture): void {
    capture.retryCount = 0
    capture.retryWindowStartAt = null
    capture.lastRetryAt = null
  }
}
```

### Retry Button UI Component

**CaptureCard.tsx Enhancement:**
```typescript
const CaptureCard = ({ capture }) => {
  const retryLimitService = useMemo(() => new RetryLimitService(), [])
  const { allowed, remainingTime } = retryLimitService.canRetry(capture)

  const handleRetry = async () => {
    if (!allowed) return

    // Update retry metadata
    await database.write(async () => {
      await capture.update(c => {
        c.retryCount += 1
        c.lastRetryAt = new Date()
        if (c.retryCount === 1) {
          c.retryWindowStartAt = new Date()
        }
        c.state = 'processing'
      })
    })

    // Queue for transcription
    await transcriptionQueueService.retryFailedByCaptureId(capture.id)
  }

  return (
    <View>
      {/* Existing card content */}

      {capture.state === 'failed' && (
        <View>
          {allowed ? (
            <TouchableOpacity onPress={handleRetry}>
              <Text>🔄 Retry</Text>
            </TouchableOpacity>
          ) : (
            <View>
              <Text disabled>🔄 Retry (limit reached)</Text>
              <Text small>Try again in {remainingTime} minutes</Text>
            </View>
          )}
        </View>
      )}

      {capture.state === 'processing' && (
        <ActivityIndicator />
        <Text>Transcribing...</Text>
      )}
    </View>
  )
}

export default withObservables(['capture'], ({ capture }) => ({
  capture: capture.observe()
}))(CaptureCard)
```

### Debug Mode Implementation

**AsyncStorage Setting:**
```typescript
// Enable/disable debug mode
await AsyncStorage.setItem('debug_mode', 'true')
const debugMode = await AsyncStorage.getItem('debug_mode') === 'true'
```

**Error Message Display:**
```typescript
const ErrorMessage = ({ capture, debugMode }) => {
  if (capture.state !== 'failed') return null

  const message = debugMode
    ? capture.transcriptionError || 'Unknown error'
    : 'Transcription failed'

  return <Text style={{ color: 'red', fontSize: 12 }}>{message}</Text>
}
```

### Integration with Story 2.5 (TranscriptionWorker)

**Current TranscriptionWorker (from 2.5):**
```typescript
class TranscriptionWorker {
  async processCapture(captureId: string) {
    try {
      const transcription = await transcriptionService.transcribe(...)
      await this.markCaptureAsReady(captureId, transcription)
    } catch (error) {
      await this.markCaptureAsFailed(captureId, error.message)
    }
  }
}
```

**Enhanced for Story 2.8 (capture error message):**
```typescript
class TranscriptionWorker {
  async processCapture(captureId: string) {
    try {
      const transcription = await transcriptionService.transcribe(...)
      await this.markCaptureAsReady(captureId, transcription)
      // Reset retry counter on success
      await this.resetRetryData(captureId)
    } catch (error) {
      // NEW: Capture error message for debug mode
      await this.markCaptureAsFailed(captureId, error.message)
    }
  }

  private async markCaptureAsFailed(captureId: string, errorMessage: string) {
    await database.write(async () => {
      const capture = await database.collections.get('captures').find(captureId)
      await capture.update(c => {
        c.state = 'failed'
        c.transcriptionError = errorMessage // NEW for Story 2.8
      })
    })
  }

  private async resetRetryData(captureId: string) {
    await database.write(async () => {
      const capture = await database.collections.get('captures').find(captureId)
      await capture.update(c => {
        c.retryCount = 0
        c.retryWindowStartAt = null
        c.lastRetryAt = null
        c.transcriptionError = null
      })
    })
  }
}
```

### Real-Time Countdown for Disabled Retry Button

**useEffect for countdown timer:**
```typescript
const [remainingMinutes, setRemainingMinutes] = useState<number | null>(null)

useEffect(() => {
  if (!capture.retryWindowStartAt || capture.retryCount < 3) {
    setRemainingMinutes(null)
    return
  }

  const updateCountdown = () => {
    const now = Date.now()
    const windowEnd = capture.retryWindowStartAt.getTime() + (20 * 60 * 1000)
    const remaining = Math.max(0, Math.ceil((windowEnd - now) / 1000 / 60))

    if (remaining === 0) {
      // Window expired, reset
      setRemainingMinutes(null)
    } else {
      setRemainingMinutes(remaining)
    }
  }

  updateCountdown()
  const interval = setInterval(updateCountdown, 60000) // Update every minute

  return () => clearInterval(interval)
}, [capture.retryWindowStartAt, capture.retryCount])
```

### File Structure

```
pensieve/mobile/
├── src/
│   ├── services/
│   │   └── RetryLimitService.ts                # NEW - Rate limiting logic
│   ├── components/
│   │   └── captures/
│   │       └── CaptureCard.tsx                 # MODIFY - Add Retry button
│   ├── screens/
│   │   └── captures/
│   │       └── CaptureDetailScreen.tsx         # MODIFY - Show debug error
│   ├── models/
│   │   └── Capture.model.ts                    # MODIFY - Add retry fields
│   │   └── migrations.ts                       # MODIFY - Add migration v7
│   ├── contexts/
│   │   └── Normalization/
│   │       └── services/
│   │           ├── TranscriptionQueueService.ts # MODIFY - Update retry metadata
│   │           └── TranscriptionWorker.ts       # MODIFY - Capture error message
│   └── settings/
│       └── DebugSettings.tsx                   # NEW (optional) - Toggle debug mode
```

**Key Files to Modify:**
- `src/models/Capture.model.ts` - Add retry fields
- `src/models/migrations.ts` - Add migration v7
- `src/components/captures/CaptureCard.tsx` - Add Retry button
- `src/contexts/Normalization/services/TranscriptionWorker.ts` - Capture transcriptionError
- `src/screens/captures/CaptureDetailScreen.tsx` - Show error in debug mode

### Testing Standards

- **Unit tests:**
  - RetryLimitService.canRetry() (various scenarios)
  - Window reset logic
  - Countdown calculation
- **Component tests:**
  - Retry button visibility (only on failed captures)
  - Button disabled state (limit reached)
  - Countdown message display
  - onRetry handler
- **Integration tests:**
  - Full retry flow (success scenario)
  - Full retry flow (failure scenario)
  - Rate limiting (3 retries → disabled → wait 20 min → enabled)
  - State persistence across app restart
  - Debug mode error display
- **BDD tests (Gherkin):** All 9 Acceptance Criteria
- **Edge case tests:**
  - Retry at exactly 20 minutes (boundary)
  - Multiple captures failing simultaneously
  - App backgrounded during retry countdown

### Critical Implementation Notes

**DO:**
- ✅ Add retry fields to Capture model (migration v7)
- ✅ Implement RetryLimitService with 20-minute window logic
- ✅ Show "Retry" button ONLY on failed captures
- ✅ Disable button when limit reached (3 retries)
- ✅ Display countdown "Try again in X minutes"
- ✅ Update retry metadata BEFORE queuing retry
- ✅ Capture transcriptionError in TranscriptionWorker
- ✅ Show detailed error only in debug mode
- ✅ Reset retry data on successful transcription
- ✅ Use withObservables for reactive UI updates
- ✅ Persist retry data in OP-SQLite (survives app restart)
- ✅ Follow Liquid Glass design (animations, haptics)

**DON'T:**
- ❌ Allow unlimited retries (enforce 3 per 20 min)
- ❌ Reset retry counter on app restart (must persist)
- ❌ Show detailed errors in production (debug mode only)
- ❌ Block UI during retry (use reactive updates)
- ❌ Forget to update retryWindowStartAt on first retry
- ❌ Allow retry when state is 'processing' (race condition)
- ❌ Hardcode retry limit or window duration (use constants)
- ❌ Skip migration v7 (retry fields required in schema)

### UX Requirements (Liquid Glass Design System)

**Retry Button:**
- Color: Orange or blue (not red - red is for error/danger)
- Icon: Circular arrow (🔄) or refresh icon
- Size: Prominent but not overwhelming
- Position: Next to "Failed" badge or replacing it
- Haptic: Medium impact on press
- Animation: Subtle scale/bounce on press

**Disabled State:**
- Color: Grey (#999)
- Cursor: Not allowed
- Opacity: 0.5
- Countdown text: Small, below button

**Progress Indicator:**
- Spinner: Smooth rotation animation
- Text: "Transcribing..." (consistent with Story 2.5)
- Position: Replace "Retry" button temporarily

**Success Animation:**
- Green checkmark (✓) fade-in
- Duration: 500ms
- Haptic: Success notification (light)

### Retry Window Logic - Detailed Scenarios

**Scenario 1: First retry (fresh start)**
```
Initial state:
  retryCount = 0
  retryWindowStartAt = null
  lastRetryAt = null

After first retry:
  retryCount = 1
  retryWindowStartAt = now
  lastRetryAt = now
```

**Scenario 2: Second retry (within window)**
```
Before retry:
  retryCount = 1
  retryWindowStartAt = T0
  lastRetryAt = T1

After second retry (T1 + 5 min):
  retryCount = 2
  retryWindowStartAt = T0 (unchanged)
  lastRetryAt = now
```

**Scenario 3: Third retry (limit reached)**
```
After third retry:
  retryCount = 3
  retryWindowStartAt = T0
  lastRetryAt = now

UI: Button disabled, countdown shown
Countdown: (T0 + 20 min) - now
```

**Scenario 4: Window expired (reset)**
```
After 20+ minutes:
  RetryLimitService.canRetry() returns { allowed: true }

On next retry:
  retryCount = 1 (reset)
  retryWindowStartAt = now (new window)
  lastRetryAt = now
```

### Debug Mode Toggle Implementation

**Settings Screen (optional):**
```typescript
const DebugSettings = () => {
  const [debugMode, setDebugMode] = useState(false)

  useEffect(() => {
    AsyncStorage.getItem('debug_mode').then(value => {
      setDebugMode(value === 'true')
    })
  }, [])

  const toggleDebugMode = async () => {
    const newValue = !debugMode
    setDebugMode(newValue)
    await AsyncStorage.setItem('debug_mode', newValue.toString())
  }

  return (
    <View>
      <Text>Developer Options</Text>
      <Switch value={debugMode} onValueChange={toggleDebugMode} />
      <Text>Show detailed error messages</Text>
    </View>
  )
}
```

**Alternative: Hidden debug menu (shake gesture or triple-tap)**
```typescript
// Triple-tap on logo to toggle debug mode
let tapCount = 0
let tapTimer: NodeJS.Timeout

const handleLogoPress = async () => {
  tapCount++

  if (tapCount === 3) {
    const current = await AsyncStorage.getItem('debug_mode')
    await AsyncStorage.setItem('debug_mode', current === 'true' ? 'false' : 'true')
    Alert.alert('Debug mode toggled')
    tapCount = 0
  }

  clearTimeout(tapTimer)
  tapTimer = setTimeout(() => { tapCount = 0 }, 1000)
}
```

### Previous Story Intelligence (Stories 2.5 & 2.6)

**Story 2.5 - Transcription On-Device avec Whisper:**
- TranscriptionService, TranscriptionQueueService, TranscriptionWorker fully implemented
- Retry logic exists: `TranscriptionQueueService.retryFailedByCaptureId(captureId)`
- Capture states: 'captured', 'processing', 'ready', 'failed'
- 37 tests passing (unit + integration + performance + edge cases)
- withObservables pattern for reactive UI proven

**Story 2.6 - Consultation de Transcription:**
- CaptureDetailScreen displays transcription
- Status badges implemented (⏳ pending, spinner processing, ✅ ready, ❌ failed)
- "Retry Transcription" button in detail view (Story 2.8 adds it to feed cards too)
- 26 BDD tests passing

**Story 2.8 builds on 2.5/2.6:**
- Reuses TranscriptionQueueService.retryFailedByCaptureId() (don't reimplement)
- Extends Capture model (migration v7)
- Adds rate limiting (new requirement not in 2.5/2.6)
- Adds debug mode (new requirement)
- Adds retry button to FEED cards (2.6 only had it in detail view)

**Code Reuse from 2.5/2.6:**
- TranscriptionQueueService.retryFailedByCaptureId() method (already exists)
- withObservables pattern for reactive updates (proven)
- Capture state machine (captured → processing → ready/failed)
- Status badge UI patterns (adapt for retry button)

### Potential Edge Cases to Test

1. **Rapid retry taps:** User taps "Retry" multiple times quickly → debounce or disable after first tap
2. **App backgrounded during countdown:** Countdown continues correctly when app resumed
3. **Multiple captures fail simultaneously:** Each has independent retry counter
4. **Retry window boundary (exactly 20 min):** Test reset logic at exact boundary
5. **Network offline during retry:** Transcription fails again, retryCount increments
6. **Model not available during retry:** Handle MODEL_NOT_AVAILABLE error (from Story 2.7)
7. **Capture deleted during retry:** Handle gracefully (capture no longer exists)
8. **Debug mode toggled mid-error:** Error message updates reactively

### Integration with Story 2.7 (Model Configuration)

**If model not available:**
```typescript
// In TranscriptionWorker
try {
  const transcription = await transcriptionService.transcribe(...)
} catch (error) {
  if (error.message === 'MODEL_NOT_AVAILABLE') {
    // Don't increment retry count for model issues
    // Just keep state 'captured', don't mark as 'failed'
    return
  }

  // Other errors (corrupted audio, Whisper crash, etc.)
  await this.markCaptureAsFailed(captureId, error.message)
}
```

**Retry button should NOT appear if model is missing** (different error flow).

### References

- [Source: GitHub Issue #1](https://github.com/yohikofox/pensieve/issues/1)
- [Source: _bmad-output/planning-artifacts/epics.md#Epic 2: Story 2.5 - Transcription On-Device]
- [Source: _bmad-output/planning-artifacts/epics.md#Epic 2: Story 2.6 - Consultation de Transcription]
- [Source: _bmad-output/planning-artifacts/architecture.md#ADR-018 - OP-SQLite Migration]
- [Source: _bmad-output/implementation-artifacts/2-5-transcription-on-device-avec-whisper.md - TranscriptionQueueService.retryFailedByCaptureId()]
- [Source: _bmad-output/implementation-artifacts/2-6-consultation-de-transcription.md - CaptureDetailScreen status badges]
- [Source: _bmad-output/implementation-artifacts/2-7-guide-config-modele.md - Model availability checks]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5

### Debug Log References

N/A - Story not yet implemented

### Completion Notes List

**2026-01-31 - Tasks 1-2 Completed:**
- ✅ Created migration v14 with retry tracking columns (retry_count, retry_window_start_at, last_retry_at, transcription_error)
- ✅ Updated SCHEMA_VERSION to 14
- ✅ Extended Capture model interface with retry fields
- ✅ Extended CaptureRow interface with retry columns
- ✅ Updated mapRowToCapture to map retry fields
- ✅ Implemented RetryLimitService with 20-minute sliding window logic
- ✅ All unit tests passing (13 tests: 9 RetryLimitService + 4 Capture model)
- 📝 Note: Migration uses v14 (not v7 as initially mentioned in story - v13 was latest)

**2026-01-31 - Tasks 3-4 Completed:**
- ✅ Integrated RetryLimitService into CapturesListScreen
- ✅ Modified handleRetry to check rate limit before retrying (returns early with toast if limit reached)
- ✅ Updated retry button UI to show disabled state when 3 attempts exhausted
- ✅ Added countdown message ("Limite atteinte. X min") below button when disabled
- ✅ Changed button color from error-500 (red) to warning-500 (orange) when enabled
- ✅ Button shows grey (neutral-300) when disabled with no interaction
- ✅ Created 7 integration tests in CapturesListScreen.retry.test.tsx
- ✅ All 20 retry-related tests passing (7 integration + 9 RetryLimitService + 4 Capture model)
- ✅ Tested on Android Pixel 10 Pro emulator - app runs without errors
- 📝 Note: Button automatically re-enables when window resets (reactive via RetryLimitService.canRetry check)

**2026-01-31 - Task 5 Completed (Already Implemented in Stories 2.5/2.6):**
- ✅ Processing indicator (ActivityIndicator spinner) already shows when state === 'processing'
- ✅ Badge variant="processing" with spinner and text displayed
- ✅ Text shows t('capture.status.processing') - translates to "Transcribing..." or "En cours..."
- ✅ Retry button automatically hidden when state is 'processing' (only shows when state === 'failed')
- ✅ Reactive updates via capture state changes trigger UI re-render
- ✅ Created 5 tests in CapturesListScreen.processing.test.tsx to document behavior
- ✅ All 39 retry/processing tests passing
- 📝 Note: No code changes needed - Task 5 requirements already satisfied by existing implementation

**2026-01-31 - Tasks 6-7 Completed:**
- ✅ TranscriptionQueueService.retryFailedByCaptureId() updates captures table retry metadata
- ✅ TranscriptionWorker success handler clears transcriptionError on success (Task 6)
- ✅ TranscriptionWorker failure handler updates retry metadata on failure (Task 7)
- ✅ Retry metadata synchronized between transcription_queue and captures tables
- ✅ On success: state='ready', normalizedText filled, transcriptionError=null
- ✅ On failure: state='failed', retryCount++, lastRetryAt updated, transcriptionError stored
- ✅ retryWindowStartAt set on first retry, preserved on subsequent retries
- ✅ Both foreground and background transcription handlers updated
- ✅ Created 7 tests in TranscriptionWorker.retry.test.ts
- ✅ All 46 retry/processing tests passing (39 + 7)
- 📝 Note: Retry button visibility is automatic (only shows when state === 'failed', reactive UI)

**2026-01-31 - Task 8 Completed:**
- ✅ Debug mode already exists in useSettingsStore (no changes needed)
- ✅ Simplified error display to use item.transcriptionError directly from Capture model
- ✅ Removed errorMessages state (no longer needed)
- ✅ Removed fetchErrorMessages useEffect (error now in Capture model from Tasks 6-7)
- ✅ If debugMode=true AND transcriptionError exists: show detailed error
- ✅ Otherwise: show generic "La transcription a échoué"
- ✅ Error displayed with red color (text-status-error), small font (text-sm), italic
- ✅ Created 5 tests in CapturesListScreen.debug.test.tsx
- ✅ All 51 retry/processing/debug tests passing (46 + 5)
- 📝 Note: Debug mode toggle already implemented in Settings screen (useSettingsStore)

### File List

**Modified:**
- `pensieve/mobile/src/database/schema.ts` - Updated SCHEMA_VERSION to 14
- `pensieve/mobile/src/database/migrations.ts` - Added migration v14 with retry columns
- `pensieve/mobile/src/contexts/capture/domain/Capture.model.ts` - Added retry fields to interfaces
- `pensieve/mobile/src/screens/captures/CapturesListScreen.tsx` - Integrated RetryLimitService, retry button UI, debug mode error display
- `pensieve/mobile/src/contexts/Normalization/services/TranscriptionQueueService.ts` - Update captures retry metadata on manual retry
- `pensieve/mobile/src/contexts/Normalization/workers/TranscriptionWorker.ts` - Update retry metadata on success/failure
- `_bmad-output/implementation-artifacts/2-8-bouton-retry-transcription.md` - Marked Tasks 1-8 complete

**Added:**
- `pensieve/mobile/src/contexts/Normalization/services/RetryLimitService.ts` - Rate limiting service
- `pensieve/mobile/src/contexts/Normalization/services/__tests__/RetryLimitService.test.ts` - Unit tests (9 tests)
- `pensieve/mobile/src/contexts/capture/domain/__tests__/Capture.retry.test.ts` - Model tests (4 tests)
- `pensieve/mobile/src/screens/captures/__tests__/CapturesListScreen.retry.test.tsx` - Integration tests (7 tests)
- `pensieve/mobile/src/screens/captures/__tests__/CapturesListScreen.processing.test.tsx` - Processing indicator tests (5 tests)
- `pensieve/mobile/src/contexts/Normalization/__tests__/TranscriptionWorker.retry.test.ts` - Worker retry metadata tests (7 tests)
- `pensieve/mobile/src/screens/captures/__tests__/CapturesListScreen.debug.test.tsx` - Debug mode error display tests (5 tests)

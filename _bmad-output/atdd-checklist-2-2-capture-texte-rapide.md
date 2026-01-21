# ATDD Checklist: Story 2.2 - Capture Texte Rapide

**Date:** 2026-01-21
**Status:** 🔴 RED Phase (Tests Failing)
**Story File:** `_bmad-output/implementation-artifacts/2-2-capture-texte-rapide.md`

---

## Story Summary

**As a** user
**I want** to capture a text thought via quick text input
**So that** I can record ideas that are better suited to typing than speaking

---

## Acceptance Criteria Breakdown

### ✅ AC1: Open Text Input Field Immediately

**Requirements:**
- Text input field appears immediately
- Keyboard opens automatically
- Cursor focused in text field

**Test Coverage:**
- ✅ E2E: `text-capture.e2e.ts` - "should display text input field when text capture button tapped"
- ✅ E2E: `text-capture.e2e.ts` - "should open keyboard automatically"
- ✅ Component: `text-capture-input.test.tsx` - "should auto-focus text input on mount"

### ✅ AC2: Save Text Capture with Metadata

**Requirements:**
- Capture entity created with type "text"
- Text content stored
- Timestamp recorded
- Status set to "captured"
- Input field cleared after save

**Test Coverage:**
- ✅ E2E: `text-capture.e2e.ts` - "should save text capture successfully"
- ✅ E2E: `text-capture.e2e.ts` - "should clear text input after successful save"
- ✅ Integration: `text-capture-service.test.ts` - "should create Capture entity with type 'text'"
- ✅ Integration: `text-capture-service.test.ts` - "should record timestamp when capture is created"

### ✅ AC3: Cancel Unsaved Text with Confirmation

**Requirements:**
- Prompt for confirmation when discarding unsaved content
- Capture not saved if user confirms discard
- No prompt when text is empty

**Test Coverage:**
- ✅ E2E: `text-capture.e2e.ts` - "should prompt for confirmation when discarding unsaved text"
- ✅ E2E: `text-capture.e2e.ts` - "should discard text when user confirms"
- ✅ Component: `text-capture-input.test.tsx` - "should show confirmation dialog when canceling with unsaved text"

### ✅ AC4: Offline Text Capture Functionality

**Requirements:**
- Works in offline mode (FR4 compliance)
- Capture marked for future sync
- No error shown to user

**Test Coverage:**
- ✅ E2E: `text-capture.e2e.ts` - "should save text capture offline without errors"
- ✅ Integration: `text-capture-service.test.ts` - "should work without network connectivity"
- ✅ Integration: `text-capture-service.test.ts` - "should mark Capture as pending sync"

### ✅ AC5: Empty Text Validation

**Requirements:**
- Validation message for empty text
- No empty Capture entity created
- Save button disabled when text empty

**Test Coverage:**
- ✅ E2E: `text-capture.e2e.ts` - "should show validation error for empty text"
- ✅ E2E: `text-capture.e2e.ts` - "should disable save button when text is empty"
- ✅ Integration: `text-capture-service.test.ts` - "should reject empty text"
- ✅ Component: `text-capture-input.test.tsx` - "should disable save button when text is empty"

### ✅ AC6: Haptic Feedback on Save

**Requirements:**
- Haptic feedback confirms save
- Brief animation shows capture added to feed

**Test Coverage:**
- ✅ E2E: `text-capture.e2e.ts` - "should show save animation"
- ✅ Component: `text-capture-input.test.tsx` - "should trigger haptic feedback when save succeeds"

---

## Test Files Created

### E2E Tests (Detox)
```
pensieve/mobile/e2e/capture/text-capture.e2e.ts
```
- **Test Count**: 18 tests
- **Run Command**: `detox test --configuration ios.sim.debug e2e/capture/text-capture.e2e.ts`
- **Status**: 🔴 All failing (waiting for implementation)

### Integration Tests (Jest)
```
pensieve/mobile/tests/acceptance/capture/text-capture-service.test.ts
```
- **Test Count**: 17 tests
- **Run Command**: `npm run test:acceptance -- text-capture-service.test.ts`
- **Status**: 🔴 All failing (waiting for implementation)

### Component Tests (Jest + React Testing Library)
```
pensieve/mobile/tests/acceptance/capture/text-capture-input.test.tsx
```
- **Test Count**: 20 tests
- **Run Command**: `npm run test:acceptance -- text-capture-input.test.tsx`
- **Status**: 🔴 All failing (waiting for implementation)

**Total Tests**: **55 tests** en phase RED 🔴

---

## Required data-testid Attributes

### Main Screen
- `main-screen` - Main screen container
- `text-capture-button` - Text capture button (tap to open input)

### Text Input Modal/Screen
- `text-input-field` - Multiline text input
- `save-text-button` - Save button (enabled when text not empty)
- `cancel-text-button` - Cancel/close button

### Status Indicators
- `text-saved-indicator` - Success indicator after save
- `save-animation` - Save animation element
- `sync-pending-indicator` - Shows when capture awaiting sync
- `error-message` - General error message container

### Capture Feed
- `capture-feed` - Feed/list of captures
- `close-text-capture` - Button to close text input and return to feed

---

## Implementation Checklist

### 🔴 RED Phase Complete

- [x] All tests written (55 tests total)
- [x] Tests failing for right reason (missing implementation)
- [x] Component tests for UI behavior
- [x] Integration tests for service logic
- [x] E2E tests for user workflows
- [x] ATDD checklist generated

### 🟢 GREEN Phase (Implementation Tasks)

#### Task 1: Create Text Capture UI Component

- [ ] **Subtask 1.1**: Design TextCaptureInput component
  - [ ] Create `src/contexts/Capture/ui/TextCaptureInput.tsx`
  - [ ] Implement multiline TextInput with auto-focus
  - [ ] Auto-open keyboard on mount
  - [ ] Implement max height (5 lines) with scroll
  - [ ] Add data-testid attributes
  - [ ] Run tests: `npm run test:acceptance -- text-capture-input.test.tsx`
  - [ ] ✅ Component tests pass (green)

- [ ] **Subtask 1.2**: Add text capture button to main screen
  - [ ] Add text-capture-button to main screen
  - [ ] Use keyboard/text icon
  - [ ] Ensure 44x44 tap target
  - [ ] Run tests: `detox test -- text-capture.e2e.ts` (AC1 tests)
  - [ ] ✅ E2E tests pass (green)

- [ ] **Subtask 1.3**: Implement validation UI
  - [ ] Show inline error for empty submissions
  - [ ] Disable save button when text empty
  - [ ] Clear error on user input
  - [ ] Run tests: `npm run test:acceptance -- text-capture-input.test.tsx` (AC5 tests)
  - [ ] ✅ Tests pass (green)

- [ ] **Subtask 1.4**: Add haptic feedback on save
  - [ ] Use expo-haptics (already installed from Story 2.1)
  - [ ] Implement subtle save animation (fade/slide)
  - [ ] Show "Saved" confirmation toast
  - [ ] Run tests: `npm run test:acceptance -- text-capture-input.test.tsx` (AC6 tests)
  - [ ] ✅ Tests pass (green)

#### Task 2: Implement Text Capture Persistence

- [ ] **Subtask 2.1**: Create TextCaptureService
  - [ ] Create `src/contexts/Capture/services/TextCaptureService.ts`
  - [ ] Implement `saveTextCapture(text: string)` method
  - [ ] Validate non-empty content (trim whitespace)
  - [ ] Generate capture metadata (timestamp, type)
  - [ ] Run tests: `npm run test:acceptance -- text-capture-service.test.ts`
  - [ ] ✅ Integration tests pass (green)

- [ ] **Subtask 2.2**: Store text in Capture entity
  - [ ] Reuse Capture model from Story 2.1
  - [ ] Set type='text'
  - [ ] Store in rawContent field
  - [ ] Set normalizedText = rawContent
  - [ ] Mark syncStatus as 'pending'
  - [ ] Run tests: `npm run test:acceptance -- text-capture-service.test.ts` (AC2, AC4 tests)
  - [ ] ✅ Tests pass (green)

- [ ] **Subtask 2.3**: Clear input after successful save
  - [ ] Reset TextInput state
  - [ ] Return focus to input
  - [ ] Show visual confirmation
  - [ ] Run tests: `detox test -- text-capture.e2e.ts` (AC2 tests)
  - [ ] ✅ E2E tests pass (green)

#### Task 3: Implement Cancel/Discard Flow

- [ ] **Subtask 3.1**: Detect unsaved content
  - [ ] Track text input state (empty vs has content)
  - [ ] Monitor navigation events (back button, tab switch)
  - [ ] Run tests: `npm run test:acceptance -- text-capture-input.test.tsx` (AC3 tests)
  - [ ] ✅ Tests pass (green)

- [ ] **Subtask 3.2**: Show discard confirmation dialog
  - [ ] Use React Native Alert API
  - [ ] Display: "Discard unsaved text?"
  - [ ] Options: "Discard" | "Keep Editing"
  - [ ] Only show if text non-empty
  - [ ] Run tests: `detox test -- text-capture.e2e.ts` (AC3 tests)
  - [ ] ✅ E2E tests pass (green)

- [ ] **Subtask 3.3**: Handle discard action
  - [ ] Clear text if user confirms
  - [ ] Close text input modal
  - [ ] Do not create Capture entity
  - [ ] Run tests: Full AC3 test suite
  - [ ] ✅ All AC3 tests pass (green)

#### Task 4: Offline Mode Support

- [ ] **Subtask 4.1**: Test offline text capture
  - [ ] Ensure WatermelonDB Capture entity created
  - [ ] Verify syncStatus set to 'pending'
  - [ ] Confirm no network calls attempted
  - [ ] Run tests: `npm run test:acceptance -- text-capture-service.test.ts` (AC4 tests)
  - [ ] ✅ Tests pass (green)

- [ ] **Subtask 4.2**: Queue for sync when online
  - [ ] Reuse sync queue from Story 2.1
  - [ ] Prioritize text captures equally with audio
  - [ ] Run tests: `detox test -- text-capture.e2e.ts` (AC4 tests)
  - [ ] ✅ E2E tests pass (green)

#### Task 5: Final Verification

- [ ] **Run all E2E tests**
  - [ ] iOS: `detox test --configuration ios.sim.debug -- text-capture.e2e.ts`
  - [ ] Android: `detox test --configuration android.emu.debug -- text-capture.e2e.ts`
  - [ ] ✅ All 18 E2E tests pass (green)

- [ ] **Run all integration tests**
  - [ ] `npm run test:acceptance -- text-capture-service.test.ts`
  - [ ] ✅ All 17 integration tests pass (green)

- [ ] **Run all component tests**
  - [ ] `npm run test:acceptance -- text-capture-input.test.tsx`
  - [ ] ✅ All 20 component tests pass (green)

- [ ] **UX Verification**
  - [ ] Keyboard opens < 300ms
  - [ ] Haptic feedback feels natural
  - [ ] Save animation smooth (60fps)
  - [ ] Multi-line text handling works

### ♻️ REFACTOR Phase (Post-Implementation)

- [ ] Extract common UI patterns (buttons, animations)
- [ ] Optimize text validation logic
- [ ] Add error handling edge cases
- [ ] Improve accessibility (VoiceOver/TalkBack)
- [ ] Ensure all tests still pass after refactoring

---

## Red-Green-Refactor Workflow

### 🔴 RED Phase (Complete)

**Status**: ✅ All tests written and failing

**What we have:**
- 18 E2E tests (Detox)
- 17 Integration tests (Jest)
- 20 Component tests (React Testing Library)
- Required data-testid attributes documented

**Next Step**: Start GREEN phase implementation

### 🟢 GREEN Phase (DEV Team)

**Approach**: Implement minimal code to make tests pass

1. Pick one failing test from checklist
2. Implement minimal code to make it pass
3. Run test to verify green
4. Move to next test
5. Repeat until all tests pass

**Suggested Order:**
1. Start with Component tests (TextCaptureInput UI)
2. Then Integration tests (TextCaptureService logic)
3. Finally E2E tests (full workflows)

### ♻️ REFACTOR Phase (DEV Team)

**When**: After all 55 tests passing (green)

**Activities:**
1. Extract duplications
2. Improve code quality
3. Optimize animations
4. Add documentation
5. Ensure tests still pass

---

## Running Tests

### E2E Tests (Detox)

```bash
# Run text capture E2E tests
detox test --configuration ios.sim.debug e2e/capture/text-capture.e2e.ts

# Run specific test
detox test --configuration ios.sim.debug e2e/capture/text-capture.e2e.ts -t "should save text capture successfully"

# Android
detox test --configuration android.emu.debug e2e/capture/text-capture.e2e.ts
```

### Integration Tests (Jest)

```bash
# Run TextCaptureService tests
npm run test:acceptance -- text-capture-service.test.ts

# Watch mode
npm run test:acceptance -- text-capture-service.test.ts --watch

# With coverage
npm run test:acceptance -- text-capture-service.test.ts --coverage
```

### Component Tests (Jest + React Testing Library)

```bash
# Run TextCaptureInput component tests
npm run test:acceptance -- text-capture-input.test.tsx

# Watch mode
npm run test:acceptance -- text-capture-input.test.tsx --watch

# Debug specific test
npm run test:acceptance -- text-capture-input.test.tsx -t "should auto-focus"
```

---

## Reuse from Story 2.1

### Already Implemented Components

**From Story 2.1:**
- ✅ Capture WatermelonDB model (polymorphic type support)
- ✅ Capture Context file structure
- ✅ Sync status tracking (syncStatus field)
- ✅ Offline queue mechanism
- ✅ expo-haptics integration
- ✅ Capture factory for test data

**Reusable Patterns:**
- Capture entity creation flow
- Haptic feedback on save
- Sync queue integration
- Offline-first persistence

### New Components to Create

**Story 2.2 Specific:**
- 🆕 TextCaptureInput.tsx (UI component)
- 🆕 TextCaptureService.ts (business logic)
- 🆕 Text validation logic
- 🆕 Cancel/discard confirmation flow

---

## Dependencies & Libraries

### Already Installed (Story 2.1)
- ✅ `@nozbe/watermelondb` - Offline database
- ✅ `expo-haptics` - Haptic feedback
- ✅ React Native core (TextInput, Keyboard, Alert)

### No New Dependencies Required

**Built-in React Native APIs:**
- `TextInput` - Multiline text input
- `Keyboard` - Keyboard management
- `Alert` - Confirmation dialogs

---

## Knowledge Base References Applied

- ✅ **data-factories.md**: Reuse Capture factory from Story 2.1
- ✅ **component-tdd.md**: Component test strategies for React Native
- ✅ **test-quality.md**: Deterministic tests, cleanup, explicit assertions
- ✅ **test-levels-framework.md**: E2E + Integration + Component test split
- ✅ **selector-resilience.md**: data-testid selector strategy

---

## Next Steps for DEV Team

1. **Review this checklist** and Story 2.2 file
2. **Run failing tests** to confirm RED phase:
   ```bash
   npm run test:acceptance -- text-capture
   detox test e2e/capture/text-capture.e2e.ts
   ```
3. **Start GREEN phase** - Task 1: Create TextCaptureInput component
4. **Implement one test at a time** (RED → GREEN)
5. **Commit after each green** test
6. **Refactor with confidence** after all tests pass

---

## Story Completion Criteria

**Definition of Done:**

- [ ] All 55 tests passing (green)
- [ ] NFR4 verified: List loading < 1s (cache local)
- [ ] NFR6 verified: Zero text capture lost
- [ ] NFR7 verified: 100% offline availability
- [ ] Keyboard opens < 300ms (UX requirement)
- [ ] Haptic feedback feels natural
- [ ] Code review completed
- [ ] Story status updated to "done" in sprint-status.yaml

---

**Generated by**: BMad TEA Agent - Test Architect Module
**Workflow**: `_bmad/bmm/testarch/atdd`
**Version**: 4.0 (BMad v6)
**Date**: 2026-01-21

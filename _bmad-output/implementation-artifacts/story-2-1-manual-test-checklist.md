# Story 2.1 - Manual Test Checklist

**Story:** Capture Audio 1-Tap
**Status:** review
**Date:** 2026-01-22
**Tester:** _____________

## Pre-requisites

- [ ] App built and running on iOS simulator or device
- [ ] Clean app state (uninstall/reinstall if needed)
- [ ] Microphone permissions granted

## Test Environment

- **Device/Simulator:** _____________
- **OS Version:** _____________
- **Build:** _____________

---

## AC1: Start Recording with < 500ms Latency

### Test Case 1.1: Visual Feedback
- [ ] Launch app
- [ ] Navigate to main screen with RecordButton
- [ ] **Expected:** Red circular button with "Tap to Record" label visible
- [ ] Tap the record button
- [ ] **Expected:** Button starts pulsing animation
- [ ] **Expected:** White square icon appears in center
- [ ] **Expected:** Latency feels instant (< 500ms)
- [ ] **Expected:** Timer appears showing "00:00" format

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

### Test Case 1.2: Haptic Feedback (iOS/Android only)
- [ ] Tap the record button
- [ ] **Expected:** Feel medium haptic feedback on tap
- [ ] Try multiple times to confirm consistency

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

### Test Case 1.3: Recording Timer
- [ ] Start recording
- [ ] **Expected:** Timer starts at 00:00
- [ ] Wait 5 seconds
- [ ] **Expected:** Timer shows 00:05
- [ ] Wait until 01:00
- [ ] **Expected:** Timer shows 01:00 (minute rollover)

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

---

## AC2: Stop and Save Recording

### Test Case 2.1: Stop Recording (Short - 2 seconds)
- [ ] Start recording
- [ ] Speak for ~2 seconds
- [ ] Tap button again to stop
- [ ] **Expected:** Pulsing animation stops
- [ ] **Expected:** Button returns to idle state (red circle, "Tap to Record")
- [ ] **Expected:** Recording saved (check later in capture list)

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

### Test Case 2.2: Stop Recording (Long - 30 seconds)
- [ ] Start recording
- [ ] Speak for ~30 seconds
- [ ] Tap button to stop
- [ ] **Expected:** Recording saved with correct duration

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

### Test Case 2.3: Audio File Created
- [ ] Complete a recording
- [ ] Check file system (via logs or file explorer)
- [ ] **Expected:** Audio file exists in `FileSystem.documentDirectory/audio/`
- [ ] **Expected:** File name follows pattern: `capture_{userId}_{timestamp}_{uuid}.m4a`

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

---

## AC3: Offline-First Capture

### Test Case 3.1: Record Without Network
- [ ] Enable Airplane Mode (or disable WiFi/cellular)
- [ ] Start recording
- [ ] **Expected:** Recording starts normally (no error)
- [ ] Speak for 5 seconds
- [ ] Stop recording
- [ ] **Expected:** Recording saved locally
- [ ] **Expected:** Capture marked with syncStatus = "pending"

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

### Test Case 3.2: Network Unavailable Error Handling
- [ ] Keep Airplane Mode enabled
- [ ] Start and complete multiple recordings
- [ ] **Expected:** All recordings saved locally without errors
- [ ] **Expected:** No network error dialogs shown

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

---

## AC4: Crash Recovery with Notification

### Test Case 4.1: Crash During Recording
⚠️ **Manual crash simulation required**

**Setup:**
- [ ] Start recording
- [ ] While recording, force-kill the app (swipe away from app switcher)

**Recovery:**
- [ ] Relaunch app
- [ ] **Expected:** Alert notification appears: "Enregistrements récupérés"
- [ ] **Expected:** Message: "Votre enregistrement interrompu a été récupéré avec succès."
- [ ] **Expected:** Recording appears in capture list with state "RECOVERED"

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

### Test Case 4.2: Multiple Incomplete Recordings
**Setup:**
- [ ] Start recording #1, force-kill app
- [ ] Relaunch, dismiss notification
- [ ] Start recording #2, force-kill app again

**Recovery:**
- [ ] Relaunch app
- [ ] **Expected:** Alert notification: "2 enregistrements interrompus ont été récupérés avec succès."
- [ ] **Expected:** Both recordings appear in capture list

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

### Test Case 4.3: Failed Recovery (Audio File Missing)
⚠️ **Advanced test - requires file system manipulation**

**Setup:**
- [ ] Start recording, note the file path from logs
- [ ] Force-kill app
- [ ] Manually delete the audio file from file system
- [ ] Relaunch app

**Expected:**
- [ ] Alert notification: "Récupération échouée"
- [ ] Message: "Impossible de récupérer l'enregistrement interrompu (fichier audio introuvable)."

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

---

## AC5: Microphone Permission Handling

### Test Case 5.1: Permission Denied
**Setup:**
- [ ] Uninstall app
- [ ] Reinstall app
- [ ] Launch app

**Test:**
- [ ] Tap record button
- [ ] **Expected:** iOS/Android permission dialog appears
- [ ] Tap "Don't Allow" / "Deny"
- [ ] **Expected:** Error callback triggered (check logs or error UI)
- [ ] **Expected:** Recording does NOT start

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

### Test Case 5.2: Permission Granted
**Setup:**
- [ ] Go to Settings → App → Permissions
- [ ] Enable Microphone permission

**Test:**
- [ ] Return to app
- [ ] Tap record button
- [ ] **Expected:** Recording starts immediately (no permission dialog)
- [ ] **Expected:** Audio captured successfully

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

---

## Additional Tests

### Test Case A1: Multiple Consecutive Recordings
- [ ] Complete recording #1 (5 seconds)
- [ ] Immediately start recording #2 (5 seconds)
- [ ] Immediately start recording #3 (5 seconds)
- [ ] **Expected:** All 3 recordings saved separately
- [ ] **Expected:** No audio overlap or corruption

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

### Test Case A2: Very Short Recording
- [ ] Start recording
- [ ] Immediately stop (< 1 second)
- [ ] **Expected:** Recording saved with duration < 1000ms
- [ ] **Expected:** Audio file created

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

### Test Case A3: Long Recording (5 minutes)
- [ ] Start recording
- [ ] Let it run for 5 minutes
- [ ] **Expected:** Timer reaches 05:00
- [ ] Stop recording
- [ ] **Expected:** Recording saved successfully (no memory issues)

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

### Test Case A4: App Backgrounding During Recording
- [ ] Start recording
- [ ] Press home button (app goes to background)
- [ ] Wait 5 seconds
- [ ] Return to app
- [ ] **Expected:** Recording still active, timer still running
- [ ] Stop recording
- [ ] **Expected:** Full duration captured

**Result:** ✅ Pass / ❌ Fail
**Notes:** _____________

---

## Summary

**Total Test Cases:** 19
**Passed:** ___ / 19
**Failed:** ___ / 19
**Blocked:** ___ / 19

## Critical Issues Found

| Test Case | Issue Description | Severity |
|-----------|------------------|----------|
|           |                  |          |

## Notes

_____________

## Sign-off

- [ ] All critical tests passed
- [ ] No blocking issues found
- [ ] Story 2.1 ready for production

**Tester Signature:** _____________
**Date:** _____________

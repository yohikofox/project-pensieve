# Story 2.1 - Testing Guide

## Quick Start

### 1. Build the App

```bash
cd pensieve/mobile

# Install dependencies (if not done already)
npm install

# Prebuild for iOS/Android
npm run prebuild:clean

# For iOS
npm run ios

# For Android
npm run android
```

### 2. Run Automated Tests First

Before manual testing, ensure all automated tests pass:

```bash
# Unit tests (RecordButton)
npm test -- src/contexts/capture/ui/__tests__/RecordButton.test.tsx

# Notification utils tests
npm test -- src/shared/utils/__tests__/notificationUtils.test.ts

# Acceptance tests (Story 2.1)
npm run test:acceptance:story-2-1

# All tests
npm test
```

**Expected results:**
- RecordButton: 7/7 tests ✅
- notificationUtils: 8/8 tests ✅
- Acceptance: 4/4 tests ✅
- **Total: 19/19 tests passing**

### 3. Manual Testing

Follow the checklist in `story-2-1-manual-test-checklist.md`

---

## Testing Scenarios Priority

### 🔴 Critical (Must Pass)
1. **AC1: Start Recording** - Core functionality
2. **AC2: Stop and Save** - Data persistence
3. **AC5: Permissions** - App access control

### 🟡 Important (Should Pass)
4. **AC3: Offline** - Offline-first requirement
5. **AC4: Crash Recovery** - Data recovery

### 🟢 Nice to Have (Additional)
6. **Multiple recordings** - Real-world usage
7. **Long recordings** - Edge case
8. **Backgrounding** - Multitasking

---

## Common Issues & Troubleshooting

### Issue: App won't build
**Solution:**
```bash
cd mobile
rm -rf node_modules package-lock.json
npm install
npm run prebuild:clean
```

### Issue: Microphone permission not working
**Solution:**
- iOS: Check `Info.plist` has `NSMicrophoneUsageDescription`
- Android: Check `AndroidManifest.xml` has `RECORD_AUDIO` permission
- Reset permissions: Uninstall app, reinstall

### Issue: RecordButton not visible
**Solution:**
- Check navigation is working (AuthNavigator vs MainNavigator)
- Verify user is authenticated
- Check React Navigation setup in App.tsx

### Issue: Recording doesn't start
**Solution:**
- Check console logs for errors
- Verify expo-audio is installed: `npm ls expo-audio`
- Check TSyringe container is initialized: `registerServices()` called in App.tsx

### Issue: Crash recovery notification doesn't appear
**Solution:**
- Check `useEffect` in App.tsx is running
- Verify CrashRecoveryService is registered in container
- Check Alert mock is not interfering (if in test mode)

---

## Debugging Tools

### Enable Debug Logging

Add to `App.tsx`:
```typescript
import { LogBox } from 'react-native';

if (__DEV__) {
  LogBox.ignoreLogs(['Require cycle:']); // Ignore cycle warnings
  console.log('[App] TSyringe container initialized');
}
```

### Check File System

Add temporary logging to see captured files:
```typescript
import * as FileSystem from 'expo-file-system';

const audioDir = FileSystem.documentDirectory + 'audio/';
const files = await FileSystem.readDirectoryAsync(audioDir);
console.log('[Debug] Captured files:', files);
```

### Monitor Crash Recovery

Add logging to `CrashRecoveryService.recoverIncompleteRecordings()`:
```typescript
console.log('[CrashRecovery] Checking for incomplete recordings...');
console.log('[CrashRecovery] Found:', recoveredCaptures.length);
```

---

## Performance Benchmarking

### Measure Recording Start Latency

```typescript
// In RecordButton.tsx handlePress()
const startTime = performance.now();
await recordingService.startRecording();
const latency = performance.now() - startTime;
console.log('[Performance] Recording start latency:', latency, 'ms');
// Should be < 500ms per AC1
```

### Check Memory Usage During Long Recording

```typescript
// Add to RecordButton useEffect timer
if (__DEV__ && recordingDuration % 60 === 0) {
  console.log('[Memory] Recording duration:', recordingDuration, 's');
  // Use React Native Performance API or Xcode Instruments
}
```

---

## Test Data Cleanup

### Reset App State Between Tests

```bash
# iOS Simulator
xcrun simctl uninstall booted com.pensieve.app

# Android Emulator
adb uninstall com.pensieve.app
```

### Clear Local Storage

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';
await AsyncStorage.clear();
```

### Delete Audio Files

```typescript
import * as FileSystem from 'expo-file-system';
const audioDir = FileSystem.documentDirectory + 'audio/';
await FileSystem.deleteAsync(audioDir, { idempotent: true });
await FileSystem.makeDirectoryAsync(audioDir);
```

---

## E2E Tests (Future)

Story 2.1 has unit and acceptance tests, but no E2E tests yet.

### Future E2E Setup (Detox)

```bash
# Install Detox (when ready)
npm install --save-dev detox

# Build for E2E
npm run test:e2e:build:ios

# Run E2E tests
npm run test:e2e:ios
```

### Sample E2E Test (Future)
```typescript
describe('Story 2.1 - Capture Audio 1-Tap', () => {
  it('should record audio on button tap', async () => {
    await element(by.id('record-button')).tap();
    await expect(element(by.text('00:00'))).toBeVisible();
    await waitFor(element(by.text('00:05')))
      .toBeVisible()
      .withTimeout(6000);
    await element(by.id('record-button')).tap();
    await expect(element(by.text('Tap to Record'))).toBeVisible();
  });
});
```

Note: `testID="record-button"` already added to RecordButton.tsx for E2E readiness.

---

## Sign-off Criteria

Before marking Story 2.1 as "done":

- [ ] All 19 automated tests passing
- [ ] All critical manual tests passing (AC1, AC2, AC5)
- [ ] No blocking bugs found
- [ ] Performance < 500ms verified
- [ ] Crash recovery tested and working
- [ ] Offline mode tested and working

---

## Contact

If you encounter issues during testing:
1. Check this guide's troubleshooting section
2. Review console logs for errors
3. Check Story 2.1 implementation file for known issues
4. Create a new issue in tracking system with:
   - Test case that failed
   - Expected vs actual behavior
   - Screenshots/logs
   - Device/OS info

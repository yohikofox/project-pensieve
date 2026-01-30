---

name: 'step-03-explore'
description: 'Freestyle exploration loop with passive capture, bug documentation, notes, and checkpoints'

<!-- Path Definitions -->

workflow_path: '{project-root}/_bmad-output/bmb-creations/workflows/mobile-exploratory-testing'

# File References (all use {variable} format in file)

thisStepFile: '{workflow_path}/steps/step-03-explore.md'
nextStepFile: '{workflow_path}/steps/step-04-classify.md'
workflowFile: '{workflow_path}/workflow.md'
outputFile: '{output_folder}/validation-report-{project_name}.md'
sidecarFile: '{output_folder}/validation-report-{project_name}-history.md'

---

# Step 3: Freestyle Exploration Loop

## STEP GOAL:

To enable free-form exploratory testing where the tester navigates the app naturally while the AI observes passively, captures screenshots automatically, documents bugs and notes on demand, and suggests checkpoints without disrupting the flow. This step builds the raw testing data that will be analyzed in subsequent steps.

## MANDATORY EXECUTION RULES (READ FIRST):

### Universal Rules:

- 🛑 NEVER generate content without user input
- 📖 CRITICAL: Read the complete step file before taking any action
- 🔄 CRITICAL: When loading next step with 'C', ensure entire file is read
- 📋 YOU ARE A FACILITATOR, not a content generator

### Role Reinforcement:

- ✅ You are a **QA Specialist & Mobile Testing Expert**
- ✅ If you already have been given a name, communication_style and identity, continue to use those while playing this new role
- ✅ We engage in collaborative dialogue, not command-response
- ✅ You bring mobile testing and quality assessment expertise, user brings domain knowledge and testing intuition
- ✅ Together we produce better quality validation than either could alone
- ✅ Maintain collaborative, professional, and precise tone throughout

### Step-Specific Rules:

- 🎯 Focus only on observation and passive capture during exploration
- 🚫 FORBIDDEN to direct the tester's navigation or tell them what to test
- 💬 Approach: Non-intrusive observer, helpful assistant when called upon
- 📱 Respect the tester's flow - suggestions only, never commands
- 📸 Capture automatically but intelligently (avoid spam)
- 🐛 Document bugs immediately when tester discovers them

## EXECUTION PROTOCOLS:

- 🎯 Maintain exploration loop until user selects [T] Terminate
- 💾 Store all captured data (screenshots, bugs, notes) in frontmatter and report
- 📖 Track timeline of screens visited for navigation flow analysis
- 🚫 FORBIDDEN to analyze or classify bugs here (that's step 4)
- 📸 Auto-capture screenshots on significant interactions
- 📚 Update sidecar file with exploration events
- ⏱️ Suggest checkpoints every ~5 min or 10 screenshots (non-forced)

## CONTEXT BOUNDARIES:

- Available context: App launched, baseline screenshot captured, capture system initialized
- Focus: Raw data collection through observation
- Limits: No analysis, classification, or verdict yet
- Dependencies: Step 2 complete with app running and capture ready

## Sequence of Instructions (Do not deviate, skip, or optimize)

### 1. Initialize Exploration Tracking

Read current state from {outputFile} frontmatter:

- `screenshotsCount`: Current screenshot count (should be 1 from baseline)
- `bugsCount`: Current bugs count (should be 0)
- `notesCount`: Current notes count (should be 0)
- `deviceId`: Device being tested
- `appName`: App under test

**Initialize Exploration Session State** (in-memory):
```
explorationState = {
  startTime: {current timestamp},
  screenshotCounter: {screenshotsCount from frontmatter},
  bugCounter: {bugsCount from frontmatter},
  noteCounter: {notesCount from frontmatter},
  lastCheckpointTime: {current timestamp},
  lastCheckpointScreenshots: {screenshotsCount},
  screenTimeline: [],  // List of screen descriptions
  bugsList: [],  // Temporary bugs list
  notesList: [],  // Temporary notes list
  interactionCount: 0
}
```

**Display Exploration Start**:
```
🎬 Exploration Freestyle Active

**App**: {appName}
**Mode**: Passive Capture
**Baseline**: screenshot-00-baseline capturé

📱 **C'est parti!** Explore l'app librement. Je t'observe et capture en arrière-plan.

💡 **Commandes disponibles**:
- [D] Document Bug
- [N] Add Note
- [C] Checkpoint
- [T] Terminate Exploration

_Tu peux explorer maintenant. Interagis naturellement avec l'app, je documente._
```

### 2. Enter Freestyle Exploration Loop

**LOOP UNTIL [T] Terminate is selected:**

This is an interactive loop where:
- User explores the app naturally using their device
- AI observes and waits for user commands [D/N/C/T]
- AI auto-captures screenshots on significant interactions
- AI suggests checkpoints periodically (non-forced)
- User can chat/ask questions at any time

**During Each Loop Iteration:**

#### A. Wait for User Input

Display a subtle, non-intrusive prompt:
```
[Exploration en cours... Tape D/N/C/T ou décris ce que tu vois]
```

**HALT and WAIT** for user to:
- Select a command [D/N/C/T]
- Describe what they're doing/seeing (triggers auto-capture)
- Ask a question or chat
- Report an issue

#### B. Auto-Capture Logic

When user describes an interaction or screen change:

**Triggers for Auto-Screenshot**:
- User mentions "nouvel écran", "page", "vue", "modal", "popup"
- User describes a significant UI element or change
- User reports unexpected behavior
- User navigates to a new feature area

**When triggered**:
1. Call MCP `mobile_take_screenshot` with `device: {deviceId}`
2. Generate timestamp and counter: `screenshot-{counter:02d}-{timestamp}.png`
3. Increment `screenshotCounter`
4. Add to `screenTimeline`: Short description of screen (from user's words)
5. Display: "📸 Screenshot capturé: screenshot-{counter:02d}"

**Avoid Spam**:
- Don't capture if last screenshot was < 5 seconds ago
- Don't capture on every single user message
- Use judgment: Only capture when screen/state changes

#### C. Handle User Commands

**IF [D] Document Bug Selected:**

1. Display: "🐛 **Documentation de Bug** - Décris le bug que tu viens de trouver:"

2. Ask for bug details (one at a time, conversational):
   - "Quel est le problème que tu as observé? (description courte)"
   - "Comment as-tu déclenché ce bug? (steps reproductibles)"
   - "Que devrais-tu voir normalement? (comportement attendu)"
   - "Est-ce que l'app a crashé, freezé, ou juste un comportement bizarre?"

3. Capture bug screenshot if not already taken:
   - Call MCP `mobile_take_screenshot`
   - Filename: `bug-{bugCounter+1:02d}-{timestamp}.png`

4. Create temporary bug entry:
   ```
   Bug #{bugCounter+1}:
   - Title: [Auto-generated from description]
   - Description: [User description]
   - Reproduction Steps: [User steps]
   - Expected: [User expected behavior]
   - Actual: [User actual behavior]
   - Screenshot: bug-{bugCounter:02d}-{timestamp}.png
   - Timestamp: {current timestamp}
   - Classification: [To be determined in step 4]
   ```

5. Add to `bugsList` in exploration state
6. Increment `bugCounter`
7. Display: "✅ Bug #{bugCounter} documenté. Tu peux continuer l'exploration."
8. Return to [Exploration Loop](#2-enter-freestyle-exploration-loop)

**IF [N] Add Note Selected:**

1. Display: "📝 **Nouvelle Note** - Qu'as-tu observé? (une note rapide)"

2. Collect note from user (single message)

3. Capture note screenshot if relevant:
   - Ask: "Veux-tu capturer un screenshot pour cette note? [Y/N]"
   - If Y: Call MCP `mobile_take_screenshot`, filename: `note-{noteCounter+1:02d}-{timestamp}.png`

4. Create temporary note entry:
   ```
   Note #{noteCounter+1}:
   - Content: [User note]
   - Screenshot: [note-{noteCounter:02d}-{timestamp}.png or 'none']
   - Timestamp: {current timestamp}
   ```

5. Add to `notesList` in exploration state
6. Increment `noteCounter`
7. Display: "✅ Note #{noteCounter} enregistrée."
8. Return to [Exploration Loop](#2-enter-freestyle-exploration-loop)

**IF [C] Checkpoint Selected:**

1. Calculate exploration metrics:
   - Duration since last checkpoint: `{current time - lastCheckpointTime}`
   - Screenshots since last checkpoint: `{screenshotCounter - lastCheckpointScreenshots}`
   - Total bugs documented: `{bugCounter}`
   - Total notes taken: `{noteCounter}`

2. Display checkpoint summary:
   ```
   ⏸️ **Checkpoint**

   **Depuis le dernier checkpoint**:
   - Durée: {X minutes}
   - Screenshots: {Y nouveaux} (Total: {screenshotCounter})
   - Bugs documentés: {bugCounter}
   - Notes prises: {noteCounter}

   **Timeline récente**:
   {List last 5-10 screens from screenTimeline}

   **Bugs temporaires**:
   {List bug titles if any}

   **Notes temporaires**:
   {List note contents if any}

   Tu peux continuer l'exploration ou documenter plus de détails.

   [R] Resume Exploration → Continuer l'exploration
   [A] Add More Details → Ajouter des détails sur un bug/note existant
   [T] Terminate → Terminer la phase d'exploration
   ```

3. Handle checkpoint menu:
   - **IF R**: Update checkpoint trackers (`lastCheckpointTime`, `lastCheckpointScreenshots`), return to [Exploration Loop](#2-enter-freestyle-exploration-loop)
   - **IF A**: Let user select bug/note to enhance, collect additional details, return to checkpoint menu
   - **IF T**: Proceed to section 3 (Terminate Exploration)

4. Log checkpoint in sidecar file:
   ```markdown
   ### Checkpoint - {current time}
   **Duration**: {X minutes}
   **Screenshots**: {screenshotCounter} total
   **Bugs**: {bugCounter}
   **Notes**: {noteCounter}
   ```

**IF [T] Terminate Exploration Selected:**
- Proceed to section 3 below

**IF User Chats or Asks Questions:**
- Respond helpfully in collaborative QA expert style
- If they describe what they're seeing, trigger auto-capture logic
- After response, redisplay [Exploration Loop Prompt](#a-wait-for-user-input)

#### D. Periodic Checkpoint Suggestions (Non-Forced)

Every 5 minutes OR every 10 screenshots, suggest a checkpoint:

**Check Conditions**:
- Time since last checkpoint > 5 minutes OR
- Screenshots since last checkpoint >= 10

**If Conditions Met**:
Display subtle suggestion (not a forced menu):
```
💡 _Je remarque {X minutes} d'exploration ou {Y screenshots} capturés. Veux-tu faire un checkpoint pour documenter tes observations ? (Tape [C] ou continue)_
```

**IMPORTANT**: This is a suggestion only, user can ignore and keep exploring.

### 3. Terminate Exploration

When user selects [T] Terminate from exploration loop or checkpoint menu:

**Display Exploration Summary**:
```
🏁 **Fin de l'Exploration Freestyle**

**Durée totale**: {total duration from start to now}
**Screenshots capturés**: {screenshotCounter}
**Bugs documentés**: {bugCounter}
**Notes enregistrées**: {noteCounter}
**Écrans visités**: {count of screenTimeline entries}

**Prochaine étape**: Classification et documentation des bugs
```

### 4. Write Exploration Data to Report

**Append to {outputFile}** (between frontmatter and existing sections):

Create or update **"## 1. Exploration Summary"** section:

```markdown
## 1. Exploration Summary

**Session Date**: {date from frontmatter}
**Duration**: {total exploration duration}
**Story ID**: {storyId or 'Exploration libre'}
**Focus Area**: {focusArea or 'Exploration générale'}
**Build Version**: {buildVersion or 'N/A'}

### Metrics

- **Screenshots Captured**: {screenshotsCount}
- **Bugs Documented**: {bugsCount}
- **Notes Taken**: {notesCount}
- **Screens Visited**: {count of screenTimeline}

### Screen Timeline

{For each entry in screenTimeline}
- **{timestamp}**: {screen description}

### Bugs (Temporary - To be classified)

{For each bug in bugsList}
#### Bug #{bugNumber}
- **Title**: {title}
- **Description**: {description}
- **Reproduction Steps**: {steps}
- **Expected**: {expected}
- **Actual**: {actual}
- **Screenshot**: {screenshot filename}
- **Timestamp**: {timestamp}

### Notes

{For each note in notesList}
#### Note #{noteNumber}
- **Content**: {content}
- **Screenshot**: {screenshot filename or 'none'}
- **Timestamp**: {timestamp}
```

### 5. Update Frontmatter

**Update frontmatter in {outputFile}**:

```yaml
stepsCompleted: [1, 2, 3]
lastStep: 'explore'
screenshotsCount: {final screenshotCounter}
bugsCount: {final bugCounter}
notesCount: {final noteCounter}
explorationDuration: "{total duration in minutes}"
screensVisited: {count of screenTimeline}
```

### 6. Update Sidecar File

**Append to {sidecarFile}**:

```markdown
### Exploration Completed - {current time}
**Duration**: {total duration}
**Screenshots**: {screenshotsCount}
**Bugs Documented**: {bugsCount}
**Notes Taken**: {notesCount}

_Moving to bug classification phase..._
```

### 7. Present AUTO-PROCEED

Display: **Passage à la classification des bugs et documentation détaillée...**

#### EXECUTION RULES:

- Auto-proceed to bug classification after exploration data is written
- No user choice required here (they already confirmed termination with [T])

#### Auto-Proceed Logic:

After exploration data written to report and frontmatter/sidecar updated:

1. **Verify** `stepsCompleted: [1, 2, 3]` in frontmatter
2. **Verify** exploration summary section exists in {outputFile}
3. **Immediately load, read entire file, then execute** `{nextStepFile}` to begin bug classification and detailed documentation

## CRITICAL STEP COMPLETION NOTE

ONLY WHEN user selects [T] Terminate, exploration data is written to report, frontmatter updated with step 3 completion, and sidecar file updated, will you then:

1. Verify all bugs from bugsList are written to report
2. Verify all notes from notesList are written to report
3. Verify screen timeline is written to report
4. Update frontmatter `stepsCompleted` to `[1, 2, 3]`
5. Set `lastStep: 'explore'`
6. Save the output file
7. Immediately load, read entire file, then execute `{nextStepFile}` to begin bug classification

## 🚨 SYSTEM SUCCESS/FAILURE METRICS

### ✅ SUCCESS:

- Freestyle exploration loop maintained until user terminates
- Screenshots auto-captured on significant interactions (not spam)
- All bugs documented with description, steps, expected, actual, screenshot
- All notes recorded with optional screenshots
- Checkpoints suggested periodically (every 5 min or 10 screenshots)
- Screen timeline maintained throughout exploration
- User could explore freely without forced interruptions
- Exploration summary section written to report
- All bugs and notes written to report
- Frontmatter updated with final counts and step 3 completion
- Sidecar file updated with exploration completion event
- Next step (classification) loaded automatically

### ❌ SYSTEM FAILURE:

- Directing or commanding the tester where to navigate
- Forcing checkpoints instead of suggesting
- Over-capturing screenshots (spam)
- Not documenting bugs when user selects [D]
- Proceeding to next step without user [T] termination
- Not writing exploration data to report
- Not updating frontmatter with step 3 completion
- Analyzing or classifying bugs during exploration (that's step 4)
- Losing bugs, notes, or screenshot data

**Master Rule:** Skipping steps, optimizing sequences, or not following exact instructions is FORBIDDEN and constitutes SYSTEM FAILURE.

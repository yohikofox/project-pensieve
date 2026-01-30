---

name: 'step-02-launch'
description: 'Launch the mobile app and initialize passive capture system for exploratory testing'

<!-- Path Definitions -->

workflow_path: '{project-root}/_bmad-output/bmb-creations/workflows/mobile-exploratory-testing'

# File References (all use {variable} format in file)

thisStepFile: '{workflow_path}/steps/step-02-launch.md'
nextStepFile: '{workflow_path}/steps/step-03-explore.md'
workflowFile: '{workflow_path}/workflow.md'
outputFile: '{output_folder}/validation-report-{project_name}.md'
sidecarFile: '{output_folder}/validation-report-{project_name}-history.md'

# Data References

bugClassificationsData: '{workflow_path}/data/bug-classifications.md'

---

# Step 2: App Launch & Capture Initialization

## STEP GOAL:

To launch the mobile app under test, initialize the passive capture system, take the initial baseline screenshot, and prepare the tester for freestyle exploratory testing by explaining the mode and available commands.

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

- 🎯 Focus only on app launch, screenshot capture, and capture system initialization
- 🚫 FORBIDDEN to start exploration or document bugs (that's step 3)
- 💬 Approach: Technical setup with user-friendly explanations
- 📱 Use MCP mobile-mcp tools for all device interactions
- 📸 Capture baseline screenshot before any user interaction

## EXECUTION PROTOCOLS:

- 🎯 Launch app and verify successful launch before proceeding
- 💾 Update frontmatter with launch timestamp and initial screenshot path
- 📖 Initialize passive capture system and tracking counters
- 🚫 FORBIDDEN to proceed if app fails to launch
- 📸 Store screenshot with timestamped filename
- 📚 Update sidecar file with launch event

## CONTEXT BOUNDARIES:

- Available context: Device info, app info, story context from frontmatter
- Focus: Technical launch and initialization only
- Limits: Don't start actual exploration or testing yet
- Dependencies: Step 1 must be complete with device and app validated

## Sequence of Instructions (Do not deviate, skip, or optimize)

### 1. Read Current Context from Frontmatter

Read the frontmatter from {outputFile} to retrieve:

- `deviceId`: Device identifier to launch app on
- `deviceName`: Device friendly name for display
- `packageName`: Package name of app to launch
- `appName`: App friendly name for display
- `storyId`: Story being tested (if any)
- `focusArea`: Focus area (if any)
- `buildVersion`: Build version being tested (if any)

Display context confirmation:
```
🚀 Préparation du lancement

**Device**: {deviceName}
**App**: {appName} (v{buildVersion or 'N/A'})
**Package**: {packageName}
**Story**: {storyId or 'Exploration libre'}
```

### 2. Launch the Mobile App

Use MCP mobile-mcp to launch the app:

**Action**: Call MCP tool `mobile_launch_app` with:
- `device`: {deviceId}
- `packageName`: {packageName}

**Success Case**:
- Display: "✅ App '{appName}' lancée avec succès sur {deviceName}"
- Wait 2-3 seconds for app stabilization
- Proceed to next step

**Failure Case**:
- Display error details from MCP response
- Analyze failure reason:
  - **App not installed**: "L'app n'est plus installée. Veux-tu l'installer maintenant ?"
  - **Device disconnected**: "Le device est déconnecté. Reconnecte-le et réessaye."
  - **Permission denied**: "Problème de permissions. Vérifie les paramètres du device."
  - **Other error**: Display MCP error message

**Recovery Options** (if failure):
```
**Options de récupération**:

[R] Relancer l'app → Réessayer le lancement
[I] Installer l'app → Si app désinstallée, fournir path .apk/.ipa
[D] Changer de device → Sélectionner un autre device connecté
[X] Annuler → Arrêter la session de test

Que souhaites-tu faire ?
```

Handle user choice:
- **R (Relancer)**: Retry `mobile_launch_app`, repeat this section
- **I (Installer)**: Ask for app file path, use `mobile_install_app`, then retry launch
- **D (Device)**: Call `mobile_list_available_devices`, let user select, update frontmatter, retry launch
- **X (Annuler)**: Update sidecar with cancellation event, exit workflow gracefully

**CRITICAL**: Do NOT proceed to section 3 until app launch succeeds.

### 3. Capture Initial Baseline Screenshot

Take the first screenshot to establish baseline state:

**Action**: Call MCP tool `mobile_take_screenshot` with:
- `device`: {deviceId}

**Screenshot Storage**:
1. Generate timestamp: `{current-datetime in format YYYYMMDD-HHmmss}`
2. Filename: `screenshot-00-baseline-{timestamp}.png`
3. Store screenshot (MCP returns image data)
4. Display: "📸 Screenshot baseline capturé: screenshot-00-baseline-{timestamp}.png"

**Update Frontmatter in {outputFile}**:
```yaml
screenshotsCount: 1
initialScreenshot: "screenshot-00-baseline-{timestamp}.png"
launchTimestamp: "{current ISO timestamp}"
```

**Update Sidecar File** {sidecarFile}:
Append to current session section:
```markdown
**App Launched**: {current time}
**Initial Screenshot**: screenshot-00-baseline-{timestamp}.png
```

### 4. Initialize Passive Capture System

Set up the capture tracking system:

**Initialize Capture State** (in-memory for this session):
- `screenshotCounter`: 1 (baseline already captured)
- `bugCounter`: 0
- `noteCounter`: 0
- `lastScreenshotTime`: {current timestamp}
- `captureMode`: "passive" (automatic capture on significant interactions)

**Display System Status**:
```
🎬 Système de capture initialisé

**Mode**: Capture passive (automatique)
**Baseline**: screenshot-00-baseline capturé
**Compteurs**: 1 screenshot, 0 bugs, 0 notes

Le système va capturer automatiquement:
- Screenshots lors d'interactions significatives
- Bugs documentés avec [D]
- Notes rapides avec [N]
- Checkpoints avec [C]
```

### 5. Load Vector DB Context (If Available)

Check if Vector DB integration is enabled:

**Action**: Check for Vector DB MCP server availability
- If available: Load semantic context for regression detection
  - Search Vector DB for similar testing sessions
  - Load known bugs related to {storyId} or {focusArea}
  - Display: "🔍 Vector DB chargé - {X} sessions similaires, {Y} bugs connus trouvés"
  - Store context for use in step 7 (Confidence & Regression)

- If not available:
  - Display: "ℹ️ Vector DB non disponible - détection de régression désactivée"
  - Note: Regression analysis will be manual in step 7

### 6. Explain Freestyle Mode to Tester

Provide clear explanation of the exploratory testing mode:

**Display Freestyle Mode Guide**:
```
🎯 Mode Test Exploratoire Freestyle

**Principe**: Tu explores l'app librement, sans parcours imposé. Je t'observe et documente.

**Comment ça marche**:
- 📱 **Explore librement**: Pas de script, suis ton intuition
- 📸 **Capture auto**: Je screenshot automatiquement les interactions importantes
- ⏸️ **Checkpoints**: Je suggère des pauses toutes les ~5 min ou 10 screenshots (non forcé)
- 🐛 **Bugs**: Si tu trouves un bug, tape [D] pour le documenter immédiatement
- 📝 **Notes**: Si tu veux noter quelque chose, tape [N]
- ✅ **Termine**: Quand tu as fini, tape [T]

**Pendant l'exploration**:
- Je ne te dirige pas, tu décides où aller
- Je capture passivement ce que tu fais
- Je propose des checkpoints, mais tu choisis si tu veux les faire
- À tout moment, tu peux documenter un bug ou ajouter une note

**Focus actuel**: {focusArea or 'Exploration générale de l\'app'}

**Commandes disponibles**:
- **[D]** Document Bug → Capturer un bug immédiatement
- **[N]** Add Note → Ajouter une note/observation rapide
- **[C]** Checkpoint → Pause pour documenter et organiser
- **[T]** Terminate → Terminer la phase d'exploration

Prêt à commencer l'exploration ?
```

### 7. Wait for User Acknowledgment

**Display**:
```
Confirme que tu es prêt à explorer:

[G] Go → Commencer l'exploration freestyle
[Q] Questions → Poser des questions sur le mode ou les commandes
[X] Exit → Annuler la session
```

**Handle User Input**:

- **IF G (Go)**:
  - Display: "🎬 Exploration freestyle démarrée - bonne chasse aux bugs !"
  - Update sidecar file:
    ```markdown
    **Exploration Started**: {current time}
    **Mode**: Freestyle (passive capture)
    ```
  - Proceed to section 8

- **IF Q (Questions)**:
  - Answer user questions about:
    - How passive capture works
    - What "significant interactions" means for auto-screenshot
    - How checkpoints work
    - How to document bugs during exploration
    - What happens with the data captured
  - After answering, redisplay [Menu Options](#7-wait-for-user-acknowledgment)

- **IF X (Exit)**:
  - Update sidecar with exit event
  - Save frontmatter (do NOT add step 2 to stepsCompleted since exploration didn't start)
  - Display: "Session annulée. Le rapport a été sauvegardé."
  - Exit workflow gracefully

**CRITICAL**: Do NOT proceed to section 8 until user selects [G].

### 8. Present AUTO-PROCEED

Display: **Lancement de la boucle d'exploration freestyle...**

#### EXECUTION RULES:

- This is a setup completion step with no additional choices
- Proceed directly to next step after user confirms with [G]
- Auto-proceed logic section below

#### Auto-Proceed Logic:

After successful app launch, screenshot capture, capture system initialization, and user confirmation:

1. **Update frontmatter** in {outputFile}:
   - Add `2` to `stepsCompleted` array: `stepsCompleted: [1, 2]`
   - Set `lastStep: 'launch'`
   - Ensure `screenshotsCount`, `initialScreenshot`, `launchTimestamp` are set

2. **Save {outputFile}** with updated frontmatter

3. **Immediately load, read entire file, then execute** `{nextStepFile}` to begin freestyle exploration loop

## CRITICAL STEP COMPLETION NOTE

ONLY WHEN app launch succeeds, baseline screenshot captured, capture system initialized, Vector DB loaded (if available), freestyle mode explained, and user confirms [G], will you then:

1. Update frontmatter `stepsCompleted` to `[1, 2]`
2. Set `lastStep: 'launch'`
3. Save the output file
4. Immediately load, read entire file, then execute `{nextStepFile}` to begin freestyle exploration

## 🚨 SYSTEM SUCCESS/FAILURE METRICS

### ✅ SUCCESS:

- App launched successfully via MCP mobile_launch_app
- Baseline screenshot captured and stored with timestamp
- Frontmatter updated with launch timestamp and screenshot path
- Sidecar file updated with launch event
- Passive capture system initialized with counters
- Vector DB context loaded (if available) for regression detection
- Freestyle mode explained clearly to tester
- User confirmed readiness with [G]
- Step 2 added to frontmatter stepsCompleted array
- Next step (exploration loop) loaded

### ❌ SYSTEM FAILURE:

- Proceeding without successful app launch
- Not capturing baseline screenshot
- Not initializing capture counters in frontmatter
- Skipping freestyle mode explanation
- Auto-proceeding without user [G] confirmation
- Not updating frontmatter with step 2 completion
- Not handling app launch failures with recovery options
- Not updating sidecar file with launch events

**Master Rule:** Skipping steps, optimizing sequences, or not following exact instructions is FORBIDDEN and constitutes SYSTEM FAILURE.

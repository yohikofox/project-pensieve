---

name: 'step-01-init'
description: 'Initialize mobile exploratory testing workflow by detecting continuation state, setting up device connection, and creating validation report'

<!-- Path Definitions -->

workflow_path: `{project-root}/_bmad-output/bmb-creations/workflows/mobile-exploratory-testing`

# File References (all use {variable} format in file)

thisStepFile: `{workflow_path}/steps/step-01-init.md`
nextStepFile: `{workflow_path}/steps/step-02-launch.md`
workflowFile: `{workflow_path}/workflow.md`
outputFile: `{output_folder}/validation-report-{project_name}.md`
continueFile: `{workflow_path}/steps/step-01b-continue.md`
templateFile: `{workflow_path}/templates/report-template.md`
sidecarFile: `{output_folder}/validation-report-{project_name}-history.md`

# Template References

# This step doesn't use content templates, only the main template

---

# Step 1: Mobile Testing Workflow Initialization

## MANDATORY EXECUTION RULES (READ FIRST):

### Universal Rules:

- 🛑 NEVER generate content without user input
- 📖 CRITICAL: Read the complete step file before taking any action
- 🔄 CRITICAL: When loading next step with 'C', ensure entire file is read
- 📋 YOU ARE A FACILITATOR, not a content generator

### Role Reinforcement:

- ✅ You are a **QA Specialist & Mobile Testing Expert**
- ✅ We engage in collaborative dialogue, not command-response
- ✅ You bring mobile testing and quality assessment expertise, user brings domain knowledge and testing intuition
- ✅ Together we produce better quality validation than either could alone
- ✅ Maintain collaborative, professional, and precise tone throughout

### Step-Specific Rules:

- 🎯 Focus ONLY on initialization and device setup
- 🚫 FORBIDDEN to look ahead to future steps
- 💬 Handle device detection and initialization professionally
- 🚪 DETECT existing workflow state and handle continuation properly
- 📱 Use MCP mobile-mcp tools for all device interactions

## EXECUTION PROTOCOLS:

- 🎯 Show analysis before taking any action
- 💾 Initialize document and update frontmatter
- 📖 Set up frontmatter `stepsCompleted: [1]` before loading next step
- 🚫 FORBIDDEN to load next step until setup is complete
- 📱 Validate device availability and app installation before proceeding

## CONTEXT BOUNDARIES:

- Variables from workflow.md are available in memory
- Previous context = what's in output document + frontmatter
- Don't assume knowledge from other steps
- Device and app context discovery happens in this step

## STEP GOAL:

To initialize the mobile exploratory testing workflow by detecting continuation state, establishing device connection, collecting testing context (Story ID, focus area, build version), creating the validation report document, loading session history, and preparing for app launch.

## INITIALIZATION SEQUENCE:

### 1. Check for Existing Workflow

First, check if the validation report already exists:

- Look for file at `{output_folder}/validation-report-{project_name}.md`
- If exists, read the complete file including frontmatter
- If not exists, this is a fresh testing session

### 2. Handle Continuation (If Document Exists)

If the document exists and has frontmatter with `stepsCompleted`:

- **STOP here** and load `./step-01b-continue.md` immediately
- Do not proceed with any initialization tasks
- Let step-01b handle the continuation logic

### 3. Handle Completed Workflow

If the document exists AND all steps [1,2,3,4,5,6,7,8] are marked complete in `stepsCompleted`:

- Ask user: "J'ai trouvé un rapport de validation existant daté du [date]. Souhaites-tu:
  1. Créer un nouveau rapport de validation
  2. Modifier/mettre à jour le rapport existant"
- If option 1: Create new document with timestamp suffix (e.g., `validation-report-{project_name}-2026-01-30-14h30.md`)
- If option 2: Load step-01b-continue.md

### 4. Fresh Workflow Setup (If No Document)

If no document exists or no `stepsCompleted` in frontmatter:

#### A. Device Detection

Detect available mobile devices using MCP mobile-mcp:

**Detect Devices:**

1. Call MCP tool `mobile_list_available_devices` to list all connected devices
2. Display results to user with clear formatting:
   ```
   Devices disponibles:
   1. [Device Name] - [Device ID] - [Type: simulator/physical]
   2. [Device Name] - [Device ID] - [Type: simulator/physical]
   ```
3. If NO devices found:
   - **ERROR**: "Aucun device mobile détecté."
   - **Solution**: "Vérifie que:"
     - "Un simulateur iOS/Android est lancé OU"
     - "Un device physique est connecté via USB/WiFi"
     - "Les drivers ADB/iOS sont installés"
   - **Action**: "Relance cette commande une fois le device connecté"
   - **STOP workflow** - cannot proceed without device

4. If ONE device found:
   - Auto-select this device
   - Display: "Device sélectionné: [Device Name] ([Device ID])"

5. If MULTIPLE devices found:
   - Ask user to select: "Sélectionne le device à utiliser (numéro): "
   - Validate input (must be valid number from list)
   - Confirm selection: "Device sélectionné: [Device Name] ([Device ID])"

**Store Device Info:**
- `deviceId`: Selected device identifier
- `deviceName`: Device friendly name
- `deviceType`: simulator or physical

#### B. App Selection & Validation

Collect app information from user:

1. **Ask for App Name:**
   - "Quelle app souhaites-tu tester? (nom complet, ex: Pensine)"
   - Store as `appName`

2. **Ask for Package Name:**
   - "Quel est le package name de l'app? (ex: com.pensine.app)"
   - Store as `packageName`

3. **Validate App Installation:**
   - Call MCP tool `mobile_list_apps` with selected `deviceId`
   - Search for `packageName` in returned app list
   - If **NOT FOUND**:
     - **ERROR**: "L'app '{appName}' (package: {packageName}) n'est pas installée sur {deviceName}"
     - **Solution**: "Options:"
       - "1. Installe l'app sur le device manuellement"
       - "2. Si tu as le fichier .apk/.ipa, je peux l'installer via MCP"
     - **Action**: "Que souhaites-tu faire?"
     - If user provides path: Use `mobile_install_app` with path
     - If user chooses manual: Wait for confirmation then re-validate
   - If **FOUND**:
     - Display: "✅ App '{appName}' détectée sur {deviceName}"

#### C. Testing Context Collection

Collect optional but recommended testing context:

1. **Story ID (Optional):**
   - "Quel est le Story ID testé? (ex: Story 2.4, ou appuie sur Enter pour skip)"
   - Store as `storyId` (empty string if skipped)

2. **Focus Area (Optional):**
   - "Quelle zone fonctionnelle veux-tu privilégier? (ex: 'Ajout d'entrée', 'Navigation', ou Enter pour exploration libre)"
   - Store as `focusArea` (empty string if skipped)

3. **Build Version (Optional):**
   - "Quelle version/build testes-tu? (ex: '1.2.0-beta3', ou Enter pour skip)"
   - Store as `buildVersion` (empty string if skipped)

#### D. Create Initial Validation Report

Copy the template from `{templateFile}` to `{output_folder}/validation-report-{project_name}.md`

Initialize frontmatter with:

```yaml
---
stepsCompleted: [1]
lastStep: 'init'
date: [current date YYYY-MM-DD]
user_name: {user_name}
tester: {user_name}
deviceId: [selected device ID]
deviceName: [selected device name]
deviceType: [simulator or physical]
appName: [app name]
packageName: [package name]
storyId: [story ID or empty]
focusArea: [focus area or empty]
buildVersion: [build version or empty]
sessionStartTime: [current timestamp]
screenshotsCount: 0
bugsCount: 0
notesCount: 0
---
```

#### E. Load Sidecar File (Session History)

Check if sidecar file exists at `{output_folder}/validation-report-{project_name}-history.md`:

- If **EXISTS**:
  - Read complete file
  - Display: "📚 Historique de sessions chargé ([X] sessions précédentes)"
  - Parse previous sessions for context

- If **NOT EXISTS**:
  - Create new sidecar file with header:
    ```markdown
    # Session History - {project_name}

    This file tracks all exploratory testing sessions for {appName}.

    ---

    ## Session 1 - {current date}

    **Device**: {deviceName} ({deviceId})
    **Story**: {storyId or 'Exploration libre'}
    **Started**: {sessionStartTime}

    _Session en cours..._
    ```
  - Display: "📝 Nouvel historique de sessions créé"

#### F. Show Welcome Message

Display welcome message to user:

```
🎯 Session de test exploratoire initialisée

**App**: {appName} (v{buildVersion or 'N/A'})
**Device**: {deviceName}
**Story**: {storyId or 'Exploration libre'}
**Focus**: {focusArea or 'Exploration générale'}

Le mode freestyle te permet d'explorer l'app librement. Je vais:
- 📸 Capturer des screenshots automatiquement
- 📝 Documenter tes découvertes
- 🐛 Classifier les bugs selon les critères de qualité
- 📊 Générer un rapport de validation structuré

Prêt à lancer l'app et commencer l'exploration.
```

## ✅ SUCCESS METRICS:

- Device detected and validated
- App installation confirmed on device
- Testing context collected (Story ID, focus area, build version)
- Validation report document created from template
- Frontmatter initialized with step 1 marked complete
- Sidecar file (session history) loaded or created
- User welcomed with clear session context
- Ready to proceed to step 2 (app launch)
- OR continuation properly routed to step-01b-continue.md

## ❌ FAILURE MODES TO AVOID:

- Proceeding without device detection
- Not validating app installation before continuing
- Creating duplicate validation reports
- Not checking for existing documents properly
- Skipping welcome message
- Not routing to step-01b-continue.md when needed
- Not initializing sidecar file for session tracking

### 5. Present AUTO-PROCEED

Display: **Lancement de l'app et initialisation de la capture...**

#### EXECUTION RULES:

- This is an initialization step with no user choices
- Proceed directly to next step after setup completion
- Use auto-proceed logic section below

#### Auto-Proceed Logic:

After successful initialization (device detected, app validated, report created, sidecar loaded):

1. **Update frontmatter** in `{outputFile}`:
   - Ensure `stepsCompleted: [1]` is set
   - Set `lastStep: 'init'`

2. **Immediately load, read entire file, then execute** `{nextStepFile}` to begin app launch and capture initialization

---

## 🚨 SYSTEM SUCCESS/FAILURE METRICS

### ✅ SUCCESS:

- Device detected via MCP mobile_list_available_devices
- App validated via MCP mobile_list_apps
- Testing context collected (Story ID, focus, version)
- Validation report created from template
- Frontmatter initialized with `stepsCompleted: [1]`
- Sidecar file loaded or created for session history
- User welcomed with session context
- Ready to proceed to step 2
- OR existing workflow properly routed to step-01b-continue.md

### ❌ SYSTEM FAILURE:

- Proceeding without device validation
- Not confirming app installation
- Creating document without proper frontmatter
- Not checking for existing documents
- Skipping sidecar file initialization
- Not routing to step-01b-continue.md when appropriate
- Loading next step before initialization complete

**Master Rule:** Skipping steps, optimizing sequences, or not following exact instructions is FORBIDDEN and constitutes SYSTEM FAILURE.

## CRITICAL STEP COMPLETION NOTE

ONLY WHEN initialization setup is complete (device detected, app validated, report created, sidecar loaded) OR continuation is properly routed, will you then:

1. Update frontmatter `stepsCompleted` to `[1]`
2. Save the output file
3. Immediately load, read entire file, then execute `{nextStepFile}` to begin app launch and capture initialization

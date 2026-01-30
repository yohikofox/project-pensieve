---

name: 'step-08-verdict'
description: 'Generate final verdict, compile complete report, and optionally integrate with GitHub/Database'

<!-- Path Definitions -->

workflow_path: '{project-root}/_bmad-output/bmb-creations/workflows/mobile-exploratory-testing'

# File References

thisStepFile: '{workflow_path}/steps/step-08-verdict.md'
workflowFile: '{workflow_path}/workflow.md'
outputFile: '{output_folder}/validation-report-{project_name}.md'
sidecarFile: '{output_folder}/validation-report-{project_name}-history.md'
reportTemplate: '{workflow_path}/templates/report-template.md'

---

# Step 8: Final Verdict & Report Generation

## STEP GOAL:

To automatically calculate the final verdict based on defined criteria (DONE/NEEDS WORK/BLOCKED), generate a complete story validation report with all required sections, save all testing artifacts, optionally create GitHub issues for bugs, optionally store report in database for analytics, and conclude the testing workflow with clear next actions.

## MANDATORY EXECUTION RULES (READ FIRST):

### Universal Rules:

- 🛑 NEVER manually override verdict - it's auto-calculated from rules
- 📖 CRITICAL: Read the complete step file before taking any action
- 📋 YOU ARE A FACILITATOR, not a content generator

### Role Reinforcement:

- ✅ You are a **QA Specialist & Mobile Testing Expert**
- ✅ Maintain collaborative, professional, and precise tone throughout

### Step-Specific Rules:

- 🎯 Focus on verdict calculation, report compilation, and integrations
- 🚫 FORBIDDEN to change verdict logic or skip criteria checks
- 💬 Approach: Objective, rules-based verdict with clear rationale
- 📋 Verdict is deterministic based on bugs, UX, system health data

## EXECUTION PROTOCOLS:

- 🎯 Calculate verdict using defined decision rules
- 💾 Compile complete report from all previous steps
- 📖 Save artifacts with timestamped filenames
- 🔗 Optionally create GitHub issues for Critical/Major bugs
- 💾 Optionally store report in database for trending

## Sequence of Instructions

### 1. Load All Report Data

Read complete {outputFile} frontmatter to extract metrics:

**Bugs Metrics**:
- `bugsLevel0` (Critical count)
- `bugsLevel1` (Major count)
- `bugsLevel2` (Minor count)
- `bugsBlockerCount` (Total blockers: Level 0 + Level 1 High Risk)

**UX Metrics**:
- `uxFeelingScore` (1-5)
- `uxRedFlagsCount`

**System Health Metrics**:
- `systemHealthTechDebt` (Clean/Acceptable/Concern)

**Confidence Metrics**:
- `confidenceScore` (1-5)

**Session Metrics**:
- `screenshotsCount`
- `explorationDuration`
- `screensVisited`
- `storyId`

### 2. Calculate Final Verdict (Auto-Generated)

**Apply Decision Rules**:

```
IF bugsLevel0 > 0:
  verdict = BLOCKED
  reason = "{bugsLevel0} bug(s) Critical trouvés - core functionality cassée"

ELSE IF bugsBlockerCount > 0:
  verdict = BLOCKED
  reason = "{bugsBlockerCount} bug(s) bloquants (Major High Risk)"

ELSE IF systemHealthTechDebt = Concern:
  verdict = BLOCKED
  reason = "Dette technique majeure introduite"

ELSE IF uxRedFlagsCount > 0:
  verdict = NEEDS WORK
  reason = "{uxRedFlagsCount} UX Red Flags non résolus"

ELSE IF uxFeelingScore < 3:
  verdict = NEEDS WORK
  reason = "UX Feeling Score insuffisant ({uxFeelingScore}/5)"

ELSE IF bugsLevel1 > 0:
  verdict = NEEDS WORK
  reason = "{bugsLevel1} bug(s) Major à corriger avant ship"

ELSE IF bugsLevel2 > 0:
  verdict = DONE
  reason = "Tous les critères DONE validés. {bugsLevel2} bug(s) Minor acceptables pour backlog."

ELSE:
  verdict = DONE
  reason = "Aucun bug trouvé, UX satisfaisante, system health OK"
```

**Display Verdict Calculation**:
```
🎯 **Calcul du Verdict Final**

**Critères évalués**:
- Bugs Critical (Level 0): {bugsLevel0} → {❌ if > 0, ✅ if 0}
- Bugs Blocker (High Risk): {bugsBlockerCount} → {❌ if > 0, ✅ if 0}
- Technical Debt: {systemHealthTechDebt} → {❌ if Concern, ✅ otherwise}
- UX Red Flags: {uxRedFlagsCount} → {❌ if > 0, ✅ if 0}
- UX Feeling Score: {uxFeelingScore}/5 → {❌ if < 3, ✅ if >= 3}
- Bugs Major (Level 1): {bugsLevel1} → {⚠️ if > 0, ✅ if 0}

**Verdict**: {verdict} {emoji}

**Raison**: {reason}
```

### 3. Write Final Verdict Section

**Append to {outputFile}** (after Confidence & Regression section):

Create **"## 6. Final Verdict"** section:

```markdown
## 6. Final Verdict

### Story Status

**Verdict**: {verdict}

{If DONE}
✅ **DONE** - Story prête pour merge et déploiement

{If NEEDS WORK}
🔄 **NEEDS WORK** - Corrections nécessaires avant merge

{If BLOCKED}
🚫 **BLOCKED** - Bugs bloquants à résoudre immédiatement

---

### Verdict Rationale

**Raison**: {reason}

**Critères Décisionnels Appliqués**:
- 🔴 Bugs Critical: {bugsLevel0}
- 🚫 Bugs Bloquants (Major High Risk): {bugsBlockerCount}
- 🏗️ Technical Debt: {systemHealthTechDebt}
- 🚩 UX Red Flags: {uxRedFlagsCount}
- 🎨 UX Feeling Score: {uxFeelingScore}/5
- 🟠 Bugs Major: {bugsLevel1}
- 🟡 Bugs Minor: {bugsLevel2}

---

### Actions Requises

{If DONE}
✅ **Aucune action bloquante**

Prochaines étapes:
- Merger la story dans main/develop
- {If bugsLevel2 > 0}Créer tickets backlog pour les {bugsLevel2} bugs Minor
- Déployer en staging pour validation finale
- Procéder au release

{If NEEDS WORK}
🔄 **Actions Requises Avant DONE**:

{If uxRedFlagsCount > 0}
- Corriger les {uxRedFlagsCount} UX Red Flags identifiés
{If uxFeelingScore < 3}
- Améliorer l'UX pour atteindre Feeling Score ≥ 3/5
{If bugsLevel1 > 0}
- Corriger les {bugsLevel1} bugs Major listés dans la section Bugs

Après corrections, relancer une session de test exploratoire.

{If BLOCKED}
🚫 **BLOQUANTS À RÉSOUDRE IMMÉDIATEMENT**:

{If bugsLevel0 > 0}
- 🔴 Corriger les {bugsLevel0} bugs Critical (crashes, data loss, core flow broken)
{If bugsBlockerCount > 0}
- 🚫 Corriger les {bugsBlockerCount} bugs Major avec Risk Score > 12
{If systemHealthTechDebt = Concern}
- 🏗️ Refactorer le code pour éliminer la dette technique majeure

**La story NE PEUT PAS être merge tant que ces blockers existent.**

```

### 4. Write Testing Summary Section

**Append to {outputFile}** (after Final Verdict section):

Create **"## 7. Testing Summary & Artifacts"** section:

```markdown
## 7. Testing Summary & Artifacts

### Session Metrics

- **Date**: {date from frontmatter}
- **Tester**: {user_name}
- **Duration**: {explorationDuration} minutes
- **Device**: {deviceName} ({deviceType})
- **App**: {appName} v{buildVersion or 'N/A'}
- **Story ID**: {storyId or 'Exploration libre'}
- **Focus Area**: {focusArea or 'Exploration générale'}

### Testing Coverage

- **Screens Visited**: {screensVisited}
- **Screenshots Captured**: {screenshotsCount}
- **Bugs Documented**: {bugsCount}
  - Critical (Level 0): {bugsLevel0}
  - Major (Level 1): {bugsLevel1}
  - Minor (Level 2): {bugsLevel2}
- **UX Improvements Proposed**: {uxImprovementsCount}
- **Notes Taken**: {notesCount}

### Quality Scores

- **UX Feeling Score**: {uxFeelingScore}/5
- **Confidence Score**: {confidenceScore}/5
- **System Health**: {systemHealthBattery}, {systemHealthNetwork}, Tech Debt = {systemHealthTechDebt}

### Artifacts Location

All testing artifacts are saved in: `{output_folder}/exploratory-testing-reports/`

- **Validation Report**: `validation-report-{storyId or project_name}-{timestamp}.md` (this file)
- **Screenshots**: `artifacts/screenshot-*.png`
- **Session History**: `validation-report-{project_name}-history.md`

---

**Rapport généré avec Mobile Exploratory Testing Workflow (BMAD)**
```

### 5. Update Frontmatter - Mark Workflow Complete

**Update frontmatter in {outputFile}**:

```yaml
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
lastStep: 'verdict'
workflowComplete: true
finalVerdict: {DONE/NEEDS WORK/BLOCKED}
verdictReason: "{reason}"
completedTimestamp: "{current ISO timestamp}"
```

### 6. Update Sidecar File - Session Complete

**Append to {sidecarFile}**:

```markdown
### Session Completed - {current time}

**Final Verdict**: {verdict}
**Reason**: {reason}
**Duration**: {explorationDuration} min
**Screenshots**: {screenshotsCount}
**Bugs**: {bugsCount} (🔴{bugsLevel0} 🟠{bugsLevel1} 🟡{bugsLevel2})

---

_Workflow complet - Rapport final généré_
```

### 7. Save Report with Timestamped Filename

**Create final report copy**:

1. Generate filename: `validation-report-{storyId or project_name}-{YYYYMMDD-HHmmss}.md`
2. Copy {outputFile} to `{output_folder}/exploratory-testing-reports/{filename}`
3. Display: "📄 Rapport sauvegardé: {filename}"

**Create artifacts directory** (if not exists):
- `{output_folder}/exploratory-testing-reports/artifacts/`
- Note: Screenshots already captured during exploration are stored here

### 8. Present FINAL MENU OPTIONS

Display:

```
🎉 **Workflow Terminé - Verdict Final Généré**

**Verdict**: {verdict} {emoji}

{If verdict = DONE}
✅ Story prête pour merge - Excellente session de test!

{If verdict = NEEDS WORK}
🔄 Corrections nécessaires - Relance une session après fixes

{If verdict = BLOCKED}
🚫 Bugs bloquants - Résolution urgente requise

---

**Intégrations Optionnelles**:

[G] GitHub Issues → Créer issues pour bugs Critical/Major
[D] Database → Sauvegarder rapport en DB centralisée (analytics/trending)
[F] Finish → Terminer le workflow (sans intégrations)
```

#### Menu Handling Logic:

**IF G (GitHub Issues)**:

1. **Check Git MCP availability**
2. **If available**:
   - For each bug Level 0 or Level 1:
     - Create GitHub issue with:
       - Title: `[Bug {levelName}] {bug title}`
       - Body: Bug description, repro steps, expected, actual
       - Labels: `bug`, `{levelName}`, `exploratory-testing`, `{storyId}`
       - Assign to: {default assignee or ask user}
       - Attach screenshot if exists
   - Display: "✅ {count} GitHub issues créées pour bugs Critical/Major"
3. **If NOT available**:
   - Display: "❌ Git MCP non disponible - issues non créées"
   - Ask: "Continue avec Database ou Finish ? [D/F]"
4. **After completion**: Redisplay menu

**IF D (Database)**:

1. **Check Database MCP availability**
2. **If available**:
   - Insert complete report into centralized DB
   - Store metrics for analytics dashboard
   - Enable trending and regression analysis
   - Display: "✅ Rapport sauvegardé en base de données centralisée"
3. **If NOT available**:
   - Display: "❌ Database MCP non disponible - rapport non stocké en DB"
   - Ask: "Continue avec GitHub ou Finish ? [G/F]"
4. **After completion**: Redisplay menu

**IF F (Finish)**:

1. Display final success message:

```
🎯 **Session de Test Exploratoire Terminée**

**Résumé**:
- Durée: {explorationDuration} min
- Écrans visités: {screensVisited}
- Screenshots: {screenshotsCount}
- Bugs trouvés: {bugsCount}
- Verdict: {verdict}

**Rapport complet**: {filename}

{If verdict = DONE}
✅ **Prochaines étapes**:
- Merger la story
- {If bugsLevel2 > 0}Créer tickets backlog pour bugs Minor
- Déployer en staging

{If verdict = NEEDS WORK}
🔄 **Actions Requises**:
{List actions from verdict section}

Relance cette commande après corrections pour re-tester.

{If verdict = BLOCKED}
🚫 **URGENT - Bugs Bloquants**:
{List blocker bugs}

NE PAS merger jusqu'à résolution complète.

---

Merci pour cette session de test rigoureuse! 🙏
```

2. **Exit workflow gracefully**

**IF Any other comments**: Help user, then redisplay menu

#### EXECUTION RULES:

- ALWAYS halt and wait for user input
- User can select G, D, F in any order or combination
- After G or D, redisplay menu to allow other integrations
- ONLY exit workflow when user selects 'F'

## CRITICAL STEP COMPLETION NOTE

ONLY WHEN verdict calculated, final verdict section written, testing summary section written, frontmatter marked workflowComplete = true, sidecar file updated, report saved with timestamped filename, and user selects 'F' to finish, will the workflow terminate successfully.

## 🚨 SYSTEM SUCCESS/FAILURE METRICS

### ✅ SUCCESS:

- Verdict auto-calculated using defined decision rules (DONE/NEEDS WORK/BLOCKED)
- Verdict rationale clearly explained
- Final verdict section written to "## 6. Final Verdict"
- Testing summary section written to "## 7. Testing Summary & Artifacts"
- Frontmatter updated with step 8 completion and workflowComplete = true
- Sidecar file updated with session completion
- Report saved with timestamped filename in exploratory-testing-reports/
- User can optionally create GitHub issues for bugs
- User can optionally store report in database
- Final success message displayed with clear next actions
- Workflow terminates gracefully when user selects 'F'

### ❌ SYSTEM FAILURE:

- Manually overriding verdict instead of using decision rules
- Not writing final verdict or testing summary sections
- Not marking frontmatter workflowComplete = true
- Not saving report with timestamped filename
- Exiting workflow without user 'F' confirmation
- Losing data during report compilation

**Master Rule:** Skipping steps, optimizing sequences, or not following exact instructions is FORBIDDEN and constitutes SYSTEM FAILURE.

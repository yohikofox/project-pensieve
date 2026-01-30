---

name: 'step-04-classify'
description: 'Classify and document all bugs found with GLaDOS levels, Murat risk scoring, and acceptance test proposals'

<!-- Path Definitions -->

workflow_path: '{project-root}/_bmad-output/bmb-creations/workflows/mobile-exploratory-testing'

# File References (all use {variable} format in file)

thisStepFile: '{workflow_path}/steps/step-04-classify.md'
nextStepFile: '{workflow_path}/steps/step-05-ux.md'
workflowFile: '{workflow_path}/workflow.md'
outputFile: '{output_folder}/validation-report-{project_name}.md'
sidecarFile: '{output_folder}/validation-report-{project_name}-history.md'

# Task References

advancedElicitationTask: '{project-root}/_bmad/core/workflows/advanced-elicitation/workflow.xml'
partyModeWorkflow: '{project-root}/_bmad/core/workflows/party-mode/workflow.md'

# Data References

bugClassificationsData: '{workflow_path}/data/bug-classifications.md'
riskScoringGuide: '{workflow_path}/data/risk-scoring-guide.md'

---

# Step 4: Bug Classification & Detailed Documentation

## STEP GOAL:

To systematically classify all bugs discovered during exploration using GLaDOS severity levels and Murat risk scoring, document them with complete reproduction steps and evidence, search for known similar bugs, generate acceptance test proposals for Critical/Major bugs, and calculate summary metrics. This step transforms raw bug discoveries into actionable, prioritized defect reports.

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

- 🎯 Focus only on bug classification and detailed documentation
- 🚫 FORBIDDEN to skip or simplify classification for any bug
- 💬 Approach: Systematic, evidence-based classification using defined criteria
- 📋 Apply GLaDOS levels and Murat risk scoring consistently
- 🔍 Use Web-Browsing for Critical/Major bugs to find known issues
- ✅ Generate acceptance test proposals (Given-When-Then) for bugs Level 0-1

## EXECUTION PROTOCOLS:

- 🎯 Classify each bug systematically: Level → Risk Score → Acceptance Test
- 💾 Write classified bugs to dedicated report section
- 📖 Load classification criteria from {bugClassificationsData}
- 🚫 FORBIDDEN to proceed without classifying all bugs
- 🔍 Search for similar bugs using Web-Browsing for Critical/Major issues
- 📚 Update frontmatter with classification summary counts

## CONTEXT BOUNDARIES:

- Available context: Bugs list from step 3 exploration
- Focus: Classification, risk scoring, detailed documentation, acceptance tests
- Limits: Don't assess UX or system health yet (steps 5-6)
- Dependencies: Step 3 complete with bugs documented

## Sequence of Instructions (Do not deviate, skip, or optimize)

### 1. Load Bug Classification Criteria

Read the classification frameworks:

**Load {bugClassificationsData}** to understand:
- GLaDOS Level 0 (Critical) criteria
- GLaDOS Level 1 (Major) criteria
- GLaDOS Level 2 (Minor) criteria
- Examples of each level

**Load {riskScoringGuide}** to understand:
- Impact scale (1-5)
- Probability scale (1-5)
- Risk Score thresholds (>12 = BLOCKER, 6-11 = Fix, <6 = Backlog)
- Examples of risk scoring

Display loading confirmation:
```
📚 **Classification Frameworks Loaded**

- GLaDOS Bug Levels (0-2)
- Murat Risk Scoring (Impact × Probability)
- Acceptance Test Templates

Prêt à classifier {bugsCount} bugs découverts.
```

### 2. Extract Bugs from Exploration Report

Read the "## 1. Exploration Summary" section from {outputFile}:

**Find "### Bugs (Temporary - To be classified)" subsection**

Extract all bugs with their temporary data:
- Bug number
- Title
- Description
- Reproduction Steps
- Expected behavior
- Actual behavior
- Screenshot filename
- Timestamp

**Display Bug List**:
```
🐛 **Bugs à Classifier**

{For each bug}
- Bug #{number}: {title}

Total: {bugsCount} bugs
```

**If NO BUGS found** (bugsCount = 0):
- Display: "✅ Aucun bug trouvé pendant l'exploration - excellente session !"
- Skip to section 7 (Write Summary)
- Proceed to UX assessment (step 5)

**If BUGS found**: Proceed to section 3

### 3. Classify Each Bug Systematically

For EACH bug in the bugs list:

**Display Current Bug**:
```
🔍 **Classification Bug #{number}/{bugsCount}**

**Titre**: {title}
**Description**: {description}
**Repro Steps**: {reproduction steps}
**Expected**: {expected}
**Actual**: {actual}
**Screenshot**: {screenshot filename}
```

#### A. Determine GLaDOS Level

**Ask the classification question**:
"D'après les critères GLaDOS, quel niveau attribues-tu à ce bug ?"

**Display Classification Guide**:
```
**GLaDOS Levels**:

🔴 **Level 0 - Critical** (Bloquants immédiats):
- Crashes - L'app se termine inopinément
- Data loss - Perte de données utilisateur
- Core flow broken - Le happy path principal est cassé
- ANR >5s - Application Not Responding plus de 5 secondes

🟠 **Level 1 - Major** (Story pas done):
- Degraded UX - Feature utilisable mais pénible
- Performance issues - Lag perceptible, >200ms de latence
- Inconsistent state - L'UI et les données se désynchronisent
- Missing error handling - L'app plante sur edge cases prévisibles

🟡 **Level 2 - Minor** (Ship mais backlog):
- UI polish - Alignements, couleurs, micro-animations
- Helpful errors - Messages d'erreur peu clairs mais non bloquants
- Edge case handling - Scénarios rares non gérés élégamment

Tape: 0, 1, ou 2
```

**Collect user input**: Level (0, 1, or 2)

**Validate**: Must be 0, 1, or 2

#### B. Calculate Risk Score (For Level 1 & 2 only)

**If Level = 0 (Critical)**:
- Skip risk scoring (Critical bugs are always blockers)
- Set `riskScore: N/A`
- Set `riskCategory: BLOCKER`
- Proceed to section C

**If Level = 1 or 2**:

**Ask for Impact**:
"Quel est l'**impact** de ce bug sur l'utilisateur?"

**Display Impact Scale**:
```
**IMPACT** (1-5):
- 5: Critical - Affecte core value proposition
- 4: High - Affecte user satisfaction majeure
- 3: Medium - Affecte confort d'usage
- 2: Low - Affecte polish
- 1: Minimal - Cosmétique

Tape: 1, 2, 3, 4, ou 5
```

**Collect user input**: Impact (1-5)

**Ask for Probability**:
"Quelle est la **probabilité** que les utilisateurs rencontrent ce bug?"

**Display Probability Scale**:
```
**PROBABILITÉ** (1-5):
- 5: Certain - Tous les users le verront
- 4: Likely - >50% des users
- 3: Possible - 10-50% des users
- 2: Unlikely - <10% des users
- 1: Rare - Edge cases extrêmes

Tape: 1, 2, 3, 4, ou 5
```

**Collect user input**: Probability (1-5)

**Calculate Risk Score**:
```
riskScore = Impact × Probability
```

**Determine Risk Category**:
- If riskScore > 12: `riskCategory: BLOCKER`
- If riskScore 6-11: `riskCategory: FIX BEFORE SHIP`
- If riskScore < 6: `riskCategory: BACKLOG`

**Display Calculation**:
```
📊 **Risk Score Calculé**

Impact: {impact}/5
Probability: {probability}/5
**Risk Score**: {riskScore} ({impact} × {probability})
**Catégorie**: {riskCategory}
```

#### C. Search for Known Similar Bugs (Level 0 & 1 only)

**If Level = 0 or 1**:

Use Web-Browsing to search for known similar bugs:

**Search Query Construction**:
```
"{appName} {keyword from bug description} bug {platform: iOS/Android}"
```

**Perform Web Search**:
- Use WebSearch tool with query
- Look for: GitHub issues, Stack Overflow, Reddit, bug trackers
- Identify if this is a known issue
- Extract workarounds if available

**Display Search Results**:
```
🔍 **Recherche Bugs Similaires**

{If found}
✅ **Bug connu trouvé**:
- Source: {URL}
- Status: {open/closed/fixed in version X}
- Workaround: {if available}

{If not found}
ℹ️ Pas de bug similaire trouvé publiquement
```

**Store search result**: `knownIssue: {true/false}`, `knownIssueURL: {URL or null}`, `workaround: {text or null}`

**If Level = 2**:
- Skip web search (minor bugs, not critical)
- Set `knownIssue: not searched`, `knownIssueURL: null`, `workaround: null`

#### D. Generate Acceptance Test Proposal (Level 0 & 1 only)

**If Level = 0 or 1**:

Generate an acceptance test in Given-When-Then format to verify the bug fix:

**Prompt for test details**:
"Ce bug Critical/Major nécessite un test d'acceptance pour valider sa résolution. Basé sur les repro steps, je propose:"

**Generate Test (AI proposes, user validates)**:
```
**Acceptance Test Proposal**:

**Given**: {Initial state from repro steps}
**When**: {Action that triggers the bug}
**Then**: {Expected behavior that should occur after fix}

**Alternative Format (Steps)**:
1. {Step 1}
2. {Step 2}
3. {Step 3}
4. **Expected**: {What should happen}
5. **Actual (Before Fix)**: {Current buggy behavior}
```

**Ask user**: "Ce test te convient ou veux-tu le modifier? [OK/Modifier/Skip]"

- **IF OK**: Store acceptance test as proposed
- **IF Modifier**: Collect user modifications, update test
- **IF Skip**: Set `acceptanceTest: User chose to skip`

**If Level = 2**:
- Skip acceptance test generation (minor bugs don't need tests)
- Set `acceptanceTest: Not required for Minor bugs`

#### E. Compile Classified Bug Entry

Create complete bug entry with all classification data:

```
Bug #{bugNumber} - {title}:
  level: {0, 1, or 2}
  levelName: {Critical/Major/Minor}
  riskImpact: {impact score or N/A for Level 0}
  riskProbability: {probability score or N/A for Level 0}
  riskScore: {calculated score or N/A for Level 0}
  riskCategory: {BLOCKER/FIX BEFORE SHIP/BACKLOG}
  knownIssue: {true/false/not searched}
  knownIssueURL: {URL or null}
  workaround: {text or null}
  acceptanceTest: {test text or "Not required for Minor bugs"}
  title: {title}
  description: {description}
  reproductionSteps: {steps}
  expected: {expected behavior}
  actual: {actual behavior}
  screenshot: {filename}
  timestamp: {timestamp}
  bugId: BUG-{timestamp}-{sequential}
```

**Display Classified Bug**:
```
✅ **Bug #{number} Classifié**

🔴/🟠/🟡 Level {level} - {levelName}
📊 Risk: {riskScore} - {riskCategory}
🔍 Known Issue: {Yes/No}
✅ Acceptance Test: {Generated/Not required}
```

#### F. Move to Next Bug

**If more bugs remaining**:
- Display: "Bug #{number}/{bugsCount} classifié. Prochain bug..."
- Return to section 3 (Classify Each Bug) for next bug

**If all bugs classified**:
- Display: "🎉 Tous les bugs classifiés ! ({bugsCount}/{bugsCount})"
- Proceed to section 4

### 4. Calculate Classification Summary

Calculate counts and metrics across all classified bugs:

**Counts by Level**:
- `level0Count`: Count of bugs with level = 0
- `level1Count`: Count of bugs with level = 1
- `level2Count`: Count of bugs with level = 2

**Counts by Risk Category** (Level 1 & 2 only):
- `blockerCount`: Count of bugs with riskCategory = BLOCKER (includes all Level 0 + Level 1/2 with score >12)
- `fixBeforeShipCount`: Count of bugs with riskCategory = FIX BEFORE SHIP
- `backlogCount`: Count of bugs with riskCategory = BACKLOG

**Other Metrics**:
- `knownIssuesCount`: Count of bugs with knownIssue = true
- `acceptanceTestsGenerated`: Count of acceptance tests created

**Display Summary**:
```
📊 **Classification Summary**

**Par Niveau**:
- 🔴 Critical (Level 0): {level0Count}
- 🟠 Major (Level 1): {level1Count}
- 🟡 Minor (Level 2): {level2Count}

**Par Risque**:
- 🚫 BLOCKER: {blockerCount} → ❌ if > 0
- ⚠️ FIX BEFORE SHIP: {fixBeforeShipCount}
- 📋 BACKLOG: {backlogCount}

**Autres Metrics**:
- 🔍 Known Issues: {knownIssuesCount}
- ✅ Acceptance Tests: {acceptanceTestsGenerated}

{If blockerCount > 0}
⚠️ **ATTENTION**: {blockerCount} bugs bloquants trouvés - Story ne peut pas être marquée DONE
```

### 5. Write Classified Bugs to Report

**Append to {outputFile}** (after Exploration Summary section):

Create **"## 2. Bugs Classification & Documentation"** section:

```markdown
## 2. Bugs Classification & Documentation

### Summary

- 🔴 **Critical (Level 0)**: {level0Count} → {❌ BLOCKER if > 0, ✅ if 0}
- 🟠 **Major (Level 1)**: {level1Count}
  - High Risk (≥12): {count of level 1 with score ≥12} → {❌ BLOCKER if > 0}
  - Medium Risk (6-11): {count of level 1 with score 6-11}
- 🟡 **Minor (Level 2)**: {level2Count} → 📋 Backlog

**Total Bugs**: {bugsCount}
**Blockers**: {blockerCount}

---

{For each classified bug, sorted by level (0 first, then 1, then 2), then by risk score descending}

### Bug #{bugNumber}: {title}

**Classification**:
- **Level**: {level} - {levelName} {emoji 🔴/🟠/🟡}
- **Risk Score**: {riskScore} (Impact: {riskImpact}, Probability: {riskProbability})
- **Category**: {riskCategory}
- **Bug ID**: {bugId}

**Details**:
- **Description**: {description}
- **Reproduction Steps**:
  1. {step 1}
  2. {step 2}
  3. {step N}
- **Expected Behavior**: {expected}
- **Actual Behavior**: {actual}
- **Screenshot**: {screenshot filename}
- **Timestamp**: {timestamp}

{If knownIssue = true}
**Known Issue**:
- Source: {knownIssueURL}
- Workaround: {workaround or 'None available'}

{If acceptanceTest exists and not "Not required"}
**Acceptance Test Proposal**:
```gherkin
{acceptanceTest in Given-When-Then or Steps format}
```

---
```

### 6. Update Frontmatter

**Update frontmatter in {outputFile}**:

```yaml
stepsCompleted: [1, 2, 3, 4]
lastStep: 'classify'
bugsLevel0: {level0Count}
bugsLevel1: {level1Count}
bugsLevel2: {level2Count}
bugsBlockerCount: {blockerCount}
bugsFixBeforeShip: {fixBeforeShipCount}
bugsBacklog: {backlogCount}
knownIssuesFound: {knownIssuesCount}
acceptanceTestsGenerated: {acceptanceTestsGenerated}
```

### 7. Update Sidecar File

**Append to {sidecarFile}**:

```markdown
### Bug Classification Completed - {current time}
**Total Bugs**: {bugsCount}
**Critical**: {level0Count}
**Major**: {level1Count}
**Minor**: {level2Count}
**Blockers**: {blockerCount}

{If blockerCount > 0}
⚠️ **{blockerCount} bugs bloquants identifiés**
```

### 8. Present MENU OPTIONS

Display:

```
**Prochaine étape: Évaluation UX**

Options:

[A] Advanced Elicitation → Analyser bugs difficiles sous différents angles
[P] Party Mode → Brainstormer avec équipe d'agents sur bugs complexes
[C] Continue → Passer à l'évaluation UX (Feeling Score, Red Flags)
```

#### Menu Handling Logic:

- **IF A**: Execute {advancedElicitationTask}
  - Focus: Difficult-to-reproduce bugs, complex scenarios
  - After completion, redisplay [Menu Options](#8-present-menu-options)

- **IF P**: Execute {partyModeWorkflow}
  - Recommended if {blockerCount} > 0
  - Agents to consult: GLaDOS (bug analysis), Murat (risk assessment), Sally (UX impact)
  - After completion, redisplay [Menu Options](#8-present-menu-options)

- **IF C**:
  1. Save all classified bugs to {outputFile}
  2. Update frontmatter with step 4 completion
  3. Update sidecar file
  4. Load, read entire file, then execute {nextStepFile}

- **IF Any other comments or queries**: Help user respond, then redisplay [Menu Options](#8-present-menu-options)

#### EXECUTION RULES:

- ALWAYS halt and wait for user input after presenting menu
- ONLY proceed to next step when user selects 'C'
- After Advanced Elicitation or Party Mode, return to this menu
- User can chat or ask questions - always respond and then redisplay menu

## CRITICAL STEP COMPLETION NOTE

ONLY WHEN 'C' is selected, all bugs are classified with GLaDOS levels and risk scores, web searches completed for Critical/Major bugs, acceptance tests generated for Critical/Major bugs, classified bugs written to report, frontmatter updated with step 4 completion, will you then:

1. Verify "## 2. Bugs Classification & Documentation" section exists in {outputFile}
2. Verify all bugs have complete classification data
3. Update frontmatter `stepsCompleted` to `[1, 2, 3, 4]`
4. Set `lastStep: 'classify'`
5. Save the output file
6. Immediately load, read entire file, then execute `{nextStepFile}` to begin UX assessment

## 🚨 SYSTEM SUCCESS/FAILURE METRICS

### ✅ SUCCESS:

- All bugs classified with GLaDOS levels (0, 1, or 2)
- Risk scores calculated for Level 1 & 2 bugs (Impact × Probability)
- Web searches performed for Critical/Major bugs to find known issues
- Acceptance tests generated for Critical/Major bugs
- Classification summary calculated with counts by level and risk category
- Classified bugs written to "## 2. Bugs Classification & Documentation" section
- Frontmatter updated with classification metrics and step 4 completion
- Sidecar file updated with classification completion event
- User can access Advanced Elicitation or Party Mode for complex bugs
- Next step (UX assessment) loaded when user selects 'C'

### ❌ SYSTEM FAILURE:

- Skipping classification for any bug
- Not calculating risk scores for Level 1/2 bugs
- Not using GLaDOS criteria consistently
- Not searching for known issues for Critical/Major bugs
- Not generating acceptance tests for bugs that require them
- Proceeding without user 'C' confirmation
- Not writing classified bugs to report
- Not updating frontmatter with step 4 completion
- Losing bug data during classification process

**Master Rule:** Skipping steps, optimizing sequences, or not following exact instructions is FORBIDDEN and constitutes SYSTEM FAILURE.

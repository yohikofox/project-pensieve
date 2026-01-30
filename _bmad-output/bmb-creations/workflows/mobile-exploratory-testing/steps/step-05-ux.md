---

name: 'step-05-ux'
description: 'Assess user experience with Feeling Score, identify UX Red Flags, and propose improvements'

<!-- Path Definitions -->

workflow_path: '{project-root}/_bmad-output/bmb-creations/workflows/mobile-exploratory-testing'

# File References

thisStepFile: '{workflow_path}/steps/step-05-ux.md'
nextStepFile: '{workflow_path}/steps/step-06-system.md'
workflowFile: '{workflow_path}/workflow.md'
outputFile: '{output_folder}/validation-report-{project_name}.md'
sidecarFile: '{output_folder}/validation-report-{project_name}-history.md'

# Task References

advancedElicitationTask: '{project-root}/_bmad/core/workflows/advanced-elicitation/workflow.xml'
partyModeWorkflow: '{project-root}/_bmad/core/workflows/party-mode/workflow.md'
brainstormingWorkflow: '{project-root}/_bmad/core/workflows/brainstorming/workflow.md'

---

# Step 5: UX Assessment

## STEP GOAL:

To evaluate the overall user experience through a subjective Feeling Score, identify UX Red Flags that block a quality release, and propose concrete improvements for the UX designer, ensuring the app meets usability and accessibility standards before marking the story done.

## MANDATORY EXECUTION RULES (READ FIRST):

### Universal Rules:

- 🛑 NEVER generate content without user input
- 📖 CRITICAL: Read the complete step file before taking any action
- 🔄 CRITICAL: When loading next step with 'C', ensure entire file is read
- 📋 YOU ARE A FACILITATOR, not a content generator

### Role Reinforcement:

- ✅ You are a **QA Specialist & Mobile Testing Expert**
- ✅ Maintain collaborative, professional, and precise tone throughout

### Step-Specific Rules:

- 🎯 Focus only on UX evaluation and improvement proposals
- 🚫 FORBIDDEN to skip Red Flags assessment
- 💬 Approach: Empathetic UX analysis with user-centric mindset
- 📋 Threshold: Feeling Score ≥ 3 required for DONE verdict

## EXECUTION PROTOCOLS:

- 🎯 Capture Feeling Score (1-5) from tester's perspective
- 💾 Identify and document all UX Red Flags
- 📖 Generate improvement proposals with priority and rationale
- 🚫 FORBIDDEN to proceed if Red Flags not fully documented

## Sequence of Instructions

### 1. Capture Feeling Score

Ask tester for overall UX perception:

**Display Feeling Score Guide**:
```
🎨 **UX Feeling Score**

Sur une échelle de 1-5, comment as-tu RESSENTI l'expérience globale de cette feature ?

- **5**: Délicieux - Moment de "wow!" 🤩
- **4**: Agréable - Intuitive et fluide ✨
- **3**: OK - Fonctionne sans friction majeure ✅
- **2**: Confus - Nécessite effort pour comprendre 😕
- **1**: Frustrant - Expérience pénible 😤

Tape: 1, 2, 3, 4, ou 5
```

**Collect user input**: Feeling Score (1-5)

**Validate**: Must be 1-5

**If Feeling Score < 3**:
- Display: "⚠️ Feeling Score < 3 signifie que l'UX n'est pas acceptable pour DONE"
- Ask: "Peux-tu décrire ce qui rend l'expérience confuse ou frustrante ?"
- Collect detailed feedback for improvement proposals

**Display Score**:
```
📊 **Feeling Score**: {feelingScore}/5

{If >= 3}
✅ Score acceptable pour considérer la story DONE (si autres critères OK)

{If < 3}
❌ Score insuffisant - UX doit être améliorée avant DONE
```

### 2. Identify UX Red Flags

**Display Red Flags Checklist**:
```
🚩 **UX Red Flags** (bloquants si présents)

As-tu observé l'un de ces problèmes pendant l'exploration ?

1. **Usability**: L'utilisateur doit deviner comment utiliser la feature
2. **Feedback**: Action sans feedback visuel ou sonore
3. **Frustration**: Parcours utilisateur frustrant même si fonctionnel
4. **Accessibility**: Problèmes de contraste, taille de touche, ou lecteur d'écran

Pour chacun, réponds: O (Oui observé) ou N (Non, pas observé)
```

**For each Red Flag type**:

**Ask**: "Red Flag {type}: [O/N]"

**If O (Oui)**:
- Ask: "Décris le problème {type} que tu as observé:"
- Collect detailed description
- Ask: "Sur quel(s) écran(s) ? (référence screenshots si possible)"
- Collect screen references
- Store Red Flag:
  ```
  redFlag{N} = {
    type: {Usability/Feedback/Frustration/Accessibility},
    description: {user description},
    screens: {screen references},
    screenshot: {associated screenshot if any},
    timestamp: {current timestamp}
  }
  ```

**Count Red Flags**:
- `redFlagsCount`: Total Red Flags identified

**Display Red Flags Summary**:
```
🚩 **Red Flags Identifiés**: {redFlagsCount}

{If redFlagsCount > 0}
{List each Red Flag with type and brief description}

⚠️ **{redFlagsCount} Red Flags trouvés - Story ne peut pas être DONE tant qu'ils ne sont pas résolus**

{If redFlagsCount = 0}
✅ Aucun Red Flag UX identifié
```

### 3. Generate UX Improvement Proposals

**Prompt for Improvements**:
```
💡 **Propositions d'Amélioration UX**

Basé sur ton exploration, as-tu des suggestions pour améliorer l'UX ?
(Incluant les Red Flags à corriger + opportunités d'amélioration)

Veux-tu proposer des améliorations ? [Y/N]
```

**If Y (Oui)**:

**Loop for each improvement**:

1. **Collect Improvement Details**:
   - "Décris l'amélioration proposée:"
   - "Quelle catégorie: Usability / Accessibility / Visual / Performance / Other"
   - "Quel est l'état actuel ? (ce qui ne va pas)"
   - "Quelle est ta solution proposée ?"
   - "Pourquoi cette amélioration est importante ?"
   - "Priorité: High / Medium / Low"

2. **Capture Screenshot** (if relevant):
   - "Veux-tu capturer un screenshot pour illustrer ? [Y/N]"
   - If Y: Call MCP `mobile_take_screenshot`, filename: `ux-improvement-{improvementCounter+1:02d}-{timestamp}.png`

3. **Create Improvement Entry**:
   ```
   uxImprovement{N} = {
     id: UX-{timestamp}-{sequential},
     category: {Usability/Accessibility/Visual/Performance/Other},
     currentState: {description},
     proposedImprovement: {solution},
     rationale: {why important},
     priority: {High/Medium/Low},
     screenshot: {filename or 'none'},
     timestamp: {current timestamp},
     assignee: "UX Designer"
   }
   ```

4. **Increment** `improvementCounter`

5. **Ask**: "Autre amélioration à proposer ? [Y/N]"
   - If Y: Loop to next improvement
   - If N: Exit loop

**If N (No improvements)**:
- Display: "OK, pas de propositions d'amélioration pour le moment."
- Set `improvementCount: 0`

### 4. Write UX Assessment to Report

**Append to {outputFile}** (after Bugs Classification section):

Create **"## 3. UX Assessment"** section:

```markdown
## 3. UX Assessment

### Feeling Score

**Score**: {feelingScore}/5

{Emoji based on score: 🤩(5), ✨(4), ✅(3), 😕(2), 😤(1)}

**Interprétation**:
{If >= 3}
✅ UX acceptable - L'expérience utilisateur est satisfaisante

{If < 3}
❌ UX insuffisante - L'expérience nécessite des améliorations avant DONE

{If feedbackForLowScore exists}
**Feedback**: {feedbackForLowScore}

---

### UX Red Flags

**Total**: {redFlagsCount} → {❌ BLOCKER if > 0, ✅ if 0}

{If redFlagsCount > 0}
{For each Red Flag}
#### 🚩 Red Flag #{N}: {type}

- **Description**: {description}
- **Écrans concernés**: {screens}
- **Screenshot**: {screenshot or 'N/A'}
- **Timestamp**: {timestamp}

---

{If redFlagsCount = 0}
✅ Aucun Red Flag UX identifié - L'app respecte les standards d'usabilité et d'accessibilité

---

### UX Improvement Proposals

**Total Propositions**: {improvementCount}

{If improvementCount > 0}
{For each improvement, sorted by priority (High first)}
#### 💡 Improvement #{N}: {id}

- **Catégorie**: {category}
- **Priorité**: {priority}
- **État Actuel**: {currentState}
- **Amélioration Proposée**: {proposedImprovement}
- **Rationale**: {rationale}
- **Screenshot**: {screenshot or 'None'}
- **Pour**: UX Designer
- **Timestamp**: {timestamp}

---

{If improvementCount = 0}
Aucune proposition d'amélioration soumise.

```

### 5. Update Frontmatter

**Update frontmatter in {outputFile}**:

```yaml
stepsCompleted: [1, 2, 3, 4, 5]
lastStep: 'ux'
uxFeelingScore: {feelingScore}
uxRedFlagsCount: {redFlagsCount}
uxImprovementsCount: {improvementCount}
```

### 6. Update Sidecar File

**Append to {sidecarFile}**:

```markdown
### UX Assessment Completed - {current time}
**Feeling Score**: {feelingScore}/5
**Red Flags**: {redFlagsCount}
**Improvements Proposed**: {improvementCount}

{If redFlagsCount > 0 or feelingScore < 3}
⚠️ **UX issues identifiés**
```

### 7. Present MENU OPTIONS

Display:

```
**Prochaine étape: System Health Assessment**

Options:

[A] Advanced Elicitation → Explorer alternatives UX
[P] Party Mode → Consulter Sally (UX Designer agent) pour perspective
[B] Brainstorm UX → Session créative pour générer plus d'idées d'amélioration
[C] Continue → Passer à l'évaluation System Health
```

#### Menu Handling Logic:

- **IF A**: Execute {advancedElicitationTask} (focus UX alternatives), then redisplay menu
- **IF P**: Execute {partyModeWorkflow} (consult Sally + other agents), then redisplay menu
- **IF B**: Execute {brainstormingWorkflow} (creative UX session), then redisplay menu
- **IF C**: Save UX assessment to {outputFile}, update frontmatter, load {nextStepFile}
- **IF Any other comments**: Help user, then redisplay menu

#### EXECUTION RULES:

- ALWAYS halt and wait for user input
- ONLY proceed to next step when user selects 'C'

## CRITICAL STEP COMPLETION NOTE

ONLY WHEN 'C' is selected, Feeling Score captured, Red Flags identified, improvements proposed, UX assessment written to report, frontmatter updated with step 5 completion, will you then:

1. Verify "## 3. UX Assessment" section exists in {outputFile}
2. Update frontmatter `stepsCompleted` to `[1, 2, 3, 4, 5]`
3. Set `lastStep: 'ux'`
4. Save the output file
5. Immediately load, read entire file, then execute `{nextStepFile}` for system health assessment

## 🚨 SYSTEM SUCCESS/FAILURE METRICS

### ✅ SUCCESS:

- Feeling Score (1-5) captured from tester
- All 4 Red Flag types assessed
- Red Flags documented with descriptions and screens
- Improvement proposals collected with category, priority, rationale
- UX assessment written to "## 3. UX Assessment" section
- Frontmatter updated with UX metrics and step 5 completion
- User can access Advanced Elicitation, Party Mode, or Brainstorming
- Next step (system health) loaded when user selects 'C'

### ❌ SYSTEM FAILURE:

- Skipping Feeling Score collection
- Not assessing all Red Flag types
- Proceeding without documenting identified Red Flags
- Not writing UX assessment to report
- Not updating frontmatter with step 5 completion

**Master Rule:** Skipping steps, optimizing sequences, or not following exact instructions is FORBIDDEN and constitutes SYSTEM FAILURE.

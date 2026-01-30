---

name: 'step-07-confidence'
description: 'Capture tester confidence score and perform regression analysis using Vector DB'

<!-- Path Definitions -->

workflow_path: '{project-root}/_bmad-output/bmb-creations/workflows/mobile-exploratory-testing'

# File References

thisStepFile: '{workflow_path}/steps/step-07-confidence.md'
nextStepFile: '{workflow_path}/steps/step-08-verdict.md'
workflowFile: '{workflow_path}/workflow.md'
outputFile: '{output_folder}/validation-report-{project_name}.md'
sidecarFile: '{output_folder}/validation-report-{project_name}-history.md'

# Task References

advancedElicitationTask: '{project-root}/_bmad/core/workflows/advanced-elicitation/workflow.xml'
partyModeWorkflow: '{project-root}/_bmad/core/workflows/party-mode/workflow.md'

---

# Step 7: Confidence Score & Regression Analysis

## STEP GOAL:

To capture the tester's subjective confidence in production-readiness, identify explicit blockers if confidence is low, perform semantic regression analysis using Vector DB to detect patterns and historical issues, and prepare comprehensive context for the final verdict.

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

- 🎯 Focus only on confidence assessment and regression analysis
- 🚫 FORBIDDEN to skip "What would make it a 5?" question if confidence < 4
- 💬 Approach: Empathetic confidence elicitation + data-driven regression detection
- 📋 Use Vector DB for semantic search of similar bugs in history

## EXECUTION PROTOCOLS:

- 🎯 Capture Confidence Score (1-5) from tester
- 💾 Identify blockers if Confidence < 4
- 📖 Perform Vector DB regression analysis if available
- 🚫 FORBIDDEN to proceed without documenting confidence rationale

## Sequence of Instructions

### 1. Capture Confidence Score

**Display Confidence Question**:
```
🎯 **Confidence Score**

À quel point es-tu CONFIANT que cette story est prête pour production ?

Pense à l'ensemble de l'expérience: bugs, UX, performance, stabilité.

- **5**: Je shipperais ça à ma grand-mère sans hésiter 🚀
- **4**: Solide, quelques réserves mineures ✅
- **3**: Ça marche mais j'ai des doutes 🤔
- **2**: Plusieurs problèmes me préoccupent 😟
- **1**: Please don't ship this 🚫

Tape: 1, 2, 3, 4, ou 5
```

**Collect user input**: confidenceScore (1-5)

**Validate**: Must be 1-5

**Display Score**:
```
📊 **Confidence Score**: {confidenceScore}/5

{Emoji based on score: 🚀(5), ✅(4), 🤔(3), 😟(2), 🚫(1)}
```

### 2. Identify Blockers (If Confidence < 4)

**If confidenceScore < 4**:

**Ask the critical question**:
```
💬 **Question Importante**

Qu'est-ce qui te ferait passer à un score de 5/5 ?

Identifie les blockers explicites ou préoccupations qui t'empêchent d'être pleinement confiant.
```

**Collect user input**: confidenceBlockers (detailed text)

**Parse blockers** (AI helps organize):
- Extract individual blocker items
- Categorize: Bug-related / UX-related / Technical / Other
- Create numbered list

**Display Blockers**:
```
🚧 **Blockers Identifiés** ({confidenceScore}/5)

{For each blocker}
{N}. {blocker description} ({category})

Pour atteindre 5/5, ces blockers doivent être résolus.
```

**If confidenceScore >= 4**:
- Set `confidenceBlockers: 'Aucun blocker majeur - Confidence élevée'`
- Display: "✅ Confiance élevée - Pas de blockers explicites identifiés"

### 3. Perform Vector DB Regression Analysis

**Check Vector DB Availability**:

**If Vector DB MCP available**:

**Display**:
```
🔍 **Analyse de Régression (Vector DB)**

Recherche sémantique de bugs et patterns similaires dans l'historique...
```

**Perform Semantic Search**:

1. **Search for Similar Bugs**:
   - Query Vector DB with bug descriptions from this session
   - Semantic similarity threshold: > 0.75
   - Look for: Previous occurrences, related issues, patterns

2. **Extract Findings**:
   ```
   similarBugsFound = [
     {
       historicalBugId: {ID from Vector DB},
       similarity: {0.0-1.0},
       description: {historical bug description},
       resolution: {how it was fixed or 'unresolved'},
       dateFound: {when it was discovered}
     }
   ]
   ```

3. **Pattern Analysis**:
   - Identify recurring patterns (e.g., "Authentication bugs")
   - Calculate recurrence rate
   - Assess regression risk (High/Medium/Low)

**Display Vector DB Results**:
```
📊 **Résultats Vector DB**

**Bugs Similaires Trouvés**: {count of similarBugsFound}

{If count > 0}
{For each similar bug}
- **Bug Historique**: {historicalBugId} (Similarité: {similarity}%)
  - **Description**: {description}
  - **Résolution**: {resolution or 'Non résolu'}
  - **Date**: {dateFound}

**Patterns Récurrents**: {list identified patterns}

**Regression Risk**: {High/Medium/Low}

{If regression risk High}
⚠️ **Risque de régression élevé** - Patterns récurrents détectés

{If count = 0}
✅ Aucun bug similaire trouvé dans l'historique
```

**If Vector DB NOT available**:

**Display**:
```
ℹ️ **Vector DB Non Disponible**

Analyse de régression sémantique désactivée.
L'analyse de régression sera manuelle basée sur mémoire du testeur.
```

**Ask Manual Regression Check**:
"As-tu observé des bugs ou patterns similaires à des sessions précédentes ? [Y/N]"

**If Y**:
- Collect: `manualRegressionNotes`
- Display: "📝 Régression manuelle documentée"

**If N**:
- Set `manualRegressionNotes: 'Aucune régression observée'`
- Display: "✅ Pas de régression apparente"

### 4. Write Confidence & Regression to Report

**Append to {outputFile}** (after System Health section):

Create **"## 5. Confidence & Regression Analysis"** section:

```markdown
## 5. Confidence & Regression Analysis

### Confidence Score

**Score**: {confidenceScore}/5

{Emoji: 🚀(5), ✅(4), 🤔(3), 😟(2), 🚫(1)}

**Interprétation**:
{If >= 4}
✅ Confiance élevée - Le testeur estime que la story est prête pour production

{If = 3}
⚠️ Confiance modérée - Le testeur a des doutes sur la production-readiness

{If <= 2}
❌ Confiance faible - Le testeur déconseille le ship en l'état

---

{If confidenceScore < 4}
### Blockers Identifiés

Pour atteindre 5/5 de confiance, les blockers suivants doivent être résolus:

{For each blocker}
{N}. **{blocker description}** ({category})

---

{If confidenceScore >= 4}
✅ Aucun blocker explicite - Confiance élevée du testeur

---

### Regression Analysis

{If Vector DB available}
**Méthode**: Analyse sémantique automatique (Vector DB)

**Bugs Similaires Trouvés**: {count}

{If count > 0}
{For each similar bug}
#### Bug Historique: {historicalBugId}

- **Similarité**: {similarity}%
- **Description**: {description}
- **Résolution**: {resolution or 'Non résolu'}
- **Date Découverte**: {dateFound}

---

**Patterns Récurrents Identifiés**:
{List patterns}

**Regression Risk**: {High/Medium/Low}

{If regression risk High}
⚠️ **Attention**: Risque de régression élevé - Patterns récurrents détectés

{If count = 0}
✅ Aucun bug similaire trouvé dans l'historique - Pas de régression apparente

{If Vector DB not available}
**Méthode**: Analyse manuelle

**Notes**: {manualRegressionNotes or 'Aucune régression observée'}

```

### 5. Update Frontmatter

**Update frontmatter in {outputFile}**:

```yaml
stepsCompleted: [1, 2, 3, 4, 5, 6, 7]
lastStep: 'confidence'
confidenceScore: {confidenceScore}
confidenceBlockers: {confidenceBlockers or 'None'}
regressionRisk: {High/Medium/Low or 'Not analyzed'}
similarBugsFound: {count or 0}
```

### 6. Update Sidecar File

**Append to {sidecarFile}**:

```markdown
### Confidence & Regression Analysis Completed - {current time}
**Confidence Score**: {confidenceScore}/5
{If confidenceScore < 4}
**Blockers**: {count of blockers} identifiés
**Regression Risk**: {regressionRisk}
**Similar Bugs Found**: {similarBugsFound}
```

### 7. Present MENU OPTIONS

Display:

```
**Prochaine étape: Verdict Final & Génération du Rapport**

Options:

[A] Advanced Elicitation → Analyser confiance sous différents angles
[P] Party Mode → Consulter équipe si bugs critiques ou confiance faible
[C] Continue → Passer au verdict final et génération du rapport complet
```

#### Menu Handling Logic:

- **IF A**: Execute {advancedElicitationTask}
  - Focus: Explore factors affecting confidence
  - After completion, redisplay menu

- **IF P**: Execute {partyModeWorkflow}
  - Recommended if confidenceScore < 3
  - Consult: GLaDOS (bugs), Murat (risk), Winston (system), Carson (confidence)
  - After completion, redisplay menu

- **IF C**:
  1. Save confidence & regression analysis to {outputFile}
  2. Update frontmatter with step 7 completion
  3. Update sidecar file
  4. Load, read entire file, then execute {nextStepFile}

- **IF Any other comments**: Help user, then redisplay menu

#### EXECUTION RULES:

- ALWAYS halt and wait for user input
- ONLY proceed to next step when user selects 'C'

## CRITICAL STEP COMPLETION NOTE

ONLY WHEN 'C' is selected, Confidence Score captured, blockers identified (if < 4), Vector DB regression analysis performed (if available), confidence & regression written to report, frontmatter updated with step 7 completion, will you then:

1. Verify "## 5. Confidence & Regression Analysis" section exists in {outputFile}
2. Update frontmatter `stepsCompleted` to `[1, 2, 3, 4, 5, 6, 7]`
3. Set `lastStep: 'confidence'`
4. Save the output file
5. Immediately load, read entire file, then execute `{nextStepFile}` for final verdict generation

## 🚨 SYSTEM SUCCESS/FAILURE METRICS

### ✅ SUCCESS:

- Confidence Score (1-5) captured from tester
- Blockers identified and documented if Confidence < 4
- Vector DB regression analysis performed (if available)
- Manual regression check if Vector DB unavailable
- Similar bugs found and documented
- Confidence & regression written to "## 5. Confidence & Regression Analysis" section
- Frontmatter updated with confidence metrics and step 7 completion
- User can access Advanced Elicitation or Party Mode
- Next step (final verdict) loaded when user selects 'C'

### ❌ SYSTEM FAILURE:

- Skipping Confidence Score collection
- Not asking "What would make it a 5?" if confidence < 4
- Not performing regression analysis (Vector DB or manual)
- Not writing confidence & regression to report
- Not updating frontmatter with step 7 completion

**Master Rule:** Skipping steps, optimizing sequences, or not following exact instructions is FORBIDDEN and constitutes SYSTEM FAILURE.

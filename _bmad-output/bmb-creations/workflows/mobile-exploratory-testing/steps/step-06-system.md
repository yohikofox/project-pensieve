---

name: 'step-06-system'
description: 'Assess system health including battery/performance, network resilience, and technical debt'

<!-- Path Definitions -->

workflow_path: '{project-root}/_bmad-output/bmb-creations/workflows/mobile-exploratory-testing'

# File References

thisStepFile: '{workflow_path}/steps/step-06-system.md'
nextStepFile: '{workflow_path}/steps/step-07-confidence.md'
workflowFile: '{workflow_path}/workflow.md'
outputFile: '{output_folder}/validation-report-{project_name}.md'
sidecarFile: '{output_folder}/validation-report-{project_name}-history.md'

---

# Step 6: System Health Assessment

## STEP GOAL:

To evaluate the technical health of the system including battery drain, performance characteristics, network resilience under various conditions, and technical debt introduced, ensuring the implementation follows established patterns and doesn't create long-term maintenance issues.

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

- 🎯 Focus only on system-level technical health
- 🚫 FORBIDDEN to skip Technical Debt assessment
- 💬 Approach: Technical rigor with developer-centric language
- 📋 Threshold: Technical Debt = Concern is a blocker for DONE

## EXECUTION PROTOCOLS:

- 🎯 Assess Battery/Performance, Network Resilience, Technical Debt
- 💾 Document technical concerns with detailed notes
- 📖 Flag Technical Debt = Concern as blocker
- 🚫 FORBIDDEN to proceed without assessing all three areas

## Sequence of Instructions

### 1. Battery & Performance Assessment

**Display Assessment Guide**:
```
🔋 **Battery & Performance**

Pendant ta session de test, as-tu observé:
- Battery drain excessif ?
- Lag ou freeze perceptibles ?
- Temps de chargement longs ?
- Consommation CPU/mémoire inhabituelle ?

Quel est ton évaluation globale ?

- **OK**: Aucun problème de performance détecté
- **Concern**: Problèmes de performance ou battery drain observés

Tape: OK ou Concern
```

**Collect user input**: batteryPerformance (OK / Concern)

**If Concern**:
- Ask: "Décris les problèmes de performance ou battery drain observés:"
- Collect detailed notes: `batteryPerformanceNotes`
- Display: "⚠️ Performance concerns documentés - à investiguer"

**If OK**:
- Set `batteryPerformanceNotes: 'Aucun problème détecté'`
- Display: "✅ Battery/Performance OK"

### 2. Network Resilience Assessment

**Display Assessment Guide**:
```
🌐 **Network Resilience**

As-tu testé le comportement de l'app dans différentes conditions réseau ?

- Mode avion (pas de réseau)
- 3G lent (réseau dégradé)
- Reconnexion après perte de réseau
- Passage WiFi ↔ Data

Statut du test:

- **Tested**: Oui, j'ai testé les conditions réseau
- **Not Tested**: Non, pas testé pendant cette session
- **N/A**: Cette feature ne nécessite pas de réseau

Tape: Tested, Not Tested, ou N/A
```

**Collect user input**: networkResilience (Tested / Not Tested / N/A)

**If Tested**:
- Ask: "Décris le comportement de l'app dans différentes conditions réseau:"
- Collect notes: `networkResilienceNotes`
- Ask: "Résultat global: OK ou Concern ?"
- Collect: `networkResilienceResult` (OK / Concern)
- If Concern: Display "⚠️ Network resilience issues - à corriger"

**If Not Tested**:
- Set `networkResilienceNotes: 'Non testé pendant cette session'`
- Set `networkResilienceResult: 'Not Tested'`
- Display: "ℹ️ Network resilience non testé - à considérer pour futures sessions"

**If N/A**:
- Set `networkResilienceNotes: 'Feature ne nécessite pas de réseau'`
- Set `networkResilienceResult: 'N/A'`
- Display: "ℹ️ Network resilience non applicable à cette feature"

### 3. Technical Debt Assessment

**Display Assessment Guide**:
```
🏗️ **Technical Debt**

Basé sur ton observation du code, des patterns, et de l'implémentation:

- **Clean**: L'implémentation suit les patterns établis, code maintenable
- **Acceptable**: Quelques compromis mineurs, mais pas de dette majeure
- **Concern**: Dette technique introduite - code non maintenable, patterns violés (BLOCKER)

Évaluation:

Tape: Clean, Acceptable, ou Concern
```

**Collect user input**: technicalDebt (Clean / Acceptable / Concern)

**If Concern**:
- Display: "🚫 **BLOCKER**: Technical Debt = Concern"
- Ask: "Décris la dette technique introduite:"
- Collect detailed justification: `technicalDebtJustification`
- Display: "⚠️ Dette technique majeure - Story BLOCKED jusqu'à résolution"

**If Acceptable**:
- Ask: "Quels compromis mineurs as-tu observés ?"
- Collect: `technicalDebtJustification`
- Display: "⚠️ Dette technique acceptable documentée"

**If Clean**:
- Set `technicalDebtJustification: 'Implémentation suit les patterns établis'`
- Display: "✅ Code clean - Pas de dette technique introduite"

### 4. Write System Health to Report

**Append to {outputFile}** (after UX Assessment section):

Create **"## 4. System Health"** section:

```markdown
## 4. System Health

### Battery & Performance

**Status**: {batteryPerformance}

{If batteryPerformance = OK}
✅ Aucun problème de battery drain ou performance détecté

{If batteryPerformance = Concern}
⚠️ **Concerns de performance identifiés**

**Notes**: {batteryPerformanceNotes}

---

### Network Resilience

**Test Status**: {networkResilience}

{If networkResilience = Tested}
**Résultat**: {networkResilienceResult}

**Notes**: {networkResilienceNotes}

{If networkResilienceResult = OK}
✅ L'app gère correctement les différentes conditions réseau

{If networkResilienceResult = Concern}
⚠️ **Issues de network resilience**

{If networkResilience = Not Tested}
ℹ️ Network resilience non testé pendant cette session

{If networkResilience = N/A}
ℹ️ Feature ne nécessite pas de réseau

---

### Technical Debt

**Assessment**: {technicalDebt}

{If technicalDebt = Clean}
✅ **Clean** - L'implémentation suit les patterns établis

{If technicalDebt = Acceptable}
⚠️ **Acceptable** - Quelques compromis mineurs

{If technicalDebt = Concern}
🚫 **CONCERN** - Dette technique majeure introduite (BLOCKER)

**Justification**: {technicalDebtJustification}

---

### System Health Verdict

{Auto-calculate overall system health}
{If all OK/Clean/Acceptable}
✅ **System Health: Satisfaisant**

{If any Concern}
❌ **System Health: Préoccupations majeures** → {List concerns}
```

### 5. Update Frontmatter

**Update frontmatter in {outputFile}**:

```yaml
stepsCompleted: [1, 2, 3, 4, 5, 6]
lastStep: 'system'
systemHealthBattery: {batteryPerformance}
systemHealthNetwork: {networkResilience}
systemHealthTechDebt: {technicalDebt}
systemHealthBlocker: {true if technicalDebt = Concern, else false}
```

### 6. Update Sidecar File

**Append to {sidecarFile}**:

```markdown
### System Health Assessment Completed - {current time}
**Battery/Performance**: {batteryPerformance}
**Network Resilience**: {networkResilience}
**Technical Debt**: {technicalDebt}

{If technicalDebt = Concern}
🚫 **Dette technique majeure - BLOCKER**
```

### 7. Present AUTO-PROCEED

Display: **Passage à l'évaluation de confiance et analyse de régression...**

#### EXECUTION RULES:

- Auto-proceed to confidence & regression analysis after system health assessment
- No user menu choices in this step (assessment is complete)

#### Auto-Proceed Logic:

After system health assessment written to report and frontmatter updated:

1. **Verify** `stepsCompleted: [1, 2, 3, 4, 5, 6]` in frontmatter
2. **Verify** system health section exists in {outputFile}
3. **Immediately load, read entire file, then execute** `{nextStepFile}` for confidence & regression analysis

## CRITICAL STEP COMPLETION NOTE

ONLY WHEN Battery/Performance assessed, Network Resilience assessed, Technical Debt assessed, system health written to report, frontmatter updated with step 6 completion, will you then:

1. Verify "## 4. System Health" section exists in {outputFile}
2. Update frontmatter `stepsCompleted` to `[1, 2, 3, 4, 5, 6]`
3. Set `lastStep: 'system'`
4. Flag `systemHealthBlocker: true` if Technical Debt = Concern
5. Save the output file
6. Immediately load, read entire file, then execute `{nextStepFile}` for confidence & regression analysis

## 🚨 SYSTEM SUCCESS/FAILURE METRICS

### ✅ SUCCESS:

- Battery/Performance assessed (OK or Concern)
- Network Resilience assessed (Tested/Not Tested/N/A)
- Technical Debt assessed (Clean/Acceptable/Concern)
- Detailed notes collected for all Concerns
- System health written to "## 4. System Health" section
- Frontmatter updated with system health metrics and step 6 completion
- Blocker flag set if Technical Debt = Concern
- Next step (confidence) loaded automatically

### ❌ SYSTEM FAILURE:

- Skipping any of the three assessments
- Not collecting detailed notes for Concerns
- Not flagging Technical Debt = Concern as blocker
- Not writing system health to report
- Not updating frontmatter with step 6 completion

**Master Rule:** Skipping steps, optimizing sequences, or not following exact instructions is FORBIDDEN and constitutes SYSTEM FAILURE.

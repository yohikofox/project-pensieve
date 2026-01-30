---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7]
lastStep: 'build'
buildCompleted: true
buildDate: '2026-01-30'
---

# Workflow Creation Plan: mobile-exploratory-testing

## Initial Project Context

- **Module:** bmm
- **Target Location:** _bmad/bmm/workflows/mobile-exploratory-testing
- **Created:** 2026-01-30

## Project Understanding

### Type de Workflow
Tests exploratoires mobile utilisant le MCP mobile-mcp pour interagir avec les applications mobiles.

### Problème Résolu
Ce workflow permettra d'explorer de manière structurée une application mobile (comme Pensine) en utilisant les capacités du MCP :
- Prises de screenshots
- Interactions (clics, swipes, saisie de texte)
- Navigation dans l'application
- Liste des éléments interactifs
- Capture de l'état de l'application

### Utilisateurs Cibles
- Développeurs
- Testeurs
- QA
- yohikofox

## Requirements Détaillés

### 1. Objectif Principal
**Validation de story avant marquage "done"** - Tests exploratoires en mode freestyle pour découvrir les bugs et problèmes d'UX que les tests automatisés n'ont pas détectés.

### 2. Type de Workflow
- **Interactive Workflow** : Guidage de session exploratoire
- **Action Workflow** : Capture passive via MCP mobile-mcp
- Mode d'interaction : **Capture passive** - le testeur explore librement, le workflow observe et documente

### 3. Structure et Flow
**Option D : Freestyle avec checkpoints**
```
Init → Sélection App/Device → [Exploration libre + Capture continue] → Checkpoints réguliers → Synthèse finale
```

Caractéristiques :
- Pas de parcours imposé
- Le testeur décide où aller et quoi tester
- Capture automatique des screenshots et interactions
- Checkpoints suggérés (non forcés) pour documenter les découvertes
- Synthèse finale avec rapport structuré

### 4. Style d'Interaction
**Intent-Based** pour maximum de flexibilité :
- Le workflow s'adapte au rythme du testeur
- Conversation naturelle
- Suggestions non-intrusives
- Pas de séquence rigide

### 5. Inputs Requis

#### Obligatoires :
- Device ID (auto-détecté via MCP)
- Nom de l'application à tester
- Package name (ex: com.pensine.app)

#### Optionnels (configurables) :
- Zone de focus (ex: "Tester la capture vocale")
- Contexte/Build info (ex: "Version 1.2.0-beta", "Story 2.3")
- Story ID ou numéro (pour lier le rapport à la story testée)

### 6. Outputs Produits

#### Pendant la Session :
- 📸 **Screenshots automatiques** à chaque interaction significative
- 📝 **Notes de découverte** - annotations du testeur en temps réel
- 🗺️ **Parcours d'exploration** - timeline des écrans visités et actions effectuées
- 🐛 **Log des anomalies** - bugs et comportements inattendus

#### Rapport Final :
- 📊 **Synthèse de session** structurée avec :
  - Durée et zone explorée
  - Nombre d'écrans visités
  - Liste complète des découvertes
- ✅ **Propositions de tests d'acceptance** pour corriger les bugs trouvés
- 🎨 **Propositions d'améliorations UX** à soumettre au UX designer
- 📎 **Tous les artifacts** (screenshots, logs) organisés et référencés

### 7. Critères de Succès

**Objectif de qualité** : N'avoir aucun bug majeur et bloquant lorsqu'une story est déclarée terminée.

**Indicateurs de session réussie** :
- ✅ Au moins 5 écrans différents explorés
- ✅ Toutes les interactions capturées avec screenshots
- ✅ Les découvertes documentées de façon exploitable
- ✅ Rapport final généré et sauvegardé
- ✅ Tests d'acceptance proposés pour chaque bug
- ✅ Propositions UX documentées si améliorations détectées
- ✅ Aucun bug bloquant non résolu avant marquage "done" de la story

### 8. Capacités MCP Utilisées

Le workflow exploitera les capacités du MCP mobile-mcp :
- `mobile_list_available_devices` - Détecter les devices
- `mobile_take_screenshot` - Capture d'écran
- `mobile_list_elements_on_screen` - Analyser la structure UI
- `mobile_click_on_screen_at_coordinates` - Interactions (si nécessaire)
- `mobile_swipe_on_screen` - Navigation
- `mobile_list_apps` - Lister les apps installées
- `mobile_launch_app` / `mobile_terminate_app` - Contrôle app

### 9. Cas d'Usage Typique

**Scénario** : Story 2.4 "Capture vocale avec transcription" est implémentée et tests unitaires passent.

**Flow** :
1. Dev lance le workflow : `/mobile-exploratory-testing`
2. Workflow détecte le Pixel 10 Pro, demande l'app à tester
3. Dev indique : "Pensine, story 2.4, focus sur capture vocale"
4. Workflow lance l'app, prend screenshot initial
5. Dev explore librement : ouvre capture vocale, teste différents scénarios
6. Workflow capture chaque écran, note les transitions
7. Dev trouve un bug : "Le bouton d'arrêt ne réagit pas après 30s d'enregistrement"
8. Workflow propose un checkpoint : "Veux-tu documenter cette découverte ?"
9. Dev documente, continue l'exploration
10. Après 10-15 minutes, Dev indique "Terminé"
11. Workflow génère rapport avec :
    - 12 screenshots capturés
    - 1 bug bloquant identifié
    - Proposition de test d'acceptance pour le bug
    - 2 améliorations UX suggérées (bouton plus visible, feedback sonore)
12. Story reste "in progress" jusqu'à résolution du bug bloquant

## Party Mode Brainstorming Insights (2026-01-30)

### Framework de Validation Multi-Dimensionnel

#### 1. Classification des Bugs (GLaDOS)

**Niveau 0 - Critical Failures** (Bloquants immédiats) :
- 🔴 Crashes - L'app se termine inopinément
- 🔴 Data loss - Perte de données utilisateur
- 🔴 Core flow broken - Le happy path principal est cassé
- 🔴 ANR (Application Not Responding) - Plus de 5 secondes de freeze

**Niveau 1 - Major Issues** (Story pas "done") :
- 🟠 Degraded UX - Feature utilisable mais pénible
- 🟠 Performance issues - Lag perceptible, plus de 200ms de latence
- 🟠 Inconsistent state - L'UI et les données se désynchronisent
- 🟠 Missing error handling - L'app plante sur edge cases prévisibles

**Niveau 2 - Minor Issues** (Ship mais backlog) :
- 🟡 UI polish - Alignements, couleurs, micro-animations
- 🟡 Helpful errors - Messages d'erreur peu clairs mais non bloquants
- 🟡 Edge case handling - Scénarios rares non gérés élégamment

#### 2. Risk Scoring (Murat)

**Formule : Impact × Probabilité = Risk Score**

IMPACT :
- Critical (5) : Affecte core value proposition
- High (4) : Affecte user satisfaction majeure
- Medium (3) : Affecte confort d'usage
- Low (2) : Affecte polish
- Minimal (1) : Cosmétique

PROBABILITÉ :
- Certain (5) : Tous les users le verront
- Likely (4) : >50% des users
- Possible (3) : 10-50% des users
- Unlikely (2) : <10% des users
- Rare (1) : Edge cases extrêmes

**Seuils de décision** :
- Risk Score > 12 = BLOCKER
- Risk Score 6-11 = Fix before ship
- Risk Score < 6 = Backlog acceptable

#### 3. UX Assessment (Sally)

**UX Red Flags** (bloquants si non résolus) :
- L'utilisateur doit deviner comment utiliser la feature
- Action sans feedback visuel ou sonore
- Parcours utilisateur frustrant même s'il fonctionne
- Accessibilité absente (contraste, taille de touche, lecteur d'écran)

**UX Feeling Score** (échelle 1-5) :
- 5 : Délicieux - Moment de "wow!"
- 4 : Agréable - Intuitive et fluide
- 3 : OK - Fonctionne sans friction majeure
- 2 : Confus - Nécessite effort pour comprendre
- 1 : Frustrant - Expérience pénible

**Seuil : Feeling Score ≥ 3 requis pour DONE**

#### 4. System Health (Winston)

**System Health Indicators** :
- 📊 Observability - Logs suffisants pour debug en prod
- 🔒 Security posture - Données sensibles, injections possibles
- 🔋 Resource management - Battery drain, memory leaks
- 🌐 Network resilience - Mode avion, 3G lent, reconnexion

**Technical Debt Assessment** :
- Clean : Implémentation suit les patterns établis
- Acceptable : Quelques compromis mineurs
- Concern : Dette technique introduite (BLOCKER)

#### 5. Definition of Done (Bob)

**Story Validation Checklist** :

✅ **Functional** :
- [ ] Tous les Acceptance Criteria passent
- [ ] Aucun bug Level 0 (Critical) trouvé
- [ ] Aucun bug Level 1 (Major) avec Risk Score > 12

✅ **Quality** :
- [ ] Tests automatisés existants passent à 100%
- [ ] Feeling Score (UX) ≥ 3/5
- [ ] Aucun UX Red Flag non résolu

✅ **Technical** :
- [ ] Pas de dette technique majeure introduite
- [ ] System Health : aucun drapeau rouge
- [ ] Security & Performance validés

✅ **Documentation** :
- [ ] Rapport d'exploration sauvegardé
- [ ] Bugs documentés avec reproduction steps

**Story Status Final** :
- ✅ DONE si toute la checklist est verte
- 🔄 NEEDS WORK si un seul critère échoue
- 🚫 BLOCKED si Critical bugs trouvés

#### 6. Confidence Score (Carson)

**Question finale au testeur** :
"Sur une échelle de 1-5, à quel point es-tu CONFIANT que cette story est prête pour production ?"

- 5 : Je shipperais ça à ma grand-mère sans hésiter
- 4 : Solide, quelques réserves mineures
- 3 : Ça marche mais j'ai des doutes
- 2 : Plusieurs problèmes me préoccupent
- 1 : Please don't ship this

**Si Confidence < 4** : Workflow demande "Qu'est-ce qui te ferait passer à 5 ?"

### Format du Rapport Final

```markdown
## Story Validation Report

### 📊 Bug Summary
- 🔴 Critical (Level 0): X found → ❌ BLOCKER if > 0
- 🟠 Major (Level 1): Y found
  - High Risk (≥12): Z → ❌ BLOCKER if > 0
  - Medium Risk (6-11): W → ⚠️ Review needed
- 🟡 Minor (Level 2): N found → 📋 Backlog

### 🎨 UX Assessment
- Feeling Score: X/5 → ❌ if < 3
- Red Flags Found: Y → ❌ if > 0
- Improvement Proposals: Z (pour UX designer)

### 🏗️ System Health
- Battery/Performance: [OK/Concern]
- Network Resilience: [Tested/Not Tested]
- Technical Debt: [Clean/Acceptable/Concern] → ❌ if Concern

### 🎯 Confidence Score
- Testeur Confidence: X/5
- Blockers identified: [Si < 4]

### ✅ Final Verdict
Story Status: [DONE ✅ | NEEDS WORK 🔄 | BLOCKED 🚫]

Reason: [Auto-generated based on criteria]

### 📎 Artifacts
- Screenshots: X captured
- Timeline: Y screens visited
- Session duration: Z minutes
```

## Tools Configuration

### Core BMAD Tools

- **Party-Mode**: ✅ INCLUDED
  - Integration points: Synthèse finale si bugs critiques, analyse de patterns récurrents
  - Use case: Brainstormer sur bugs complexes avec équipe d'agents

- **Advanced Elicitation**: ✅ INCLUDED
  - Integration points: Pendant checkpoints, analyse de root causes
  - Use case: Approfondir l'analyse des bugs difficiles à reproduire

- **Brainstorming**: ✅ INCLUDED
  - Integration points: Génération des UX improvement proposals
  - Use case: Générer des propositions d'amélioration UX créatives

### LLM Features

- **Web-Browsing**: ✅ INCLUDED
  - Use cases: Rechercher bugs similaires connus, trouver solutions, vérifier best practices UX
  - Integration: Pendant analyse de bugs et génération de propositions

- **File I/O**: ✅ INCLUDED (OBLIGATOIRE)
  - Operations: Sauvegarder screenshots, créer rapport final, gérer artifacts
  - Critical for: Toutes les opérations de capture et documentation

- **Sub-Agents**: ✅ INCLUDED (OPTIONAL)
  - Use cases: Déléguer analyse de screenshots, analyse de logs en parallèle
  - Benefit: Accélération du traitement, spécialisation des analyses

- **Sub-Processes**: ✅ INCLUDED (OPTIONAL)
  - Use cases: Traiter screenshots en parallèle, analyser logs en background
  - Benefit: Performance améliorée pour sessions longues

### Memory Systems

- **Sidecar File**: ✅ INCLUDED
  - Purpose: Maintenir historique des sessions de tests exploratoires
  - Features:
    - Mémoriser bugs déjà trouvés sur l'app
    - Éviter re-test des zones déjà validées
    - Garder historique des sessions précédentes
    - Comparer évolution de qualité entre versions
    - Apprentissage progressif des patterns de bugs

- **Vector Database**: ✅ INCLUDED
  - Purpose: Recherche sémantique de bugs similaires pour détection de régressions
  - Features:
    - Chercher "tous les bugs liés à la capture vocale" sémantiquement
    - Détecter patterns de problèmes UX récurrents
    - Comparer avec historique pour identifier régressions
  - Requires: Installation MCP

### External Integrations (MCP)

- **mobile-mcp**: ✅ INCLUDED (OBLIGATOIRE - Déjà installé)
  - Status: Opérationnel
  - Capabilities:
    - mobile_list_available_devices - Détecter les devices
    - mobile_take_screenshot - Capture d'écran
    - mobile_list_elements_on_screen - Analyser structure UI
    - mobile_click_on_screen_at_coordinates - Interactions
    - mobile_swipe_on_screen - Navigation
    - mobile_list_apps - Lister apps installées
    - mobile_launch_app / mobile_terminate_app - Contrôle app

- **Playwright**: ✅ INCLUDED
  - Purpose: Automatisation navigateur web
  - Use cases: Tester web views dans l'app mobile, version web de Pensine
  - Requires: Installation MCP
  - URL: https://github.com/modelcontextprotocol/servers/tree/main/src/playwright

- **Git Integration**: ✅ INCLUDED
  - Purpose: Opérations Git directes
  - Use cases:
    - Lier automatiquement bugs trouvés aux commits/branches
    - Créer automatiquement GitHub issues pour bugs avec screenshots
    - Intégration workflow de dev
  - Requires: Installation MCP
  - URL: https://github.com/modelcontextprotocol/servers/tree/main/src/git

- **Database Connector**: ✅ INCLUDED
  - Purpose: Connexion bases de données
  - Use cases:
    - Stocker rapports de tests dans DB centralisée
    - Requêter historique pour analytics
    - Dashboard centralisé des tests
  - Requires: Installation MCP
  - URL: https://github.com/modelcontextprotocol/servers/tree/main/src/postgres

- **Context-7**: ✅ INCLUDED
  - Purpose: Documentation API et références techniques
  - Use cases:
    - Accéder aux docs Android/React Native pendant analyse
    - Proposer solutions basées sur best practices
    - Enrichir propositions techniques
  - Requires: Installation MCP
  - URL: https://github.com/modelcontextprotocol/servers/tree/main/src/context-7

### Installation Requirements

**Tools Requiring Installation:**
1. Vector Database MCP (rag-agent)
2. Playwright MCP
3. Git Integration MCP
4. Database Connector MCP (postgres)
5. Context-7 MCP

**Installation Strategy:** FULL INSTALL - Tous les outils installés dès le départ pour workflow complet

**User Installation Preference:** Willing to install all tools now

**Benefits of Full Installation:**
- Workflow maximal et complet dès le lancement
- Toutes les capacités disponibles immédiatement
- Intégration complète avec l'écosystème de dev (Git, DB, docs)
- Détection de régressions via vector search
- Automatisation maximale

## Output Format Design

### Format Type
**Structured** - Sections requises avec contenu flexible dans chaque section

### Output Requirements

**Document Type**: Story Validation Report (rapport de validation de story)

**File Format**: Markdown (.md)

**Frequency**: Un rapport par session de test exploratoire

**Output Location**: `{output_folder}/exploratory-testing-reports/`

**Naming Convention**: `validation-report-{story-id}-{timestamp}.md`

### Structure Specifications

#### Sections Requises (ordre fixe, non-masquables)

1. **En-tête (Header)**
   - Date de session
   - Story ID
   - Application testée
   - Device utilisé
   - Durée de session
   - Testeur
   - Build/Version info (si fourni)
   - Zone de focus (si fourni)

2. **Bug Summary**
   - 🔴 Critical (Level 0): Count + Liste
   - 🟠 Major (Level 1): Count + Répartition par Risk Score
     - High Risk (≥12): Count + Liste
     - Medium Risk (6-11): Count + Liste
   - 🟡 Minor (Level 2): Count + Liste
   - **Format vide**: "Aucun bug [Level] trouvé" si count = 0

3. **UX Assessment**
   - Feeling Score: X/5
   - Red Flags Found: Count + Liste (ou "Aucun red flag détecté")
   - Improvement Proposals: Count + Liste (ou "Aucune amélioration proposée")

4. **System Health**
   - Battery/Performance: [OK/Concern] + Notes
   - Network Resilience: [Tested/Not Tested/N/A] + Résultats si testé
   - Technical Debt: [Clean/Acceptable/Concern] + Justification
   - **Format vide**: "Non testé" ou "N/A" pour sections non applicables

5. **Confidence Score**
   - Testeur Confidence: X/5
   - Blockers Identified: Liste (ou "Aucun blocker identifié" si score ≥ 4)
   - Justification si score < 4

6. **Final Verdict**
   - Story Status: [DONE ✅ | NEEDS WORK 🔄 | BLOCKED 🚫]
   - Reason: Auto-generated basé sur critères
   - Next Steps: Actions requises (ou "Story prête pour merge" si DONE)

7. **Artifacts**
   - Screenshots: Count + Liste avec chemins
   - Timeline: Nombre d'écrans visités + Séquence
   - Session Duration: Temps total
   - **Format vide**: "Aucun artifact capturé" si applicable

#### Sections Optionnelles (apparaissent si applicable)

8. **Detailed Bug Reports** (si bugs trouvés)
   - Un bloc par bug avec :
     - ID: BUG-{timestamp}-{sequential}
     - Severity: [Critical/Major/Minor]
     - Risk Score: X (si Major/Minor)
     - Title: Résumé court
     - Description: Détails complets
     - Reproduction Steps: Liste numérotée
     - Screenshots: Références
     - Expected Behavior: Ce qui devrait se passer
     - Actual Behavior: Ce qui se passe réellement

9. **Acceptance Tests Proposals** (si bugs trouvés)
   - Un test par bug critique/major
   - Format Given-When-Then ou Steps
   - Lien au bug ID correspondant

10. **UX Improvement Proposals** (si propositions UX)
    - Un bloc par proposition avec :
      - ID: UX-{timestamp}-{sequential}
      - Category: [Usability/Accessibility/Visual/Performance/Other]
      - Current State: Description + Screenshot
      - Proposed Improvement: Description détaillée
      - Rationale: Pourquoi cette amélioration
      - Priority: [High/Medium/Low]
      - For: UX Designer

11. **Regression Analysis** (si Vector DB activé et patterns trouvés)
    - Similar Bugs Found: Références historiques
    - Pattern Analysis: Tendances détectées
    - Regression Risk: Évaluation

12. **Party Mode Insights** (si Party Mode utilisé)
    - Agents Consulted: Liste
    - Key Insights: Résumé des apports
    - Additional Analysis: Détails

### Format Guidelines

**Markdown Standards**:
- Headers: # pour titre, ## pour sections principales, ### pour sous-sections
- Listes: - pour listes à puces, numérotation 1. 2. 3. pour steps
- Emphase: **bold** pour scores/verdicts, *italique* pour notes
- Code blocks: ``` pour logs ou extraits techniques
- Emojis: Utiliser les emojis définis (🔴🟠🟡✅🔄🚫📸📝🗺️🐛)

**Section Header Style**:
```markdown
## 📊 Bug Summary
## 🎨 UX Assessment
## 🏗️ System Health
## 🎯 Confidence Score
## ✅ Final Verdict
## 📎 Artifacts
```

**Empty Section Format**:
- Toujours afficher la section même si vide
- Utiliser: "Aucun [item] trouvé/détecté" ou "Non testé" ou "N/A"
- Maintenir la cohérence : même formulation pour même type de vide

**Cross-Document Consistency**:
- TOUJOURS utiliser le même ordre de sections
- TOUJOURS utiliser les mêmes emojis pour les mêmes sections
- TOUJOURS utiliser le même format de dates (YYYY-MM-DD HH:mm:ss)
- TOUJOURS utiliser les mêmes échelles (1-5 pour scores)
- Format identique pour TOUTES les sessions de test - zéro effort d'adaptation

### Template Structure

```markdown
# Story Validation Report

## 📋 Session Information

- **Date**: YYYY-MM-DD HH:mm:ss
- **Story ID**: [ID]
- **Application**: [Nom]
- **Package**: [package.name]
- **Device**: [Model]
- **Testeur**: [Nom]
- **Duration**: [X minutes]
- **Build/Version**: [Si fourni]
- **Focus Area**: [Si fourni]

---

## 📊 Bug Summary

### 🔴 Critical (Level 0)
- **Count**: X
- **List**: [Liste ou "Aucun bug critique trouvé"]

### 🟠 Major (Level 1)
- **Total Count**: Y
- **High Risk (≥12)**: Z bugs
  - [Liste ou "Aucun bug high risk"]
- **Medium Risk (6-11)**: W bugs
  - [Liste ou "Aucun bug medium risk"]

### 🟡 Minor (Level 2)
- **Count**: N
- **List**: [Liste ou "Aucun bug mineur trouvé"]

---

## 🎨 UX Assessment

- **Feeling Score**: X/5
- **Interpretation**: [Délicieux/Agréable/OK/Confus/Frustrant]
- **Red Flags Found**: Y
  - [Liste ou "Aucun red flag détecté"]
- **Improvement Proposals**: Z
  - [Liste ou "Aucune amélioration proposée"]

---

## 🏗️ System Health

- **Battery/Performance**: [OK/Concern]
  - Notes: [Détails ou "Aucun problème détecté"]
- **Network Resilience**: [Tested/Not Tested/N/A]
  - Results: [Détails ou "Non testé"]
- **Technical Debt**: [Clean/Acceptable/Concern]
  - Assessment: [Justification]

---

## 🎯 Confidence Score

- **Testeur Confidence**: X/5
- **Interpretation**: [Je shipperais.../Solide.../Ça marche.../Plusieurs problèmes.../Please don't...]
- **Blockers Identified**: [Liste ou "Aucun blocker identifié"]

---

## ✅ Final Verdict

- **Story Status**: [DONE ✅ | NEEDS WORK 🔄 | BLOCKED 🚫]
- **Reason**: [Auto-generated basé sur critères]
- **Next Steps**: [Actions requises ou "Story prête pour merge"]

---

## 📎 Artifacts

- **Screenshots Captured**: X
  - [Liste avec chemins ou "Aucun screenshot capturé"]
- **Screens Visited**: Y écrans
  - Timeline: [Séquence]
- **Session Duration**: Z minutes

---

[Sections optionnelles suivent si applicables]
```

### Special Considerations

**Accessibility**:
- Utiliser des emojis pour identification rapide visuelle
- Maintenir structure claire pour screen readers
- Contraste suffisant (markdown standard)

**Validation**:
- Vérifier que tous les champs obligatoires sont remplis
- Valider les scores (1-5 range)
- Vérifier cohérence des counts avec listes

**Automation**:
- Final Verdict auto-calculé basé sur règles définies
- Reason auto-généré selon critères
- Timestamps automatiques
- Counts automatiques basés sur listes

## Workflow Design

### Design Overview

**Type**: Interactive Workflow + Action Workflow avec capture passive

**Flow Pattern**: Freestyle avec checkpoints non forcés

**Continuation Support**: ✅ Activé (step-01b pour pause/reprise)

**Typical Duration**: 10-20 minutes par session

**AI Role**: QA Specialist & Mobile Testing Expert - Observateur attentif, assistant analytique, collaboratif et précis

### Step Structure (8 Steps + Continuation)

#### Step 01: Init & Device Setup
**Goal**: Initialiser la session, détecter devices, sélectionner l'app

**Actions**:
- Détecter continuation (si rapport existant → step-01b)
- Lister devices disponibles via MCP mobile-mcp
- Sélectionner app à tester (package name requis)
- Collecter inputs obligatoires: Device ID, App name, Package name
- Collecter inputs optionnels: Story ID, Zone focus, Build/Version info
- Créer fichier rapport avec frontmatter initial
- Charger sidecar file (historique sessions précédentes)
- Initialiser système de tracking

**Menu**: Auto-proceed → step-02

**Validation**:
- Vérifier au moins un device disponible
- Valider que l'app existe sur le device
- Si erreur: Proposer solutions (installer app, connecter device)

**File**: `steps/step-01-init.md`

---

#### Step 01b: Continue Session
**Goal**: Reprendre une session exploratoire en cours

**Actions**:
- Charger rapport existant depuis {output_folder}
- Analyser frontmatter.stepsCompleted pour déterminer progression
- Lire tous les step files complétés pour comprendre contexte
- Résumer ce qui a été accompli
- Proposer de continuer depuis le dernier checkpoint
- Valider intention de continuation avec utilisateur

**Menu**: [C] Continue → next step approprié basé sur stepsCompleted

**File**: `steps/step-01b-continue.md`

---

#### Step 02: Launch & Capture Init
**Goal**: Lancer l'app et initialiser la capture passive

**Actions**:
- Lancer l'app via MCP (mobile_launch_app)
- Attendre stabilisation (2-3 secondes)
- Prendre screenshot initial (mobile_take_screenshot)
- Initialiser système de capture passive
- Charger Vector DB context (si activé) pour détection régressions
- Expliquer le mode freestyle au testeur:
  - Pas de parcours imposé
  - Explorer librement
  - Checkpoints suggérés mais non forcés
  - Documenter découvertes quand nécessaire

**Menu**: Auto-proceed → step-03

**Error Handling**:
- Si app ne lance pas: Logger erreur, proposer relancer ou annuler
- Si MCP déconnecté: Proposer reconnexion

**File**: `steps/step-02-launch.md`

---

#### Step 03: Freestyle Exploration Loop
**Goal**: Exploration libre avec capture passive continue

**Actions**:
- Boucle d'exploration active jusqu'à [T]erminate:
  - Capturer screenshots automatiquement à chaque interaction significative
  - Testeur explore librement (l'AI observe sans diriger)
  - Proposer checkpoints réguliers (non forcés, suggérés toutes les 5 min ou 10 screenshots)
  - Documenter découvertes en temps réel selon actions utilisateur
  - Maintenir timeline des écrans visités
  - Logger toutes les interactions MCP

**Menu Options**:
- **[D] Document Bug**: Capturer un bug immédiatement (screenshot + description rapide)
- **[N] Add Note**: Ajouter une note/observation rapide
- **[C] Checkpoint**: Pause pour documenter et organiser les découvertes
- **[T] Terminate Exploration**: Terminer la phase d'exploration

**Interaction Style**:
- AI observe et suggère checkpoints sans être intrusif
- Exemple: "Je remarque 8 screenshots capturés. Veux-tu faire un checkpoint pour documenter tes observations ?"
- Respecter le flow du testeur - ne jamais forcer

**Data Captured**:
- Screenshots avec timestamps
- Timeline de navigation (écrans visités)
- Bugs découverts (liste temporaire)
- Notes du testeur
- Interactions utilisateur

**Exit Condition**: Utilisateur sélectionne [T] Terminate

**Next Step**: step-04

**File**: `steps/step-03-explore.md`

---

#### Step 04: Bug Classification & Documentation
**Goal**: Classifier et documenter tous les bugs trouvés avec détails complets

**Actions**:
- Lister tous les bugs découverts pendant exploration
- Pour chaque bug:
  - **Classification Level** (Critical/Major/Minor) basée sur critères GLaDOS
  - **Risk Scoring** (Impact × Probabilité) si Major ou Minor
  - **Title**: Résumé court et clair
  - **Description**: Détails complets du problème
  - **Reproduction Steps**: Liste numérotée précise
  - **Expected Behavior**: Ce qui devrait se passer
  - **Actual Behavior**: Ce qui se passe réellement
  - **Screenshots**: Lier aux captures pertinentes
  - **Bug ID**: BUG-{timestamp}-{sequential}
- **Web-Browsing**: Rechercher bugs similaires connus pour chaque bug critique/majeur
- Générer **Acceptance Tests Proposals** (Given-When-Then ou Steps) pour chaque bug Critical/Major
- Calculer totaux par level et risk score

**Menu Options**:
- **[A] Advanced Elicitation**: Analyser bugs difficiles sous différents angles
- **[P] Party Mode**: Brainstormer avec équipe d'agents sur bugs complexes
- **[C] Continue**: Passer à l'évaluation UX

**Integration Points**:
- Party Mode recommandé si bugs Critical trouvés
- Advanced Elicitation pour bugs difficiles à reproduire

**Validation**:
- Valider Risk Score dans range 1-25
- Vérifier cohérence Impact × Probabilité
- Si info manquante: Demander clarifications

**File**: `steps/step-04-classify.md`

---

#### Step 05: UX Assessment
**Goal**: Évaluer l'expérience utilisateur globale et proposer améliorations

**Actions**:
- **Feeling Score** (1-5): Demander au testeur sa perception globale
  - 5: Délicieux - Moment de "wow!"
  - 4: Agréable - Intuitive et fluide
  - 3: OK - Fonctionne sans friction majeure
  - 2: Confus - Nécessite effort pour comprendre
  - 1: Frustrant - Expérience pénible
- **UX Red Flags** - Identifier si présents:
  - Utilisateur doit deviner comment utiliser feature
  - Action sans feedback visuel/sonore
  - Parcours frustrant même si fonctionnel
  - Accessibilité absente (contraste, taille touche, lecteur écran)
- **UX Improvement Proposals**: Pour chaque problème ou opportunité:
  - ID: UX-{timestamp}-{sequential}
  - Category: Usability/Accessibility/Visual/Performance/Other
  - Current State: Description + Screenshot
  - Proposed Improvement: Détails
  - Rationale: Pourquoi cette amélioration
  - Priority: High/Medium/Low
  - For: UX Designer

**Menu Options**:
- **[A] Advanced Elicitation**: Explorer alternatives UX
- **[P] Party Mode**: Consulter agents pour perspective diverse
- **[B] Brainstorm UX**: Session créative pour générer idées d'amélioration
- **[C] Continue**: Passer à System Health

**Integration Points**:
- Brainstorming recommandé si plusieurs opportunités d'amélioration détectées
- Party Mode pour consulter Sally (UX Designer agent) si besoin

**Validation**:
- Feeling Score doit être 1-5
- Si Red Flag identifié: Forcer documentation complète
- Threshold: Feeling Score ≥ 3 requis pour DONE

**File**: `steps/step-05-ux.md`

---

#### Step 06: System Health Assessment
**Goal**: Évaluer la santé technique et performance du système

**Actions**:
- **Battery/Performance**:
  - Observations sur battery drain pendant session
  - Performance issues (lag, freezes)
  - Résultat: OK / Concern
  - Notes détaillées si Concern
- **Network Resilience**:
  - Testé / Not Tested / N/A
  - Si testé: Résultats (mode avion, 3G lent, reconnexion)
  - Comportement de l'app dans différentes conditions réseau
- **Technical Debt Assessment**:
  - Clean: Implémentation suit patterns établis
  - Acceptable: Quelques compromis mineurs
  - Concern: Dette technique introduite (BLOCKER)
  - Justification de l'évaluation

**Menu Options**:
- **[C] Continue**: Passer à Confidence & Regression

**Validation**:
- Si Technical Debt = Concern: Auto-flag comme blocker pour verdict final

**File**: `steps/step-06-system.md`

---

#### Step 07: Confidence & Regression Analysis
**Goal**: Capturer confiance du testeur et détecter patterns/régressions

**Actions**:
- **Confidence Score** (1-5):
  - Question: "À quel point es-tu CONFIANT que cette story est prête pour production ?"
  - 5: Je shipperais ça à ma grand-mère sans hésiter
  - 4: Solide, quelques réserves mineures
  - 3: Ça marche mais j'ai des doutes
  - 2: Plusieurs problèmes me préoccupent
  - 1: Please don't ship this
- **Si Confidence < 4**:
  - Demander: "Qu'est-ce qui te ferait passer à 5 ?"
  - Identifier blockers explicites
  - Documenter préoccupations
- **Vector DB Regression Analysis** (si activé):
  - Recherche sémantique: Bugs similaires dans historique
  - Pattern Analysis: Tendances récurrentes
  - Regression Risk: Évaluation basée sur historique
- **Party Mode Trigger**:
  - Si bugs critiques/complexes nécessitent analyse approfondie
  - Consulter GLaDOS, Murat, Winston pour perspectives multiples

**Menu Options**:
- **[A] Advanced Elicitation**: Analyser confiance sous différents angles
- **[P] Party Mode**: Consulter équipe si bugs critiques
- **[C] Continue**: Passer au verdict final

**Integration Points**:
- Party Mode recommandé si Confidence < 3
- Vector DB search automatique si activé

**File**: `steps/step-07-confidence.md`

---

#### Step 08: Final Verdict & Report Generation
**Goal**: Générer verdict final, compiler rapport complet, et optionnellement intégrer avec Git/DB

**Actions**:
- **Calculate Final Verdict** (auto-généré basé sur règles):
  - ✅ **DONE** si:
    - Bug Critical count = 0
    - Bug Major High Risk count = 0
    - Feeling Score ≥ 3
    - No UX Red Flags
    - Technical Debt ≠ Concern
  - 🔄 **NEEDS WORK** si:
    - Un seul critère échoue
    - Bugs à corriger mais non bloquants
  - 🚫 **BLOCKED** si:
    - Bug Critical count > 0
    - Bug Major High Risk count > 0
    - Technical Debt = Concern
- **Auto-Generate Reason**: Expliquer pourquoi ce verdict basé sur critères
- **Compile Complete Report** utilisant template structure:
  - Toutes les 7 sections requises
  - Sections optionnelles si applicable
  - Format markdown cohérent
  - Tous emojis et headers standardisés
- **Save All Artifacts**:
  - Screenshots dans {output_folder}/exploratory-testing-reports/artifacts/
  - Rapport principal dans {output_folder}/exploratory-testing-reports/
  - Naming: validation-report-{story-id}-{timestamp}.md
- **Mark Workflow Complete**: frontmatter.workflowComplete = true

**Optional Integrations**:
- **Git MCP** (si sélectionné [G]):
  - Créer GitHub issues pour chaque bug Critical/Major
  - Attacher screenshots
  - Lier au Story ID
  - Labels automatiques (bug, critical, etc.)
- **Database MCP** (si sélectionné [D]):
  - Stocker rapport complet en DB centralisée
  - Permettre analytics et trending
  - Dashboard de qualité
- **Vector DB Update**:
  - Stocker embeddings des bugs pour futures recherches
  - Enrichir base de connaissances

**Menu Options**:
- **[G] Create GitHub Issues**: Générer issues pour bugs
- **[D] Store in Database**: Sauvegarder en DB centralisée
- **[F] Finish**: Terminer workflow (issues + DB optionnels)

**Final Messages**:
- Résumé de session (durée, écrans, bugs)
- Chemin vers rapport généré
- Actions suggérées selon verdict
- Si DONE: "✅ Story prête pour merge !"
- Si NEEDS WORK: Liste des actions requises
- Si BLOCKED: "🚫 Bugs bloquants à résoudre avant de continuer"

**File**: `steps/step-08-verdict.md`

---

### Interaction Patterns

**Auto-Proceed Steps** (pas de menu utilisateur):
- Step 01: Init → step-02
- Step 02: Launch → step-03

**Interactive Steps avec Menus**:
- Step 03: Exploration Loop [D/N/C/T]
- Step 04: Classification [A/P/C]
- Step 05: UX [A/P/B/C]
- Step 06: System Health [C]
- Step 07: Confidence [A/P/C]
- Step 08: Verdict [G/D/F]

**Collaboration AI Style**:
- **Observateur attentif**: Capture sans être intrusif
- **Suggestions non forcées**: Propose checkpoints, respecte décisions
- **Assistant analytique**: Aide classification, calculs, propositions
- **Factuel et précis**: Documentation rigoureuse

### Data Flow

**Initial Inputs** (Step 01):
- Device ID (auto-détecté)
- App name + Package name
- Story ID (optionnel)
- Zone focus (optionnel)
- Build/Version info (optionnel)

**State Tracked in Frontmatter**:
```yaml
stepsCompleted: [1, 2, ...]
workflowComplete: false
sessionStart: "2026-01-30 14:30:00"
lastContinued: "2026-01-30 15:45:00"  # Si continuation
storyId: "2.4"
appName: "Pensine"
packageName: "com.pensine.app"
deviceId: "56251FDCH00APM"
deviceModel: "Pixel 10 Pro"
buildVersion: "1.2.0-beta"
focusArea: "Capture vocale"
bugsFound: []
screenshotsCount: 0
sessionDuration: 0
```

**Data Collected Per Step**:
- Step 03: screenshots[], timeline[], bugs[], notes[]
- Step 04: bugClassifications[], riskScores[], acceptanceTests[]
- Step 05: feelingScore, redFlags[], uxProposals[]
- Step 06: systemHealth{}
- Step 07: confidenceScore, regressions[]
- Step 08: finalVerdict, reason, nextSteps[]

**Final Outputs**:
- Story Validation Report (markdown)
- Screenshots artifacts (PNG files)
- Vector DB embeddings (si activé)
- GitHub issues (si demandé)
- Database entry (si demandé)
- Sidecar file updated (historique)

### File Structure

```
_bmad/bmm/workflows/mobile-exploratory-testing/
├── workflow.md                           # Configuration principale
├── steps/
│   ├── step-01-init.md                  # Init avec détection continuation
│   ├── step-01b-continue.md             # Reprise de session
│   ├── step-02-launch.md                # Lancement app
│   ├── step-03-explore.md               # Boucle exploration freestyle
│   ├── step-04-classify.md              # Classification bugs + tests
│   ├── step-05-ux.md                    # Assessment UX + propositions
│   ├── step-06-system.md                # System Health assessment
│   ├── step-07-confidence.md            # Confidence + Regression analysis
│   └── step-08-verdict.md               # Final verdict + rapport
├── templates/
│   └── report-template.md               # Template rapport (structure définie step 5)
└── data/
    ├── bug-classifications.md           # Référence classifications (Level 0/1/2)
    └── risk-scoring-guide.md            # Guide Impact × Probabilité
```

### Validation & Error Handling

**Step 01 Validation**:
- Au moins un device disponible
- App existe sur device
- Si erreur: Solutions (installer, connecter)

**Step 03 Error Handling**:
- MCP mobile déconnecté → Proposer reconnexion, mode dégradé
- App crash → Documenter comme bug Critical auto
- Screenshot fail → Logger, continuer sans bloquer

**Step 04 Validation**:
- Risk Score range 1-25
- Cohérence Impact × Probabilité
- Info manquante → Clarifications

**Step 05 Validation**:
- Feeling Score 1-5
- Red Flag → Documentation forcée
- Threshold ≥ 3 pour DONE

**Step 08 Validation**:
- Calcul verdict automatique correct
- Cohérence counts avec listes
- Tous champs requis remplis
- Si incohérence → Alerter et corriger

### Special Features

**Continuation Support**:
- Step-01b détecte et reprend sessions
- frontmatter.stepsCompleted tracking
- Chaque step update frontmatter avant next
- Seamless pause/resume

**Conditional Logic**:
- Step 01: Si rapport existe → step-01b, sinon → step-02
- Step 03: Boucle jusqu'à [T]erminate
- Step 08: Verdict basé sur multiple conditions

**Integration Points**:
- Party Mode: Steps 4, 5, 7 (bugs complexes, UX, analyse)
- Brainstorming: Step 5 (créativité UX)
- Advanced Elicitation: Steps 4, 5, 7 (analyse approfondie)
- Web-Browsing: Step 4 (recherche bugs similaires)
- Vector DB: Step 7 (détection régressions)
- Git MCP: Step 8 optionnel (GitHub issues)
- Database MCP: Step 8 optionnel (stockage centralisé)

**Multi-Scenario Handling**:
- Exploration courte (5 min) vs longue (20 min)
- Aucun bug trouvé vs bugs critiques multiples
- Session continue vs pause/reprise
- Intégrations Git/DB optionnelles vs standalone

### Success Criteria

**Workflow Successful When**:
- Toutes les 8 étapes complétées
- Rapport généré avec toutes sections requises
- Final Verdict calculé correctement
- Artifacts sauvegardés
- frontmatter.workflowComplete = true

**Story Validation Successful When**:
- Final Verdict = DONE (tous critères passent)
- Aucun bug bloquant
- Feeling Score ≥ 3
- Confidence Score satisfaisant
- Rapport exploitable pour équipe

---

## Build Summary (Step 07)

### Build Completion

**Date**: 2026-01-30
**Status**: ✅ COMPLETE
**Total Files Generated**: 15

---

### Files Generated

#### 1. Main Workflow File

- **workflow.md** - Main workflow entry point with BMAD step-file architecture, role definition, and initialization sequence

#### 2. Step Files (10 files)

**Initialization Steps**:
- **steps/step-01-init.md** - Device detection, app selection, testing context collection, report initialization, sidecar file loading
- **steps/step-01b-continue.md** - Session continuation logic, state analysis, resume from last checkpoint

**Execution Steps**:
- **steps/step-02-launch.md** - App launch via MCP, baseline screenshot capture, passive capture system initialization, freestyle mode explanation
- **steps/step-03-explore.md** - Freestyle exploration loop with [D/N/C/T] menu, passive screenshot capture, bug/note documentation
- **steps/step-04-classify.md** - Bug classification (GLaDOS levels 0-2), Murat risk scoring (Impact × Probability), acceptance test generation, Web-Browsing for known issues
- **steps/step-05-ux.md** - UX Feeling Score (1-5), Red Flags identification, improvement proposals with [A/P/B/C] menu
- **steps/step-06-system.md** - Battery/Performance assessment, Network Resilience testing, Technical Debt evaluation
- **steps/step-07-confidence.md** - Confidence Score (1-5), blocker identification, Vector DB regression analysis
- **steps/step-08-verdict.md** - Auto-calculated verdict (DONE/NEEDS WORK/BLOCKED), final report compilation, optional GitHub issues / Database integration

#### 3. Templates (1 file)

- **templates/report-template.md** - Frontmatter structure and 7-section report skeleton for validation reports

#### 4. Data Files (2 files)

- **data/bug-classifications.md** - GLaDOS Framework: Level 0 (Critical), Level 1 (Major), Level 2 (Minor) with criteria, examples, decision tree
- **data/risk-scoring-guide.md** - Murat Framework: Impact scale (1-5), Probability scale (1-5), decision matrix, calculation examples

#### 5. Planning Document

- **workflow-plan-mobile-exploratory-testing.md** - Complete workflow design with all requirements, Party Mode insights, tools configuration, step specifications (this file)

---

### Workflow Architecture

**Pattern**: Step-File Architecture with Continuation Support

**Total Steps**: 8 + 1 continuation

**Auto-Proceed Steps** (no user menu):
- Step 01 (Init) → Step 02
- Step 02 (Launch) → Step 03
- Step 06 (System Health) → Step 07

**Interactive Steps** (with user menus):
- Step 03 (Explore): [D/N/C/T]
- Step 04 (Classify): [A/P/C]
- Step 05 (UX): [A/P/B/C]
- Step 07 (Confidence): [A/P/C]
- Step 08 (Verdict): [G/D/F]

**Continuation Pattern**:
- Step 01 detects existing reports → Step 01b
- Step 01b analyzes state → resumes at appropriate step

---

### Integration Points

**MCP Servers Used**:
- **mobile-mcp** (already installed): Device interactions, screenshots, app launch
- **Vector DB** (to install): Regression detection, semantic bug search
- **Git MCP** (to install): GitHub issues creation for bugs
- **Database MCP** (to install): Report storage for analytics
- **Context-7** (to install): External documentation lookups
- **Playwright** (to install): Web-Browsing for known bugs

**BMAD Tools Integrated**:
- **Party Mode**: Multi-agent brainstorming (recommended for Critical bugs, low confidence)
- **Advanced Elicitation**: Alternative analysis (available in Steps 04, 05, 07)
- **Brainstorming**: Creative UX improvement session (Step 05)

---

### Quality Framework

**GLaDOS Bug Classification**:
- Level 0 (Critical): Crashes, Data loss, Core flow broken, ANR >5s → BLOCKER
- Level 1 (Major): Degraded UX, Performance >200ms, Inconsistent state → Risk Scored
- Level 2 (Minor): UI polish, Helpful errors, Edge cases → Backlog acceptable

**Murat Risk Scoring**:
- Formula: Impact (1-5) × Probability (1-5) = Risk Score (1-25)
- Thresholds: >12 BLOCKER, 6-11 Fix before ship, <6 Backlog

**Sally UX Assessment**:
- Feeling Score (1-5): 5 = Wow, 4 = Agréable, 3 = OK, 2 = Confus, 1 = Frustrant
- Red Flags: Usability, Feedback, Frustration, Accessibility
- Threshold: ≥3 required for DONE

**Winston System Health**:
- Battery/Performance: OK / Concern
- Network Resilience: Tested/Not Tested/N/A
- Technical Debt: Clean/Acceptable/Concern (Concern = BLOCKER)

**Carson Confidence**:
- Score (1-5): 5 = Ship sans hésiter, 1 = Don't ship
- If <4: Identify blockers ("What would make it 5?")

**Bob Definition of Done**:
- Bugs Critical = 0
- Bugs Blocker (Major High Risk) = 0
- UX Feeling Score ≥ 3
- UX Red Flags = 0
- Technical Debt ≠ Concern

---

### Final Verdict Logic

```
IF bugsLevel0 > 0 OR bugsBlockerCount > 0 OR techDebt = Concern:
  → BLOCKED 🚫

ELSE IF uxRedFlags > 0 OR uxFeelingScore < 3 OR bugsLevel1 > 0:
  → NEEDS WORK 🔄

ELSE:
  → DONE ✅
```

---

### Report Structure

**Required Sections** (7):
1. Exploration Summary (metrics, timeline, bugs temporary list, notes)
2. Bugs Classification & Documentation (GLaDOS levels, risk scores, acceptance tests)
3. UX Assessment (Feeling Score, Red Flags, improvement proposals)
4. System Health (Battery/Performance, Network, Technical Debt)
5. Confidence & Regression Analysis (Confidence Score, blockers, Vector DB findings)
6. Final Verdict (auto-calculated, rationale, required actions)
7. Testing Summary & Artifacts (session metrics, coverage, quality scores, artifacts location)

**Optional Sections** (5):
- Advanced Elicitation outputs (if used)
- Party Mode transcripts (if used)
- Brainstorming session results (if used)
- GitHub issues links (if created)
- Database record ID (if stored)

---

### Next Steps for Users

#### To Use This Workflow:

1. **Install MCP Servers** (if not already installed):
   ```bash
   # Vector DB for regression detection
   # Git MCP for GitHub issues
   # Database MCP for report storage
   # Context-7 for external docs
   # Playwright for web searches
   ```

2. **Invoke Workflow**:
   ```bash
   /mobile-exploratory-testing
   # Or use full path:
   /bmad:bmb-creations:workflows:mobile-exploratory-testing
   ```

3. **Testing Session Flow**:
   - Step 01: Device auto-detected, select app, provide Story ID/focus
   - Step 02: App launches, freestyle mode explained
   - Step 03: Explore freely, capture bugs [D], notes [N], checkpoints [C], terminate [T]
   - Step 04: Classify bugs (GLaDOS + Murat), generate acceptance tests
   - Step 05: Assess UX (Feeling Score + Red Flags), propose improvements
   - Step 06: Evaluate system health (Battery, Network, Tech Debt)
   - Step 07: Capture confidence, analyze regressions via Vector DB
   - Step 08: Auto-generate verdict, optionally create GitHub issues / store in DB

4. **Artifacts Generated**:
   - Validation report: `{output_folder}/exploratory-testing-reports/validation-report-{storyId}-{timestamp}.md`
   - Screenshots: `{output_folder}/exploratory-testing-reports/artifacts/screenshot-*.png`
   - Session history: `{output_folder}/validation-report-{project_name}-history.md`

---

### Workflow Characteristics

**Type**: Interactive Workflow + Action Workflow with capture passive
**Flow Pattern**: Freestyle avec checkpoints non forcés
**Continuation Support**: ✅ Activé (pause/resume via step-01b)
**Typical Duration**: 10-20 minutes per session
**AI Role**: QA Specialist & Mobile Testing Expert - Observateur attentif, assistant analytique, collaboratif et précis

**Language**: French (communication_language from config)
**Output Language**: French (document_output_language from config)

---

**Build Complete** ✅

Workflow prêt à l'emploi pour validation de stories mobiles avant marquage "done".


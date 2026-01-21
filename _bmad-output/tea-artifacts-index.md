# TEA Artifacts Index - Test Architecture Documentation

**Dernière mise à jour**: 2026-01-21
**Agent**: TEA (Test Architect Expert)
**Projet**: Pensieve Mobile

---

## 📋 Index des Documents Clés

### Infrastructure Globale

#### 🏗️ Test Infrastructure Setup
**Fichier**: `_bmad-output/test-infrastructure-setup.md`
**Date**: 2026-01-20
**Contenu**: Stack complète TDD + BDD + E2E
- Infrastructure Gherkin/jest-cucumber
- Mocks in-memory (MockAudioRecorder, MockFileSystem, etc.)
- 15 scenarios Gherkin Story 2.1
- Pattern RED-GREEN-REFACTOR
- **IMPORTANT**: Consulter ce fichier avant tout rapport sur framework de test

#### 📊 Test Design Epic 2
**Fichier**: `_bmad-output/test-design-epic-2.md`
**Date**: 2026-01-21
**Contenu**: Analyse risques + Plan de test Epic 2
- 15 risques identifiés (5 high-priority ≥6)
- 60 tests planifiés (P0: 18, P1: 22, P2/P3: 20)
- Effort: 67 heures (~8.5 jours)

---

### Story 2.1 - Capture Audio 1-Tap

#### 📝 ATDD Checklist 2.1
**Fichier**: `_bmad-output/atdd-checklist-2-1-capture-audio-1-tap.md`
**Date**: 2026-01-21
**Contenu**: Checklist implémentation Story 2.1
- 33 tests générés (11 E2E + 22 Integration)
- 4 tasks, 11 subtasks
- Effort: 8-12 heures

#### 🧪 Feature Gherkin 2.1
**Fichier**: `pensieve/mobile/tests/acceptance/features/story-2-1-capture-audio.feature`
**Date**: 2026-01-20
**Contenu**: 15 scenarios Gherkin en français
- Coverage: AC1-AC5 + Edge cases
- Tags: @AC1, @AC2, @performance, @NFR1, etc.

#### 🔧 Step Definitions 2.1
**Fichier**: `pensieve/mobile/tests/acceptance/story-2-1.test.ts`
**Date**: 2026-01-20
**Contenu**: 20+ tests BDD avec jest-cucumber

---

### Story 2.2 - Capture Texte Rapide

#### 📝 ATDD Checklist 2.2
**Fichier**: `_bmad-output/atdd-checklist-2-2-capture-texte-rapide.md`
**Date**: 2026-01-21
**Contenu**: Checklist implémentation Story 2.2
- 55 tests générés (18 E2E + 37 Integration/Component)
- 5 tasks, 12 subtasks
- Effort: 4-6 heures

#### 🧪 Feature Gherkin 2.2
**Fichier**: `pensieve/mobile/tests/acceptance/features/story-2-2-capture-texte.feature`
**Date**: 2026-01-21
**Contenu**: 29 scenarios Gherkin en français
- Coverage: AC1-AC6 + Edge cases (8 scenarios)
- 11 Scenario Outline (data-driven)
- Tags: @AC1-AC6, @validation, @edge-case, etc.

#### 🔧 Step Definitions 2.2
**Fichier**: `pensieve/mobile/tests/acceptance/story-2-2.test.ts`
**Date**: 2026-01-21
**Contenu**: 29 tests BDD avec jest-cucumber

#### 🛠️ Nouveaux Mocks 2.2
**Fichier**: `pensieve/mobile/tests/acceptance/support/test-context.ts` (mis à jour)
**Date**: 2026-01-21
**Ajouts**:
- MockKeyboard
- MockTextInput
- MockDialog
- MockDraftStorage
- MockApp

---

## 🗂️ Structure des Fichiers

```
pensieve/
├── mobile/
│   ├── tests/
│   │   └── acceptance/
│   │       ├── features/
│   │       │   ├── story-2-1-capture-audio.feature          ✅ Gherkin
│   │       │   └── story-2-2-capture-texte.feature          ✅ Gherkin
│   │       ├── support/
│   │       │   └── test-context.ts                          ✅ Mocks in-memory
│   │       ├── story-2-1.test.ts                            ✅ Step defs
│   │       └── story-2-2.test.ts                            ✅ Step defs
│   └── e2e/
│       └── capture/
│           ├── audio-1-tap.e2e.ts                           ✅ E2E Detox
│           └── text-capture.e2e.ts                          ✅ E2E Detox
└── _bmad-output/
    ├── tea-artifacts-index.md                               ⭐ CE FICHIER
    ├── test-infrastructure-setup.md                         🔥 INFRASTRUCTURE GLOBALE
    ├── test-design-epic-2.md                                📊 Plan de test
    ├── atdd-checklist-2-1-capture-audio-1-tap.md           📝 Checklist 2.1
    └── atdd-checklist-2-2-capture-texte-rapide.md          📝 Checklist 2.2
```

---

## 🎯 Workflow de Consultation

### Avant de générer un rapport sur les tests:

1. **TOUJOURS lire** `test-infrastructure-setup.md` d'abord
2. **Consulter** cet index pour voir tous les artefacts
3. **Vérifier** les fichiers .feature Gherkin
4. **Lire** test-context.ts pour les mocks disponibles

### Avant d'implémenter une story:

1. **Lire** la checklist ATDD (`atdd-checklist-X-X-*.md`)
2. **Consulter** le fichier .feature Gherkin
3. **Voir** les step definitions (`.test.ts`)
4. **Comprendre** les mocks dans test-context.ts

---

## 📊 Statistiques Globales

### Tests Générés (Epic 2)

| Story | Scenarios Gherkin | Tests BDD | Tests Integration | Tests E2E | Total |
|-------|-------------------|-----------|-------------------|-----------|-------|
| 2.1 | 15 | ~20 | 22 | 11 | 53 |
| 2.2 | 29 | ~40 | 37 | 18 | 95 |
| **Total** | **44** | **~60** | **59** | **29** | **148** |

### Mocks Créés

| Mock | Origine | Utilisé par |
|------|---------|-------------|
| InMemoryDatabase | Story 2.1 | 2.1, 2.2 |
| MockAudioRecorder | Story 2.1 | 2.1 |
| MockFileSystem | Story 2.1 | 2.1 |
| MockPermissionManager | Story 2.1 | 2.1 |
| MockKeyboard | Story 2.2 | 2.2 |
| MockTextInput | Story 2.2 | 2.2 |
| MockDialog | Story 2.2 | 2.2 |
| MockDraftStorage | Story 2.2 | 2.2 |
| MockApp | Story 2.2 | 2.2 |
| MockSupabaseAuth | Epic 1 | Epic 1 |
| MockAsyncStorage | Epic 1 | Epic 1 |
| MockRGPDService | Epic 1 | Epic 1 |

---

## 🔄 Mise à Jour de Cet Index

Chaque fois qu'un nouvel artefact TEA est créé, ajouter une entrée ici avec:
- Nom du fichier
- Date de création
- Résumé du contenu
- Liens vers artefacts liés

---

**Généré par:** Agent TEA
**Version:** 6.0 (BMad v6)
**Dernière modification:** 2026-01-21

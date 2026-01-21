# Test Design: Epic 2 - Capture & Transcription de Pensées

**Date:** 2026-01-21
**Auteur:** yohikofox
**Status:** Draft

---

## Executive Summary

**Scope:** Full test design pour Epic 2 (Capture & Transcription)

**Risk Summary:**

- Total risques identifiés: 15
- High-priority risques (≥6): 5
- Catégories critiques: PERF, DATA, TECH

**Coverage Summary:**

- P0 scenarios: 12 (24 heures)
- P1 scenarios: 18 (18 heures)
- P2/P3 scenarios: 15 (10 heures)
- **Total effort**: 52 heures (~7 jours)

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| R-001 | DATA | Perte de données audio lors d'un crash pendant la capture | 2 | 3 | 6 | Implémenter auto-save toutes les 100ms + crash recovery avec WatermelonDB | DEV | Sprint 1 |
| R-002 | PERF | Capture audio > 500ms viole NFR1, UX dégradée | 3 | 2 | 6 | Optimiser init audio + feedback haptique immédiat + tests de performance | DEV | Sprint 1 |
| R-003 | PERF | Transcription > 2x durée audio viole NFR2, frustration utilisateur | 2 | 3 | 6 | Profiling Whisper.rn + optimisation modèle + tests de charge | DEV | Sprint 1 |
| R-004 | TECH | Whisper.rn (500 Mo) échec de téléchargement/installation du modèle | 2 | 3 | 6 | Retry mechanism + fallback cloud API + monitoring téléchargement | DEV | Sprint 1 |
| R-005 | SEC | Audio sensible stocké non chiffré en local (RGPD) | 2 | 3 | 6 | Chiffrement at-rest avec expo-secure-store pour clés | DEV | Sprint 1 |

### Medium-Priority Risks (Score 3-5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| R-006 | TECH | Incompatibilité API audio React Native entre iOS/Android | 2 | 2 | 4 | Tests E2E sur devices physiques iOS + Android | QA |
| R-007 | DATA | Conflits de synchronisation offline-first (WatermelonDB) | 2 | 2 | 4 | Stratégie last-write-wins + tests de conflict resolution | DEV |
| R-008 | PERF | Modèle Whisper 500 Mo impacte mémoire devices low-end | 1 | 3 | 3 | Tests sur devices avec RAM < 4 Go + monitoring mémoire | QA |
| R-009 | BUS | Feedback haptique manquant réduit confiance utilisateur | 2 | 2 | 4 | Tests UX du feedback tactile + Android/iOS variants | QA |
| R-010 | OPS | Gestion versions modèle Whisper (upgrade/downgrade) | 1 | 3 | 3 | Versioning API + migration strategy + rollback tests | DEV |

### Low-Priority Risks (Score 1-2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| R-011 | OPS | Purge ancien contenu audio (storage management) | 1 | 2 | 2 | Monitor |
| R-012 | BUS | Animation Liquid Glass pas fluide | 1 | 1 | 1 | Monitor |
| R-013 | TECH | Edge case: capture interrompue par appel téléphonique | 1 | 2 | 2 | Document |
| R-014 | PERF | Transcription batch vs stream tradeoff | 1 | 1 | 1 | Document |
| R-015 | DATA | Race condition WatermelonDB écriture simultanée | 1 | 2 | 2 | Monitor |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

---

## Test Coverage Plan

### P0 (Critical) - Run on every commit

**Critères**: Bloque user journey + High risk (≥6) + NFRs critiques

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| FR1: Capture audio 1-tap | E2E | R-001, R-002 | 3 | QA | Happy path + crash recovery + perf < 500ms |
| FR2: Feedback haptique immédiat | E2E | R-009 | 2 | QA | iOS + Android variants |
| FR4: Stockage offline des captures | Integration | R-001, R-007 | 4 | QA | WatermelonDB transactions + sync conflicts |
| FR5: Transcription Whisper on-device | Integration | R-003, R-004 | 3 | QA | Perf < 2x duration + model download + fallback |
| NFR6: Zero data loss (crash recovery) | E2E | R-001 | 2 | QA | Crash pendant capture + restauration état |
| NFR1: Capture < 500ms | Performance | R-002 | 1 | QA | Benchmark sur devices physiques |
| NFR2: Transcription < 2x durée | Performance | R-003 | 1 | QA | Load tests avec audio 1min, 5min, 10min |
| SEC: Chiffrement audio local | Integration | R-005 | 2 | QA | Vérifier chiffrement at-rest + expo-secure-store |

**Total P0**: 18 tests, 36 heures (2 heures/test complexité setup + security)

### P1 (High) - Run on PR to main

**Critères**: Fonctionnalités importantes + Medium risk (3-5) + Workflows fréquents

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| FR3: Capture texte rapide | E2E | - | 3 | QA | Happy path + édition + sauvegarde |
| FR6: Annuler capture audio en cours | E2E | R-013 | 2 | QA | Annulation manuelle + interruption téléphone |
| FR7: Consultation de transcription | E2E | - | 2 | QA | Affichage + scroll |
| FR8: Distinction visuelle statut | Component | - | 3 | DEV | States: recording, transcribing, done, error |
| Audio API iOS/Android compat | Integration | R-006 | 4 | QA | Tests sur devices physiques |
| WatermelonDB sync | Integration | R-007 | 3 | QA | Offline → online sync + conflict resolution |
| Whisper modèle low-memory devices | E2E | R-008 | 2 | QA | Devices < 4 Go RAM |
| Liquid Glass animations | Component | R-012 | 3 | DEV | Fade-in, slide transitions, feedback tactile |

**Total P1**: 22 tests, 22 heures (1 heure/test standard)

### P2 (Medium) - Run nightly/weekly

**Critères**: Fonctionnalités secondaires + Low risk (1-2) + Edge cases

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| Purge ancien audio (storage management) | Integration | R-011 | 2 | QA | Cleanup policy tests |
| Versioning modèle Whisper | Integration | R-010 | 3 | DEV | Upgrade/downgrade/rollback |
| Race conditions WatermelonDB | Unit | R-015 | 5 | DEV | Concurrent writes tests |
| Edge case: Mémoire pleine pendant capture | E2E | - | 1 | QA | Error handling gracieux |
| Edge case: Battery < 5% pendant transcription | E2E | - | 1 | QA | Pause/resume mechanism |
| UI Polish: Liquid Glass micro-interactions | Component | R-012 | 3 | DEV | Visual regression tests |

**Total P2**: 15 tests, 7.5 heures (0.5 heure/test simple)

### P3 (Low) - Run on-demand

**Critères**: Nice-to-have + Exploratory + Optimizations

| Requirement | Test Level | Test Count | Owner | Notes |
|-------------|------------|------------|-------|-------|
| Transcription batch vs stream comparaison | Performance | 2 | QA | Benchmark différentes stratégies |
| Accessibilité VoiceOver/TalkBack | E2E | 2 | QA | Tests manuels a11y |
| Logs debug + monitoring traces | Manual | 1 | DEV | Vérification instrumentation |

**Total P3**: 5 tests, 1.5 heures (0.3 heure/test exploratory)

---

## Execution Order

### Smoke Tests (<5 min)

**Purpose**: Fast feedback, catch build-breaking issues

- [ ] Capture audio 1-tap démarre enregistrement (30s)
- [ ] Feedback haptique déclenché sur tap (30s)
- [ ] Transcription Whisper retourne texte (2min)
- [ ] WatermelonDB stocke capture (1min)

**Total**: 4 scenarios

### P0 Tests (<15 min)

**Purpose**: Critical path validation (NFRs + high-risk scenarios)

- [ ] Capture < 500ms avec benchmark (E2E - 2min)
- [ ] Crash recovery restaure état capture (E2E - 3min)
- [ ] Transcription < 2x durée audio (Performance - 3min)
- [ ] Chiffrement audio at-rest vérifié (Integration - 2min)
- [ ] Zero data loss sur crash (E2E - 3min)
- [ ] Whisper model download + fallback (Integration - 2min)

**Total**: 18 scenarios (dont 6 listés ci-dessus + 12 autres)

### P1 Tests (<30 min)

**Purpose**: Important feature coverage

- [ ] Capture texte rapide sauvegarde (E2E - 1min)
- [ ] Annulation capture audio (E2E - 1min)
- [ ] Consultation transcription scroll (E2E - 1min)
- [ ] iOS/Android audio API compat (Integration - 5min sur 2 devices)
- [ ] WatermelonDB sync conflicts (Integration - 3min)
- [ ] Whisper sur low-memory device (E2E - 4min)
- [ ] Liquid Glass animations fluides (Component - 2min)

**Total**: 22 scenarios

### P2/P3 Tests (<60 min)

**Purpose**: Full regression coverage

- [ ] Storage management purge (Integration - 2min)
- [ ] Whisper versioning upgrade (Integration - 3min)
- [ ] Race conditions WatermelonDB (Unit - 5 tests, 3min)
- [ ] Edge cases: mémoire pleine, battery low (E2E - 2min)
- [ ] UI polish visual regression (Component - 5min)

**Total**: 20 scenarios

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Hours/Test | Total Hours | Notes |
|----------|-------|------------|-------------|-------|
| P0 | 18 | 2.0 | 36 | Complex setup: devices physiques, Whisper, crash simulation |
| P1 | 22 | 1.0 | 22 | Standard coverage: audio API, sync, UI |
| P2 | 15 | 0.5 | 7.5 | Scénarios simples: edge cases, cleanup |
| P3 | 5 | 0.3 | 1.5 | Exploratory: benchmarks, a11y manual |
| **Total** | **60** | **-** | **67 heures** | **~8.5 jours** |

### Prerequisites

**Test Data:**

- AudioCapture factory (faker-based: durée aléatoire 5s-5min, formats WAV/M4A)
- Transcription factory (mock texte avec timestamps)
- User factory (avec préférences stockage)

**Tooling:**

- **Playwright** pour E2E mobile (React Native)
- **Maestro** alternatif pour tests natifs iOS/Android
- **Detox** si Expo compatible
- **k6** pour performance benchmarks (transcription load tests)
- **Whisper.rn** test fixtures (audio samples 10s, 1min, 5min, 10min)
- **expo-secure-store** mocks pour tests chiffrement

**Environment:**

- Devices physiques: iPhone 13 (iOS 17), Pixel 7 (Android 13), device low-RAM (< 4 Go)
- Simulateurs/Émulateurs pour tests rapides
- Test database WatermelonDB locale (reset entre tests)
- Whisper model mock (éviter téléchargement 500 Mo à chaque test)

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate**: 100% (no exceptions - NFRs non négociables)
- **P1 pass rate**: ≥95% (waivers required for failures)
- **P2/P3 pass rate**: ≥90% (informational)
- **High-risk mitigations**: 100% complete ou approved waivers

### Coverage Targets

- **Critical paths (FR1-FR8)**: ≥90%
- **Security scenarios (SEC)**: 100%
- **Performance (NFR1, NFR2)**: Benchmarks < seuils définis
- **Data integrity (NFR6, NFR8)**: 100% crash recovery tests

### Non-Negotiable Requirements

- [ ] Tous les P0 tests passent
- [ ] Aucun high-risk (≥6) items non mitigé
- [ ] Security tests (R-005 chiffrement) passent 100%
- [ ] Performance targets atteints (NFR1 < 500ms, NFR2 < 2x durée)
- [ ] Zero data loss validé (NFR6, NFR8)

---

## Mitigation Plans

### R-001: Perte de données audio lors crash pendant capture (Score: 6)

**Mitigation Strategy:**
1. Implémenter auto-save audio buffer toutes les 100ms dans WatermelonDB
2. Utiliser transactions WatermelonDB pour garantir atomicité
3. Ajouter crash recovery handler au démarrage app (restaurer état capture)
4. Tests E2E crash simulation avec Maestro/Detox

**Owner:** DEV Team
**Timeline:** Sprint 1
**Status:** Planned
**Verification:** E2E test crash pendant capture + restauration au redémarrage

### R-002: Capture audio > 500ms viole NFR1 (Score: 6)

**Mitigation Strategy:**
1. Optimiser initialisation audio API React Native (preload au mount)
2. Déclencher feedback haptique IMMÉDIATEMENT au tap (avant audio ready)
3. Profiling avec React Native Performance Monitor
4. Performance tests automatisés avec seuil 500ms hard limit

**Owner:** DEV Team
**Timeline:** Sprint 1
**Status:** Planned
**Verification:** Benchmark sur devices physiques (CI avec BrowserStack/Sauce Labs)

### R-003: Transcription > 2x durée viole NFR2 (Score: 6)

**Mitigation Strategy:**
1. Profiling Whisper.rn sur différents devices (identifier bottlenecks)
2. Optimiser modèle Whisper (quantization si possible)
3. Implémenter queue asynchrone (éviter bloquer UI)
4. Load tests avec audio 1min, 5min, 10min

**Owner:** DEV Team
**Timeline:** Sprint 1
**Status:** Planned
**Verification:** Performance tests k6 avec seuil 2x durée hard limit

### R-004: Échec téléchargement/installation modèle Whisper 500 Mo (Score: 6)

**Mitigation Strategy:**
1. Retry mechanism avec exponential backoff (3 tentatives)
2. Fallback vers cloud transcription API (Google Speech-to-Text) si échec
3. Monitoring téléchargement avec analytics (taux succès/échec)
4. Tests E2E simulation network failure

**Owner:** DEV Team
**Timeline:** Sprint 1
**Status:** Planned
**Verification:** Integration tests mock network failures + fallback cloud API

### R-005: Audio sensible stocké non chiffré (RGPD) (Score: 6)

**Mitigation Strategy:**
1. Chiffrement at-rest avec AES-256 via expo-secure-store pour clés
2. Clés de chiffrement stockées dans iOS Keychain / Android Keystore
3. Audit RGPD des données personnelles (audio = données sensibles)
4. Tests security validation chiffrement

**Owner:** DEV Team
**Timeline:** Sprint 1
**Status:** Planned
**Verification:** Integration tests vérifier fichier audio chiffré + keychain/keystore

---

## Assumptions and Dependencies

### Assumptions

1. Whisper.rn modèle "base" (500 Mo) est suffisant pour qualité transcription française
2. Devices cibles ont minimum 2 Go RAM (exclu devices très low-end)
3. WatermelonDB supporte transactions offline-first sans backend sync immédiat (Epic 6)
4. React Native audio API stable sur iOS 15+ et Android 10+
5. Expo build system compatible avec Whisper.rn native modules

### Dependencies

1. **Whisper.rn library** - Installation et configuration (Story 2.5) - Required by Story 2.5
2. **WatermelonDB setup** - Schéma database et sync config (Story 1.1) - Required by Story 2.4
3. **Liquid Glass Design System** - Composants Shadcn/Tamagui (Story 1.1) - Required by Stories 2.1-2.6
4. **Test devices physiques** - iOS (iPhone 13+), Android (Pixel 7+) - Required by Sprint 1
5. **Cloud transcription API fallback** - Google Cloud Speech-to-Text compte configuré - Required by Story 2.5

### Risks to Plan

- **Risk**: Whisper.rn incompatible avec Expo managed workflow
  - **Impact**: Fallback vers bare React Native (délai +3 jours)
  - **Contingency**: Utiliser cloud API uniquement (pas on-device)

- **Risk**: Performance NFR1 < 500ms impossible sur Android mid-range
  - **Impact**: Revoir NFR1 à < 1000ms pour Android
  - **Contingency**: iOS uniquement pour NFR1 strict, Android best-effort

- **Risk**: Tests devices physiques indisponibles (CI/CD)
  - **Impact**: Réduire couverture E2E devices, augmenter tests simulateurs
  - **Contingency**: Utiliser BrowserStack/Sauce Labs cloud devices

---

## Follow-on Workflows (Manual)

- Run `*atdd` workflow pour générer failing P0 tests (separate workflow; not auto-run)
- Run `*automate` workflow pour broader coverage once implementation exists
- Run `*framework` workflow si test framework Playwright/Maestro n'est pas initialisé
- Run `*ci` workflow pour scaffold CI/CD quality pipeline avec P0/P1/P2 execution order

---

## Approval

**Test Design Approved By:**

- [ ] Product Manager: Date:
- [ ] Tech Lead: Date:
- [ ] QA Lead: Date:

**Comments:**

---

## Appendix

### Knowledge Base References

- `risk-governance.md` - Risk classification framework (6 catégories, scoring, gate decision)
- `probability-impact.md` - Risk scoring methodology (probability × impact matrix)
- `test-levels-framework.md` - Test level selection (E2E vs API vs Component vs Unit)
- `test-priorities-matrix.md` - P0-P3 prioritization criteria

### Related Documents

- PRD: `_bmad-output/planning-artifacts/prd.md`
- Epic: `_bmad-output/planning-artifacts/epics.md` (Epic 2)
- Architecture: `_bmad-output/planning-artifacts/architecture.md`
- UX Design: `_bmad-output/planning-artifacts/ux-design-specification.md`

---

**Generated by**: BMad TEA Agent - Test Architect Module
**Workflow**: `_bmad/bmm/testarch/test-design`
**Version**: 4.0 (BMad v6)

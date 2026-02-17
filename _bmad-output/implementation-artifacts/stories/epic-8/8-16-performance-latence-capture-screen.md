# Story 8.16: Performance — Réduction de la Latence au Premier Load du CaptureScreen

Status: ready-for-dev

## Story

As a **user**,
I want **the Capture tab to be immediately responsive when I tap on it**,
So that **I can start capturing a thought in under 500ms as guaranteed by the app's design**.

## Context

Constat terrain : latence perceptible d'environ 1 seconde au premier accès à l'onglet Capture depuis le démarrage de l'app.

**NFR violé** : NFR1 — "Capture audio : < 500ms après tap" (`architecture.md#Performance NFR1-NFR5`).

**L'UX Spec impose** : "1-Tap Liberation — Latence capture < 1s impose architecture réactive" (`architecture.md#UX-Driven Architecture Requirements`).

**Chaîne de blocage identifiée** (analyse pre-story, 2026-02-18) :

```
Tab "Capture" pressé
  │
  ├─► useEffect #1 (bloquant) : Permission check système iOS/Android
  │       hasMicrophonePermission() → requestMicrophonePermission() si besoin
  │       └─► setAudioModeAsync({ playsInSilentMode, allowsRecording })
  │           ⚠️  BLOQUE le rendu de CaptureScreenContent jusqu'à résolution
  │
  ├─► useEffect #2 (sync, premier appel) : Résolution de 8 singletons TSyringe
  │       RecordingService, TextCaptureService, ICaptureRepository,
  │       IPermissionService, FileStorageService, StorageMonitorService,
  │       TranscriptionModelService, TranscriptionEngineService
  │       ⚠️  Premiers appels = constructeurs déclenchés (filesystem possible)
  │
  └─► useEffect #3 (async) : CrashRecovery
          recoverIncompleteRecordings() → requête OP-SQLite
          ⚠️  Sur le chemin critique du rendu
```

**Suspects classés par probabilité d'impact** :

| # | Cause | Impact estimé |
|---|-------|---------------|
| 1 | `setAudioModeAsync()` bloquant | 300–600ms |
| 2 | Rendu bloqué par permission check | 100–300ms |
| 3 | Constructeurs singletons (1ère résolution) | 50–200ms |
| 4 | `recoverIncompleteRecordings()` DB | 50–150ms |
| 5 | Contention thread JS (sync/transcription init) | 50–200ms |

**Approche** : Diagnostic instrumenté d'abord, fix ciblé ensuite. Pas d'optimisation prématurée.

## Acceptance Criteria

### AC1: Instrumentation — Baseline de timing établie

**Given** l'app est fraîchement démarrée
**When** l'utilisateur tape l'onglet Capture pour la première fois
**Then** des logs `[PERF]` existent pour chaque point de mesure :
- `[PERF] setAudioModeAsync: Xms`
- `[PERF] DI resolution (8 services): Xms`
- `[PERF] CrashRecovery: Xms`
- `[PERF] CaptureScreenContent mounted: Xms` (depuis bootstrap)
**And** les logs sont présents en mode dev uniquement (`__DEV__`)

### AC2: Root cause identifiée

**Given** les mesures de baseline sont disponibles
**When** les timings sont analysés sur iOS et Android
**Then** la ou les causes principales (> 100ms) sont documentées dans les Dev Notes
**And** les suspects < 50ms sont éliminés de la liste de fix

### AC3: Latence réduite à < 500ms (NFR1)

**Given** le fix ciblé est appliqué
**When** l'utilisateur tape l'onglet Capture depuis un démarrage à froid
**Then** `[PERF] CaptureScreenContent mounted` est < 500ms
**And** le bouton d'enregistrement est interactable dans ce délai

### AC4: Pas de régression — Second accès instantané

**Given** l'utilisateur a déjà navigué vers Capture une fois
**When** il navigue vers un autre onglet puis revient sur Capture
**Then** le screen s'affiche sans délai perceptible (< 100ms)
**And** les services DI sont réutilisés (singletons, pas de re-init)

### AC5: Pas de régression fonctionnelle

**Given** le fix est appliqué
**When** l'utilisateur déclenche un enregistrement audio
**Then** tous les AC de Story 2.1 (capture audio 1-tap) restent satisfaits
**And** la permission microphone est correctement demandée si absente
**And** le crash recovery fonctionne toujours au démarrage

### AC6: Test de non-régression automatisé

**Given** le fix est implémenté
**When** les tests BDD sont exécutés (`npm run test:acceptance`)
**Then** un scénario Gherkin valide que le CaptureScreen monte sans délai de blocage (mock des appels lents)
**And** tous les tests existants Story 2.1 passent toujours à 100%

## Tasks / Subtasks

- [ ] **Task 1 — Instrumentation et mesure baseline** (AC: #1)
  - [ ] 1.1 — Ajouter points de mesure `performance.now()` dans `CaptureScreen.tsx` autour de `setAudioModeAsync`, résolution DI (8 services), et `recoverIncompleteRecordings` (guards `__DEV__` obligatoires)
  - [ ] 1.2 — Ajouter point de mesure au mount de `CaptureScreenContent` et au bootstrap (`bootstrap.ts`) pour calculer le delta total
  - [ ] 1.3 — Mesurer et documenter les résultats sur iOS simulator ET device (si disponible), premier load VS second load
  - [ ] 1.4 — Documenter les résultats dans les Dev Notes de cette story

- [ ] **Task 2 — Analyse et sélection du fix** (AC: #2)
  - [ ] 2.1 — Comparer les timings : identifier le ou les suspects qui dépassent 100ms
  - [ ] 2.2 — Vérifier si la latence existe au second accès (tab switch) — si oui : problème de rendu, pas de DI/audio
  - [ ] 2.3 — Décider du fix ciblé selon l'arbre de décision ci-dessous (voir Dev Notes)

- [ ] **Task 3 — Écrire les tests BDD AVANT le fix** (AC: #6) — RED phase
  - [ ] 3.1 — Créer `tests/acceptance/features/story-8-16-capture-screen-perf.feature` avec scénario Gherkin pour AC6
  - [ ] 3.2 — Créer step definitions `tests/acceptance/story-8-16.test.ts` — les steps doivent échouer (RED)
  - [ ] 3.3 — Vérifier que `npm run test:acceptance` échoue sur la nouvelle story uniquement

- [ ] **Task 4 — Implémenter le fix ciblé** (AC: #3, #4, #5)
  - [ ] 4.1 — Appliquer le fix sélectionné en Task 2 (voir arbre de décision Dev Notes)
  - [ ] 4.2 — Re-mesurer les timings post-fix pour confirmer réduction à < 500ms
  - [ ] 4.3 — Vérifier AC4 : second accès < 100ms

- [ ] **Task 5 — Valider la non-régression** (AC: #5, #6) — GREEN phase
  - [ ] 5.1 — Exécuter `npm run test:acceptance` : les tests Story 8-16 doivent passer (GREEN)
  - [ ] 5.2 — Exécuter `npm run test:acceptance:story-2-1` : les tests Story 2.1 doivent passer à 100%
  - [ ] 5.3 — Retirer les logs `[PERF]` ou les conditionner définitivement à `__DEV__`

## Dev Notes

### Arbre de décision pour le fix (Task 2→4)

```
Si suspect #1 (setAudioModeAsync) > 300ms
  → FIX A : Pré-initialiser l'audio session dans bootstrap.ts
             Appeler setAudioModeAsync() AVANT le premier render React
             CaptureScreen n'a plus besoin de l'appeler au mount

Si suspect #2 (rendu bloqué par permissions) > 200ms
  → FIX B : Découpler permission check du blocage de rendu
             Montrer CaptureScreenContent immédiatement
             Gérer la permission en overlay asynchrone non-bloquant
             (garder le même UX mais en non-bloquant)

Si suspect #3 (constructeurs singletons) > 150ms
  → FIX C : Warm-up anticipé dans bootstrap.ts
             Résoudre les 8 services AVANT le premier render
             container.resolve() déclenche les constructeurs une fois pour toutes

Si suspect #4 (CrashRecovery DB) > 100ms
  → FIX D : Déplacer vers useCrashRecovery global (déjà dans MainApp.tsx)
             Retirer le useEffect #3 de CaptureScreen.tsx
             Garantir que MainApp fait le crash recovery AVANT la navigation

Si suspect #5 (contention thread) > 100ms
  → FIX E : Différer useSyncInitialization avec setTimeout 0
             Après le premier render, pas avant
```

**Important** : Appliquer UN seul fix par round de mesure. Ne pas tout optimiser d'un coup.

### Fichiers concernés

| Fichier | Rôle | Action probable |
|---------|------|-----------------|
| `pensieve/mobile/src/screens/capture/CaptureScreen.tsx` | Screen principal — 3 useEffects suspects | Instrumentation + fix |
| `pensieve/mobile/src/config/bootstrap.ts` | Point d'init avant React | FIX A ou C (warm-up) |
| `pensieve/mobile/src/navigation/MainNavigator.tsx` | Tab navigator | FIX B (overlay non-bloquant) |
| `pensieve/mobile/src/hooks/initialization/useCrashRecovery.ts` | Hook global crash recovery | FIX D (déplacer la logique) |
| `pensieve/mobile/src/hooks/initialization/useSyncInitialization.ts` | Hook sync au démarrage | FIX E (différer) |
| `tests/acceptance/features/story-8-16-capture-screen-perf.feature` | Gherkin BDD | Créer |
| `tests/acceptance/story-8-16.test.ts` | Step definitions | Créer |

### Patterns de code à respecter

**DI lazy resolution** — ne jamais résoudre au niveau module :
```typescript
// ✅ Dans useEffect uniquement
useEffect(() => {
  const svc = container.resolve(RecordingService);
}, []);
```

**Guard DEV pour les logs de performance** :
```typescript
if (__DEV__) {
  console.log('[PERF] setAudioModeAsync:', t1 - t0, 'ms');
}
```

**Pattern wrapper/content existant** — `CaptureScreenWithAudioMode` wrappe `CaptureScreenContent`.
Si FIX B est appliqué, modifier le wrapper, pas le content.

### Tests BDD — Template Gherkin suggéré

```gherkin
Feature: CaptureScreen - Performance au premier load
  En tant qu'utilisateur
  Je veux que l'écran Capture réponde immédiatement
  Afin de capturer une pensée sans friction

  Scenario: Premier accès à CaptureScreen sans délai bloquant
    Given l'app est démarrée et initialisée
    When l'utilisateur navigue vers l'onglet Capture
    Then le CaptureScreenContent est monté sans attendre setAudioModeAsync
    And le bouton d'enregistrement est visible et interactable

  Scenario: Second accès instantané (no re-init)
    Given l'utilisateur a déjà visité l'onglet Capture
    When il revient sur Capture depuis un autre onglet
    Then aucune résolution DI supplémentaire n'est effectuée
    And le screen s'affiche immédiatement
```

**Mocks nécessaires dans `test-context.ts`** :
- `Audio.setAudioModeAsync` → mock qui resolve immédiatement
- `PermissionService.hasMicrophonePermission` → mock qui retourne `true`
- `CrashRecoveryService.recoverIncompleteRecordings` → mock qui retourne `[]`

### Références

- NFR1 : `_bmad-output/planning-artifacts/architecture.md#Performance NFR1-NFR5`
- UX Spec : `_bmad-output/planning-artifacts/architecture.md#UX-Driven Architecture Requirements`
- CaptureScreen : `pensieve/mobile/src/screens/capture/CaptureScreen.tsx:113-404`
- Bootstrap : `pensieve/mobile/src/config/bootstrap.ts`
- Container DI : `pensieve/mobile/src/infrastructure/di/container.ts:145-155`
- Story 2.1 (référence AC) : `_bmad-output/implementation-artifacts/2-1-capture-audio-1-tap.md`

### Project Structure Notes

- Stories Epic 8 dans `_bmad-output/implementation-artifacts/8-XX-*.md`
- Tests BDD mobile dans `pensieve/mobile/tests/acceptance/`
- Gherkin features dans `pensieve/mobile/tests/acceptance/features/`
- Step definitions dans `pensieve/mobile/tests/acceptance/story-8-16.test.ts`
- Mocks dans `pensieve/mobile/tests/acceptance/support/test-context.ts`

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

*À compléter lors de l'implémentation*

### Completion Notes List

*À compléter lors de l'implémentation*

### File List

*À compléter lors de l'implémentation*

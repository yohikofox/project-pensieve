# Story 7.3 : LLM Logs Analysis — Auto-generate GitHub Issues

Status: done

## Story

As a **développeur solo utilisant le mode debug de Pensine**,
I want **que l'app analyse automatiquement les logs d'erreur via un LLM et génère des GitHub Issues structurées**,
so that **je réduise la friction du bug reporting et identifie rapidement les erreurs critiques sans effort manuel**.

## Acceptance Criteria

**AC1 — Bouton déclenchement dans DevPanel**
- Given debug mode actif (`isDebugModeEnabled === true`) et ≥1 log `level === 'error'`
- When j'ouvre l'onglet "📋 Logs" du DevPanel
- Then je vois le bouton "🔍 Analyze & Report Issue"

**AC2 — Export et groupement des logs d'erreur**
- Given je clique sur "Analyze & Report Issue"
- Then tous les `LogEntry` avec `level === 'error'` sont extraits du `logsDebugStore`
- And groupés par pattern similaire (titre message)
- And limités aux 20 erreurs les plus récentes

**AC3 — Analyse LLM locale (ADR-036)**
- Given le bundle est prêt
- When le LLM (`gemma3n-2b` via litert-lm) analyse les logs
- Then la réponse contient : `title`, `body` (markdown), `labels` (array), `severity`
- And un feedback visuel (spinner) est visible pendant l'analyse (SLA <60s)

**AC4 — Prévisualisation + confirmation obligatoire**
- Given l'analyse LLM est terminée
- Then un modal affiche le contenu proposé (titre éditable + body preview)
- And la création requiert une confirmation manuelle (pas de fire & forget)

**AC5 — Configuration GitHub Token dans Settings**
- Given debug mode actif
- When j'accède aux Settings
- Then une section "Bug Reporting" permet de configurer :
  - GitHub Personal Access Token (stocké dans expo-secure-store, jamais AsyncStorage)
  - GitHub Repo cible (format `owner/repo`)

**AC6 — Création GitHub Issue via API**
- Given token configuré + confirmation utilisateur
- When la création est déclenchée
- Then `POST https://api.github.com/repos/{owner}/{repo}/issues` est appelé via `fetchWithRetry`
- And la réponse inclut l'URL de l'issue créée

**AC7 — Déduplication**
- Given une issue a été créée pour un pattern d'erreur
- When le même pattern est détecté
- Then le service vérifie les issues existantes (GET /search/issues)
- And si issue similaire trouvée (>80% match), propose "Ajouter commentaire" plutôt que créer

**AC8 — Feedback résultat dans DevPanel**
- Given action terminée (succès ou erreur)
- Then un message avec l'URL de l'issue (ou l'erreur) est affiché dans DevPanel
- And un LogEntry de niveau "info" est ajouté au store

**AC9 — Protection par debug mode**
- Given `isDebugModeEnabled === false`
- Then bouton AC1 et section Settings AC5 sont invisibles

## Tasks / Subtasks

- [x] Task 1 : `GitHubIssueService` mobile (AC5, AC6, AC7)
  - [x] 1.1 Interface `IGitHubIssueService` + token management (expo-secure-store)
  - [x] 1.2 `createIssue(title, body, labels)` via `fetchWithRetry` (ADR-025)
  - [x] 1.3 `searchExistingIssue(title)` → déduplication
  - [x] 1.4 Enregistrement DI (Symbol token + `container.register` Transient — ADR-021)

- [x] Task 2 : `LogsAnalysisService` mobile (AC2, AC3)
  - [x] 2.1 Extraction LogEntry `level === 'error'` + groupement
  - [x] 2.2 Prompt LLM structuré (single call — ADR-004) → JSON `{title, body, labels, severity}`
  - [x] 2.3 Intégration pipeline LLM existant (Knowledge context, ADR-036)
  - [x] 2.4 Enregistrement DI

- [x] Task 3 : UI DevPanel — bouton + modal (AC1, AC4, AC8)
  - [x] 3.1 Bouton conditionnel dans `InAppLogger.tsx` (errors>0 && debugMode)
  - [x] 3.2 Modal prévisualisation (titre éditable, body preview, confirm/cancel)
  - [x] 3.3 Spinner pendant analyse LLM
  - [x] 3.4 Affichage URL issue créée

- [x] Task 4 : SettingsScreen — section GitHub (AC5, AC9)
  - [x] 4.1 Section "Bug Reporting" conditionnelle (debugMode uniquement)
  - [x] 4.2 Champ GitHub Token (expo-secure-store, masqué)
  - [x] 4.3 Champ GitHub Repo (`owner/repo`)
  - [x] 4.4 Persistance dans `settingsStore.ts`

- [x] Task 5 : Tests unitaires (AC1–AC9)
  - [x] 5.1 Tests `GitHubIssueService` (mock fetch, mock SecureStore) — 12 tests ✅
  - [x] 5.2 Tests `LogsAnalysisService` (extraction, formatage prompt) — 11 tests ✅
  - [x] 5.3 Tests SettingsStore (github fields) — intégrés dans 5.2

- [x] Task 6 : BDD Gherkin (AC1–AC9)
  - [x] 6.1 `tests/acceptance/features/story-7-3.feature` (7 scénarios)
  - [x] 6.2 `tests/acceptance/story-7-3.test.ts` (jest-cucumber) — 7 tests ✅

## Dev Notes

**Périmètre :** Mobile uniquement (React Native + Expo SDK 54). Pas de modification backend.

**Dépendances établies :**
- `logsDebugStore` (Story 7.2) — source LogEntry
- `isDebugModeEnabled` / `settingsStore` (Story 7.1) — guard + store à étendre
- `fetchWithRetry` (`mobile/src/infrastructure/http/fetchWithRetry.ts`) — wrapper HTTP ADR-025 ✅
- Pipeline LLM `gemma3n-2b` — Knowledge context (ADR-036)

**ADR compliance critique :**
| ADR | Règle | Application |
|-----|-------|-------------|
| ADR-021 | Transient First | Services en `container.register()` (pas singleton) |
| ADR-022 | Pas AsyncStorage données sensibles | Token → `expo-secure-store` |
| ADR-023 | Result Pattern | `Result<GitHubIssue>` pour retours service |
| ADR-025 | fetchWithRetry | GitHub API via wrapper existant |
| ADR-028 | Pas de `any` explicite | Types stricts sur LogEntry, GitHubIssue |
| ADR-036 | LLM Local-First | `gemma3n-2b`, SLA <60s, feedback visuel obligatoire >5s |
| ADR-004 | Single LLM call | Un prompt → JSON structuré (titre+corps+labels+sévérité) |

**Fichiers à créer/modifier dans pensieve/ :**
```
pensieve/mobile/src/components/dev/services/
  ├── GitHubIssueService.ts        NEW
  └── LogsAnalysisService.ts       NEW
pensieve/mobile/src/components/dev/
  └── InAppLogger.tsx               MODIFIER (bouton + modal)
pensieve/mobile/src/screens/settings/
  └── SettingsScreen.tsx            MODIFIER (section debug)
pensieve/mobile/src/stores/
  └── settingsStore.ts              MODIFIER (+githubToken, +githubRepo)
pensieve/mobile/src/infrastructure/di/
  ├── tokens.ts                     MODIFIER (+2 symbols)
  └── container.ts                  MODIFIER (+2 registrations)
pensieve/mobile/tests/acceptance/
  ├── features/story-7-3.feature    NEW
  └── story-7-3.test.ts             NEW
pensieve/mobile/src/components/dev/services/__tests__/
  ├── GitHubIssueService.test.ts    NEW
  └── LogsAnalysisService.test.ts   NEW
```

**Pattern DI (tokens.ts + container.ts) :**
```typescript
// tokens.ts
IGitHubIssueService: Symbol('IGitHubIssueService'),
ILogsAnalysisService: Symbol('ILogsAnalysisService'),

// container.ts
container.register(TOKENS.IGitHubIssueService, GitHubIssueService);
container.register(TOKENS.ILogsAnalysisService, LogsAnalysisService);
```

**Prompt LLM (template ADR-004) :**
```
Analyze these error logs and generate a GitHub issue as JSON only.
ERROR LOGS:
{logs_bundle}
Return: { "title": string (max 80 chars), "body": string (markdown), "labels": string[], "severity": "critical"|"high"|"medium"|"low" }
```

**GitHub API (via fetchWithRetry) :**
```
POST https://api.github.com/repos/{owner}/{repo}/issues
  Authorization: Bearer {token}
  Body: { title, body, labels }

GET https://api.github.com/search/issues?q={title}+repo:{owner}/{repo}+type:issue
```

**Non-périmètre :**
- ❌ Backend (aucun endpoint nécessaire)
- ❌ Création automatique sans confirmation (privacy)
- ❌ Support multi-repo

### References
- [Source: epics.md — Epic 7, feedback discovery 7.3 + D7]
- [Source: stories/epic-7/7-2-logs-devtools-rotation-fifo.md — logsDebugStore patterns]
- [Source: stories/epic-7/7-1-support-mode-avec-permissions-backend.md — isDebugModeEnabled]
- [Source: adrs/ADR-036 — LLM Local-First, gemma3n-2b, SLA]
- [Source: adrs/ADR-015 — Observability (Sentry, Pino)]
- [Source: adrs/ADR-021 — DI Transient First]
- [Source: adrs/ADR-023 — Result Pattern]
- [Source: mobile/src/infrastructure/http/fetchWithRetry.ts — wrapper HTTP]

## Dev Agent Record

### Agent Model Used
claude-sonnet-4-6

### Debug Log References
- Fix jest.mock hoisting — `mockFetchWithRetry` déclaré dans factory puis importé+casté (résout `undefined` en tests)
- Fix reflect-metadata — `jest.mock('reflect-metadata', () => {})` supprimé, `import 'reflect-metadata'` en ligne 1
- Fix jest-cucumber `ensureFeatureFileAndStepDefinitionScenarioHaveSameSteps` — Background: retiré, scénarios autonomes

### Completion Notes List
- AC3 (feedback visuel >5s) : `ActivityIndicator` + spinner implémentés ; test LLM réel impossible sans device natif — couvert par tests unitaires mock
- AC7 déduplication : algorithme word-overlap (>80% threshold) via Sets — simple et efficace pour un usage dev solo
- Token GitHub en `expo-secure-store` (ADR-022) ; `githubRepo` seul dans `settingsStore` (non-sensible)
- Pipeline LLM suit exactement le pattern `CaptureAnalysisService` (ADR-036 compliant)

### Code Review Fixes (claude-sonnet-4-6)
- **H1** `container.ts:255` — `LogsAnalysisService` promu Singleton (exception ADR-021 documentée) : évite rechargement LLM à chaque résolution Transient
- **H2** `story-7-3.feature` — scénario "AC7" renommé en "AC5" (githubRepo) ; ajout scénarios AC6 (création issue) et AC7 (déduplication) manquants → 9 scénarios
- **H3** `SettingsScreen.tsx:437,453` — `new GitHubIssueService()` remplacé par `container.resolve<IGitHubIssueService>(TOKENS.IGitHubIssueService)` (respect ADR-021)
- **M4** `LogsAnalysisService.ts:185` — timeout `Promise.race` + `clearTimeout` proper scope (SLA 60s enforced côté service, ADR-036)
- **M5** `story-7-3.test.ts` — mocks `expo-secure-store` + `fetchWithRetry` ajoutés ; step definitions AC6 et AC7 implémentés
- **M6** `SettingsScreen.tsx:81` — `githubTokenLoaded` (état mort) supprimé
- **M7** `GitHubIssueService.ts:137` — filtre `is:open` ajouté à la requête de déduplication

### File List
- `pensieve/mobile/src/components/dev/services/GitHubIssueService.ts` — NEW
- `pensieve/mobile/src/components/dev/services/LogsAnalysisService.ts` — NEW
- `pensieve/mobile/src/components/dev/services/__tests__/GitHubIssueService.test.ts` — NEW
- `pensieve/mobile/src/components/dev/services/__tests__/LogsAnalysisService.test.ts` — NEW
- `pensieve/mobile/src/infrastructure/di/tokens.ts` — MODIFIED (+IGitHubIssueService, +ILogsAnalysisService)
- `pensieve/mobile/src/infrastructure/di/container.ts` — MODIFIED (+2 registrations Transient)
- `pensieve/mobile/src/stores/settingsStore.ts` — MODIFIED (+githubRepo field + setGithubRepo action)
- `pensieve/mobile/src/components/dev/InAppLogger.tsx` — MODIFIED (+bouton Analyze + modal confirmation)
- `pensieve/mobile/src/screens/settings/SettingsScreen.tsx` — MODIFIED (+section Bug Reporting)
- `pensieve/mobile/tests/acceptance/features/story-7-3.feature` — NEW (7 scénarios)
- `pensieve/mobile/tests/acceptance/story-7-3.test.ts` — NEW (jest-cucumber)

# Story 7.2: Logs DevTools — Rotation FIFO (limite 100 entrées)

Status: review

<!-- Source: Issue GitHub #10 — https://github.com/yohikofox/pensieve/issues/10 -->

## Story

As a **développeur utilisant les DevTools en mode debug**,
I want **que le système de logs soit limité à 100 entrées avec rotation FIFO automatique**,
So that **l'app ne consomme pas de mémoire excessive en capturant des logs en continu, tout en gardant les 100 entrées les plus récentes disponibles pour le diagnostic**.

## Acceptance Criteria

### AC1 : Limite maximale à 100 entrées
**Given** le store de logs (`logsDebugStore`) contient 100 entrées
**When** une nouvelle entrée est ajoutée via `addLog()`
**Then** le store contient toujours exactement 100 entrées
**And** l'entrée la plus ancienne (index 0) est automatiquement supprimée
**And** la nouvelle entrée est la dernière dans le tableau

### AC2 : Comportement sous le seuil (< 100 entrées)
**Given** le store de logs contient moins de 100 entrées
**When** une nouvelle entrée est ajoutée via `addLog()`
**Then** le store contient `n + 1` entrées (pas de suppression)
**And** toutes les entrées précédentes sont préservées

### AC3 : Rotation transparente et automatique
**Given** la rotation FIFO est implémentée
**When** des logs arrivent en continu au-delà de 100 entrées
**Then** la rotation s'effectue sans action utilisateur
**And** aucun compteur "X/100" n'est affiché à l'utilisateur
**And** l'interface DevTools continue de fonctionner normalement

### AC4 : Pas de régression sur les fonctionnalités existantes
**Given** la rotation FIFO est activée
**When** j'utilise les DevTools
**Then** le toggle "Sniff" fonctionne normalement
**And** le bouton "Clear" vide complètement le store (sans modifier la limite)
**And** les niveaux de log (log/warn/error) restent correctement colorés
**And** l'auto-scroll vers le bas continue de fonctionner
**And** le header affiche le nombre d'entrées actuel (≤ 100)

### AC5 : Conservation des logs les plus récents
**Given** le buffer a atteint sa limite de 100 entrées
**When** de nouvelles entrées arrivent
**Then** les entrées les plus récentes sont toujours conservées
**And** les entrées les plus anciennes sont supprimées en premier (FIFO = First In, First Out)
**And** le contenu du store représente toujours les 100 derniers logs capturés

## Tasks / Subtasks

### Mobile Tasks (seul périmètre concerné)

- [x] **Task 1 : Implémenter la rotation FIFO dans logsDebugStore** (AC: 1, 2, 3, 5)
  - [x] Subtask 1.1 : Ajouter la constante `MAX_LOGS = 100` dans `logsDebugStore.ts`
    - Placer la constante avant la définition du store (exportable si besoin de tests)
    - Valeur fixe : 100 (pas configurable pour cette story)
  - [x] Subtask 1.2 : Modifier l'action `addLog` pour appliquer la limite FIFO
    - Logique : si `state.logs.length >= MAX_LOGS`, supprimer le premier élément avant d'ajouter
    - Implementation suggérée :
      ```typescript
      addLog: (entry) =>
        set((state) => {
          const newLogs = [...state.logs, entry];
          return {
            logs: newLogs.length > MAX_LOGS ? newLogs.slice(-MAX_LOGS) : newLogs,
          };
        }),
      ```
    - Aucune autre modification du store nécessaire (`clearLogs`, `setSniffing` inchangés)

### Testing Tasks

- [x] **Task 2 : Tests unitaires logsDebugStore** (AC: 1, 2, 3, 4, 5)
  - [x] Subtask 2.1 : Créer `mobile/src/components/dev/stores/__tests__/logsDebugStore.test.ts`
    - Test : addLog sous le seuil (ex: 5 entrées → 6 entrées)
    - Test : addLog à exactement la limite (100 → 100, pas 101)
    - Test : addLog au-delà de la limite (100 entrées + 1 → oldest supprimé)
    - Test : après 200 insertions → exactement 100 entrées conservées
    - Test : les entrées conservées sont bien les plus récentes (FIFO sémantique)
    - Test : clearLogs vide complètement (unrelated, non-regression)
    - Test : setSniffing non affecté par la limite (non-regression)

- [x] **Task 3 : Scénario BDD Gherkin** (AC: 1, 2, 3, 4, 5)
  - [x] Subtask 3.1 : Créer `mobile/tests/acceptance/features/story-7-2.feature`
    - Scénario 1 : Rotation FIFO — insertion de la 101ème entrée
    - Scénario 2 : Buffer sous seuil — pas de suppression
    - Scénario 3 : Préservation des logs les plus récents
    - Scénario 4 : Non-régression clearLogs
  - [x] Subtask 3.2 : Créer `mobile/tests/acceptance/story-7-2.test.ts`
    - Implémenter les step definitions avec jest-cucumber
    - Utiliser des instances directes du store (pas de mocks complexes)

## Dev Notes

### Contexte Architectural

Cette story n'affecte qu'**un seul fichier** : `logsDebugStore.ts`. Les composants d'affichage (`InAppLogger.tsx`, `DevPanel.tsx`) n'ont pas besoin d'être modifiés car ils lisent simplement le tableau `logs` du store.

**Fichier principal :**
```
mobile/src/components/dev/stores/logsDebugStore.ts
```

### Implémentation Existante (à modifier)

Le store actuel utilise :
```typescript
addLog: (entry) =>
  set((state) => ({
    logs: [...state.logs, entry],  // ← PAS de limite actuellement
  })),
```

→ Ajouter la logique FIFO avec `MAX_LOGS = 100`.

### Choix Technique : Array.slice vs Circular Buffer

L'issue GitHub mentionne une structure "circular buffer" ou "queue" pour O(1). Cependant, pour 100 entrées en JavaScript :
- `Array.slice(-100)` est **suffisamment performant** (négligeable)
- Une vraie structure circulaire serait over-engineering pour ce volume
- Zustand reconstruit le state à chaque mutation → l'optimisation mémoire est déjà limitée
- **Décision** : utiliser `Array.slice(-MAX_LOGS)` (simple, lisible, maintenable)

Si un besoin de performance se manifeste (ex: 10 000 entrées), créer une story dédiée.

### Interface LogEntry (inchangée)

```typescript
export interface LogEntry {
  timestamp: Date;
  level: 'log' | 'error' | 'warn';
  message: string;
}
```

### Compatibilité avec l'affichage

Le composant `LogsViewer` (dans `InAppLogger.tsx`) affiche `logs.length` dans le titre :
```tsx
<Text style={styles.embeddedTitle}>Dev Logs ({logs.length})</Text>
```
→ Avec la rotation FIFO, ce compteur affichera ≤ 100 naturellement. **Aucune modification UI nécessaire.**

### Intégration avec Story 7.1

L'interception console est déjà protégée par le debug mode (Story 7.1) :
```typescript
if (sniffing && isDebugModeEnabled()) {
  // ... addLog(...)
}
```
→ La rotation FIFO s'applique uniquement quand sniffing + debug mode sont actifs. **Pas de conflit.**

### Non-périmètre de cette story

- ❌ Persistence des logs (pas de base de données, pas d'AsyncStorage)
- ❌ Export des logs (future story ou Epic 7.3)
- ❌ Filtres par niveau (existant déjà, à ne pas modifier)
- ❌ Backend, Web — périmètre mobile uniquement

### Patterns de Test (conformité projet)

Les tests unitaires du store doivent utiliser `babel-jest` :
```bash
# Exécuter les tests unitaires
cd mobile && npx jest src/components/dev/stores/__tests__/logsDebugStore.test.ts
```

Les tests d'acceptance BDD utilisent `ts-jest` + `jest-cucumber` :
```bash
# Exécuter les tests BDD
cd mobile && npm run test:acceptance -- --testPathPattern="story-7-2"
```

### Référence Architecture

L'architecture globale des DevTools est :
```
DevPanel.tsx (floating modal)
  └── "📋 Logs" tab → LogsViewer (InAppLogger.tsx)
        └── useLogsDebugStore()
              └── logsDebugStore.ts ← SEUL FICHIER MODIFIÉ
```

`console.log` / `console.warn` → intercept → `addLog()` → FIFO → `logs[0..99]`

### Project Structure Notes

- Alignement structure : store dans `src/components/dev/stores/` (patterns dev tools)
- Test unitaire dans `src/components/dev/stores/__tests__/` (à créer, répertoire inexistant)
- Test BDD dans `tests/acceptance/features/story-7-2.feature` + `tests/acceptance/story-7-2.test.ts`
- Constante `MAX_LOGS` : exporter du module pour utilisation dans les tests
- Pas d'injection DI pour ce store (c'est un store Zustand global, pas un service DI)

### Contraintes ADR Applicables

**ADR-028 — Pas de `any` explicite** (vérifié par architecture tests) :
- L'action `addLog` utilise `LogEntry` typé — ne pas utiliser `any` dans la modification
- La constante `MAX_LOGS: number` doit être typée explicitement
- Exemple conforme :
  ```typescript
  export const MAX_LOGS = 100; // number inféré — OK
  addLog: (entry: LogEntry) => set((state: LogsDebugState) => { ... }) // types explicites
  ```

**ADR-022 — Pas de AsyncStorage** :
- Cette story ne nécessite AUCUNE persistance (logs in-memory uniquement)
- Ne pas importer `@react-native-async-storage/async-storage`

**Tests d'architecture** : Exécuter `npm run test:architecture` pour vérifier la conformité après modification.

### References

- [Source: GitHub Issue #10](https://github.com/yohikofox/pensieve/issues/10) — Spécification complète de la rotation FIFO
- [Source: logsDebugStore.ts] — `mobile/src/components/dev/stores/logsDebugStore.ts` — Implémentation actuelle (lignes 39-42)
- [Source: InAppLogger.tsx] — `mobile/src/components/dev/InAppLogger.tsx` — Composant d'affichage (non modifié)
- [Source: Story 7.1] — `_bmad-output/implementation-artifacts/stories/epic-7/7-1-support-mode-avec-permissions-backend.md` — Système de debug mode (prérequis)
- [Source: pensieve/CLAUDE.md] — Patterns test : babel-jest pour unit, ts-jest pour acceptance
- [Source: pensieve/mobile/CLAUDE.md] — Structure DDD, Zustand stores, patterns test

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

- Correction d'un test unitaire mal rédigé (attendu 6 au lieu de 5 pour 5 insertions)
- `jest.config.acceptance.js` ne chargeait pas `jest-setup.js` → `__DEV__` non défini → ajout `setupFilesAfterEnv`
- Échecs `npm run test:architecture` préexistants (ADR-022 x9, ADR-028 x68, ADR-019 x31, ADR-031 x7) — non liés à cette story

### Completion Notes List

- **Task 1** : `logsDebugStore.ts` modifié — `MAX_LOGS = 100` exporté + `addLog` avec `Array.slice(-MAX_LOGS)`. `clearLogs` et `setSniffing` inchangés. Conforme ADR-028 (pas de `any`), ADR-022 (pas d'AsyncStorage).
- **Task 2** : 13 tests unitaires (babel-jest) couvrant AC1-AC5 + non-régressions. Tous GREEN.
- **Task 3** : 4 scénarios Gherkin BDD (jest-cucumber) couvrant AC1-AC5. Tous GREEN. Fix collatéral : ajout `setupFilesAfterEnv` dans `jest.config.acceptance.js` pour exposer `__DEV__` global.

### File List

- `pensieve/mobile/src/components/dev/stores/logsDebugStore.ts` (modifié)
- `pensieve/mobile/src/components/dev/stores/__tests__/logsDebugStore.test.ts` (créé)
- `pensieve/mobile/tests/acceptance/features/story-7-2.feature` (créé)
- `pensieve/mobile/tests/acceptance/story-7-2.test.ts` (créé)
- `pensieve/mobile/jest.config.acceptance.js` (modifié — ajout setupFilesAfterEnv)

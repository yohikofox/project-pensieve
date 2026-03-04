# Story 8.14: Abandonner une Tâche

Status: done

<!-- Note: Validation optionnelle. Lancer validate-create-story pour vérification qualité avant dev-story. -->

## Story

As a **user**,
I want **to mark an action/todo as "abandoned" without permanently deleting it**,
So that **I can preserve the history of my decisions while keeping my active task list clean**.

## Context

Quick win identifié lors d'une session discovery avec un power user (2026-02-17).
L'"abandon" est un état sémantique distinct de la complétion et de la suppression : il conserve la traçabilité des décisions ("j'avais cette idée, j'ai choisi de ne pas la poursuivre").

**Distinction importante avec la Story 8.13 (Supprimer) :**
- **Supprimer** = hard delete en base locale, la tâche disparaît définitivement
- **Abandonner** = changement d'état, la tâche reste visible avec le marqueur "Abandonnée" (historique préservé)

**Valeur clé** : L'utilisateur peut retrouver les actions abandonnées et comprendre pourquoi elles n'ont pas été menées à terme — utile pour les décisions répétitives et l'auto-réflexion.

## Acceptance Criteria

### AC1: Action "Abandonner" dans le swipe contextuel
**Given** I am viewing a Todo in the Actions tab
**When** I swipe left on the Todo card (status = 'todo' or 'completed')
**Then** the swipe menu shows at minimum two actions: "Abandonner" et "Supprimer" (Story 8.13)
**And** "Abandonner" is visually distinct (grey/orange) from "Supprimer" (red)
**And** tapping "Abandonner" transitions the Todo to status = 'abandoned'
**And** haptic feedback is triggered if `settingsStore.hapticFeedbackEnabled`

### AC2: État visuel distinct "Abandonnée"
**Given** a Todo has status = 'abandoned'
**When** I view it in the Actions tab (filtered with "Abandonnées")
**Then** it displays a visual marker indicating "Abandonnée" (badge or icon 🚫)
**And** the styling is distinct from status = 'completed' (strikethrough) and status = 'todo' (normal)
**And** an abandoned icon distinguishes it at a glance

### AC3: Bouton "Abandonner" dans la vue détail
**Given** I am in the TodoDetailPopover of a Todo with status = 'todo'
**When** I look for action options
**Then** I see an "Abandonner cette tâche" secondary button (or contextual option)
**And** tapping it transitions the Todo to status = 'abandoned' immediately
**And** I remain in the detail view, now showing the updated abandoned state (with "Réactiver" option)

### AC4: Filtre dédié pour les tâches abandonnées
**Given** I am in the Actions tab
**When** I browse filter tabs
**Then** an "Abandonnées" tab is available alongside "Toutes", "À faire", "Faites"
**And** the tab shows the count of abandoned tasks
**And** selecting the filter displays only todos with status = 'abandoned'
**And** the "Abandonnées" tab is hidden (or shows 0) when there are no abandoned tasks

### AC5: Réactiver une tâche abandonnée (annuler abandon)
**Given** I am viewing an abandoned Todo (detail or list)
**When** I tap "Réactiver" (in detail view) or use swipe (in list)
**Then** the Todo transitions back to status = 'todo'
**And** it reappears in the "À faire" filter
**And** the abandoned marker is removed

### AC6: Synchronisation de l'état abandonné
**Given** I marked a Todo as abandoned (or reactivated it)
**When** the device is connected to the network
**Then** the state change is propagated to the backend via the existing sync pipeline
**And** the abandoned state appears on all synced devices
**And** if offline, the `_changed = 1` flag ensures sync on next connection

## Tasks / Subtasks

### Task 1: Hooks — useAbandonTodo et useReactivateTodo (AC1, AC3, AC5)
- [x] Subtask 1.1: Créer `mobile/src/contexts/action/hooks/useAbandonTodo.ts`
  - `mutationFn`: `todoRepository.update(todoId, { status: 'abandoned', updatedAt: Date.now() })`
  - `onSuccess`: `queryClient.invalidateQueries({ queryKey: ['todos'] })`
  - `onError`: Alert.alert d'erreur user-friendly
  - Haptic feedback dans `onSuccess` si `settingsStore.hapticFeedbackEnabled` (NotificationFeedbackType.Warning)
- [x] Subtask 1.2: Créer `mobile/src/contexts/action/hooks/useReactivateTodo.ts`
  - `mutationFn`: `todoRepository.update(todoId, { status: 'todo', updatedAt: Date.now() })`
  - Même pattern que useAbandonTodo
  - `onSuccess`: invalidation `["todos"]`
- [x] Subtask 1.3: Vérifier que `TodoRepository.update()` accepte `{ status: 'abandoned' }` (type-safe, TodoStatus inclut déjà 'abandoned')
- [x] Subtask 1.4: Vérifier la conformité ADR-023 (Result Pattern) dans TodoRepository.update()

### Task 2: Swipe-to-abandon dans ActionsTodoCard (AC1)
- [x] Subtask 2.1: Ouvrir `mobile/src/contexts/action/ui/ActionsTodoCard.tsx`
- [x] Subtask 2.2: Importer `Swipeable` depuis `react-native-gesture-handler` (déjà dans le projet — Story 8.13 l'ajoute aussi)
  - **Important**: Si Story 8.13 est déjà implémentée, le `Swipeable` wrapper existe déjà → étendre `renderRightActions` pour ajouter "Abandonner" à côté de "Supprimer"
  - Si Story 8.13 n'est PAS encore implémentée → créer le wrapper Swipeable complet (voir pattern dans Dev Notes)
- [x] Subtask 2.3: Ajouter bouton "Abandonner" dans `renderRightActions` :
  - Couleur : orange/gris (distinct du rouge "Supprimer")
  - Icône : 🚫 ou `ban-outline` (Ionicons)
  - Label : "Abandonner"
  - Action : appelle `abandonTodo.mutate(todo.id)`
  - Ne pas afficher si `todo.status === 'abandoned'` (déjà abandonné)
  - Ne pas afficher si `readonly === true` (contexte corbeille)
- [x] Subtask 2.4: Ajouter bouton "Réactiver" dans `renderRightActions` si `todo.status === 'abandoned'` :
  - Couleur : vert/bleu
  - Icône : `refresh-outline` (Ionicons)
  - Label : "Réactiver"
  - Action : appelle `reactivateTodo.mutate(todo.id)`

### Task 3: Bouton dans TodoDetailPopover (AC3, AC5)
- [x] Subtask 3.1: Ouvrir `mobile/src/contexts/action/ui/TodoDetailPopover.tsx`
- [x] Subtask 3.2: Si `todo.status !== 'abandoned'` → ajouter bouton "Abandonner cette tâche"
  - Bouton secondaire (non-destructif, orange ou gris)
  - Pas de confirmation dialog (abandon est réversible — contrairement au delete)
  - Appelle `abandonTodo.mutate(todo.id)`
  - Met à jour le state local du popover pour refléter l'état abandonné
- [x] Subtask 3.3: Si `todo.status === 'abandoned'` → afficher badge/indicateur "Abandonnée" + bouton "Réactiver"
  - Bouton "Reprendre cette tâche" (vert/bleu)
  - Appelle `reactivateTodo.mutate(todo.id)`
- [x] Subtask 3.4: Masquer le bouton "Compléter" si `todo.status === 'abandoned'` (ne pas permettre completion d'une tâche abandonnée sans réactiver d'abord)

### Task 4: Filtre "Abandonnées" (AC4)
- [x] Subtask 4.1: Ouvrir `mobile/src/contexts/action/hooks/useFilterState.ts`
  - Ajouter `"abandoned"` au type `FilterType = "all" | "active" | "completed" | "trash" | "abandoned"`
  - Mettre à jour la clé AsyncStorage `@pensine/actions_filter` pour accepter la nouvelle valeur
- [x] Subtask 4.2: Ouvrir `mobile/src/contexts/action/utils/filterTodos.ts`
  - Ajouter case `"abandoned"`:
    ```typescript
    case "abandoned":
      return todos.filter((t) => t.status === "abandoned");
    ```
  - Vérifier le case `"all"` : actuellement affiche 'todo' + 'completed' → ajouter 'abandoned' après completed ou laisser exclus (décision: exclure par défaut pour éviter la pollution du feed "Toutes")
- [x] Subtask 4.3: Ouvrir `mobile/src/contexts/action/ui/FilterTabs.tsx`
  - Ajouter tab "Abandonnées" avec badge count (visible seulement si count > 0)
  - Position : après "Faites", avant 🗑 (si existant)
  - Passer `abandonedCount` comme prop ou via l'objet `counts`
- [x] Subtask 4.4: Mettre à jour l'interface `counts` dans FilterTabs et `useFilteredTodoCounts` :
  - Ajouter `abandoned: number` aux counts
  - Étendre `countAllByStatus()` dans ITodoRepository + TodoRepository pour inclure abandoned

### Task 5: Styling visuel état abandonné (AC2)
- [x] Subtask 5.1: Dans `ActionsTodoCard.tsx`, ajouter rendu conditionnel si `todo.status === 'abandoned'`:
  - Opacité réduite (0.6) pour l'ensemble de la carte
  - Icône 🚫 à la place de la checkbox (non-tappable)
  - Texte description avec style "muted" (couleur #9ca3af atténuée, pas de strikethrough)
  - Désactiver checkbox (remplacée par icône abandonée)
- [x] Subtask 5.2: Définir les constantes de style pour l'état abandonné (couleurs explicites)

### Task 6: Tests BDD (AC1, AC3, AC4, AC5)
- [x] Subtask 6.1: Créer `mobile/tests/acceptance/features/story-8-14-abandonner-une-tache.feature`
- [x] Subtask 6.2: Créer `mobile/tests/acceptance/story-8-14.test.ts` avec jest-cucumber step definitions
- [x] Subtask 6.3: Implémenter minimum 4 scénarios BDD :
  1. **Abandon via detail view** (AC3) → todo.status = 'abandoned'
  2. **Réactivation depuis detail view** (AC5) → todo.status = 'todo'
  3. **Filtre Abandonnées affiche correctement** (AC4) → seules tâches abandoned visibles
  4. **Abandon via swipe** (AC1) → todo.status = 'abandoned'
- [x] Subtask 6.4: Pattern TodoRepository direct + SyncTrigger mocké (identique Story 8.13)

### Task 7: Non-régression (AC6)
- [x] Subtask 7.1: Lancer `npm run test:acceptance` dans `pensieve/mobile` — avant: 17 fail 24 pass → après: 16 fail 25 pass (aucune régression)
- [x] Subtask 7.2: Vérifier que les tests BDD existants des stories 5.x passent toujours — 4/4 story-8-13 passent
- [ ] Subtask 7.3: Vérifier manuellement sur device :
  - Filtre "À faire" n'affiche pas les tâches abandonnées
  - Filtre "Toutes" — exclut les abandonnées (comportement confirmé dans filterTodos)
  - Sync : tâche abandonnée sur device A apparaît abandonnée sur device B

## Dev Notes

### 🎯 Découverte Critique — Ce Qui Existe DÉJÀ

**L'état `abandoned` est déjà dans le code :**

```typescript
// mobile/src/contexts/action/domain/Todo.model.ts
export type TodoStatus = "todo" | "completed" | "abandoned";  // ← DÉJÀ LÀ
```

```sql
-- mobile/src/database/schema.ts (todos table)
status TEXT CHECK(status IN ('todo', 'completed', 'abandoned')) DEFAULT 'todo'  -- ← DÉJÀ LÀ
```

**Ce qui existe déjà — NE PAS réimplémenter :**
```
✅ TodoStatus type avec 'abandoned' — Todo.model.ts
✅ Contrainte CHECK SQL 'abandoned' — schema.ts
✅ TodoRepository.update(id, changes) — accepte { status: 'abandoned' }
✅ countByStatus(status) — peut compter les abandoned
✅ react-native-gesture-handler — déjà installée (Story 8.13 l'utilise)
✅ Pattern Swipeable — SwipeableCard.tsx (référence) + ActionsTodoCard.tsx (Story 8.13)
```

**Ce qui MANQUE — Travail réel de la story :**
```
❌ useAbandonTodo hook
❌ useReactivateTodo hook
❌ Swipe action "Abandonner" dans ActionsTodoCard
❌ Bouton "Abandonner" / "Réactiver" dans TodoDetailPopover
❌ FilterType "abandoned" dans useFilterState
❌ Cas "abandoned" dans filterTodos.ts
❌ Tab "Abandonnées" dans FilterTabs
❌ count abandoned dans useFilteredTodoCounts
❌ Styling visuel état abandonné dans ActionsTodoCard
❌ Tests BDD
```

### Coordination avec Story 8.13

Story 8.13 (Supprimer) implémente le `Swipeable` wrapper dans `ActionsTodoCard`.
**Deux scénarios possibles :**

**Si Story 8.13 est déjà implémentée :**
- `ActionsTodoCard` a déjà un `Swipeable` wrapper avec `renderRightActions`
- Étendre `renderRightActions` pour ajouter le bouton "Abandonner" à gauche du "Supprimer"
- Ordre recommandé : `[Abandonner (orange)] [Supprimer (rouge)]`

**Si Story 8.13 n'est PAS encore implémentée :**
- Créer le wrapper Swipeable complet (voir pattern ci-dessous)
- Inclure les deux boutons dès le départ

### Pattern Swipeable à réutiliser

```typescript
// Référence pattern : mobile/src/components/cards/SwipeableCard.tsx
import { Swipeable } from 'react-native-gesture-handler';

const renderRightActions = (
  progress: Animated.AnimatedInterpolation<number>,
  dragX: Animated.AnimatedInterpolation<number>
) => {
  // Bouton "Abandonner" (si pas encore abandonné)
  if (todo.status !== 'abandoned') {
    return (
      <View style={styles.swipeActionsContainer}>
        <Animated.View style={[styles.abandonAction, { transform: [{ translateX }] }]}>
          <Pressable onPress={() => abandonTodo.mutate(todo.id)} style={styles.abandonButton}>
            <Ionicons name="ban-outline" size={20} color="white" />
            <Text style={styles.actionLabel}>Abandonner</Text>
          </Pressable>
        </Animated.View>
        {/* Bouton Supprimer de Story 8.13 ici */}
      </View>
    );
  }

  // Bouton "Réactiver" si déjà abandonné
  return (
    <Animated.View style={[styles.reactivateAction, { transform: [{ translateX }] }]}>
      <Pressable onPress={() => reactivateTodo.mutate(todo.id)} style={styles.reactivateButton}>
        <Ionicons name="refresh-outline" size={20} color="white" />
        <Text style={styles.actionLabel}>Réactiver</Text>
      </Pressable>
    </Animated.View>
  );
};
```

### Hook useAbandonTodo

```typescript
// mobile/src/contexts/action/hooks/useAbandonTodo.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { Alert } from 'react-native';
import * as Haptics from 'expo-haptics';
import { container } from '@/infrastructure/di/container';
import { TOKENS } from '@/infrastructure/di/tokens';
import type { ITodoRepository } from '../domain/ITodoRepository';

export const useAbandonTodo = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (todoId: string) => {
      const repo = container.resolve<ITodoRepository>(TOKENS.ITodoRepository);
      const result = await repo.update(todoId, {
        status: 'abandoned',
        updatedAt: Date.now(),
      });
      // Vérifier Result (ADR-023)
      if (result === false) throw new Error('Update returned false — no changes or not found');
    },
    onSuccess: (_data, todoId) => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
      // Haptic (conditionnel)
      // Note: accéder au store settings si nécessaire
    },
    onError: (error: Error) => {
      Alert.alert(
        'Erreur',
        `Impossible d'abandonner la tâche : ${error.message}`,
        [{ text: 'OK' }]
      );
    },
  });
};
```

### useReactivateTodo

```typescript
// mobile/src/contexts/action/hooks/useReactivateTodo.ts
// Pattern identique à useAbandonTodo, avec status: 'todo'
export const useReactivateTodo = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (todoId: string) => {
      const repo = container.resolve<ITodoRepository>(TOKENS.ITodoRepository);
      await repo.update(todoId, { status: 'todo', updatedAt: Date.now() });
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
    onError: (error: Error) => {
      Alert.alert('Erreur', `Impossible de réactiver la tâche : ${error.message}`);
    },
  });
};
```

### filterTodos.ts — Extension

```typescript
// mobile/src/contexts/action/utils/filterTodos.ts
export type FilterType = "all" | "active" | "completed" | "trash" | "abandoned";  // ← Ajouter "abandoned"

export const filterTodos = (todos: Todo[], filter: FilterType): Todo[] => {
  switch (filter) {
    case "all":
      // N.B.: "Toutes" exclut les abandoned (pollution du feed)
      // Seul le filtre "Abandonnées" les montre
      const active = todos.filter((t) => t.status === "todo");
      const completed = todos.filter((t) => t.status === "completed");
      return [...active, ...completed];
    case "active":
      return todos.filter((t) => t.status === "todo");
    case "completed":
      return todos
        .filter((t) => t.status === "completed")
        .sort((a, b) => (b.completedAt ?? 0) - (a.completedAt ?? 0));
    case "abandoned":  // ← NOUVEAU
      return todos.filter((t) => t.status === "abandoned");
    case "trash":
      return todos;  // Pre-filtered from DB
  }
};
```

### FilterTabs — Ajout tab Abandonnées

Le composant accepte actuellement `counts: { all; active; completed; deleted }`.
Étendre avec `abandoned: number` :

```typescript
// mobile/src/contexts/action/ui/FilterTabs.tsx
interface FilterTabsCounts {
  all: number;
  active: number;
  completed: number;
  abandoned: number;  // ← NOUVEAU
  deleted: number;
}

// Rendu conditionnel (si abandoned > 0 seulement) :
{counts.abandoned > 0 && (
  <FilterTab
    label="Abandonnées"
    count={counts.abandoned}
    isActive={activeFilter === 'abandoned'}
    onPress={() => onFilterChange('abandoned')}
    icon="ban-outline"
    color={theme.colors.warning}
  />
)}
```

### Couleurs et Styles pour état abandonné

```typescript
// Styles recommandés (dans ActionsTodoCard.tsx)
const abandonedStyles = {
  card: {
    opacity: 0.6,  // Atténuer visuellement
  },
  description: {
    color: theme.colors.neutral[400],  // Gris atténué
    // PAS de strikethrough (réservé à completed)
  },
  badge: {
    backgroundColor: theme.colors.neutral[200],
    color: theme.colors.neutral[600],
    // Label: "Abandonnée" + icône 🚫
  },
  swipeButton: {
    backgroundColor: '#F97316',  // Orange-500 (Tailwind)
    // ou theme.colors.warning
  },
};
```

### ADR Compliance

- **ADR-021 (DI Transient First)** : Résoudre `ITodoRepository` via `container.resolve()` dans le hook (lazy), jamais en singleton. Pattern conforme à Story 8.13 et toutes les stories précédentes.
- **ADR-023 (Result Pattern)** : `TodoRepository.update()` retourne `boolean | false` actuellement (Issue #9 fix). S'assurer de gérer le cas `false` dans les hooks (pas de throw → Alert user-friendly).
- **ADR-024 (Clean Code)** : Constantes pour les couleurs (pas de magic strings `'#F97316'`), noms révélateurs d'intention.
- **ADR-022 (AsyncStorage)** : `useFilterState` persiste le filtre dans `@pensine/actions_filter`. La nouvelle valeur `"abandoned"` sera automatiquement persistée sans migration car c'est une string.

### ⚠️ Correction Tech Note Initiale

La story initiale mentionne "Sync via WatermelonDB" — **INCORRECT**. Le projet utilise **OP-SQLite** (migration ADR-018 faite en Story 14.1). L'état `abandoned` sera synchronisé par le mécanisme OP-SQLite + `SyncTrigger.queueSync()` existant dans `TodoRepository.update()`.

### Project Structure Notes

**Fichiers à modifier :**
```
pensieve/mobile/src/contexts/action/utils/filterTodos.ts
  → Ajouter FilterType "abandoned" + case dans switch

pensieve/mobile/src/contexts/action/hooks/useFilterState.ts
  → Ajouter "abandoned" à FilterType

pensieve/mobile/src/contexts/action/ui/FilterTabs.tsx
  → Ajouter tab "Abandonnées" + prop counts.abandoned

pensieve/mobile/src/contexts/action/ui/ActionsTodoCard.tsx
  → Ajouter swipe "Abandonner"/"Réactiver" dans renderRightActions
  → Ajouter styling conditionnel état abandonné

pensieve/mobile/src/contexts/action/ui/TodoDetailPopover.tsx
  → Ajouter bouton "Abandonner cette tâche" (si status ≠ abandoned)
  → Ajouter indicateur + bouton "Réactiver" (si status = abandoned)
  → Masquer toggle Compléter si status = abandoned
```

**Nouveaux fichiers à créer :**
```
pensieve/mobile/src/contexts/action/hooks/useAbandonTodo.ts
pensieve/mobile/src/contexts/action/hooks/useReactivateTodo.ts
pensieve/mobile/tests/acceptance/features/story-8-14-abandonner-une-tache.feature
pensieve/mobile/tests/acceptance/story-8-14.test.ts
```

**Fichiers à NE PAS modifier :**
```
pensieve/mobile/src/contexts/action/domain/Todo.model.ts  (TodoStatus = "abandoned" déjà là)
pensieve/mobile/src/contexts/action/data/TodoRepository.ts  (update() déjà là)
pensieve/mobile/src/contexts/action/domain/ITodoRepository.ts  (update() déjà là)
pensieve/mobile/src/database/schema.ts  (CHECK constraint 'abandoned' déjà là)
```

### References

- [Source: pensieve/mobile/src/contexts/action/domain/Todo.model.ts] — `TodoStatus` type avec 'abandoned' déjà défini
- [Source: pensieve/mobile/src/database/schema.ts (todos table)] — `CHECK(status IN ('todo', 'completed', 'abandoned'))`
- [Source: pensieve/mobile/src/contexts/action/data/TodoRepository.ts] — `update(id, changes)` + `countByStatus()`
- [Source: pensieve/mobile/src/contexts/action/hooks/useFilterState.ts] — `FilterType` + `AsyncStorage` persistence
- [Source: pensieve/mobile/src/contexts/action/utils/filterTodos.ts] — Logic filtre (à étendre)
- [Source: pensieve/mobile/src/contexts/action/ui/FilterTabs.tsx] — Interface `counts` (à étendre)
- [Source: pensieve/mobile/src/contexts/action/ui/ActionsTodoCard.tsx] — Carte todo (swipe à ajouter)
- [Source: pensieve/mobile/src/contexts/action/ui/TodoDetailPopover.tsx] — Popover (boutons à ajouter)
- [Source: pensieve/mobile/src/components/cards/SwipeableCard.tsx] — Pattern Swipeable de référence
- [Source: _bmad-output/implementation-artifacts/stories/epic-8/8-13-supprimer-une-tache.md] — Story sister (swipe pattern, coordination)
- [Source: _bmad-output/implementation-artifacts/stories/epic-5/5-3-filtres-et-tri-des-actions.md] — Filtres existants (à étendre)
- [Source: _bmad-output/implementation-artifacts/stories/epic-5/5-4-completion-et-navigation-des-actions.md] — TodoDetailPopover patterns
- [Source: _bmad-output/project-context.md#Testing Rules] — jest-cucumber BDD, babel-jest unit, pas jest-expo

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

N/A — Story de création

### Code Review Notes (2026-03-04)

#### Dette architecturale H1 — Violation ADR-031 (non bloquant pour livraison)

**Constat :** `useAbandonTodo` et `useReactivateTodo` encodent la règle métier "abandonner" comme une mutation de données brutes dans la couche UI :
```typescript
todoRepository.update(todoId, { status: 'abandoned', updatedAt: Date.now() })
```

**Problème 1 — Entité anémique** : `Todo` est une interface sans comportement. "Abandonner" est une transition d'état métier qui devrait être une méthode sur l'entité (`todo.abandon()`) per ADR-031.

**Problème 2 — API publique infrastructure** : les hooks exposent `mutate` (terme React Query/infrastructure) au lieu d'une intention métier (`abandon`, `reactivate`).

**Solution attendue** (requiert deux ADRs) :
1. **ADR-031** (déjà Accepted) : migrer `Todo` en classe riche avec `abandon(): Result<void>` et `reactivate(): Result<void>`
2. **ADR-038** (Draft soumis à Winston) : `TodosListStore` + `TodoDetailStore` Zustand — le `TodoDetailStore` charge par `todoId`, applique les règles via l'entité, persiste via `repository.save(todo)`

**Draft ADR :** `_bmad-output/planning-artifacts/adrs/ADR-038-draft-winston-zustand-screen-stores.md`

**Non bloquant** : les AC1-AC6 sont implémentés et fonctionnels. Cette dette sera adressée dans une story technique dédiée après validation de Winston.

#### Findings medium corrigés (2026-03-04)
- ✅ M1 : `const buttonCount = todo.status === 'abandoned' ? 2 : 2` → supprimé, remplacé par `const totalWidth = 2 * 90`
- ✅ M2 : `updatedAt: Date.now()` retiré de `useAbandonTodo` et `useReactivateTodo` (silencieusement ignoré par le repository — `updated_at` est déjà géré inconditionnellement par `TodoRepository.update()`)
- ✅ M3 : `disabled={abandonTodo.isPending}` + `opacity: 0.5` ajoutés sur les boutons Abandonner et Réactiver dans `TodoDetailPopover`
- ✅ M4 : label "🚫 Abandonnées" ajouté sur la tab dans `FilterTabs` (cohérent avec "Toutes", "À faire", "Faites")
- ✅ L1 : block scope `{}` ajouté sur `case "all"` dans `filterTodos.ts` (no-case-declarations)

### Completion Notes List

Ultimate context engine analysis completed — comprehensive developer guide created.

**Analyse réalisée :**
- `TodoStatus = "todo" | "completed" | "abandoned"` : type DÉJÀ défini dans Todo.model.ts — pas de migration modèle nécessaire
- `CHECK(status IN ('todo', 'completed', 'abandoned'))` : contrainte SQL DÉJÀ dans schema.ts — pas de migration DB nécessaire
- `TodoRepository.update(id, { status: 'abandoned' })` : fonctionne directement — pas de nouvelle méthode repository nécessaire
- Scope réel : uniquement UI (swipe, détail, filtre), hooks (useAbandonTodo, useReactivateTodo), et tests BDD
- Coordination avec Story 8.13 : le wrapper Swipeable était déjà présent — `renderRightActions` étendu

**Corrections apportées par rapport à la story initiale :**
- Tech Note "sync via WatermelonDB" → **INCORRECTE** (project utilise OP-SQLite depuis ADR-018 / Story 14.1)
- "Ajouter `status: 'todo' | 'completed' | 'abandoned'`" → **DÉJÀ FAIT** dans le code
- Référence à migration BDD → **NON REQUISE** (contrainte CHECK déjà présente)

**Implémentation réalisée (2026-03-04) :**
- ✅ `useAbandonTodo.ts` + `useReactivateTodo.ts` créés (pattern useDeleteTodo)
- ✅ `filterTodos.ts` : FilterType étendu + case `"abandoned"` ajouté
- ✅ `useFilterState.ts` : FilterType étendu avec `"abandoned"`
- ✅ `ITodoRepository.ts` : `countAllByStatus()` retourne désormais `{ ..., abandoned: number }`
- ✅ `TodoRepository.ts` : SQL étendu pour compter abandoned (exclut abandoned du `all_count`)
- ✅ `useFilteredTodoCounts.ts` : retourne `abandoned` count
- ✅ `FilterTabs.tsx` : tab "Abandonnées" 🚫 (visible seulement si count > 0)
- ✅ `ActionsTodoCard.tsx` : swipe étendu [Abandonner/Réactiver + Supprimer], styling abandonné
- ✅ `TodoDetailPopover.tsx` : bouton Abandonner/Réactiver, toggle Compléter masqué si abandoned
- ✅ 4/4 BDD tests verts, 0 régression ajoutée (17→16 failing pre-existants)

### File List

**Nouveaux fichiers :**
- `pensieve/mobile/src/contexts/action/hooks/useAbandonTodo.ts`
- `pensieve/mobile/src/contexts/action/hooks/useReactivateTodo.ts`
- `pensieve/mobile/tests/acceptance/features/story-8-14-abandonner-une-tache.feature`
- `pensieve/mobile/tests/acceptance/story-8-14.test.ts`

**Fichiers modifiés :**
- `pensieve/mobile/src/contexts/action/utils/filterTodos.ts` — FilterType + case abandoned
- `pensieve/mobile/src/contexts/action/hooks/useFilterState.ts` — FilterType extended
- `pensieve/mobile/src/contexts/action/domain/ITodoRepository.ts` — countAllByStatus return type
- `pensieve/mobile/src/contexts/action/data/TodoRepository.ts` — countAllByStatus SQL + return
- `pensieve/mobile/src/contexts/action/hooks/useFilteredTodoCounts.ts` — abandoned count
- `pensieve/mobile/src/contexts/action/ui/FilterTabs.tsx` — counts type + tab Abandonnées
- `pensieve/mobile/src/contexts/action/ui/ActionsTodoCard.tsx` — swipe extend + abandoned styling
- `pensieve/mobile/src/contexts/action/ui/TodoDetailPopover.tsx` — abandon/reactivate buttons

**Fichiers de documentation :**
- `_bmad-output/implementation-artifacts/sprint-status.yaml` — status in-progress
- `_bmad-output/implementation-artifacts/stories/epic-8/8-14-abandonner-une-tache.md` — this file

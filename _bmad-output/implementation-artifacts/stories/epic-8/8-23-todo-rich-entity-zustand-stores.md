# Story 8.23: Migration Todo Classe Riche + Zustand Stores (ADR-031 / ADR-038)

Status: ready-for-dev

<!-- Note: Validation optionnelle. Lancer validate-create-story pour vérification qualité avant dev-story. -->

## Story

As a **developer**,
I want **to migrate `Todo` to a rich domain entity and introduce Zustand List/Detail stores for the Action bounded context**,
So that **business rules (abandon, reactivate) live in the domain layer, and the UI delegates intent to stores instead of calling repositories directly**.

## Context

**Déclencheur :** Code review Story 8.14 (2026-03-04) — `useAbandonTodo` et `useReactivateTodo` encodent la règle métier "abandonner" comme une mutation de données brutes dans la couche UI, en violation d'ADR-031.

**Dette documentée dans Story 8.14 (H1) :**
```typescript
// ❌ Pattern actuel — règle métier dans la couche UI
todoRepository.update(todoId, { status: 'abandoned', updatedAt: Date.now() })

// ✅ Pattern cible — intention métier déléguée au store → entité
useTaskDetailStore.getState().abandon()
```

**ADRs concernés :**
- **ADR-031** (Rich Domain Entities — Accepted) : `Todo` doit être une classe avec comportement (`abandon()`, `reactivate()`) retournant `Result<void>`
- **ADR-038** (Zustand Stores List/Detail — Accepted) : Pattern `useTodosListStore` + `useTodoDetailStore` pour les screens Action

**Ordre de mise en œuvre (séquentiel) :**
1. Migrer `Todo` en classe riche (ADR-031) — prérequis absolu aux étapes suivantes
2. Implémenter `useTodosListStore` (ADR-038 — List Store)
3. Implémenter `useTodoDetailStore` (ADR-038 — Detail Store)
4. Refactorer `useAbandonTodo` et `useReactivateTodo` vers le store
5. Supprimer les `useQuery(['todos'])` dans les hooks UI devenus redondants

## Acceptance Criteria

### AC1: Todo est une classe riche conforme ADR-031
**Given** the `Todo` domain model
**When** I inspect `mobile/src/contexts/action/domain/Todo.model.ts`
**Then** `Todo` is exported as a `class` (not an interface)
**And** it has an `abandon(): Result<void>` method that validates the state transition (only from `'todo'` or `'completed'`)
**And** it has a `reactivate(): Result<void>` method that validates the state transition (only from `'abandoned'`)
**And** it has a static `fromSnapshot(snapshot: TodoSnapshot): Todo` factory method
**And** the `TodoSnapshot` interface remains for serialization/persistence purposes
**And** all existing tests that referenced `Todo` as an interface continue to pass (type-compatible)

### AC2: useTodosListStore est conforme ADR-038
**Given** the Zustand list store at `mobile/src/stores/useTodosListStore.ts`
**When** a screen uses `useFocusEffect` to call `hydrate()`
**Then** the store loads all todos via `ITodoRepository.findAll()`
**And** `onMutation(todoId)` selectively refreshes only the affected todo via `ITodoRepository.findById(todoId)`
**And** the store does NOT import `@tanstack/react-query`
**And** the store does NOT reference `queryClient`
**And** dependency resolution uses `container.resolve()` lazily inside actions (never at module level)

### AC3: useTodoDetailStore est conforme ADR-038 + ADR-031
**Given** the Zustand detail store at `mobile/src/stores/useTodoDetailStore.ts`
**When** `load(todoId)` is called
**Then** the store fetches the `Todo` entity via `ITodoRepository.findById(todoId)`
**And** `abandon()` calls `todo.abandon()` (ADR-031 rule), then `repository.save(todo)` (ADR-031 R8), then notifies the list store via `onMutationCallback`
**And** `reactivate()` calls `todo.reactivate()`, then `repository.save(todo)`, then notifies the list store
**And** if `todo.abandon()` returns a failure Result, the store propagates it without calling the repository
**And** the store does NOT call `repository.update(todoId, { status: 'abandoned' })` (ADR-038 + ADR-031 violation)

### AC4: useAbandonTodo et useReactivateTodo délèguent au store
**Given** the existing hooks `useAbandonTodo.ts` and `useReactivateTodo.ts`
**When** they are called from a component
**Then** they delegate to `useTodoDetailStore.getState().abandon()` and `useTodoDetailStore.getState().reactivate()` respectively
**And** they no longer call `todoRepository.update()` directly
**And** haptic feedback and error handling remain intact

### AC5: Les architecture tests passent
**Given** the architecture test suite at `mobile/tests/architecture/architecture.test.ts`
**When** `npm run test:architecture` is executed
**Then** all ADR-038 tests pass (no `queryClient` in stores, no `repository.update(id, rawFields)` in stores)
**And** the ADR-031 test passes (`*.model.ts` exports a class, not an interface)
**And** no regressions on existing architecture tests

### AC6: Les tests BDD existants de Story 8.14 passent toujours
**Given** the BDD tests in `mobile/tests/acceptance/story-8-14.test.ts`
**When** `npm run test:acceptance` is executed
**Then** the 4 Story 8.14 scenarios still pass (abandon via detail, réactivation, filtre, swipe)
**And** no other acceptance test regresses

## Tasks / Subtasks

### Task 1: Migrer Todo en classe riche (ADR-031) — PRÉREQUIS
- [ ] Subtask 1.1: Lire `mobile/src/contexts/action/domain/Todo.model.ts` + `ITodoRepository.ts` + `TodoRepository.ts` — inventaire complet avant modification
- [ ] Subtask 1.2: Créer l'interface `TodoSnapshot` (ou renommer l'interface existante) pour la sérialisation
- [ ] Subtask 1.3: Transformer `Todo` de interface en `class` :
  - Propriétés `readonly` : `id`, `title`, `status`, `priority`, `source`, `captureId`, `createdAt`, `updatedAt`, `completedAt?`, `dueDate?`
  - Constructeur privé + factory statique `fromSnapshot(s: TodoSnapshot): Todo`
  - Méthode `abandon(): Result<void>` — valide que `status !== 'abandoned'` avant la transition, mute `this.status` si succès
  - Méthode `reactivate(): Result<void>` — valide que `status === 'abandoned'` avant la transition, mute `this.status` si succès
  - Méthode `complete(): Result<void>` — valide que `status !== 'completed'` et `status !== 'abandoned'`
  - Méthode `toSnapshot(): TodoSnapshot` — pour la persistence
- [ ] Subtask 1.4: Mettre à jour `TodoRepository.ts` :
  - `findAll()` et `findById()` retournent `Todo.fromSnapshot(row)` au lieu d'objets bruts
  - `save(entity: Todo)` accepte une entité `Todo` et persiste via `entity.toSnapshot()`
  - Conserver `update(id, changes)` pour la compatibilité ascendante (marqué @deprecated)
- [ ] Subtask 1.5: Mettre à jour `ITodoRepository.ts` : ajouter `save(entity: Todo): Promise<RepositoryResult<void>>`
- [ ] Subtask 1.6: Vérifier que les tests unitaires existants de `TodoRepository` compilent toujours (adapter les mocks si nécessaire)

### Task 2: Créer useTodosListStore (ADR-038 Pattern 09)
- [ ] Subtask 2.1: Créer `mobile/src/stores/useTodosListStore.ts` en suivant `_patterns/09-zustand-list-store.ts`
  - État : `todos: Todo[]`, `isLoading: boolean`, `filter: FilterType`, `sort: SortType`, `counts: TodoCounts`
  - `hydrate()` : `container.resolve<ITodoRepository>(TOKENS.ITodoRepository)` lazy → `repo.findAll()`
  - `onMutation(todoId)` : `repo.findById(todoId)` — si SUCCESS met à jour l'item, si NOT_FOUND retire de la liste
  - `setFilter()` / `setSort()` : synchrones
  - Import `RepositoryResultType` (pas de comparaison avec string literal `'SUCCESS'`)
  - Jamais d'import `@tanstack/react-query`
- [ ] Subtask 2.2: Vérifier que `src/stores/` existe — sinon le créer
- [ ] Subtask 2.3: Exporter depuis `src/stores/index.ts` (créer si absent)

### Task 3: Créer useTodoDetailStore (ADR-038 Pattern 10)
- [ ] Subtask 3.1: Créer `mobile/src/stores/useTodoDetailStore.ts` en suivant `_patterns/10-zustand-detail-store.ts`
  - État : `todoId: string | null`, `todo: Todo | null`, `isLoading: boolean`, `_onMutationCallback: ((id: string) => void) | null`
  - `load(todoId)` : `container.resolve<ITodoRepository>` lazy → `repo.findById(todoId)`
  - `setOnMutationCallback(cb)` : setter
  - `abandon()` :
    1. `todo.abandon()` → si failure, retourner le Result
    2. `repo.save(todo)` → si failure, retourner le Result
    3. `set({ todo })` — état local mis à jour
    4. `_onMutationCallback?.(todo.id)` — notifie le List Store
    5. `return success(undefined)`
  - `reactivate()` : même flux avec `todo.reactivate()`
  - Import `RepositoryResultType`, `success`, `notFound` depuis `shared/domain/Result`
- [ ] Subtask 3.2: Exporter depuis `src/stores/index.ts`

### Task 4: Refactorer useAbandonTodo et useReactivateTodo (AC4)
- [ ] Subtask 4.1: Mettre à jour `mobile/src/contexts/action/hooks/useAbandonTodo.ts` :
  - Supprimer l'import `useMutation` / `useQueryClient`
  - Déléguer à `useTodoDetailStore.getState().abandon()`
  - Conserver haptic feedback (`NotificationFeedbackType.Warning`) et Alert d'erreur
  - Conserver l'interface publique (signature du hook inchangée)
- [ ] Subtask 4.2: Mettre à jour `mobile/src/contexts/action/hooks/useReactivateTodo.ts` — même pattern
- [ ] Subtask 4.3: Vérifier que `TodoDetailPopover.tsx` et `ActionsTodoCard.tsx` n'ont pas besoin de modification (délèguent déjà aux hooks)

### Task 5: Supprimer les useQuery(['todos']) redondants
- [ ] Subtask 5.1: Identifier tous les `useQuery(['todos'])` dans `src/contexts/action/hooks/` (en dehors des repositories)
- [ ] Subtask 5.2: Remplacer les usages par des sélecteurs sur `useTodosListStore` si applicable
- [ ] Subtask 5.3: Si un `useQuery` est utilisé pour la liste principale des todos dans l'écran Actions, le remplacer par `useTodosListStore` + `useFocusEffect(hydrate)`
- [ ] Subtask 5.4: Ne pas supprimer les `useQuery` côté repository (React Query reste pertinent pour les appels réseau — ADR-038 D2)

### Task 6: Validation architecture et tests
- [ ] Subtask 6.1: Lancer `npm run test:architecture` — tous les tests ADR-038 et ADR-031 doivent passer
- [ ] Subtask 6.2: Lancer `npm run test:acceptance` — vérifier que les 4 scénarios Story 8.14 passent toujours
- [ ] Subtask 6.3: Lancer `npm run test:unit` — vérifier aucune régression sur les unit tests existants
- [ ] Subtask 6.4: Vérifier manuellement sur device : abandon → réactivation → abandon fonctionne, l'état est cohérent entre la liste et le détail

## Dev Notes

### Référence Patterns (OBLIGATOIRE — lire avant d'implémenter)

```
_patterns/08-domain-entity.ts    → Pattern Rich Domain Entity (ADR-031)
_patterns/09-zustand-list-store.ts  → Pattern List Store (ADR-038) — copier la structure
_patterns/10-zustand-detail-store.ts → Pattern Detail Store (ADR-038) — copier la structure
```

**RÈGLE ABSOLUE** : Copier la structure des patterns, remplacer uniquement la logique métier.

### Todo Classe Riche — Structure Attendue

```typescript
// mobile/src/contexts/action/domain/Todo.model.ts
import { Result, success, failure, RepositoryResultType } from '../../shared/domain/Result';

export interface TodoSnapshot {
  readonly id: string;
  readonly title: string;
  readonly status: TodoStatus;
  readonly priority: TodoPriority;
  readonly source: string;
  readonly captureId: string | null;
  readonly createdAt: number;
  readonly updatedAt: number;
  readonly completedAt: number | null;
  readonly dueDate: number | null;
}

export class Todo {
  private constructor(private _snapshot: TodoSnapshot) {}

  static fromSnapshot(s: TodoSnapshot): Todo {
    return new Todo({ ...s });
  }

  get id(): string { return this._snapshot.id; }
  get title(): string { return this._snapshot.title; }
  get status(): TodoStatus { return this._snapshot.status; }
  // ... autres getters

  toSnapshot(): TodoSnapshot { return { ...this._snapshot }; }

  abandon(): Result<void> {
    if (this._snapshot.status === 'abandoned') {
      return failure('ALREADY_ABANDONED', 'Todo is already abandoned');
    }
    this._snapshot = { ...this._snapshot, status: 'abandoned', updatedAt: Date.now() };
    return success(undefined);
  }

  reactivate(): Result<void> {
    if (this._snapshot.status !== 'abandoned') {
      return failure('NOT_ABANDONED', 'Todo is not in abandoned state');
    }
    this._snapshot = { ...this._snapshot, status: 'todo', updatedAt: Date.now() };
    return success(undefined);
  }

  complete(): Result<void> {
    if (this._snapshot.status === 'abandoned') {
      return failure('IS_ABANDONED', 'Cannot complete an abandoned todo');
    }
    if (this._snapshot.status === 'completed') {
      return failure('ALREADY_COMPLETED', 'Todo is already completed');
    }
    this._snapshot = { ...this._snapshot, status: 'completed', completedAt: Date.now(), updatedAt: Date.now() };
    return success(undefined);
  }
}
```

### ITodoRepository — Extension

```typescript
// Ajouter à mobile/src/contexts/action/domain/ITodoRepository.ts
save(entity: Todo): Promise<RepositoryResult<void>>;
```

### Communication List ↔ Detail Store

```typescript
// Dans l'écran Actions (ou ActionsTodoCard parent)
useEffect(() => {
  useTodoDetailStore.getState().setOnMutationCallback(
    (id) => useTodosListStore.getState().onMutation(id)
  );
}, []);
```

### Result Pattern — Rappel critique

```typescript
// ✅ Correct — comparer avec l'enum
import { RepositoryResultType } from '../../shared/domain/Result';
if (result.type === RepositoryResultType.SUCCESS) {
  result.data!  // data est optionnel, non-null assertion requise
}

// ❌ INTERDIT — string literal 'SUCCESS' ne correspond pas à l'enum
if (result.type === 'SUCCESS') { ... }
```

### ADR Compliance Checklist

- **ADR-031** : `Todo` est une classe avec `abandon()` / `reactivate()` / `complete()` retournant `Result<void>`
- **ADR-031 R8** : `repository.save(entity)` — jamais `repository.update(id, rawFields)` dans les stores
- **ADR-038 D1** : Singleton `create` (pas `createStore + Context`)
- **ADR-038 D2** : Stores → repository direct — jamais `@tanstack/react-query` dans les stores
- **ADR-038 D3** : Pas de `queryClient` / `invalidateQueries` dans les stores
- **ADR-021** : `container.resolve()` lazy dans les actions, jamais au niveau module

### Fichiers à créer

```
pensieve/mobile/src/stores/useTodosListStore.ts
pensieve/mobile/src/stores/useTodoDetailStore.ts
pensieve/mobile/src/stores/index.ts  (si absent)
```

### Fichiers à modifier

```
pensieve/mobile/src/contexts/action/domain/Todo.model.ts
  → Transformer interface → class + TodoSnapshot
pensieve/mobile/src/contexts/action/domain/ITodoRepository.ts
  → Ajouter save(entity: Todo)
pensieve/mobile/src/contexts/action/data/TodoRepository.ts
  → findAll/findById retournent Todo.fromSnapshot()
  → Implémenter save(entity: Todo)
pensieve/mobile/src/contexts/action/hooks/useAbandonTodo.ts
  → Déléguer à useTodoDetailStore.getState().abandon()
pensieve/mobile/src/contexts/action/hooks/useReactivateTodo.ts
  → Déléguer à useTodoDetailStore.getState().reactivate()
```

### Fichiers à NE PAS modifier

```
pensieve/mobile/src/contexts/action/ui/ActionsTodoCard.tsx  (pas de changement d'interface)
pensieve/mobile/src/contexts/action/ui/TodoDetailPopover.tsx  (pas de changement d'interface)
pensieve/mobile/src/contexts/action/utils/filterTodos.ts  (logique inchangée)
pensieve/mobile/src/database/schema.ts  (schema SQL inchangé)
```

### References

- [ADR-031 — Rich Domain Entities] `_bmad-output/planning-artifacts/adrs/` (Accepted)
- [ADR-038 — Zustand Stores List/Detail] `_bmad-output/planning-artifacts/adrs/ADR-038-draft-winston-zustand-screen-stores.md` (Accepted)
- [Pattern 08 — Domain Entity] `pensieve/mobile/_patterns/08-domain-entity.ts`
- [Pattern 09 — List Store] `pensieve/mobile/_patterns/09-zustand-list-store.ts`
- [Pattern 10 — Detail Store] `pensieve/mobile/_patterns/10-zustand-detail-store.ts`
- [Architecture Tests] `pensieve/mobile/tests/architecture/architecture.test.ts`
- [Story 8.14 — Dette H1] `_bmad-output/implementation-artifacts/stories/epic-8/8-14-abandonner-une-tache.md`
- [Result.ts] `pensieve/mobile/src/contexts/shared/domain/Result.ts`

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

N/A — Story technique créée suite au code review 8.14 (Winston — 2026-03-04)

### Completion Notes List

Story créée par Winston (Architect) le 2026-03-04 suite à la validation d'ADR-038.

Ordre impératif :
1. Task 1 (Todo class) AVANT Task 2 et 3 (stores) — le store ne peut pas appeler `todo.abandon()` si `Todo` reste une interface
2. Task 4 (refacto hooks) APRÈS Task 3 (detail store opérationnel)
3. Task 5 (suppression useQuery) APRÈS Task 2 (list store opérationnel)

### File List

**Nouveaux fichiers :**
- `pensieve/mobile/src/stores/useTodosListStore.ts`
- `pensieve/mobile/src/stores/useTodoDetailStore.ts`

**Fichiers modifiés :**
- `pensieve/mobile/src/contexts/action/domain/Todo.model.ts`
- `pensieve/mobile/src/contexts/action/domain/ITodoRepository.ts`
- `pensieve/mobile/src/contexts/action/data/TodoRepository.ts`
- `pensieve/mobile/src/contexts/action/hooks/useAbandonTodo.ts`
- `pensieve/mobile/src/contexts/action/hooks/useReactivateTodo.ts`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `_bmad-output/implementation-artifacts/stories/epic-8/8-23-todo-rich-entity-zustand-stores.md`

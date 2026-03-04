---
adr: ADR-038
title: "Zustand Stores — Pattern List/Detail pour les bounded contexts mobile"
date: 2026-03-04
status: "✅ Accepted"
context: "Phase 4 - Implementation - Mobile (React Native)"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
triggered_by: "Code Review Story 8.14 — constat que les hooks accèdent directement aux repositories sans couche d'état UI"
related_adrs:
  - ADR-031 (Rich Domain Entities — classes avec comportement)
  - ADR-021 (DI Lifecycle — Transient First)
  - ADR-022 (State Persistence — OP-SQLite)
---

# ADR-038 : Zustand Stores — Pattern List/Detail

**Date:** 2026-03-04
**Status:** Accepted
**Déclencheur:** Code review story 8.14 (abandon/réactivation todo)

---

## Contexte & Problème

### Observation faite lors du code review 8.14

`useAbandonTodo` et `useReactivateTodo` accèdent directement au `ITodoRepository` :

```typescript
// useAbandonTodo.ts — Pattern actuel
mutationFn: async (todoId: string) => {
  const result = await todoRepository.update(todoId, {
    status: 'abandoned',   // ← règle métier encodée comme mutation de données brutes
    updatedAt: Date.now(),
  });
}
```

**Trois problèmes identifiés :**

1. **Violation ADR-031** : "abandonner" est une transition d'état métier qui devrait être une méthode sur l'entité `Todo` (`todo.abandon()`). ADR-031 prescrit ce pattern pour `Todo` en priorité 1 mais la story 8.14 a été implémentée sans l'appliquer.

2. **Le hook "invente" les données** : `{ status: 'abandoned', updatedAt: Date.now() }` est encodé en dur dans la couche UI. Si la règle change (ex: conditions sur abandon), il faut modifier tous les hooks concernés.

3. **Le hook reçoit un `todoId`** : à l'appel (`abandonTodo.mutate(todo.id)`), le composant possède déjà l'entité `Todo` complète. Passer uniquement l'ID et aller chercher l'entité en DB serait redondant. Mais l'entité en mémoire n'est pas accessible de façon unifiée.

### État actuel de la gestion d'état mobile

- **React Query** : cache serveur, source de vérité pour les données chargées
- **Zustand** : utilisé pour `settingsStore` et quelques stores ponctuels
- **Aucun store Zustand dédié aux entités de domaine** par contexte (action, knowledge, capture)

Les composants passent les entités via props, sans store centralisé. Cela fonctionne pour les cas simples mais devient fragile dès qu'une mutation dans un composant enfant (TodoDetailPopover) doit être reflétée dans le composant parent (ActionsTodoCard, liste).

---

## Proposition d'architecture

### Deux stores Zustand par screen-type (pas par entité)

Les stores sont **orientés affichage** (gabarit d'écran), pas orientés entité. Chaque "screen pattern" a ses propres besoins d'affichage.

#### TodosListStore — gabarit "liste de todos"

```typescript
interface TodosListStore {
  // Entités hydratées
  todos: Todo[];                          // entités ADR-031 (classes)

  // État d'affichage
  filter: FilterType;
  sort: SortType;
  counts: { all, active, completed, abandoned, deleted };
  isLoading: boolean;

  // Actions
  setFilter(filter: FilterType): void;
  setSort(sort: SortType): void;
  hydrate(): Promise<void>;              // charge depuis OP-SQLite via repository
  onMutation(todoId: string): void;      // callback : recharge la liste ou le todo spécifique
}
```

#### TodoDetailStore — gabarit "détail d'une todo"

```typescript
interface TodoDetailStore {
  // Référence par ID (point d'entrée unifié : liste, deep link, notification push)
  todoId: string | null;
  todo: Todo | null;                     // entité ADR-031 chargée depuis OP-SQLite

  // État d'affichage
  isLoading: boolean;
  isEditing: boolean;
  formValues: Partial<TodoSnapshot>;

  // Actions (appel via entité ADR-031 → repository.save())
  load(todoId: string): Promise<void>;
  abandon(): Promise<Result<void>>;
  reactivate(): Promise<Result<void>>;
  save(changes: Partial<TodoSnapshot>): Promise<Result<void>>;
}
```

### Pattern de mutation conforme ADR-031

```typescript
// TodoDetailStore.abandon() — conforme ADR-031
async abandon(): Promise<Result<void>> {
  if (!this.todo) return notFound('No todo loaded');

  // 1. Règle métier sur l'entité (ADR-031)
  const result = this.todo.abandon();    // Todo.abandon() valide la transition
  if (result.type !== 'SUCCESS') return result;

  // 2. Persist (repository.save conforme ADR-031)
  const saveResult = await todoRepository.save(this.todo);
  if (saveResult.type !== 'SUCCESS') return saveResult;

  // 3. Notifier le parent (callback simple, pas d'EventEmitter)
  this.onMutationCallback?.(this.todo.id);

  return success(undefined);
}
```

### Communication cross-stores : callback pattern

Le composant parent (screen liste) passe un callback au composant enfant (detail) :

```typescript
// Screen liste → passe callback
<TodoDetailPopover
  todoId={selectedTodoId}
  onMutation={(todoId) => todosListStore.onMutation(todoId)}
/>

// Store détail → trigger callback après mutation
this.onMutationCallback?.(this.todo.id);

// Store liste → recharge sélectivement
onMutation(todoId: string): void {
  // Option A : recharge uniquement ce todo dans la liste
  this.todos = this.todos.map(t =>
    t.id === todoId ? freshTodo : t
  );
  // Option B : invalide le cache React Query → re-hydrate tout
  queryClient.invalidateQueries({ queryKey: ['todos'] });
}
```

**Justification du callback vs EventEmitter :**
- Suffisant pour la complexité actuelle
- Explicite et traçable (pas de couplage caché)
- Si la complexité justifie un EventEmitter (ex: mutations cross-contexte, broadcast multi-écrans), on migre à ce moment

---

## Décisions architecturales (Winston — 2026-03-04)

### D1 — Périmètre : singleton Zustand (`create`) + `hydrate()` au focus

**Décision : les stores sont des singletons module-level créés avec `create`. L'état est rafraîchi via `hydrate()` appelé à chaque `useFocusEffect`.**

```typescript
// useTodosListStore.ts — singleton, cohérent avec settingsStore / capturesStore
export const useTodosListStore = create<TodosListState>((set) => ({
  todos: [],
  filter: 'all',
  sort: 'createdAt',
  counts: { all: 0, active: 0, completed: 0, abandoned: 0 },
  isLoading: false,
  hydrate: async () => {
    set({ isLoading: true })
    const repo = container.resolve<ITodoRepository>(TOKENS.ITodoRepository)
    const result = await repo.findAll()
    if (result.type === 'SUCCESS') set({ todos: result.value, isLoading: false })
  },
  onMutation: async (todoId: string) => {
    const repo = container.resolve<ITodoRepository>(TOKENS.ITodoRepository)
    const result = await repo.findById(todoId)
    if (result.type === 'SUCCESS') {
      set((s) => ({ todos: s.todos.map((t) => (t.id === todoId ? result.value : t)) }))
    }
  },
}))

// Dans le screen — hydratation systématique au focus
useFocusEffect(useCallback(() => {
  useTodosListStore.getState().hydrate()
}, []))
```

**Justification :** Le pattern `createStore` + Context a été évalué puis écarté après test empirique et analyse des bénéfices réels :
- L'app mobile est monoécran : jamais deux instances de `TodosListScreen` simultanées
- SQLite local est synchrone et rapide (< 5 ms) : la fenêtre de state stale entre mount et `hydrate()` est imperceptible
- Le pattern singleton est déjà utilisé partout dans le projet (`settingsStore`, `capturesStore`, `SyncStatusStore`) — cohérence > pureté théorique
- `createStore` + Context aurait ajouté du boilerplate (factory, Provider, Context, hook wrapper) sans bénéfice mesurable

Le `useFocusEffect` + `hydrate()` est le pattern standard React Navigation pour garantir des données fraîches à chaque visite d'un écran.

---

### D2 — Hydratation : store → repository direct (source de vérité unique)

**Décision : le store hydrate directement via `ITodoRepository`. React Query n'est plus la source de vérité pour les entités locales.**

Le store exprime des **intentions métier** sans savoir d'où viennent les données. Le repository implémente la stratégie (local OP-SQLite, remote API, hybride) — ce choix est opaque pour le store.

```
Composant → Store (intention) → ITodoRepository (stratégie de données)
                                        ↓
                              OP-SQLite / API / Hybride
```

**Conséquence sur React Query :** React Query reste pertinent pour les appels réseau *depuis les repositories* (sync backend, résultats AI). Mais il ne doit plus être visible en dehors de la couche repository — plus de `useQuery(['todos'])` dans les hooks UI ou les stores. Les `useQuery(['todos'])` existants sont à supprimer lors de la migration.

---

### D3 — Zustand n'est pas une "vue projetée" de React Query

**Décision : Zustand et React Query restent des couches séparées sans couplage direct.**

Le pattern "Zustand subscriber sur React Query cache" implique un couplage de `queryClient` dans le store et une triple indirection pour chaque mutation locale. Ce n'est pas acceptable. Le store Zustand appelle le repository directement — React Query n'a pas sa place dans la couche store pour des entités locales.

---

### D4 — Extension aux autres bounded contexts

**Décision : prescrire le principe pour tous les BCs avec entités navigables. Le gabarit List+Detail s'applique là où ces deux screens existent.**

| Contexte | Pattern | Raison |
|----------|---------|--------|
| **Action/Todo** | `TodosListStore` + `TodoDetailStore` | List + detail interactif, transitions d'état |
| **Knowledge/Thought** | `ThoughtsListStore` + `ThoughtDetailStore` | Même gabarit |
| **Capture** | `CapturesListStore` uniquement | Detail lecture seule — pas de store détail nécessaire dans l'immédiat |
| **Identity** | `settingsStore` existant suffit | Pas de pattern list/detail |

---

### D5 — Timing : story 8.14 reste done, story technique dédiée créée

**Décision : story 8.14 → `done` (dette H1 documentée). Story 8.23 créée pour ADR-031 Todo + ADR-038 stores.**

**Ordre de mise en œuvre (story 8.23) :**
1. Migrer `Todo` en classe riche (ADR-031, Priorité 1) — prérequis à tout le reste
2. Implémenter `TodosListStore` + `TodoDetailStore` (cette ADR)
3. Refactorer `useAbandonTodo`, `useReactivateTodo` vers le store
4. Supprimer les `useQuery(['todos'])` devenus redondants

---

### Note sur l'injection de dépendance dans les stores

Les stores résolvent leurs repositories via **lazy resolution** depuis le container TSyringe (pattern identique aux hooks existants). Le store ne reçoit pas le repository via prop ou constructeur.

```typescript
// Dans une action du store
hydrate: async () => {
  const repo = container.resolve<ITodoRepository>(ITodoRepository)
  const result = await repo.findAll()
  // ...
}
```

---

## Conformité ADR-031 (prérequis)

Cette ADR suppose que `Todo` est ou sera migré en classe riche (ADR-031, priorité 1). Si `Todo` reste une interface, le `todo.abandon()` n'existe pas et le store devrait appeler `todoRepository.update(id, { status: 'abandoned' })` — ce qui ne résout pas H1.

**Ordre de mise en œuvre suggéré :**
1. Migrer `Todo` en classe riche (story ADR-031 pilote)
2. Implémenter `TodosListStore` + `TodoDetailStore`
3. Refactorer les hooks `useAbandonTodo`, `useReactivateTodo`, etc.

---

## Dette actuelle documentée

La story 8.14 reste en `review` avec la dette suivante documentée :

- **H1** : `useAbandonTodo` et `useReactivateTodo` violent ADR-031 — transition d'état métier dans la couche UI, pas dans l'entité `Todo`
- **Bloquant** : nécessite `Todo` classe (ADR-031) + store pattern (cette ADR)
- **Non bloquant pour livraison** : les AC1-AC6 sont implémentés et fonctionnels

---

## Decision Log

**2026-03-04** — Discussion yohikofox + Claude (code review 8.14)

> Code review identifie que `useAbandonTodo.mutate(todoId)` expose une abstraction infrastructure
> (React Query `mutate`) plutôt qu'une intention métier. La règle "abandonner" est encodée
> comme mutation de données brutes dans la couche UI.
>
> Discussion converge vers :
> - Entité `Todo` avec `abandon()` / `reactivate()` (ADR-031 déjà prescrit, non encore appliqué)
> - Deux stores Zustand orientés affichage : `TodosListStore` (liste) + `TodoDetailStore` (détail via ID)
> - `TodoDetailStore` charge par `todoId` — point d'entrée unifié quel que soit le contexte appelant
> - Callback pattern pour communication cross-stores (pas d'EventEmitter à ce stade — YAGNI)
> - `repository.save(todo)` conforme ADR-031 Règle 8

**2026-03-04** — Décisions Winston (Architect) — v1

> Q1 : Zustand Context Store (`createStore` + Context), lifecycle scopé au composant.
> Q2 : store → repository direct, React Query hors couche UI/stores.
> Q3 : pas de couplage queryClient dans le store.
> Q4 : pattern List+Detail par défaut pour tous les BCs avec entités navigables.
> Q5 : story 8.14 → done. Story 8.23 créée.

**2026-03-04** — Révision D1 après test empirique et analyse — v2 (décision finale)

> Test empirique (`ZustandPersistenceTestScreen`) confirme que les singletons Zustand (`create`)
> persistent le state entre mounts. Le pattern Context Store (`createStore` + Context) a été
> évalué : lifecycle propre, mais boilerplate injustifié pour ce cas d'usage.
>
> Raisons du retour au singleton :
> - App monoécran : jamais deux instances simultanées de TodosListScreen
> - SQLite local rapide : fenêtre de stale state imperceptible avec useFocusEffect + hydrate()
> - Cohérence avec settingsStore, capturesStore, SyncStatusStore (tous singletons)
> - createStore + Context : même bénéfice qu'un useContext + useState simple, sauf pour
>   les re-renders sélectifs — optimisation prématurée à ce stade
>
> D1 révisé : singleton `create` + useFocusEffect + hydrate().

**Statut final : ✅ Accepted**

---

## References

- ADR-031 — Rich Domain Entities (Todo priorité 1 de migration)
- ADR-023 — Result Pattern (utilisé par `todo.abandon()`)
- ADR-021 — DI Lifecycle (résolution du repository dans le store)
- Story 8.14 — Déclencheur de cette discussion

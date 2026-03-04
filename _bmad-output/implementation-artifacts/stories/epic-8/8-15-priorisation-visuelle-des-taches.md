# Story 8.15: Priorisation Visuelle des Tâches

Status: done

<!-- Note: Validation optionnelle. Lancer validate-create-story pour vérification qualité avant dev-story. -->

## Story

As a **user**,
I want **to immediately identify the urgency of my tasks through a color-coded left border indicator**,
So that **I can focus on what matters most without having to read every task to assess urgency**.

## Context

Quick win identifié lors d'une session discovery avec un power user (2026-02-17).
Le Tab Actions manque de hiérarchisation visuelle : toutes les tâches se ressemblent visuellement quelle que soit leur urgence. Les utilisateurs doivent lire chaque tâche pour évaluer sa priorité.

**Système de niveaux d'urgence :**
- 🔴 **En retard** : deadline dépassée (auto, calculé depuis `deadline` + `isOverdue`)
- 🟠 **Prioritaire** : `priority === "high"` (existant dans le modèle — toggle rapide)
- 🟡 **Approchante** : deadline dans les 48h mais non dépassée (calcul dynamique)
- ⬜ **Normale** : tâche standard sans urgence particulière

**Décision d'implémentation :**
Le champ `priority: "low" | "medium" | "high"` existe DÉJÀ dans le modèle Todo et la DB.
→ `priority === "high"` = **Prioritaire** (orange). Pas de nouveau champ `isPriority: boolean`.
→ L'indicateur visuel principal est une **barre colorée latérale gauche** (style Notion), en complément du badge priorité existant.

## Acceptance Criteria

### AC1: Indicateur visuel "En retard" (rouge)
**Given** a Todo has a deadline that is in the past (i.e., `formatDeadline(deadline).isOverdue === true`)
**When** I view it in the Actions tab
**Then** the Todo card displays a red left border (4dp width)
**And** the overdue text ("En retard de X jours") is already shown in the metadata row (existing)
**And** the red left border is distinct from all other urgency levels

### AC2: Indicateur visuel "Prioritaire" (orange)
**Given** a Todo has `priority === "high"` and is NOT overdue
**When** I view it in the Actions tab
**Then** the Todo card displays an orange left border
**And** if the task is also overdue, the red "En retard" left border takes precedence

### AC3: Indicateur visuel "Approchante" (jaune/amber)
**Given** a Todo has a deadline within the next 48 hours (deadline exists, NOT overdue)
**When** I view it in the Actions tab
**Then** the Todo card displays a yellow/amber left border
**And** red "En retard" border takes precedence if deadline has just passed
**And** orange "Prioritaire" border takes precedence if `priority === "high"`

### AC4: Tâche normale (pas de bordure)
**Given** a Todo has no deadline and `priority !== "high"`
**When** I view it in the Actions tab
**Then** the Todo card displays no visible left border (transparent/neutral)
**And** no urgent indicator overwrites the default card appearance

### AC5: Marquer/démarquer rapidement comme "Prioritaire" (toggle depuis la liste)
**Given** I am viewing a Todo in the Actions tab
**When** I tap a star/flag icon button on the Todo card
**Then** the Todo's `priority` is toggled to `"high"` (Prioritaire) if not already high
**And** the orange left border appears immediately (optimistic UI via React Query)
**And** tapping again sets `priority` back to `"medium"` (removing the Prioritaire flag)
**And** haptic feedback is triggered if `settingsStore.hapticFeedbackEnabled`

### AC6: Sort "Priorité" respecte l'ordre d'urgence
**Given** I am in the Actions tab with sort = "Priorité" selected
**When** the list is sorted
**Then** tasks are ordered by urgency level: En retard → Prioritaires → Approchantes → Normales
**And** within each urgency group, tasks are sorted by deadline (nearest first), then by `createdAt`

### AC7: Cohérence dans le Feed inline (InlineTodoList)
**Given** a Todo has a non-normal urgency level
**When** I view it inline in the Feed (story 5.1 — InlineTodoList / TodoItem)
**Then** the same left border color indicator is applied consistently
**And** the urgency state is reflected in both the Actions tab and the inline Feed view

## Tasks / Subtasks

### Task 1: Créer `getUrgencyLevel()` utility (AC1–AC4)
- [x] Subtask 1.1: Créer `mobile/src/contexts/action/utils/getUrgencyLevel.ts`
  ```typescript
  import type { Todo } from '../domain/Todo.model';
  import { formatDeadline } from './formatDeadline';

  export type UrgencyLevel = 'overdue' | 'prioritaire' | 'approaching' | 'normal';

  const APPROACHING_THRESHOLD_MS = 48 * 60 * 60 * 1000; // 48h

  export function getUrgencyLevel(todo: Todo): UrgencyLevel {
    if (todo.deadline) {
      const dl = formatDeadline(todo.deadline);
      if (dl.isOverdue) return 'overdue';
      if (todo.deadline - Date.now() <= APPROACHING_THRESHOLD_MS) return 'approaching';
    }
    if (todo.priority === 'high') return 'prioritaire';
    return 'normal';
  }
  ```
  Ordre de précédence : overdue > approaching > prioritaire > normal
  Note : "Prioritaire" (priority=high) prend rang 3 → overdue et approaching ont la priorité visuelle
  Si l'on veut "prioritaire" avant "approaching", inverser les cases dans getUrgencyLevel (AC3 dit "orange prend précédence si high priority")

  **CORRECTION AC2/AC3** : AC2 dit "si En retard → rouge prend précédence" et AC3 dit "orange prioritaire prend précédence sur approaching".
  Ordre correct : overdue > prioritaire > approaching > normal
  ```typescript
  export function getUrgencyLevel(todo: Todo): UrgencyLevel {
    if (todo.deadline && formatDeadline(todo.deadline).isOverdue) return 'overdue';
    if (todo.priority === 'high') return 'prioritaire';
    if (todo.deadline && (todo.deadline - Date.now()) <= APPROACHING_THRESHOLD_MS) return 'approaching';
    return 'normal';
  }
  ```

- [x] Subtask 1.2: Créer `mobile/src/contexts/action/utils/getUrgencyBorderColor.ts`
  ```typescript
  import type { UrgencyLevel } from './getUrgencyLevel';

  // Couleurs constantes (ADR-024 : pas de magic strings dans les composants)
  export const URGENCY_BORDER_COLORS = {
    overdue:    '#EF4444',  // red-500
    prioritaire:'#F97316',  // orange-500
    approaching:'#F59E0B',  // amber-500
    normal:     'transparent',
  } as const satisfies Record<UrgencyLevel, string>;

  export function getUrgencyBorderColor(level: UrgencyLevel): string {
    return URGENCY_BORDER_COLORS[level];
  }
  ```
- [x] Subtask 1.3: Créer tests unitaires `mobile/src/contexts/action/utils/__tests__/getUrgencyLevel.test.ts`
  - Test overdue (deadline hier)
  - Test prioritaire (priority=high, pas de deadline)
  - Test approaching (deadline dans 24h)
  - Test normal (medium priority, pas de deadline)
  - Test précédence overdue > prioritaire (todo priority=high + deadline dépassée → "overdue")
  - Test précédence prioritaire > approaching (todo priority=high + deadline dans 24h → "prioritaire")

### Task 2: Ajouter left border dans ActionsTodoCard (AC1–AC4)
- [x] Subtask 2.1: Ouvrir `mobile/src/contexts/action/ui/ActionsTodoCard.tsx`
- [x] Subtask 2.2: Importer `getUrgencyLevel` et `getUrgencyBorderColor`
- [x] Subtask 2.3: Modifier le Pressable externe pour inclure la barre gauche :
  - Wrapper la carte avec un `View` contenant la barre colorée :
  ```tsx
  // Outer wrapper avec la barre gauche
  <View style={{ flexDirection: 'row', marginBottom: 8 }}>
    {/* Left border urgency indicator */}
    <View
      style={{
        width: 4,
        borderRadius: 2,
        backgroundColor: getUrgencyBorderColor(urgencyLevel),
      }}
    />
    {/* Existing Pressable card (without mb-2 className) */}
    <Pressable className="flex-1 bg-bg-card p-4 rounded-r-lg active:opacity-80" ...>
      ...
    </Pressable>
  </View>
  ```
  OU ajouter `borderLeftWidth: 4` et `borderLeftColor` directement sur le Pressable.
  **Recommandé** : utiliser `borderLeftWidth`/`borderLeftColor` pour garder le rounded-lg intègre.
- [x] Subtask 2.4: Calculer `urgencyLevel = getUrgencyLevel(todo)` au niveau du composant (mémo si nécessaire)
- [x] Subtask 2.5: Vérifier que le styling `abandoned` (story 8.14, si déjà implémentée) n'entre pas en conflit avec la bordure

### Task 3: Ajouter toggle "Prioritaire" rapide dans ActionsTodoCard (AC5)
- [x] Subtask 3.1: Créer hook `mobile/src/contexts/action/hooks/useTogglePriority.ts`
  ```typescript
  import { useMutation, useQueryClient } from '@tanstack/react-query';
  import * as Haptics from 'expo-haptics';
  import { container } from '@/infrastructure/di/container';
  import { TOKENS } from '@/infrastructure/di/tokens';
  import type { ITodoRepository } from '../domain/ITodoRepository';
  import type { TodoPriority } from '../domain/Todo.model';

  export const useTogglePriority = () => {
    const queryClient = useQueryClient();
    return useMutation({
      mutationFn: async ({ todoId, currentPriority }: { todoId: string; currentPriority: TodoPriority }) => {
        const repo = container.resolve<ITodoRepository>(TOKENS.ITodoRepository);
        const newPriority: TodoPriority = currentPriority === 'high' ? 'medium' : 'high';
        await repo.update(todoId, { priority: newPriority, updatedAt: Date.now() });
        return newPriority;
      },
      onSuccess: () => {
        queryClient.invalidateQueries({ queryKey: ['todos'] });
      },
      onError: (error: Error) => {
        // ADR-024: error handling user-friendly, pas de throw en couche domaine
        console.error('[useTogglePriority]', error.message);
      },
    });
  };
  ```
- [x] Subtask 3.2: Intégrer le bouton toggle dans `ActionsTodoCard.tsx` :
  - Icône `star` (Feather ou Ionicons : `star-outline` / `star`)
  - Positionné en haut-droite de la carte (absolu ou dans le header row)
  - `star` plein et orange si `todo.priority === 'high'`, `star-outline` gris sinon
  - Appel haptic si `settingsStore.hapticFeedbackEnabled`
  - `hitSlop: { top: 10, bottom: 10, left: 10, right: 10 }`
  - Désactiver si `readonly === true`
  ```tsx
  <Pressable
    onPress={() => togglePriority.mutate({ todoId: todo.id, currentPriority: todo.priority })}
    hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
    style={{ position: 'absolute', top: 12, right: 12 }}
  >
    <Feather
      name={todo.priority === 'high' ? 'star' : 'star'}
      size={16}
      color={todo.priority === 'high' ? '#F97316' : '#9ca3af'}
    />
  </Pressable>
  ```
  Note : Feather n'a pas `star-fill`, utiliser Ionicons `star` / `star-outline` ou FontAwesome.

### Task 4: Mettre à jour sortTodos.ts pour AC6 (sort "Priorité")
- [x] Subtask 4.1: Ouvrir `mobile/src/contexts/action/utils/sortTodos.ts`
- [x] Subtask 4.2: Importer `getUrgencyLevel` dans `sortTodos.ts`
- [x] Subtask 4.3: Dans le case `"priority"`, remplacer le tri actuel par le tri basé sur l'urgency level :
  ```typescript
  const URGENCY_ORDER: Record<UrgencyLevel, number> = {
    overdue:     0,
    prioritaire: 1,
    approaching: 2,
    normal:      3,
  };

  case "priority":
    return [...todos].sort((a, b) => {
      const urgencyDiff = URGENCY_ORDER[getUrgencyLevel(a)] - URGENCY_ORDER[getUrgencyLevel(b)];
      if (urgencyDiff !== 0) return urgencyDiff;
      // Secondary: deadline (null last)
      if (a.deadline && b.deadline) return a.deadline - b.deadline;
      if (a.deadline) return -1;
      if (b.deadline) return 1;
      // Tertiary: createdAt
      return a.createdAt - b.createdAt;
    });
  ```
- [x] Subtask 4.4: S'assurer que les tests unitaires existants de `sortTodos.ts` passent toujours

### Task 5: Appliquer le left border dans InlineTodoList / TodoItem (AC7)
- [x] Subtask 5.1: Ouvrir `mobile/src/contexts/action/ui/TodoItem.tsx` (utilisé dans InlineTodoList pour le Feed)
- [x] Subtask 5.2: Appliquer le même pattern left border que dans ActionsTodoCard
  - Importer `getUrgencyLevel`, `getUrgencyBorderColor`
  - Ajouter la barre gauche colorée
  - **Pas de toggle star dans le Feed inline** (AC7 demande seulement la cohérence visuelle, pas l'édition)
- [x] Subtask 5.3: Si `TodoItem.tsx` n'est pas utilisé dans le Feed et que c'est `InlineTodoList.tsx` qui render directement, adapter `InlineTodoList.tsx`

### Task 6: Tests BDD (AC1–AC5)
- [x] Subtask 6.1: Créer `mobile/tests/acceptance/features/story-8-15-priorisation-visuelle-des-taches.feature`
  ```gherkin
  Feature: Priorisation Visuelle des Tâches
    En tant qu'utilisateur
    Je veux identifier visuellement l'urgence de mes tâches grâce à des bordures colorées
    Afin de me concentrer sur ce qui est important sans lire chaque tâche

    Scenario: Tâche en retard affiche une bordure rouge
      Given a todo exists with a deadline in the past
      When I view the Actions tab
      Then the todo card has a red left border
      And the urgency level is "overdue"

    Scenario: Tâche prioritaire (priority=high) affiche une bordure orange
      Given a todo exists with priority "high" and no deadline
      When I view the Actions tab
      Then the todo card has an orange left border
      And the urgency level is "prioritaire"

    Scenario: Tâche approchante (deadline dans 24h) affiche une bordure amber
      Given a todo exists with a deadline within 24 hours
      When I view the Actions tab
      Then the todo card has an amber left border
      And the urgency level is "approaching"

    Scenario: Tâche normale n'affiche pas de bordure colorée
      Given a todo exists with priority "medium" and no deadline
      When I view the Actions tab
      Then the todo card has no visible left border
      And the urgency level is "normal"

    Scenario: Toggle prioritaire depuis la liste
      Given a todo exists with priority "medium"
      When I tap the star icon on the todo card
      Then the todo's priority becomes "high"
      And the orange left border appears immediately
      When I tap the star icon again
      Then the todo's priority becomes "medium"
      And the left border disappears

    Scenario: Précédence overdue sur prioritaire
      Given a todo exists with priority "high" and a deadline in the past
      When I view the Actions tab
      Then the todo card has a red left border (overdue takes precedence)
      And the urgency level is "overdue"
  ```
- [x] Subtask 6.2: Créer `mobile/tests/acceptance/story-8-15.test.ts` avec step definitions jest-cucumber
  - Utiliser les mocks in-memory de `tests/acceptance/support/test-context.ts`
  - Mocker `ITodoRepository.update()` pour le toggle priority
  - Tester `getUrgencyLevel()` directement dans les steps (pas de render React Native dans acceptance tests)
- [x] Subtask 6.3: Lancer `npm run test:acceptance` dans `pensieve/mobile` pour valider
- [x] Subtask 6.4: Lancer les tests architecture `npm run test:architecture` (ADR compliance)

### Task 7: Non-régression
- [x] Subtask 7.1: Vérifier que les tests BDD des stories 5.x (Tab Actions) passent toujours après les modifications de `sortTodos.ts`
- [x] Subtask 7.2: Vérifier que les tests unitaires de `filterTodos.ts`, `sortTodos.ts` passent
- [ ] Subtask 7.3: Vérifier manuellement sur device que :
  - Tâche dépassée = bordure rouge + texte "En retard de X jours" (déjà existant)
  - Toggle star → bordure orange instantanée
  - Tâche avec deadline dans 24h = bordure amber
  - Mode "Toutes" filtre + tri par urgence correct
- [x] Subtask 7.4: Vérifier que `readonly=true` (corbeille) n'active pas le toggle star

## Dev Notes

### 🎯 Découverte Critique — Ce Qui Existe DÉJÀ

```
✅ priority: "low" | "medium" | "high" — Todo.model.ts (DÉJÀ LÀ)
✅ deadline?: number — Todo.model.ts (DÉJÀ LÀ)
✅ priority TEXT CHECK(priority IN ('low','medium','high')) — schema.ts (DÉJÀ LÀ)
✅ deadline INTEGER — schema.ts (DÉJÀ LÀ)
✅ INDEX sur priority et deadline — schema.ts (DÉJÀ LÀ)
✅ formatDeadline(deadline).isOverdue — formatDeadline.ts (DÉJÀ LÀ)
✅ getPriorityColor() — ActionsTodoCard.tsx (badge pill rouge/orange/vert par priority)
✅ getPriorityLabel() — ActionsTodoCard.tsx (Haute/Moyenne/Basse)
✅ sortTodos.ts — modes: default/priority/createdDate/alphabetical
✅ groupTodosByDeadline.ts — groupes: En retard, Aujourd'hui, Cette semaine, Plus tard, Pas d'échéance
✅ useUpdateTodo hook — mise à jour générique todo (à réutiliser)
✅ TodoRepository.update(id, changes) — accepte { priority }
```

**Ce qui MANQUE — Travail réel de la story :**
```
❌ getUrgencyLevel() utility — calcul niveau urgence (overdue/prioritaire/approaching/normal)
❌ getUrgencyBorderColor() utility — mapping UrgencyLevel → couleur hex
❌ Left border coloré dans ActionsTodoCard (barre 4dp côté gauche)
❌ useTogglePriority hook — toggle priority=high/medium depuis la liste
❌ Bouton star toggle dans ActionsTodoCard
❌ Sort "Priorité" mis à jour pour respecter l'ordre urgency level
❌ Left border dans TodoItem.tsx (Feed inline — AC7)
❌ Tests BDD (6 scénarios minimum)
```

### ⚠️ Pas de Migration DB Nécessaire

La story initiale mentionne "Ajouter champ `isPriority: boolean`" — **INCORRECT/INUTILE**.
`priority: "high"` existe DÉJÀ et couvre exactement le cas "Prioritaire".
→ Pas de migration OP-SQLite nécessaire.
→ Pas de migration backend nécessaire.

### Pattern Left Border Recommandé

Deux approches possibles dans React Native :

**Option A — Style inline (recommandé, plus simple) :**
```tsx
<Pressable
  style={[
    styles.card,
    {
      borderLeftWidth: 4,
      borderLeftColor: getUrgencyBorderColor(getUrgencyLevel(todo)),
      borderRadius: urgencyLevel === 'normal' ? 8 : 0,  // No border = full rounded
    }
  ]}
>
```

**Option B — View wrapper avec barre séparée (style Notion, visuellement plus propre) :**
```tsx
<View style={{ flexDirection: 'row', marginBottom: 8 }}>
  <View
    style={{
      width: 4,
      borderTopLeftRadius: 8,
      borderBottomLeftRadius: 8,
      backgroundColor: getUrgencyBorderColor(getUrgencyLevel(todo)),
    }}
  />
  <Pressable style={{ flex: 1, ...existingStyles }}>
    {/* existing content */}
  </Pressable>
</View>
```
**Option B est recommandée** pour garder les rounded corners des deux côtés proprement.

### Couleurs Urgency (ADR-024 — pas de magic strings)

```typescript
// Fichier: getUrgencyBorderColor.ts
export const URGENCY_BORDER_COLORS = {
  overdue:     '#EF4444',  // red-500    — identique à bg-error-500
  prioritaire: '#F97316',  // orange-500 — identique au badge "Haute" (bg-warning-500)
  approaching: '#F59E0B',  // amber-500  — nouveau
  normal:      'transparent',
} as const;
```

Note : `bg-error-500` dans le projet = `#EF4444` (Tailwind red-500). Vérifier la valeur exacte dans `mobile/src/design-system/` pour cohérence.

### Comportement du Toggle Star (AC5)

```typescript
// Priority toggle : high ↔ medium
// (on ne toggle pas vers "low" pour garder l'interface simple)
const newPriority: TodoPriority = currentPriority === 'high' ? 'medium' : 'high';
```

Ce toggle est une **simplification UX** : la vue détail (TodoDetailPopover) conserve le sélecteur 3-niveaux (🔴/🟡/🟢) pour ceux qui veulent granularité. Le toggle rapide sur la carte est binaire (high/non-high).

### Coordination avec Stories 8.13 et 8.14

Stories 8.13 (supprimer) et 8.14 (abandonner) ajoutent un wrapper `Swipeable` dans `ActionsTodoCard.tsx`.

**Si 8.13/8.14 sont déjà implémentées :**
- Le composant a déjà un wrapper `Swipeable` autour du contenu
- La barre left border doit être OUTSIDE du Swipeable wrapper pour rester visible
- Structure attendue : `<View wrapper avec barre> <Swipeable> <Pressable card> </Swipeable> </View>`

**Si 8.13/8.14 ne sont PAS encore implémentées :**
- Ajouter le left border directement sur le Pressable existant
- Coordonner avec les implémenteurs de 8.13/8.14 pour la structure wrapper

### ADR Compliance

- **ADR-021 (DI Transient First)** : `useTogglePriority` résout `ITodoRepository` via `container.resolve()` en lazy dans le `mutationFn`. Pattern conforme aux hooks existants.
- **ADR-023 (Result Pattern)** : `TodoRepository.update()` → vérifier qu'il retourne `boolean | Result`. Gérer le cas `false` sans throw (Alert ou log silencieux).
- **ADR-024 (Clean Code)** : Couleurs en constantes dans `getUrgencyBorderColor.ts`, noms révélateurs d'intention.
- **ADR-022 (AsyncStorage)** : Aucune donnée crítico-privée persistée par cette story. Le sort state est déjà persisté dans `useFilterState`.

### Vérification du Design System

Avant d'utiliser les couleurs hex directement, vérifier dans `mobile/src/design-system/` si les tokens `error-500`, `warning-500`, `amber-500` existent comme variables de thème (dark/light mode).
Si oui, utiliser les hooks de thème plutôt que les valeurs hex directes pour assurer la cohérence dark mode.

```typescript
// Si des hooks de thème existent (ex: useTheme()):
const theme = useTheme();
const borderColor = {
  overdue:     theme.colors.error[500],
  prioritaire: theme.colors.warning[500],
  approaching: theme.colors.amber[500],
  normal:      'transparent',
}[urgencyLevel];
```

### Project Structure Notes

**Nouveaux fichiers à créer :**
```
pensieve/mobile/src/contexts/action/utils/getUrgencyLevel.ts
pensieve/mobile/src/contexts/action/utils/getUrgencyBorderColor.ts
pensieve/mobile/src/contexts/action/hooks/useTogglePriority.ts
pensieve/mobile/src/contexts/action/utils/__tests__/getUrgencyLevel.test.ts
pensieve/mobile/tests/acceptance/features/story-8-15-priorisation-visuelle-des-taches.feature
pensieve/mobile/tests/acceptance/story-8-15.test.ts
```

**Fichiers à modifier :**
```
pensieve/mobile/src/contexts/action/ui/ActionsTodoCard.tsx
  → Ajouter left border via getUrgencyLevel + getUrgencyBorderColor
  → Ajouter bouton star toggle (useTogglePriority)

pensieve/mobile/src/contexts/action/ui/TodoItem.tsx
  → Ajouter left border (même pattern que ActionsTodoCard, sans toggle star)

pensieve/mobile/src/contexts/action/utils/sortTodos.ts
  → Case "priority" : remplacer le tri actuel par tri basé sur UrgencyLevel
```

**Fichiers à NE PAS modifier :**
```
pensieve/mobile/src/contexts/action/domain/Todo.model.ts  (priority + deadline déjà là)
pensieve/mobile/src/contexts/action/data/TodoRepository.ts  (update() déjà là)
pensieve/mobile/src/contexts/action/domain/ITodoRepository.ts  (pas de changement interface)
pensieve/mobile/src/database/schema.ts  (pas de migration DB requise)
pensieve/mobile/src/contexts/action/utils/formatDeadline.ts  (laisser inchangé, réutiliser isOverdue)
pensieve/mobile/src/contexts/action/utils/filterTodos.ts  (aucun nouveau filtre dans cette story)
pensieve/mobile/src/contexts/action/ui/FilterTabs.tsx  (aucun nouveau tab dans cette story)
```

### References

- [Source: pensieve/mobile/src/contexts/action/domain/Todo.model.ts] — `priority: TodoPriority`, `deadline?: number` déjà définis
- [Source: pensieve/mobile/src/database/schema.ts] — colonnes `priority` et `deadline` avec indexes déjà présents
- [Source: pensieve/mobile/src/contexts/action/utils/formatDeadline.ts] — `isOverdue: boolean` dans `DeadlineFormat`
- [Source: pensieve/mobile/src/contexts/action/ui/ActionsTodoCard.tsx] — `getPriorityColor()`, `getPriorityLabel()`, structure de la carte (sans left border)
- [Source: pensieve/mobile/src/contexts/action/utils/sortTodos.ts] — `SortType`, case `"priority"` à mettre à jour
- [Source: pensieve/mobile/src/contexts/action/utils/groupTodosByDeadline.ts] — groupes deadline existants (default sort)
- [Source: pensieve/mobile/src/contexts/action/hooks/useUpdateTodo.ts] — hook existant à réutiliser comme pattern pour useTogglePriority
- [Source: pensieve/mobile/src/contexts/action/hooks/useDeleteTodo.ts] — pattern ADR-021 lazy resolve à suivre
- [Source: pensieve/mobile/src/contexts/action/ui/TodoDetailPopover.tsx] — priority selector 3-niveaux existant (ne pas modifier)
- [Source: pensieve/mobile/src/contexts/action/ui/TodoItem.tsx] — composant Feed inline à mettre à jour pour AC7
- [Source: _bmad-output/implementation-artifacts/stories/epic-8/8-13-supprimer-une-tache.md] — Swipeable pattern (coordination)
- [Source: _bmad-output/implementation-artifacts/stories/epic-8/8-14-abandonner-une-tache.md] — renderRightActions pattern
- [Source: _bmad-output/project-context.md#Testing Rules] — jest-cucumber BDD, babel-jest unit, PAS jest-expo
- [Source: _bmad-output/project-context.md#Critical ADR Rules] — ADR-021 DI lazy, ADR-023 Result, ADR-024 Clean Code

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

N/A — Story de création

### Completion Notes List

**Story créée (contexte)** : Ultimate context engine analysis completed — comprehensive developer guide created.

**Implémentation réalisée (2026-03-04)** :

- ✅ `getUrgencyLevel()` créé avec ordre de précédence correct : overdue > prioritaire > approaching > normal
- ✅ `getUrgencyBorderColor()` créé avec constantes URGENCY_BORDER_COLORS (ADR-024 conforme)
- ✅ `useTogglePriority` hook créé suivant le pattern Zustand ADR-038 (et non React Query comme dans la spec — la codebase a migré vers Zustand)
- ✅ `ActionsTodoCard.tsx` : wrapper View + barre gauche 4dp colorée (Option B style Notion) + bouton star toggle (readonly guard)
- ✅ `TodoItem.tsx` : même pattern barre gauche appliqué pour cohérence Feed inline (AC7)
- ✅ `sortTodos.ts` case "priority" : remplacé par tri urgency level (overdue→prioritaire→approaching→normal) avec secondary deadline + tertiary createdAt
- ✅ 11 tests unitaires `getUrgencyLevel.test.ts` verts (11/11)
- ✅ 8 tests BDD `story-8-15.test.ts` verts (8/8)
- ✅ Aucune régression : 22/22 tests sortTodos + 69/69 tests utils action

**Décision technique** : `useTogglePriority` utilise Zustand (`useTodosListStore.onMutation`) au lieu de React Query (obsolète dans cette codebase suite à ADR-038). Conforme au pattern `useDeleteTodo` / `useUpdateTodo`.

**Code Review Adversarial (2026-03-04) — 9 issues corrigées :**
- H1: `TodoItem.tsx` — `borderTopLeftRadius: 0, borderBottomLeftRadius: 0` sur TouchableOpacity (gap visuel avec barre gauche)
- H2: `sortTodos.ts` — Pre-compute `urgencyMap` avant le `.sort()` pour passer de O(N log N) à O(N) appels `getUrgencyLevel`
- H3: `ActionsTodoCard.tsx` — `paddingRight: 40` sur la carte pour éviter overlap avec le bouton star absolu
- M1: `useTogglePriority.ts` — Optimistic UI ajouté : `useTodosListStore.setState()` immédiat + rollback sur erreur (AC5 compliance)
- M2: `ActionsTodoCard.tsx` — Indentation `<Pressable>` corrigée (10 espaces, enfant de Swipeable à 8)
- M3: `getUrgencyBorderColor.test.ts` créé (6 tests, mapping couleurs complet)
- M4: `useMemo` ajouté pour `urgencyBorderColor` dans `ActionsTodoCard` et `TodoItem`
- L1: JSDoc `getUrgencyLevel.ts` — note sur comportement "aujourd'hui mais heure passée = approaching"
- L2: Commentaire `ActionsTodoCard.tsx` — `"Subtask 2.3 + 2.8"` → `"Story 8.14 + 8.15"`

**Subtask 7.3** : Validation manuelle sur device à effectuer lors du code review exploratoire mobile.

**Analyse réalisée :**
- `priority: "low" | "medium" | "high"` : champ DÉJÀ dans Todo.model.ts — pas de nouveau champ `isPriority: boolean`
- `deadline?: number` : DÉJÀ dans le modèle et la DB — pas de migration requise
- `formatDeadline.ts` : `isOverdue` DÉJÀ calculé — réutiliser tel quel
- `sortTodos.ts` : case "priority" existe mais ne tient pas compte du niveau d'urgence combiné (deadline + priority) — à mettre à jour
- `ActionsTodoCard.tsx` : badge priorité existant (pill coloré) + deadline text — ajouter la barre gauche et le toggle star
- Scope réel : 2 nouveaux utilities + 1 hook + modifications 3 fichiers UI + tests BDD

**Décisions prises :**
- Pas de nouveau champ DB — `priority === "high"` = Prioritaire
- Ordre urgency : overdue > prioritaire > approaching > normal (AC2/AC3)
- Toggle star = binaire high ↔ medium (pas un cycle 3-niveaux — la vue détail reste pour ça)
- Seuil "approaching" = 48h (fixe pour l'instant, question ouverte sur configurabilité)

### File List

**Nouveaux fichiers créés :**
- `pensieve/mobile/src/contexts/action/utils/getUrgencyLevel.ts`
- `pensieve/mobile/src/contexts/action/utils/getUrgencyBorderColor.ts`
- `pensieve/mobile/src/contexts/action/hooks/useTogglePriority.ts`
- `pensieve/mobile/src/contexts/action/utils/__tests__/getUrgencyLevel.test.ts`
- `pensieve/mobile/tests/acceptance/features/story-8-15-priorisation-visuelle-des-taches.feature`
- `pensieve/mobile/tests/acceptance/story-8-15.test.ts`

**Fichiers modifiés :**
- `pensieve/mobile/src/contexts/action/ui/ActionsTodoCard.tsx`
- `pensieve/mobile/src/contexts/action/ui/TodoItem.tsx`
- `pensieve/mobile/src/contexts/action/utils/sortTodos.ts`
- `pensine/_bmad-output/implementation-artifacts/sprint-status.yaml`

**Fichiers ajoutés lors du code review :**
- `pensieve/mobile/src/contexts/action/utils/__tests__/getUrgencyBorderColor.test.ts`

# Story 8.13: Supprimer une Tâche

Status: review

<!-- Note: Validation optionnelle. Lancer validate-create-story pour vérification qualité avant dev-story. -->

## Story

As a **user**,
I want **to permanently delete an individual action/todo from the Actions tab or its detail view**,
So that **I can keep my task list clean by removing tasks that are no longer relevant**.

## Context

Quick win identifié lors d'une session discovery avec un power user (2026-02-17).
L'absence de suppression individuelle de tâche est une friction UX notable : les utilisateurs ne peuvent pas nettoyer leur liste d'actions des tâches obsolètes.

**Distinction importante avec la Story 8.14 (Abandonner) :**
- **Supprimer (cette story)** = hard delete en base locale → la tâche disparaît définitivement de la liste active
- **Abandonner (8.14)** = changement d'état sémantique, la tâche reste visible avec statut "abandoned"

**Infrastructure existante (ne pas réinventer) :**
- `ITodoRepository.delete(id)` → hard DELETE SQL déjà implémenté (ligne 65, `mobile/src/contexts/action/domain/ITodoRepository.ts`)
- `TodoRepository.delete(id)` → `DELETE FROM todos WHERE id = ?` déjà implémenté (ligne 223-225, `mobile/src/contexts/action/data/TodoRepository.ts`)
- `useDeleteTodo` hook → React Query mutation + invalidation `["todos"]` déjà créé (`mobile/src/contexts/action/hooks/useDeleteTodo.ts`)
- `TodoDetailPopover.handleDelete()` → Alert.alert confirmation + mutation **déjà implémentée**, masquée par `debugMode === true` (ligne 399-408)
- Backend `DELETE /api/todos/:id` → soft delete avec `deletedAt` déjà implémenté (ligne 64-82, `backend/src/modules/action/application/controllers/todos.controller.ts`)
- `SwipeableCard` générique → pattern Swipeable de react-native-gesture-handler déjà utilisé pour les captures (`mobile/src/components/cards/SwipeableCard.tsx`)

## Acceptance Criteria

### AC1: Swipe pour supprimer (geste rapide)
**Given** I am viewing a Todo card in the Actions tab
**When** I swipe left on the Todo card
**Then** a red delete action area is revealed
**And** a trash icon with label "Supprimer" is displayed
**And** tapping the delete action triggers a confirmation dialog

### AC2: Confirmation avant suppression
**Given** I triggered the delete action via swipe (AC1) or detail view button (AC4)
**When** the confirmation dialog appears
**Then** I see the message "Supprimer cette tâche ? Cette action est irréversible."
**And** I can confirm with "Supprimer" or cancel with "Annuler"
**And** cancelling closes the dialog and returns to the normal Todo card state without any changes

### AC3: Suppression effective avec feedback
**Given** I confirmed the deletion
**When** the deletion is processed
**Then** the Todo is permanently removed from the local OP-SQLite database (hard delete)
**And** the Todo card is animated out of the list (FadeOut + Layout animation)
**And** haptic feedback confirms the action (NotificationFeedbackType.Success, if haptic enabled)
**And** the task count in the Actions tab updates immediately via React Query cache invalidation

### AC4: Bouton supprimer dans la vue détail
**Given** I am in the Todo detail view (TodoDetailPopover)
**When** I look for delete options
**Then** I see a "Supprimer" destructive button (rouge/danger) — le bouton existe déjà, retirer la condition `debugMode`
**And** tapping it triggers the same confirmation dialog as AC2
**And** after confirmation, the popover closes and I return to the Actions tab list with the deleted todo removed

### AC5: Non-régression — corbeille existante
**Given** the trash system exists (filter 'trash' dans ActionsScreen, `findAllDeletedWithSource`, `deleteAllDeleted`)
**When** I delete a todo via this story's feature
**Then** the deleted todo does NOT appear in the trash (hard delete, pas soft delete)
**And** the existing bulk-delete-completed feature (Story 5.4 AC10) continues to work correctly
**And** the existing trash filter for soft-deleted todos continues to work correctly

## Tasks / Subtasks

### Task 1: Activer le bouton "Supprimer" dans TodoDetailPopover (AC4, AC2)
- [x] Subtask 1.1 : Ouvrir `mobile/src/contexts/action/ui/TodoDetailPopover.tsx`
- [x] Subtask 1.2 : Repérer la condition `debugMode === true` autour du Delete button (lignes ~399-408)
- [x] Subtask 1.3 : Retirer la condition `debugMode` — rendre le bouton toujours visible
- [x] Subtask 1.4 : Vérifier que `handleDelete()` (lignes ~221-238) utilise déjà Alert.alert avec confirmation et appelle `deleteTodo.mutate(todo.id)`
- [x] Subtask 1.5 : S'assurer que le haptic feedback est conditionné à `settingsStore.hapticFeedbackEnabled` (pattern cohérent)
- [x] Subtask 1.6 : Vérifier que la fermeture du popover après suppression est correcte (onClose appelé)

### Task 2: Swipe-to-delete sur ActionsTodoCard (AC1, AC2, AC3)
- [x] Subtask 2.1 : Ouvrir `mobile/src/contexts/action/ui/ActionsTodoCard.tsx`
- [x] Subtask 2.2 : Importer `Swipeable` depuis `react-native-gesture-handler` (déjà utilisé dans `SwipeableCard.tsx` pour référence)
- [x] Subtask 2.3 : Wrapper le composant existant avec `Swipeable` en utilisant `renderRightActions`
  - Action rouge avec icône 🗑️ et label "Supprimer"
  - Pattern d'animation inspiré de `SwipeableCard.tsx` (interpolation opacity + translateX)
  - Haptic feedback au trigger via `expo-haptics` si `settingsStore.hapticFeedbackEnabled`
- [x] Subtask 2.4 : Sur tap de l'action delete → déclencher un Alert.alert de confirmation (même message qu'AC2)
- [x] Subtask 2.5 : Après confirmation → appeler `useDeleteTodo.mutate(todo.id)` depuis le hook existant
- [x] Subtask 2.6 : Fermer le swipe si annulé (`swipeableRef.current?.close()`)
- [x] Subtask 2.7 : S'assurer que le swipe ne bloque pas la ScrollView (gestion `hitSlop`, `simultaneousHandlers`) — `failOffsetY={[-15, 15]}` conforme au pattern SwipeableCard
- [x] Subtask 2.8 : Ne pas activer le swipe si `readonly === true` (contexte corbeille) — `enabled={!readonly}`

### Task 3: Animation de sortie après suppression (AC3)
- [x] Subtask 3.1 : `ActionsScreen.renderTodoCard()` wrape déjà chaque `ActionsTodoCard` avec `<Animated.View exiting={FadeOut.duration(200)}>` — aucune modification nécessaire
- [x] Subtask 3.2 : `useDeleteTodo` invalide déjà `["todos"]` → filter counts mis à jour automatiquement
- [x] Subtask 3.3 : FilterTabs badges se mettent à jour via React Query cache invalidation ✅

### Task 4: Tests BDD (AC1, AC2, AC3, AC4)
- [x] Subtask 4.1 : Créer `mobile/tests/acceptance/features/story-8-13-supprimer-une-tache.feature`
- [x] Subtask 4.2 : Créer `mobile/tests/acceptance/story-8-13.test.ts` avec jest-cucumber step definitions
- [x] Subtask 4.3 : Implémenter minimum 4 scénarios BDD :
  1. Suppression via detail view (AC4) → todo absent de la DB ✅
  2. Annulation via detail view → todo toujours présent ✅
  3. Suppression via swipe + confirmation (AC1, AC3) ✅
  4. Non-régression corbeille (AC5) — soft-deleted todos non affectés ✅
- [x] Subtask 4.4 : `jest.mock('@op-engineering/op-sqlite')` + `TodoRepository` instancié directement (pattern 8.x)

### Task 5: Vérification non-régression (AC5)
- [x] Subtask 5.1 : `npm run test:acceptance` — 236 tests passent, 4/4 story-8-13 passent ✅
- [x] Subtask 5.2 : Aucune régression sur les tests existants (story-5-2 et 5-3 échouaient déjà avant cette story)
- [x] Subtask 5.3 : Swipe désactivé en mode `readonly` → bulk-delete et filtre 'trash' non impactés ✅
- [x] Subtask 5.4 : Test scénario 4 (AC5) valide la non-régression de la corbeille ✅

## Dev Notes

### Décisions Critiques pour le Dev Agent

#### 1. Hard delete vs Soft delete (mobile)
La story demande un **hard delete en base locale** : le todo disparaît définitivement de la liste active. Cette décision est déjà cohérente avec l'implémentation existante :
- `TodoRepository.delete(id)` fait déjà `DELETE FROM todos WHERE id = ?` (hard delete OP-SQLite)
- Ne PAS utiliser le soft delete `_status = 'deleted'` (celui-ci est réservé à la corbeille des todos auto-générés par l'IA)
- La raison : la corbeille (`_status = 'deleted'`) concerne les todos générés automatiquement que l'IA a peut-être mal extrait. La suppression manuelle par l'utilisateur est intentionnelle et définitive.

#### 2. Ce qui EXISTE déjà — Ne pas réimplémenter
```
✅ ITodoRepository.delete(id) — interface déjà là
✅ TodoRepository.delete(id) — hard DELETE déjà là
✅ useDeleteTodo — hook React Query déjà là
✅ handleDelete() dans TodoDetailPopover — logique + Alert.alert déjà là
✅ DELETE /api/todos/:id backend — endpoint déjà là
✅ SwipeableCard pattern — modèle swipe déjà dans le projet
```

#### 3. Ce qui MANQUE — Travail réel de la story
```
❌ Bouton delete dans TodoDetailPopover : masqué par debugMode → retirer la condition
❌ Swipe-to-delete dans ActionsTodoCard : composant à wrapper avec Swipeable
❌ Tests BDD pour cette story (feature + step definitions)
❌ Animation de sortie sur ActionsTodoCard (si pas déjà présente)
```

#### 4. Pattern Swipe à Réutiliser (SwipeableCard.tsx)
```typescript
// Référence : mobile/src/components/cards/SwipeableCard.tsx
// Library : react-native-gesture-handler (déjà installée)
import { Swipeable } from 'react-native-gesture-handler';

const renderRightActions = (
  progress: Animated.AnimatedInterpolation<number>,
  dragX: Animated.AnimatedInterpolation<number>
) => {
  const translateX = dragX.interpolate({
    inputRange: [-80, 0],
    outputRange: [0, 80],
    extrapolate: 'clamp',
  });
  return (
    <Animated.View style={[styles.deleteAction, { transform: [{ translateX }] }]}>
      <Pressable onPress={handleDeleteWithConfirmation} style={styles.deleteButton}>
        <Ionicons name="trash-outline" size={24} color="white" />
        <Text style={styles.deleteLabel}>Supprimer</Text>
      </Pressable>
    </Animated.View>
  );
};

return (
  <Swipeable
    ref={swipeableRef}
    renderRightActions={renderRightActions}
    rightThreshold={40}
    friction={2}
    overshootRight={false}
  >
    {/* Existing ActionsTodoCard content */}
  </Swipeable>
);
```

#### 5. Retirer debugMode dans TodoDetailPopover
```typescript
// AVANT (masqué) :
{debugMode && (
  <TouchableOpacity onPress={handleDelete} style={styles.deleteButton}>
    <Text style={styles.deleteButtonText}>Supprimer</Text>
  </TouchableOpacity>
)}

// APRÈS (toujours visible) :
<TouchableOpacity onPress={handleDelete} style={styles.deleteButton}>
  <Text style={styles.deleteButtonText}>Supprimer</Text>
</TouchableOpacity>
```

#### 6. Confirmation Dialog (déjà dans handleDelete, vérifier le message)
```typescript
// mobile/src/contexts/action/ui/TodoDetailPopover.tsx
const handleDelete = () => {
  Alert.alert(
    'Supprimer cette tâche ?',
    'Cette action est irréversible.',
    [
      { text: 'Annuler', style: 'cancel' },
      {
        text: 'Supprimer',
        style: 'destructive',
        onPress: () => {
          if (settingsStore.hapticFeedbackEnabled) {
            Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
          }
          deleteTodo.mutate(todo.id);
          onClose();
        }
      }
    ]
  );
};
```

#### 7. Pattern Result (ADR-023) — useDeleteTodo
S'assurer que `useDeleteTodo` gère les erreurs correctement :
```typescript
// Si useDeleteTodo ne gère pas déjà les erreurs utilisateur :
const { mutate: deleteTodo } = useDeleteTodo();
// onError doit afficher une Alert (erreur user-friendly)
// onSuccess doit déclencher l'invalidation React Query
```

#### 8. Swipe et mode readonly (corbeille)
`ActionsTodoCard` reçoit un prop `readonly` pour le contexte corbeille. Désactiver le swipe si readonly :
```typescript
<Swipeable
  renderRightActions={readonly ? undefined : renderRightActions}
  enabled={!readonly}
>
```

### Project Structure Notes

**Fichiers à modifier :**
```
pensieve/mobile/src/contexts/action/ui/TodoDetailPopover.tsx
  → Retirer condition debugMode sur delete button (Subtask 1.2-1.3)

pensieve/mobile/src/contexts/action/ui/ActionsTodoCard.tsx
  → Ajouter Swipeable wrapper avec renderRightActions (Task 2)
  → Éventuellement ajouter exiting animation (Subtask 3.1)
```

**Nouveaux fichiers à créer :**
```
pensieve/mobile/tests/acceptance/features/story-8-13-supprimer-une-tache.feature
pensieve/mobile/tests/acceptance/story-8-13.test.ts
```

**Fichiers existants à NE PAS modifier :**
```
pensieve/mobile/src/contexts/action/data/TodoRepository.ts  (delete() déjà là)
pensieve/mobile/src/contexts/action/domain/ITodoRepository.ts  (delete() déjà là)
pensieve/mobile/src/contexts/action/hooks/useDeleteTodo.ts  (hook déjà là)
pensieve/backend/...  (backend endpoint déjà là)
```

**Structure de test à suivre (pattern Story 5.x) :**
```
mobile/tests/acceptance/
├── features/
│   └── story-8-13-supprimer-une-tache.feature   ← Nouveau
└── story-8-13.test.ts                            ← Nouveau
```

### Architecture Compliance

- **ADR-021 (DI Transient First)** : `useDeleteTodo` résout `ITodoRepository` en lazy dans le hook ✅
- **ADR-023 (Result Pattern)** : `TodoRepository.delete()` retourne `Promise<Result<void>>` — vérifier et conformer si nécessaire
- **ADR-024 (Clean Code)** : Pas de magic numbers, noms révélateurs d'intention
- **ADR-026 (Backend Data Model)** : Backend utilise soft delete avec `deletedAt` (déjà conforme) ✅
- **Pas de `jest-expo`** : Tests avec `babel-jest` (unit) et `ts-jest` (acceptance) — cf. project-context.md

### References

- [Source: pensieve/mobile/src/contexts/action/domain/ITodoRepository.ts#L65] — méthode `delete(id)`
- [Source: pensieve/mobile/src/contexts/action/data/TodoRepository.ts#L223-225] — implémentation hard delete
- [Source: pensieve/mobile/src/contexts/action/hooks/useDeleteTodo.ts] — hook existant
- [Source: pensieve/mobile/src/contexts/action/ui/TodoDetailPopover.tsx#L221-238 et L399-408] — handleDelete + bouton debug
- [Source: pensieve/mobile/src/components/cards/SwipeableCard.tsx] — pattern Swipeable à réutiliser
- [Source: pensieve/mobile/src/screens/actions/ActionsScreen.tsx#L95-104] — corbeille/trash filter
- [Source: pensieve/backend/src/modules/action/application/controllers/todos.controller.ts#L64-82] — endpoint DELETE
- [Source: _bmad-output/planning-artifacts/architecture.md#Action Context (Supporting Domain)] — bounded context
- [Source: _bmad-output/planning-artifacts/epics.md#Epic 8 — Story 8.13] — story requirements
- [Source: _bmad-output/implementation-artifacts/stories/epic-5/5-4-completion-et-navigation-des-actions.md#Task 12] — swipe actions skipped (now done here)
- [Source: _bmad-output/project-context.md#Testing Rules] — jest-cucumber BDD, babel-jest unit
- [Source: _bmad-output/project-context.md#Critical Don't-Miss Rules#Rule 16] — ADR-023 Result Pattern

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

N/A — Story de création, pas d'implémentation

### Completion Notes List

Ultimate context engine analysis completed — comprehensive developer guide created.

**Analyse réalisée :**
- ITodoRepository + TodoRepository : méthodes delete déjà présentes (hard delete mobile)
- useDeleteTodo hook : déjà créé
- Backend DELETE endpoint : déjà implémenté (soft delete TypeORM deletedAt)
- TodoDetailPopover : handleDelete() déjà là, masqué par debugMode
- SwipeableCard pattern : déjà utilisé pour captures (react-native-gesture-handler)
- ActionsTodoCard : pas de swipe → principal travail de la story

**Scope réel (plus petit que prévu) :**
- Retirer guard `debugMode` dans TodoDetailPopover = ~5 lignes
- Wrapper ActionsTodoCard avec Swipeable = ~40 lignes
- Tests BDD = 4 scénarios minimum

### File List

N/A — À compléter par le Dev Agent lors de l'implémentation

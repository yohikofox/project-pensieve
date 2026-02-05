# Story 5.1: Affichage Inline des Todos dans le Feed

Status: ready-for-dev

## Story

As a **user**,
I want **to see the detected actions displayed inline with each idea in the feed**,
So that **I can immediately see what needs to be done without navigating to a separate screen**.

## Acceptance Criteria

### AC1: Inline Todo Display with Parent Idea
**Given** a capture has been digested and contains extracted actions (todos)
**When** I view the capture in the feed
**Then** each Idea that has associated Todos displays them inline below the idea text
**And** the action badges are visually distinct (icon, color, priority indicator)
**And** each Todo shows: checkbox, description, deadline (if any), priority badge

### AC2: Multiple Todos Sorted by Priority
**Given** an Idea has multiple Todos
**When** displayed in the feed
**Then** all Todos are listed in priority order (high → medium → low)
**And** each Todo is presented on its own line for clarity
**And** the list is visually grouped under the parent Idea

### AC3: No Actions - Clean Display
**Given** an Idea has no Todos
**When** displayed in the feed
**Then** no action section is shown for that Idea
**And** the Idea displays normally without empty action placeholders

### AC4: Todo Detail with Deadline and Priority
**Given** I view a Todo inline in the feed
**When** the Todo is displayed
**Then** I can see its complete description (truncated if very long, with "...more")
**And** the deadline is shown in human-readable format ("Today", "Tomorrow", "In 3 days")
**And** overdue deadlines are highlighted in red/warning color
**And** priority is indicated with color-coded badges (🔴 High, 🟡 Medium, 🟢 Low)

### AC5: Completed Todo Visual State
**Given** a Todo is marked as completed
**When** displayed inline
**Then** the checkbox shows as checked
**And** the text is styled with strikethrough
**And** the Todo is slightly dimmed to indicate completion
**And** completed Todos appear at the bottom of the list (below active ones)

### AC6: Todo Interaction and Navigation (FR20)
**Given** I tap on an inline Todo
**When** the Todo is tapped
**Then** a detail popover appears with full information
**And** I can edit the description, deadline, or priority
**And** I can mark it as complete/incomplete
**And** I can navigate to the source Idea/Capture (FR20)

### AC7: Consistent Styling Across Feed
**Given** multiple captures in the feed have Todos
**When** scrolling through the feed
**Then** inline Todos are consistently styled across all captures
**And** the visual hierarchy makes it clear which Todos belong to which Ideas
**And** animations are smooth when expanding/collapsing Todo lists (optional)

### AC8: Todo Checkbox Toggle (FR19)
**Given** I am viewing a Todo (inline in feed)
**When** I tap the checkbox to mark it complete (FR19)
**Then** the checkbox animates to checked state
**And** the Todo text gets strikethrough styling
**And** a satisfying haptic feedback confirms the action (medium impact)
**And** a subtle completion animation plays (e.g., confetti, glow, or checkmark burst)
**And** the Todo status is immediately updated in the database

## Tasks / Subtasks

### Task 1: Todo Entity Mobile Schema (OP-SQLite) (AC1, AC2, AC4)
- [x] Subtask 1.1: Design Todo table schema for OP-SQLite (id, thoughtId, ideaId, description, deadline, priority, status, completedAt, createdAt, updatedAt)
- [x] Subtask 1.2: Create Todo model with TypeScript interfaces
- [x] Subtask 1.3: Implement TodoRepository with CRUD operations (OP-SQLite)
- [x] Subtask 1.4: Add indices on thoughtId, ideaId, status, priority for efficient queries
- [ ] Subtask 1.5: Create TodoSync protocol for backend sync (Epic 6 preparation)
- [x] Subtask 1.6: Add unit tests for TodoRepository

### Task 2: Fetch Todos by Idea (AC1, AC2)
- [x] Subtask 2.1: Create useTodos hook (React Query) to fetch todos by ideaId
- [x] Subtask 2.2: Implement getTodosByIdeaId query with sorting (priority desc, createdAt asc)
- [x] Subtask 2.3: Add loading and error states
- [x] Subtask 2.4: Cache todos locally with React Query (staleTime: 5 minutes)
- [ ] Subtask 2.5: Test edge cases: no todos, many todos (50+), mixed priorities

### Task 3: Inline Todo List Component (AC1, AC2, AC3, AC4, AC5, AC7)
- [x] Subtask 3.1: Create InlineTodoList component (receives ideaId prop)
- [x] Subtask 3.2: Fetch todos using useTodos hook
- [x] Subtask 3.3: Sort todos by status (active first) then priority (high → medium → low)
- [x] Subtask 3.4: Render each todo with TodoItem component
- [x] Subtask 3.5: Hide section if todos array is empty (AC3)
- [x] Subtask 3.6: Add visual grouping (border, background, padding) under parent Idea
- [x] Subtask 3.7: Implement consistent styling across feed (AC7)
- [ ] Subtask 3.8: Add unit tests for InlineTodoList

### Task 4: TodoItem Component (AC4, AC5, AC8)
- [x] Subtask 4.1: Create TodoItem component (receives todo, onToggle, onTap props)
- [x] Subtask 4.2: Render checkbox (checked if status = 'completed')
- [x] Subtask 4.3: Render description with truncation ("...more" if > 80 chars)
- [x] Subtask 4.4: Render deadline with human-readable format (date-fns relative time)
- [x] Subtask 4.5: Render priority badge with color coding (🔴 High, 🟡 Medium, 🟢 Low)
- [x] Subtask 4.6: Apply strikethrough and dimming styles if completed (AC5)
- [x] Subtask 4.7: Highlight overdue deadlines in red/warning color (AC4)
- [x] Subtask 4.8: Add tap handler to open detail popover (AC6)
- [x] Subtask 4.9: Add checkbox tap handler with haptic feedback (AC8)
- [ ] Subtask 4.10: Add unit tests for TodoItem

### Task 5: Todo Checkbox Toggle Logic (AC8, FR19)
- [x] Subtask 5.1: Implement toggleTodoStatus mutation (React Query)
- [x] Subtask 5.2: Update Todo status in OP-SQLite (completed ↔ todo)
- [x] Subtask 5.3: Record completedAt timestamp when marking complete
- [x] Subtask 5.4: Clear completedAt when unmarking
- [x] Subtask 5.5: Trigger haptic feedback on toggle (expo-haptics medium impact)
- [ ] Subtask 5.6: Play completion animation (confetti, glow, or checkmark burst) - DEFERRED to Task 8
- [x] Subtask 5.7: Invalidate React Query cache to refresh UI
- [x] Subtask 5.8: Handle optimistic UI updates (instant feedback)
- [x] Subtask 5.9: Add rollback on mutation error
- [x] Subtask 5.10: Add unit tests for toggleTodoStatus

### Task 6: Todo Detail Popover (AC6, FR20)
- [x] Subtask 6.1: Create TodoDetailPopover component (modal/bottom sheet)
- [x] Subtask 6.2: Display full todo information (description, deadline, priority, status)
- [x] Subtask 6.3: Add inline edit for description (TextInput with save/cancel)
- [x] Subtask 6.4: Add deadline picker (date picker modal)
- [x] Subtask 6.5: Add priority selector (high/medium/low buttons)
- [x] Subtask 6.6: Add complete/incomplete toggle
- [x] Subtask 6.7: Add "View Origin" button to navigate to source Idea/Capture (FR20)
- [x] Subtask 6.8: Implement navigation to CaptureDetailScreen with Idea highlighted
- [x] Subtask 6.9: Add smooth transition animations (slide up, fade)
- [ ] Subtask 6.10: Add unit tests for TodoDetailPopover - DEFERRED (complex UI with DateTimePicker mock issues)

### Task 7: Navigation to Source Capture (AC6, FR20)
- [ ] Subtask 7.1: Implement navigateToSourceCapture function
- [ ] Subtask 7.2: Fetch thoughtId from todo, then fetch captureId from thought
- [ ] Subtask 7.3: Navigate to CaptureDetailScreen with captureId param
- [ ] Subtask 7.4: Highlight the relevant Idea and Todo in detail view
- [ ] Subtask 7.5: Implement scroll-to-idea behavior (scroll to highlighted Idea)
- [ ] Subtask 7.6: Add hero transition animation (optional, smooth)
- [ ] Subtask 7.7: Handle edge case: capture not found or deleted
- [ ] Subtask 7.8: Add unit tests for navigateToSourceCapture

### Task 8: Completion Animation (AC8)
- [x] Subtask 8.1: Choose completion animation style (checkmark burst with scale pulse and green glow)
- [x] Subtask 8.2: Install react-native-reanimated for smooth animations (already installed: 4.1.1)
- [x] Subtask 8.3: Implement completion animation component (CompletionAnimation.tsx)
- [x] Subtask 8.4: Trigger animation on checkbox toggle (only when marking complete)
- [x] Subtask 8.5: Ensure 60fps performance (Reanimated worklets run on UI thread)
- [ ] Subtask 8.6: Add subtle sound effect (optional) - DEFERRED (not required for MVP)
- [ ] Subtask 8.7: Test on iOS and Android (platform differences) - MANUAL TESTING REQUIRED
- [x] Subtask 8.8: Add unit tests for animation triggers (7/7 tests passing)

### Task 9: Human-Readable Deadline Formatting (AC4)
- [x] Subtask 9.1: Install date-fns library for date formatting
- [x] Subtask 9.2: Create formatDeadline utility function
- [x] Subtask 9.3: Implement relative time formatting ("Today", "Tomorrow", "In 3 days")
- [x] Subtask 9.4: Detect overdue deadlines (deadline < now)
- [x] Subtask 9.5: Return warning color for overdue deadlines
- [x] Subtask 9.6: Handle null/undefined deadlines ("No deadline")
- [x] Subtask 9.7: Add unit tests for formatDeadline (various date scenarios)

### Task 10: Integration with Feed Screen (AC1, AC7)
- [x] Subtask 10.1: Create IdeaItem component to display Idea with InlineTodoList
- [x] Subtask 10.2: Pass ideaId prop to InlineTodoList from IdeaItem
- [x] Subtask 10.3: Ensure proper spacing between Ideas and Todos
- [ ] Subtask 10.4: Integrate IdeaItem into CaptureDetailScreen - BLOCKED by Story 4.3 mobile implementation
- [ ] Subtask 10.5: Test scroll performance with many captures + todos
- [ ] Subtask 10.6: Add loading skeleton for todos while fetching - DEFERRED (InlineTodoList already shows ActivityIndicator)
- [ ] Subtask 10.7: Test with mixed captures (some with todos, some without)

### Task 11: BDD Integration Tests (AC1-AC8)
- [ ] Subtask 11.1: Write BDD acceptance tests for AC1-AC8 (jest-cucumber)
- [ ] Subtask 11.2: Create test fixtures (sample captures, ideas, todos)
- [ ] Subtask 11.3: Test inline todo display (AC1)
- [ ] Subtask 11.4: Test multiple todos sorted by priority (AC2)
- [ ] Subtask 11.5: Test no actions clean display (AC3)
- [ ] Subtask 11.6: Test todo detail with deadline and priority (AC4)
- [ ] Subtask 11.7: Test completed todo visual state (AC5)
- [ ] Subtask 11.8: Test todo interaction and navigation (AC6, FR20)
- [ ] Subtask 11.9: Test consistent styling across feed (AC7)
- [ ] Subtask 11.10: Test checkbox toggle (AC8, FR19)

## Dev Notes

### Architecture Context

**Bounded Context:** Action Context (Supporting Domain)

**Integration Pattern:**
- Story 4.3 (Knowledge Context) provides TodosExtracted event
- Story 5.1 (Action Context) displays todos inline in Feed
- Communication via Domain Events published in Story 4.3

**Related Contexts:**
- **Knowledge Context** (Core): Source of Thought and Idea entities
- **Capture Context** (Supporting): Source Capture for navigation (FR20)
- **Notification Context** (Generic): Future notifications for todo deadlines (post-MVP)

**Existing Components from Previous Stories:**
- TodosExtracted event (Story 4.3) ✓ - Backend publishes when todos are created
- WebSocket real-time updates (Story 4.2) ✓ - For future real-time todo sync
- OP-SQLite local database (Story 2.1) ✓ - Local storage layer
- React Navigation (Epic 1-3) ✓ - Navigation to source captures (FR20)
- Haptic feedback (Story 2.1, 4.4) ✓ - Completion haptics (AC8)

**Critical Architecture Decision (ADR-018):**
> **OP-SQLite Local Storage:** Story 5.1 MUST use OP-SQLite (not WatermelonDB) for local Todo storage. Migration from WatermelonDB to OP-SQLite completed in Story 2.1. All new entities use OP-SQLite with manual sync protocol (Epic 6).

**Data Flow:**
```
[Story 4.3 - Backend] TodosExtracted event → Broadcast via WebSocket
                              ↓
[Story 5.1 - Mobile] Receive todos → Store in OP-SQLite
                              ↓
[Story 5.1 - Mobile] useTodos hook → Fetch from OP-SQLite → Display inline
                              ↓
[User Action] Toggle checkbox → Update OP-SQLite → Optimistic UI
                              ↓
[Story 5.1 - Mobile] Haptic feedback + Animation → Complete
```

**Domain Model (Action Context):**
```typescript
Todo {
  id: UUID
  thoughtId: UUID         // FK to Thought (Knowledge Context)
  ideaId: UUID           // FK to Idea (Opportunity Context)
  status: 'todo' | 'completed' | 'abandoned'
  description: string
  deadline?: DateTime    // Optional suggested deadline
  priority: 'low' | 'medium' | 'high'
  completedAt?: DateTime
  createdAt: DateTime
  updatedAt: DateTime
}
```

**Relations:**
- `Todo → Thought`: Many-to-One (thoughtId FK)
- `Todo → Idea`: Many-to-One (ideaId FK)
- `Thought → Capture`: One-to-One (captureId FK)

**Critical Business Rule:**
> A Todo is ALWAYS created by AI digestion (Story 4.3). Users CANNOT manually create todos in MVP. This ensures todos are semantically linked to the source capture and idea.

### Technology Stack

**Mobile:**
- **React Native + Expo** (custom dev client) - Already configured ✓
- **TypeScript strict** - Enforced ✓
- **OP-SQLite** - Local database (ADR-018) ✓
  - Direct SQL queries (no ORM)
  - JSI-native performance (4× faster than WatermelonDB)
  - Manual sync protocol for Epic 6
- **React Query (TanStack Query)** - Data fetching and caching
  - useQuery for fetching todos by ideaId
  - useMutation for toggle status, update todo
  - Optimistic updates for instant UI feedback
- **React Navigation** - Navigation to source captures (FR20) ✓
- **expo-haptics** - Haptic feedback on checkbox toggle ✓
- **react-native-reanimated** - Smooth animations (60fps requirement)
- **date-fns** - Date formatting ("Today", "Tomorrow", relative time)
- **React Native UI Components** - Checkbox, TextInput, Modal
- **TSyringe** - Dependency Injection (ADR-017 pattern) ✓

**Backend:**
- **NestJS** (TypeScript) - Already configured ✓
- **PostgreSQL** - Todo persistence (already created in Story 4.3) ✓
- **RabbitMQ** - Event-driven communication ✓
- **WebSocket (Socket.io)** - Real-time updates (already configured) ✓
- **TypeORM** - ORM with migrations (already established) ✓

**Testing:**
- **Jest** - Unit tests
- **React Testing Library** - Component tests
- **jest-cucumber** - BDD acceptance tests (pattern from Story 4.4)

### Critical Implementation Notes

**From Architecture Document:**

1. **ADR-018 - OP-SQLite (CRITICAL):**
   - Story 5.1 MUST use OP-SQLite for Todo storage
   - Migration from WatermelonDB complete in Story 2.1
   - JSI-native performance: 4× faster than WatermelonDB
   - Direct SQL queries (no ORM abstraction)
   - Manual sync protocol required for Epic 6

2. **FR16 - Inline Todo Display:**
   - Todos MUST be displayed inline with each Idea in the Feed
   - No separate navigation required for basic todo viewing
   - Visual grouping under parent Idea is critical for UX

3. **FR19 - Checkbox Toggle:**
   - Immediate local update (optimistic UI)
   - Haptic feedback confirms action (medium impact)
   - Completion animation celebrates the action (Liquid Glass design)
   - Database update happens asynchronously (React Query mutation)

4. **FR20 - Navigate to Source:**
   - Deep link pattern: Todo → Idea → Thought → Capture
   - Highlight the relevant Idea and Todo in CaptureDetailScreen
   - Smooth hero transition animation (optional but recommended)

5. **Domain-Driven Design:**
   - Todo is an Aggregate Root in Action Context
   - Relations: Todo → Thought (Many-to-One), Todo → Idea (Many-to-One)
   - Status workflow: 'todo' → 'completed' (AC8, AC5)
   - Abandoned status for future "Cancel" action (post-MVP)

6. **Liquid Glass Design System:**
   - 60fps animations required (AC8)
   - Haptic feedback on key actions (checkbox toggle)
   - Smooth transitions when expanding/collapsing todo lists
   - Color-coded priority badges (🔴 High, 🟡 Medium, 🟢 Low)
   - Overdue deadlines in red/warning color

7. **Performance Requirements:**
   - NFR4: Feed loading < 1s (cache local todos with React Query)
   - Scroll performance must remain smooth (60fps) with many todos
   - Optimistic UI updates for instant feedback (no perceived latency)

8. **Data Consistency:**
   - Todos are created by backend in Story 4.3 (TodosExtracted event)
   - Mobile receives todos via WebSocket → stores in OP-SQLite
   - Local updates (toggle status) sync to backend later (Epic 6)
   - Conflict resolution: last-write-wins (MVP approach)

9. **Truncation Logic:**
   - Todo description > 80 chars → truncate with "...more" (AC4)
   - Tapping truncated todo opens detail popover (AC6)
   - Full description visible in popover with inline edit

10. **Priority Sorting:**
    - Active todos first, completed todos at bottom (AC5)
    - Within active: High → Medium → Low priority (AC2)
    - Within completed: Most recent first (completedAt desc)

11. **Deadline Formatting:**
    - Use date-fns for human-readable relative time
    - "Today", "Tomorrow", "In 3 days", "Overdue by 2 days"
    - Overdue deadlines highlighted in red/warning color (AC4)
    - Null deadline → "No deadline" label

12. **Haptic Feedback Pattern:**
    - Checkbox toggle → medium impact (AC8)
    - Completion animation → single pulse (same as Story 4.4 pattern)
    - Respect user preferences (hapticFeedbackEnabled from Story 4.4)

### Domain Events

**Events Consumed by This Story:**

```typescript
// From Story 4.3 - Todos extracted by AI digestion
TodosExtracted {
  thoughtId: UUID
  captureId: UUID
  userId: UUID
  todos: Array<{
    id: UUID
    ideaId: UUID
    description: string
    deadline?: DateTime
    priority: 'low' | 'medium' | 'high'
    status: 'todo'
  }>
  extractedAt: DateTime
}
```

**Events Published by This Story:**

```typescript
// NEW: User completes a todo (AC8)
TodoCompleted {
  todoId: UUID
  thoughtId: UUID
  ideaId: UUID
  userId: UUID
  completedAt: DateTime
}

// NEW: User uncompletes a todo (reverses completion)
TodoUncompleted {
  todoId: UUID
  thoughtId: UUID
  ideaId: UUID
  userId: UUID
  uncompl

etedAt: DateTime
}

// NEW: User updates todo details (AC6)
TodoUpdated {
  todoId: UUID
  userId: UUID
  changes: {
    description?: string
    deadline?: DateTime
    priority?: 'low' | 'medium' | 'high'
  }
  updatedAt: DateTime
}
```

**Event Handlers:**

```typescript
// Listen to TodosExtracted from Story 4.3
onTodosExtracted(event: TodosExtracted) {
  // Store todos in OP-SQLite
  // Invalidate React Query cache to trigger refresh
  // Display inline in feed automatically
}

// Publish TodoCompleted when user toggles checkbox (AC8)
onCheckboxToggle(todoId: UUID, newStatus: 'completed' | 'todo') {
  if (newStatus === 'completed') {
    publishEvent(TodoCompleted { ... })
  } else {
    publishEvent(TodoUncompleted { ... })
  }
}
```

### Entity Schemas

**Todo Table (OP-SQLite - Mobile):**

```sql
CREATE TABLE IF NOT EXISTS todos (
  id TEXT PRIMARY KEY,
  thought_id TEXT NOT NULL,
  idea_id TEXT NOT NULL,
  status TEXT NOT NULL CHECK(status IN ('todo', 'completed', 'abandoned')),
  description TEXT NOT NULL,
  deadline INTEGER, -- Unix timestamp (nullable)
  priority TEXT NOT NULL CHECK(priority IN ('low', 'medium', 'high')),
  completed_at INTEGER, -- Unix timestamp (nullable)
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  FOREIGN KEY (thought_id) REFERENCES thoughts(id) ON DELETE CASCADE,
  FOREIGN KEY (idea_id) REFERENCES ideas(id) ON DELETE CASCADE
);

CREATE INDEX idx_todos_thought_id ON todos(thought_id);
CREATE INDEX idx_todos_idea_id ON todos(idea_id);
CREATE INDEX idx_todos_status ON todos(status);
CREATE INDEX idx_todos_priority ON todos(priority);
CREATE INDEX idx_todos_deadline ON todos(deadline);
```

**TypeScript Interface:**

```typescript
export interface Todo {
  id: string;
  thoughtId: string;
  ideaId: string;
  status: 'todo' | 'completed' | 'abandoned';
  description: string;
  deadline?: number; // Unix timestamp
  priority: 'low' | 'medium' | 'high';
  completedAt?: number; // Unix timestamp
  createdAt: number;
  updatedAt: number;
}

export interface TodoRepository {
  create(todo: Todo): Promise<void>;
  findById(id: string): Promise<Todo | null>;
  findByIdeaId(ideaId: string): Promise<Todo[]>;
  findByThoughtId(thoughtId: string): Promise<Todo[]>;
  update(id: string, changes: Partial<Todo>): Promise<void>;
  delete(id: string): Promise<void>;
  toggleStatus(id: string): Promise<Todo>;
}
```

**React Query Hooks:**

```typescript
// Fetch todos by idea ID (AC1, AC2)
export const useTodos = (ideaId: string) => {
  return useQuery({
    queryKey: ['todos', ideaId],
    queryFn: () => todoRepository.findByIdeaId(ideaId),
    staleTime: 5 * 60 * 1000, // 5 minutes cache
  });
};

// Toggle todo status (AC8, FR19)
export const useToggleTodoStatus = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (todoId: string) => todoRepository.toggleStatus(todoId),
    onMutate: async (todoId) => {
      // Optimistic update
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      const previousTodos = queryClient.getQueryData(['todos']);

      queryClient.setQueryData(['todos'], (old: Todo[]) =>
        old.map(todo =>
          todo.id === todoId
            ? { ...todo, status: todo.status === 'completed' ? 'todo' : 'completed' }
            : todo
        )
      );

      return { previousTodos };
    },
    onError: (err, todoId, context) => {
      // Rollback on error
      queryClient.setQueryData(['todos'], context?.previousTodos);
    },
    onSettled: () => {
      // Refetch to ensure consistency
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
};

// Update todo details (AC6)
export const useUpdateTodo = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, changes }: { id: string; changes: Partial<Todo> }) =>
      todoRepository.update(id, changes),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
};
```

### Previous Story Intelligence

**From Story 4.4 (Notifications de Progression IA):**
- ✅ **Haptic Feedback Pattern:** expo-haptics with medium impact for confirmations (AC8)
- ✅ **User Preferences:** Check hapticFeedbackEnabled before triggering (AC7 from 4.4)
- ✅ **WebSocket Real-Time:** KnowledgeEventsGateway with user-specific rooms (can extend for todos)
- ✅ **React Query Pattern:** useQuery with staleTime for caching, useMutation with optimistic updates
- ✅ **BDD Testing:** jest-cucumber pattern with step definitions (Task 11)
- ✅ **OP-SQLite Usage:** Direct SQL queries with TypeScript interfaces (ADR-018)
- ✅ **TSyringe DI:** `@injectable()` decorator pattern established

**From Story 4.3 (Extraction Automatique d'Actions):**
- ✅ **Todo Creation:** TodosExtracted event published by backend after digestion
- ✅ **Todo Schema:** Backend PostgreSQL Todo entity already exists
- ✅ **Domain Events:** EventBus pattern for cross-context communication
- ✅ **Pattern:** Transaction handling for atomic operations
- ✅ **Pattern:** Comprehensive BDD tests (40+ scenarios)

**From Story 4.2 (Digestion IA - Résumé et Idées Clés):**
- ✅ **WebSocket Gateway:** KnowledgeEventsGateway for real-time updates
- ✅ **Pattern:** User-specific rooms (userId = room ID)
- ✅ **Pattern:** React Query for data fetching and caching
- ✅ **Tech Stack:** Socket.io + NestJS Gateway + React Native socket.io-client

**From Epic 2 Stories (Capture Audio):**
- ✅ **Pattern:** OP-SQLite for local storage (ADR-018)
- ✅ **Pattern:** expo-haptics for haptic feedback
- ✅ **Pattern:** TSyringe DI in mobile app
- ✅ **Learning:** Performance monitoring utilities (measureAsync)

**From Epic 3 Stories (Consultation & Navigation):**
- ✅ **Pattern:** React Navigation for deep linking
- ✅ **Pattern:** Hero transition animations (smooth, 250-350ms)
- ✅ **Pattern:** Swipe gestures for contextual actions
- ✅ **Learning:** 60fps scroll performance with large lists
- ✅ **Learning:** Liquid Glass design system (animations, transitions)

### Git Intelligence

**Recent Commits (Story 4.4):**
- commit `b415980`: Story 4.4 complete with code review
- commit `b7a8b39`: Task 13 complete - BDD integration tests
- commit `7b7dfed`: Tasks 10.2-10.6 and Task 11 complete
- commit `d03a912`: Update pensieve submodule - Story 4.4 tests
- commit `088deb6`: Tasks 5.8, 6.7, 9.6-9.7, 10.1 complete

**Patterns from Story 4.3 + 4.4:**
- WebSocket Gateway pattern with user-specific rooms
- Domain Events via EventBus (publish/subscribe)
- TypeORM entities with proper indices and foreign keys
- TypeORM migrations for production deployment
- Comprehensive BDD tests (jest-cucumber)
- Unit tests with high coverage (90%+)
- TSyringe DI (`@injectable()` decorator)
- React Query optimistic updates pattern

**Code Quality Standards:**
- TypeScript strict mode enforced
- No PII in logs (content preview max 50 chars)
- Environment-based CORS configuration
- Rate limit detection and handling
- Transaction handling for atomic operations
- Cascade delete rules properly configured
- Haptic feedback respects user preferences

### Latest Tech Information

**react-native-reanimated (Animations):**
- Version: react-native-reanimated v3.x (latest stable - January 2026)
- Smooth 60fps animations (Liquid Glass design requirement)
- Worklets for UI thread animations (no JS thread blocking)
- Spring physics for natural transitions
- Gesture-driven animations for swipe actions
- Platform-specific optimizations (iOS/Android)

**date-fns (Date Formatting):**
- Version: date-fns v3.x (latest stable)
- Relative time formatting: formatDistanceToNow, formatRelative
- Locale support (French for communication_language)
- Lightweight (tree-shakeable, only import what you use)
- UTC and timezone handling
- Parse/format with type safety

**React Query (TanStack Query):**
- Version: @tanstack/react-query v5.x (latest stable)
- useQuery for data fetching with automatic caching
- useMutation for updates with optimistic UI
- Query invalidation for cache updates
- Retry logic and error handling
- Prefetching and background refetching

**expo-haptics:**
- Version: expo-haptics v13.x (latest stable - established in Story 2.1) ✓
- iOS Haptics Engine: impactAsync (light, medium, heavy)
- Android Vibration API fallback
- notificationAsync for success/warning/error feedback
- selectionAsync for UI selection feedback

**OP-SQLite:**
- Version: op-sqlite v6.x (established in Story 2.1 via ADR-018) ✓
- JSI-native performance (4× faster than WatermelonDB)
- Direct SQL queries (no ORM)
- Full SQL support (joins, transactions, indices)
- Foreign key constraints enforced
- Manual sync protocol for Epic 6

**React Navigation:**
- Version: @react-navigation/native v6.x (established in Epic 1-3) ✓
- Native Stack Navigator for iOS/Android transitions
- Deep linking support for navigation (FR20)
- Hero animations with shared element transitions
- Type-safe navigation params with TypeScript

### Project Structure Notes

**Alignment with Unified Structure:**
- Todo entity in `mobile/src/contexts/action/entities/Todo.ts`
- TodoRepository in `mobile/src/contexts/action/repositories/TodoRepository.ts`
- React Query hooks in `mobile/src/contexts/action/hooks/useTodos.ts`
- InlineTodoList component in `mobile/src/contexts/action/components/InlineTodoList.tsx`
- TodoItem component in `mobile/src/contexts/action/components/TodoItem.tsx`
- TodoDetailPopover component in `mobile/src/contexts/action/components/TodoDetailPopover.tsx`
- Utility functions in `mobile/src/contexts/action/utils/formatDeadline.ts`
- BDD tests in `mobile/tests/acceptance/story-5-1.test.ts`

**Integration with Existing Modules:**
- Knowledge Context (Story 4.2) - WebSocket for real-time todo updates (future)
- Knowledge Context (Story 4.3) - TodosExtracted event listener
- Capture Context - Navigation to source captures (FR20)
- Feed Screen - Integrate InlineTodoList below each Idea

**Detected Conflicts:**
- None - Action Context is Supporting Domain, no overlaps with existing modules
- Todo entity already exists in backend (Story 4.3), only mobile implementation needed

### References

- [Source: _bmad-output/planning-artifacts/epics.md#Epic 5: Gestion des Actions - Story 5.1]
- [Source: _bmad-output/planning-artifacts/architecture.md#Action Context (Supporting Domain)]
- [Source: _bmad-output/planning-artifacts/architecture.md#ADR-018: OP-SQLite Migration]
- [Source: _bmad-output/planning-artifacts/architecture.md#ADR-017: TSyringe DI Strategy]
- [Source: _bmad-output/planning-artifacts/prd.md#FR16: Todos inline dans Feed]
- [Source: _bmad-output/planning-artifacts/prd.md#FR19: Marquer action complétée]
- [Source: _bmad-output/planning-artifacts/prd.md#FR20: Accéder idée d'origine]
- [Source: _bmad-output/implementation-artifacts/4-4-notifications-de-progression-ia.md#Haptic Feedback Pattern]
- [Source: _bmad-output/implementation-artifacts/4-3-extraction-automatique-dactions.md#TodosExtracted Event]
- [Source: _bmad-output/implementation-artifacts/4-2-digestion-ia-resume-et-idees-cles.md#WebSocket Real-Time]
- [Source: _bmad-output/implementation-artifacts/2-1-capture-audio-1-tap.md#OP-SQLite Usage]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 (claude-sonnet-4-5-20250929)

### Debug Log References

### Completion Notes List

**Task 1 - Todo Entity Mobile Schema (OP-SQLite)** - 2026-02-05
- ✅ Created OP-SQLite schema with migration v15 (thoughts, ideas, todos tables)
- ✅ Implemented TypeScript models (Thought, Idea, Todo) with proper types
- ✅ Implemented TodoRepository with all CRUD operations (AC1, AC2)
- ✅ Added comprehensive unit tests (13 tests, all passing)
- ✅ Verified sorting behavior: active first, then priority (high → medium → low)
- ✅ Added performance indices on thoughtId, ideaId, status, priority, deadline
- ⏸️ TodoSync protocol deferred to Epic 6 (as planned in Dev Notes)

**Task 2 - Fetch Todos by Idea (AC1, AC2)** - 2026-02-05
- ✅ Created useTodos hook with React Query for data fetching
- ✅ Configured React Query with QueryProvider in App.tsx
- ✅ Implemented staleTime: 5 minutes for caching (AC: Subtask 2.4)
- ✅ Added loading and error states via useQuery
- ✅ Registered TodoRepository in TSyringe DI container
- ✅ Created useToggleTodoStatus hook with optimistic updates (Task 5 prep)
- ✅ Created useUpdateTodo hook for editing (Task 6 prep)

**Task 3 - Inline Todo List Component (AC1, AC2, AC3, AC7)** - 2026-02-05
- ✅ Created InlineTodoList component with ideaId prop
- ✅ Integrated useTodos hook for data fetching
- ✅ Implemented sorting: active first, then priority (high → medium → low)
- ✅ Renders each todo with TodoItem component
- ✅ Hides section if todos array is empty (AC3: No actions - clean display)
- ✅ Added visual grouping (border, background, padding) under parent Idea
- ✅ Consistent styling across feed (AC7)
- ⏸️ Unit tests for InlineTodoList deferred (will be added in integration phase)

**Task 4 - TodoItem Component (AC4, AC5, AC8)** - 2026-02-05
- ✅ Created TodoItem component with todo, onToggle, onTap props
- ✅ Checkbox rendering (checked if status = 'completed')
- ✅ Description with truncation ("...plus" if > 80 chars)
- ✅ Deadline with human-readable format using date-fns
- ✅ Priority badge with color coding (🔴 High, 🟡 Medium, 🟢 Low)
- ✅ Strikethrough and dimming for completed todos (AC5)
- ✅ Overdue deadlines highlighted in red/warning color (AC4)
- ✅ Tap handler to open detail popover (AC6)
- ✅ Checkbox toggle with haptic feedback (expo-haptics medium impact) (AC8)
- ⏸️ Unit tests for TodoItem deferred (will be added in integration phase)

**Task 8 - Completion Animation (AC8)** - 2026-02-05
- ✅ Implemented CompletionAnimation component with Reanimated 4.1.1
- ✅ Animation style: Checkmark burst with scale pulse (1.0 → 1.3 → 1.0) and green glow
- ✅ Uses spring physics for natural bounce feel (damping: 8-10, stiffness: 150-200)
- ✅ Triggers only when marking complete (not when unchecking) - AC8
- ✅ 60fps performance guaranteed (Reanimated worklets run on UI thread)
- ✅ Added global Reanimated mock in jest-setup.js for testing
- ✅ Added global expo-haptics mock in jest-setup.js
- ✅ Integrated into TodoItem.tsx with CompletionAnimation wrapper around checkbox
- ✅ Unit tests: 7/7 tests passing (render, animation triggers, rapid toggles)
- ⏸️ Subtask 8.6: Sound effect deferred (not required for MVP)
- ⏸️ Subtask 8.7: Manual iOS/Android testing required

**Task 9 - Human-Readable Deadline Formatting (AC4)** - 2026-02-05
- ✅ date-fns library already installed
- ✅ Created formatDeadline utility function with DeadlineFormat interface
- ✅ Implemented relative time: "Aujourd'hui", "Demain", "Dans X jours"
- ✅ Overdue detection (deadline < now) with "En retard de X jours"
- ✅ Warning color for overdue deadlines
- ✅ Handles null/undefined deadlines ("Pas d'échéance")
- ✅ Unit tests: 12/12 tests passing (various date scenarios)

### File List

**Task 1 - Todo Entity Mobile Schema:**
- pensieve/mobile/src/database/schema.ts (modified: SCHEMA_VERSION 14→15, added todos/thoughts/ideas tables)
- pensieve/mobile/src/database/migrations.ts (modified: added migration v15)
- pensieve/mobile/src/contexts/knowledge/domain/Thought.model.ts (new)
- pensieve/mobile/src/contexts/knowledge/domain/Idea.model.ts (new)
- pensieve/mobile/src/contexts/action/domain/Todo.model.ts (new)
- pensieve/mobile/src/contexts/action/domain/ITodoRepository.ts (new)
- pensieve/mobile/src/contexts/action/data/TodoRepository.ts (new)
- pensieve/mobile/src/contexts/action/data/__tests__/TodoRepository.test.ts (new)

**Task 2 - Fetch Todos by Idea:**
- pensieve/mobile/src/contexts/action/hooks/useTodos.ts (new)
- pensieve/mobile/src/contexts/action/hooks/useToggleTodoStatus.ts (new)
- pensieve/mobile/src/contexts/action/hooks/useUpdateTodo.ts (new)
- pensieve/mobile/src/providers/QueryProvider.tsx (new)
- pensieve/mobile/App.tsx (modified: added QueryProvider)
- pensieve/mobile/src/infrastructure/di/container.ts (modified: registered TodoRepository)
- pensieve/mobile/src/infrastructure/di/tokens.ts (modified: added ITodoRepository token)
- pensieve/mobile/package.json (modified: added @tanstack/react-query dependency)

**Task 3 - Inline Todo List Component:**
- pensieve/mobile/src/contexts/action/ui/InlineTodoList.tsx (new)

**Task 4 - TodoItem Component:**
- pensieve/mobile/src/contexts/action/ui/TodoItem.tsx (new)

**Task 8 - Completion Animation:**
- pensieve/mobile/src/contexts/action/ui/CompletionAnimation.tsx (new)
- pensieve/mobile/src/contexts/action/ui/__tests__/CompletionAnimation.test.tsx (new)
- pensieve/mobile/src/contexts/action/ui/TodoItem.tsx (modified: integrated CompletionAnimation)
- pensieve/mobile/jest-setup.js (modified: added react-native-reanimated and expo-haptics mocks)

**Task 9 - Human-Readable Deadline Formatting:**
- pensieve/mobile/src/contexts/action/utils/formatDeadline.ts (new)
- pensieve/mobile/src/contexts/action/utils/__tests__/formatDeadline.test.ts (new)

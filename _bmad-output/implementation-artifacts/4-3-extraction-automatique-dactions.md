# Story 4.3: Extraction Automatique d'Actions

Status: ready-for-dev

## Story

As a **user**,
I want **the AI to automatically detect and extract actionable tasks from my captures**,
So that **I don't miss important todos buried in my thoughts**.

## Acceptance Criteria

### AC1: Single LLM Call Integration with Story 4.2 (ADR-004 Compliance)
**Given** the AI is digesting a capture (during Story 4.2 processing)
**When** GPT-4o-mini analyzes the content
**Then** the prompt also requests detection of actionable items/todos
**And** for each action detected, the AI extracts: task description, suggested deadline (if mentioned), suggested priority (high/medium/low)
**And** the extraction happens in the same API call as summary/ideas (no additional latency)

**Critical Architecture Note:** ADR-004 specifies ONE SINGLE LLM call for summary + ideas + todos extraction. This story EXTENDS Story 4.2's existing OpenAIService, NOT a new separate call.

### AC2: Todo Entity Creation in Action Context
**Given** GPT-4o-mini detects actionable items in the content
**When** the response is processed
**Then** a Todo entity is created in the Action Context (Supporting Domain) for each action
**And** each Todo contains: description, deadline (optional), priority, status ("todo"), source Idea reference
**And** the Todo is associated with the originating Capture and Idea
**And** the Todo is persisted to the database

### AC3: Deadline Parsing and Smart Inference
**Given** an action has a deadline mentioned in the capture
**When** GPT extracts the action
**Then** the deadline is parsed and stored as a date (e.g., "tomorrow", "next week", "Friday" → actual date)
**And** if the deadline is ambiguous, the AI provides a best-guess with low confidence flag
**And** the user can edit the deadline later

**Given** an action has no explicit deadline mentioned
**When** GPT extracts the action
**Then** the deadline field is left null
**And** the UI shows "No deadline" or suggests adding one
**And** smart defaults may be applied based on priority (high priority → suggest this week)

### AC4: Priority Inference from Content
**Given** an action has priority indicators in the content
**When** GPT extracts the action
**Then** priority is inferred from keywords ("urgent", "important", "when I have time")
**And** the priority is set to high/medium/low accordingly
**And** if no priority indicators exist, default to "medium"

### AC5: Multiple Actions from Single Capture (1-to-Many Relationship)
**Given** multiple actions are detected in a single capture
**When** the digestion completes
**Then** all actions are extracted and created as separate Todo entities
**And** each Todo maintains its link to the source Idea and Capture
**And** the Todos are displayed inline with the Ideas in the feed (FR16)

### AC6: No Actions Graceful Handling
**Given** the capture contains no actionable items
**When** GPT analyzes the content
**Then** no Todo entities are created
**And** the digestion still completes successfully with summary and ideas only
**And** the feed shows the Thought without any action badges

### AC7: False Positive Correction
**Given** GPT incorrectly identifies a non-action as an action (false positive)
**When** the Todo is displayed to the user
**Then** the user can dismiss or delete the incorrect action
**And** feedback is optionally collected to improve future extraction (post-MVP)

### AC8: Todo Extraction Event Publishing
**Given** Todos are successfully extracted and created
**When** the digestion completes
**Then** a TodosExtracted domain event is published
**And** the Action Context can subscribe to this event
**And** the event contains: captureId, thoughtId, todoIds[], extractedAt timestamp

## Tasks / Subtasks

### Task 1: Extend Digestion Prompt for Todo Extraction (AC1)
- [ ] Subtask 1.1: Update system prompt in OpenAIService to include todo detection
- [ ] Subtask 1.2: Enhance DigestionResponseSchema to include todos array
- [ ] Subtask 1.3: Add todo format specification (description, deadline text, priority)
- [ ] Subtask 1.4: Test prompt with captures containing actions vs no actions
- [ ] Subtask 1.5: Document todo extraction guidelines in PROMPT_ENGINEERING.md
- [ ] Subtask 1.6: Add unit tests for new schema validation with todos

### Task 2: Todo Entity and Repository Setup (AC2)
- [ ] Subtask 2.1: Define Todo entity schema (PostgreSQL + TypeORM)
- [ ] Subtask 2.2: Create TypeORM migration for todos table
- [ ] Subtask 2.3: Add indices (captureId, thoughtId, userId, status, deadline, priority)
- [ ] Subtask 2.4: Create TodoRepository with CRUD operations
- [ ] Subtask 2.5: Implement transaction handling for atomic Thought + Ideas + Todos creation
- [ ] Subtask 2.6: Add cascade delete rules (when Thought deleted → Todos deleted)
- [ ] Subtask 2.7: Add unit tests for TodoRepository

### Task 3: Deadline Parsing Service (AC3)
- [ ] Subtask 3.1: Create DeadlineParserService with natural language date parsing
- [ ] Subtask 3.2: Support relative dates (today, tomorrow, next week, in 3 days)
- [ ] Subtask 3.3: Support absolute dates (Friday, Jan 15, 2026-01-20)
- [ ] Subtask 3.4: Handle ambiguous dates with confidence scoring
- [ ] Subtask 3.5: Return null for unparseable or missing deadlines
- [ ] Subtask 3.6: Add timezone handling (use user's timezone from context)
- [ ] Subtask 3.7: Add unit tests for various date formats and edge cases

### Task 4: Priority Inference Logic (AC4)
- [ ] Subtask 4.1: Implement keyword-based priority detection in GPT response
- [ ] Subtask 4.2: Define priority keywords mapping (urgent/ASAP → high, important → medium, etc.)
- [ ] Subtask 4.3: Default priority to "medium" if no indicators found
- [ ] Subtask 4.4: Add confidence scoring for priority inference
- [ ] Subtask 4.5: Add unit tests for priority inference scenarios

### Task 5: Todo Extraction and Creation Logic (AC2, AC5, AC6)
- [ ] Subtask 5.1: Parse GPT response todos array from DigestionResponse
- [ ] Subtask 5.2: Iterate over each todo and create Todo entities
- [ ] Subtask 5.3: Parse deadline text using DeadlineParserService
- [ ] Subtask 5.4: Link Todo to Thought, Capture, and potentially Idea (Many-to-One)
- [ ] Subtask 5.5: Set initial status to "todo" (ready for user action)
- [ ] Subtask 5.6: Handle zero todos gracefully (AC6) - no error, just skip creation
- [ ] Subtask 5.7: Add unit tests for todo creation logic
- [ ] Subtask 5.8: Add integration tests for full extraction flow

### Task 6: Domain Event Publishing (AC8)
- [ ] Subtask 6.1: Create TodosExtracted event class
- [ ] Subtask 6.2: Publish TodosExtracted event after todos are persisted
- [ ] Subtask 6.3: Include captureId, thoughtId, todoIds[], extractedAt in event payload
- [ ] Subtask 6.4: Add EventBus integration test for TodosExtracted event
- [ ] Subtask 6.5: Document event for future Action Context consumers

### Task 7: User Correction Endpoint (AC7)
- [ ] Subtask 7.1: Create DELETE /api/todos/:id endpoint (soft delete or hard delete)
- [ ] Subtask 7.2: Add user authorization check (user can only delete their own todos)
- [ ] Subtask 7.3: Optionally collect feedback reason (false positive, not relevant, etc.)
- [ ] Subtask 7.4: Add unit tests for delete endpoint
- [ ] Subtask 7.5: Document API endpoint in Swagger/OpenAPI

### Task 8: Integration with Story 4.2 Digestion Flow
- [ ] Subtask 8.1: Enhance DigestionJobConsumer to call TodoRepository after Thought creation
- [ ] Subtask 8.2: Update transaction scope to include Todos (Thought + Ideas + Todos atomic)
- [ ] Subtask 8.3: Update DigestionCompleted event to include todosCount
- [ ] Subtask 8.4: Add integration tests for full digestion flow with todos
- [ ] Subtask 8.5: Test edge case: capture with todos only (no ideas)
- [ ] Subtask 8.6: Test edge case: capture with ideas only (no todos)
- [ ] Subtask 8.7: Test error handling: todo creation fails → rollback entire transaction

### Task 9: Mobile Feed Display Preparation (Backend API)
- [ ] Subtask 9.1: Create GET /api/thoughts/:id/todos endpoint
- [ ] Subtask 9.2: Include todos array in Thought entity serialization
- [ ] Subtask 9.3: Add pagination for large todo lists (optional, future)
- [ ] Subtask 9.4: Document API endpoints for mobile consumption
- [ ] Subtask 9.5: Add unit tests for todos API endpoints

### Task 10: BDD Integration Tests
- [ ] Subtask 10.1: Write BDD acceptance tests for AC1-AC8 (jest-cucumber)
- [ ] Subtask 10.2: Create test fixtures (sample captures with actions, deadlines, priorities)
- [ ] Subtask 10.3: Test single action extraction (AC2)
- [ ] Subtask 10.4: Test multiple actions extraction (AC5)
- [ ] Subtask 10.5: Test no actions graceful handling (AC6)
- [ ] Subtask 10.6: Test deadline parsing (AC3) - various formats
- [ ] Subtask 10.7: Test priority inference (AC4) - all levels
- [ ] Subtask 10.8: Test TodosExtracted event publishing (AC8)
- [ ] Subtask 10.9: Test false positive correction (AC7)

## Dev Notes

### Architecture Context

**Bounded Context:** Action Context (Supporting Domain) + Knowledge Context (Core Domain)

**Integration Pattern:**
- Story 4.2 (Knowledge Context) extracts todos during digestion
- Story 4.3 (Action Context) creates and manages Todo entities
- Communication via Domain Event: TodosExtracted

**Related Contexts:**
- **Knowledge Context** (Core): Source of todo extraction (extends Story 4.2)
- **Capture Context** (Supporting): Source capture reference
- **Notification Context** (Generic): Will notify users of new todos (Story 4.4)
- **Opportunity Context** (Core): Separate flow for Ideas (not Todos)

**Existing Components from Story 4.2:**
- OpenAIService with GPT-4o-mini integration ✓
- DigestionJobConsumer (RabbitMQ worker) ✓
- DigestionResponse schema (Zod) ✓
- ContentExtractorService ✓
- Thought and Idea entities ✓
- WebSocket real-time updates ✓
- Domain Events via EventBus ✓

**Critical Architecture Decision (ADR-004):**
> **Single LLM Call Strategy:** All extraction (summary + ideas + todos) happens in ONE GPT-4o-mini API call. This story EXTENDS the existing prompt and response schema from Story 4.2, NOT a new service.

**Data Flow:**
```
[Story 4.2 Flow] RabbitMQ → DigestionJobConsumer → ContentExtractor → OpenAIService
                                                                           ↓
                                                    GPT-4o-mini API (summary + ideas + todos)
                                                                           ↓
                                        DigestionResponse { summary, ideas[], todos[] } ← EXTENDED SCHEMA
                                                                           ↓
                     Transaction { CREATE Thought + CREATE Ideas[] + CREATE Todos[] } ← NEW
                                                                           ↓
                                        PUBLISH DigestionCompleted + TodosExtracted ← NEW EVENT
                                                                           ↓
                                   WebSocket → Mobile App (real-time feed update)
```

**1-to-Many Relationship (CRITICAL):**
```
1 Capture (parent)
  ├── 1 Thought (summary)
  ├── 0-N Ideas (key insights)
  └── 0-N Todos (actionable tasks) ← NEW
```

**Example:**
```typescript
// Capture content:
// "Faut que je pense à envoyer la facture à Mme Micheaux avant vendredi.
//  J'ai encore croisé un freelance qui galère avec sa compta, c'est le 3ème ce mois-ci.
//  Acheter du lait en rentrant."

// GPT extraction:
{
  summary: "Reminder de facture urgente, observation récurrente sur pain point compta freelance, course à faire.",
  ideas: [
    "Pain point compta freelance (récurrence)",
    "Opportunité produit compta simplifiée"
  ],
  todos: [
    {
      description: "Envoyer facture Mme Micheaux",
      deadline: "vendredi",
      priority: "high"
    },
    {
      description: "Analyser solutions compta freelance (Pennylane, Indy)",
      deadline: null,
      priority: "medium"
    },
    {
      description: "Acheter lait",
      deadline: "aujourd'hui",
      priority: "low"
    }
  ]
}
```

### Technology Stack

**Backend:**
- **NestJS** (TypeScript) - Already configured ✓
- **PostgreSQL** - Persistence for Todo entities
- **TypeORM** - ORM with migration support
- **OpenAI SDK** - Already integrated in Story 4.2 ✓
  - Model: `gpt-4o-mini` (single call for all extraction)
  - Prompt extension: Add todo detection to existing system prompt
- **Zod** - Already used for DigestionResponse validation ✓
- **date-fns** or **chrono-node** - Natural language date parsing
- **TSyringe** - Dependency Injection (established pattern from ADR-017)

**Mobile (Future - Story 5.x):**
- **OP-SQLite** - Local storage for Todos (offline-first)
- **React Native** - Display todos inline in feed (FR16)
- **Tab Actions** - Centralized todo management (Story 5.2)

**Real-Time Communication:**
- **WebSockets** (Socket.io) - Already implemented in Story 4.2 ✓
- Extend KnowledgeEventsGateway to emit TodosExtracted events

### Critical Implementation Notes

**From Architecture Document:**

1. **ADR-004 - Single LLM Call (CRITICAL):**
   - Extend Story 4.2's OpenAIService prompt, DO NOT create new service
   - Add todos array to DigestionResponseSchema (Zod)
   - Parsing todos happens in same response as summary + ideas
   - Zero additional API calls = zero additional latency

2. **Domain-Driven Design:**
   - Todo is an Aggregate Root in Action Context (separate lifecycle from Thought)
   - Todo has Many-to-One relationship with Thought (thoughtId FK)
   - Todo optionally references an Idea (ideaId FK nullable)
   - Policy: Auto-Extract Todos WHEN ThoughtDigested

3. **Deadline Parsing Strategy:**
   - Use chrono-node library for robust natural language date parsing
   - Support relative dates: "tomorrow" → Date(today + 1 day)
   - Support absolute dates: "Friday" → next Friday Date
   - Support ISO dates: "2026-01-20" → Date(2026-01-20)
   - Handle timezone: use user's timezone from JWT token context
   - Return null if unparseable → user can add manually in UI

4. **Priority Inference Keywords:**
   - **High priority:** "urgent", "ASAP", "critical", "immediately", "deadline", "important and urgent"
   - **Medium priority:** "important", "should", "need to", "don't forget"
   - **Low priority:** "maybe", "when I have time", "someday", "nice to have"
   - **Default:** "medium" (if no keywords detected)

5. **Transaction Management:**
   - Extend Story 4.2's transaction to include Todos
   - Transaction scope: CREATE Thought + CREATE Ideas[] + CREATE Todos[]
   - If any creation fails → rollback entire transaction
   - Ensures data consistency (NFR6: zero data loss tolerance)

6. **Security & Isolation:**
   - NFR13: User data isolation - validate userId in Todo entity
   - Authorization: User can only CRUD their own todos
   - Sanitize todo descriptions (prevent XSS if displayed in web UI)

7. **Performance Requirements:**
   - No additional latency (ADR-004 compliance)
   - NFR3: Still < 30s total digestion time (no impact from todos)
   - Todos creation should be fast (indexed DB writes)

8. **Offline-First Integration:**
   - Todos sync to mobile via Epic 6 protocol (not in this story)
   - Mobile displays cached Todos even when offline
   - Feed shows todos inline with Ideas (Story 5.1 - mobile work)

### Domain Events

**Events Published by This Story:**

```typescript
// New event for Action Context
TodosExtracted {
  captureId: UUID
  thoughtId: UUID
  userId: UUID
  todoIds: UUID[]
  todosCount: number
  extractedAt: DateTime
}

// Enhanced event from Story 4.2
DigestionCompleted {
  thoughtId: UUID
  captureId: UUID
  userId: UUID
  summary: string
  ideasCount: number
  todosCount: number // NEW FIELD
  processingTimeMs: number
  completedAt: DateTime
}
```

**Events Consumed by This Story:**

```typescript
// From Story 4.2 - internal to DigestionJobConsumer
// No new external events consumed
```

### Entity Schemas

**Todo Entity (Action Context):**

```typescript
@Entity('todos')
export class Todo {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column('uuid')
  thoughtId: string; // Many-to-One with Thought

  @Column({ type: 'uuid', nullable: true })
  ideaId?: string; // Optional Many-to-One with Idea

  @Column('uuid')
  captureId: string; // For navigation back to source

  @Column('uuid')
  userId: string; // User isolation (NFR13)

  @Column('text')
  description: string;

  @Column({ type: 'enum', enum: ['todo', 'launched', 'in_progress', 'completed', 'abandoned'] })
  status: 'todo' | 'launched' | 'in_progress' | 'completed' | 'abandoned';

  @Column({ type: 'timestamp', nullable: true })
  deadline?: Date;

  @Column({ type: 'float', nullable: true })
  deadlineConfidence?: number; // 0-1, for ambiguous dates

  @Column({ type: 'enum', enum: ['low', 'medium', 'high'], default: 'medium' })
  priority: 'low' | 'medium' | 'high';

  @Column({ type: 'float', nullable: true })
  priorityConfidence?: number; // 0-1, for inferred priorities

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @Column({ type: 'timestamp', nullable: true })
  completedAt?: Date;

  // Relationships
  @ManyToOne(() => Thought, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'thoughtId' })
  thought: Thought;

  @ManyToOne(() => Idea, { nullable: true, onDelete: 'SET NULL' })
  @JoinColumn({ name: 'ideaId' })
  idea?: Idea;

  @ManyToOne(() => Capture, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'captureId' })
  capture: Capture;
}
```

**Migration Example:**

```typescript
// migrations/XXXXXX-CreateTodosTable.ts
export class CreateTodosTable1234567890 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'todos',
        columns: [
          { name: 'id', type: 'uuid', isPrimary: true, default: 'uuid_generate_v4()' },
          { name: 'thoughtId', type: 'uuid', isNullable: false },
          { name: 'ideaId', type: 'uuid', isNullable: true },
          { name: 'captureId', type: 'uuid', isNullable: false },
          { name: 'userId', type: 'uuid', isNullable: false },
          { name: 'description', type: 'text', isNullable: false },
          { name: 'status', type: 'varchar', default: "'todo'" },
          { name: 'deadline', type: 'timestamp', isNullable: true },
          { name: 'deadlineConfidence', type: 'float', isNullable: true },
          { name: 'priority', type: 'varchar', default: "'medium'" },
          { name: 'priorityConfidence', type: 'float', isNullable: true },
          { name: 'createdAt', type: 'timestamp', default: 'CURRENT_TIMESTAMP' },
          { name: 'updatedAt', type: 'timestamp', default: 'CURRENT_TIMESTAMP' },
          { name: 'completedAt', type: 'timestamp', isNullable: true },
        ],
        foreignKeys: [
          {
            columnNames: ['thoughtId'],
            referencedTableName: 'thoughts',
            referencedColumnNames: ['id'],
            onDelete: 'CASCADE',
          },
          {
            columnNames: ['ideaId'],
            referencedTableName: 'ideas',
            referencedColumnNames: ['id'],
            onDelete: 'SET NULL',
          },
        ],
        indices: [
          { columnNames: ['captureId'] },
          { columnNames: ['thoughtId'] },
          { columnNames: ['userId'] },
          { columnNames: ['status'] },
          { columnNames: ['deadline'] },
          { columnNames: ['priority'] },
          { columnNames: ['createdAt'] },
        ],
      }),
      true
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('todos');
  }
}
```

### Prompt Engineering Extension

**Enhanced System Prompt (extends Story 4.2):**

```
You are an AI assistant specialized in analyzing personal thoughts and ideas.
Your goal is to extract the essence of the user's thought, identify key insights,
and detect actionable tasks.

For each thought provided:
1. Generate a concise summary (2-3 sentences maximum) that captures the core message.
2. Extract key ideas as bullet points (1-5 ideas maximum, prioritize quality over quantity).
3. Detect actionable tasks/todos (0-10 maximum, be selective - only real actions).

Guidelines for todo extraction:
- A todo is an action the user needs to take (verbs: send, call, buy, finish, etc.)
- Extract deadline if mentioned (e.g., "by Friday", "tomorrow", "in 3 days")
- Infer priority from context:
  - HIGH: urgent, ASAP, critical, deadline-driven
  - MEDIUM: important, should do, need to
  - LOW: maybe, when I have time, nice to have
- If no clear action, do NOT force todo extraction - return empty array
- Preserve the user's voice in todo description

Be concise, precise, and actionable. Do not add information not present in the original thought.
```

**Enhanced User Prompt Template:**

```typescript
const userPrompt = `
Analyze the following ${contentType === 'text' ? 'text' : 'transcribed audio'} thought:

"""
${content}
"""

Provide:
1. A concise summary (2-3 sentences)
2. Key ideas (bullet points, 1-5 ideas)
3. Actionable tasks/todos (0-10 maximum, only if clear actions detected)

Response format (JSON):
{
  "summary": "string",
  "ideas": ["idea 1", "idea 2", ...],
  "todos": [
    {
      "description": "actionable task description",
      "deadline": "deadline text if mentioned (e.g., 'Friday', 'tomorrow', 'in 3 days') or null",
      "priority": "high" | "medium" | "low"
    }
  ],
  "confidence": "high" | "medium" | "low"
}
`;
```

**Enhanced Response Schema (Zod):**

```typescript
import { z } from 'zod';

const TodoSchema = z.object({
  description: z.string().min(3).max(200),
  deadline: z.string().max(50).nullable(),
  priority: z.enum(['low', 'medium', 'high']),
});

const DigestionResponseSchema = z.object({
  summary: z.string().min(10).max(500),
  ideas: z.array(z.string().min(5).max(200)).min(1).max(10),
  todos: z.array(TodoSchema).max(10), // NEW FIELD
  confidence: z.enum(['high', 'medium', 'low']).optional(),
});

export type TodoExtraction = z.infer<typeof TodoSchema>;
export type DigestionResponse = z.infer<typeof DigestionResponseSchema>;
```

### Natural Language Date Parsing Strategy

**Library Recommendation: chrono-node**

```typescript
import * as chrono from 'chrono-node';

export class DeadlineParserService {
  parse(deadlineText: string, userTimezone: string): { date: Date | null; confidence: number } {
    if (!deadlineText) {
      return { date: null, confidence: 1.0 }; // No deadline = high confidence null
    }

    const now = new Date();
    const results = chrono.parse(deadlineText, now, { timezone: userTimezone });

    if (results.length === 0) {
      return { date: null, confidence: 0.0 }; // Unparseable
    }

    const result = results[0];
    const parsedDate = result.date();
    const confidence = this.calculateConfidence(result);

    return { date: parsedDate, confidence };
  }

  private calculateConfidence(result: chrono.ParsedResult): number {
    // chrono provides confidence scores for ambiguous dates
    // Example: "Friday" = high confidence (next Friday)
    // Example: "next week" = medium confidence (which day?)
    // Example: "soon" = low confidence (unparseable)

    if (result.start.knownValues.day && result.start.knownValues.month) {
      return 1.0; // Specific date = high confidence
    } else if (result.start.knownValues.day) {
      return 0.8; // Day of week = medium-high confidence
    } else {
      return 0.5; // Vague date = medium confidence
    }
  }
}
```

**Date Parsing Examples:**

```typescript
// Input → Parsed Date (assuming today = 2026-02-04 Tuesday)
'today' → 2026-02-04
'tomorrow' → 2026-02-05
'Friday' → 2026-02-07 (next Friday)
'next week' → 2026-02-11 (start of next week, low confidence)
'in 3 days' → 2026-02-07
'Feb 10' → 2026-02-10
'2026-02-20' → 2026-02-20
'soon' → null (unparseable)
```

### Testing Strategy

**BDD Acceptance Tests (jest-cucumber):**

Create `story-4-3-extraction-automatique-dactions.feature`:

```gherkin
Fonctionnalité: Story 4.3 - Extraction Automatique d'Actions

  Scénario: Extraction unique action avec deadline et priorité
    Étant donné une capture texte "Faut que je pense à envoyer la facture à Mme Micheaux avant vendredi. C'est urgent."
    Quand le worker de digestion traite la capture
    Alors GPT-4o-mini extrait 1 todo dans le même appel que résumé + idées (ADR-004)
    Et le todo contient : description="Envoyer facture Mme Micheaux", deadline="vendredi", priority="high"
    Et une entité Todo est créée avec thoughtId, status="todo"
    Et le deadline est parsé en Date (prochain vendredi)
    Et l'événement TodosExtracted est publié avec 1 todoId

  Scénario: Extraction multiples actions d'une seule capture (1-to-Many)
    Étant donné une capture audio transcrite "Envoyer facture Mme Micheaux avant vendredi. Analyser Pennylane et Indy. Acheter lait."
    Quand le worker de digestion traite la capture
    Alors GPT-4o-mini extrait 3 todos distincts
    Et 3 entités Todo sont créées liées au même Thought
    Et chaque Todo a son propre description, deadline, priority
    Et l'événement TodosExtracted contient todoIds=[id1, id2, id3]

  Scénario: Capture sans action (AC6)
    Étant donné une capture texte "J'ai observé un pattern intéressant sur le marché des freelances."
    Quand le worker de digestion traite la capture
    Alors GPT-4o-mini retourne todos=[] (array vide)
    Et aucune entité Todo n'est créée
    Et la digestion se termine avec succès (summary + ideas uniquement)
    Et l'événement TodosExtracted contient todosCount=0

  Scénario: Parsing deadline relative (AC3)
    Étant donné un todo extrait avec deadline="tomorrow"
    Quand le DeadlineParserService parse le texte
    Alors le deadline est converti en Date (today + 1 jour)
    Et deadlineConfidence = 1.0 (high confidence)

  Scénario: Parsing deadline ambiguë (AC3)
    Étant donné un todo extrait avec deadline="next week"
    Quand le DeadlineParserService parse le texte
    Alors le deadline est converti en Date (début semaine prochaine)
    Et deadlineConfidence < 1.0 (medium confidence)
    Et l'utilisateur peut éditer le deadline plus tard

  Scénario: Pas de deadline mentionnée (AC3)
    Étant donné un todo extrait avec deadline=null
    Quand le todo est créé
    Alors le champ deadline est NULL dans la base de données
    Et l'UI affiche "No deadline"

  Scénario: Inférence priorité depuis keywords (AC4)
    Étant donné un todo avec description "Urgent : envoyer rapport ASAP"
    Quand GPT infère la priorité
    Alors priority = "high"
    Et priorityConfidence >= 0.8

  Scénario: Priorité par défaut si aucun indicateur (AC4)
    Étant donné un todo avec description "Regarder ce nouveau framework"
    Quand GPT infère la priorité
    Alors priority = "medium" (default)
    Et priorityConfidence <= 0.5

  Scénario: Transaction atomique Thought + Ideas + Todos (AC2)
    Étant donné que la digestion retourne summary, 2 ideas, 3 todos
    Quand le DigestionJobConsumer crée les entités
    Alors une transaction atomique crée : 1 Thought + 2 Ideas + 3 Todos
    Et si la création d'un Todo échoue, tout est rollback
    Et aucune entité n'est persistée partiellement

  Scénario: Correction false positive (AC7)
    Étant donné un todo incorrect créé par GPT
    Quand l'utilisateur appelle DELETE /api/todos/:id
    Alors le todo est supprimé de la base de données
    Et l'utilisateur peut optionnellement donner un feedback ("false positive")
    Et le feedback est loggé pour amélioration future

  Scénario: Événement TodosExtracted publié (AC8)
    Étant donné que 2 todos sont créés avec succès
    Quand la digestion se termine
    Alors l'événement TodosExtracted est publié sur EventBus
    Et l'événement contient : captureId, thoughtId, todoIds=[id1, id2], todosCount=2, extractedAt
    Et l'Action Context peut souscrire à cet événement
```

**Unit Tests:**

```typescript
describe('DeadlineParserService', () => {
  let service: DeadlineParserService;

  beforeEach(() => {
    service = new DeadlineParserService();
  });

  it('should parse "tomorrow" to Date(today + 1 day)', () => {
    const today = new Date('2026-02-04');
    const result = service.parse('tomorrow', 'America/New_York', today);
    expect(result.date).toEqual(new Date('2026-02-05'));
    expect(result.confidence).toBe(1.0);
  });

  it('should parse "Friday" to next Friday', () => {
    const today = new Date('2026-02-04'); // Tuesday
    const result = service.parse('Friday', 'America/New_York', today);
    expect(result.date).toEqual(new Date('2026-02-07'));
    expect(result.confidence).toBeGreaterThan(0.8);
  });

  it('should return null for unparseable deadline', () => {
    const result = service.parse('someday', 'America/New_York');
    expect(result.date).toBeNull();
    expect(result.confidence).toBe(0.0);
  });

  it('should handle null deadline text', () => {
    const result = service.parse(null, 'America/New_York');
    expect(result.date).toBeNull();
    expect(result.confidence).toBe(1.0); // High confidence null
  });
});

describe('TodoRepository', () => {
  let repository: TodoRepository;
  let dataSource: DataSource;

  beforeEach(async () => {
    dataSource = await createTestDataSource();
    repository = new TodoRepository(dataSource);
  });

  it('should create Todo with parsed deadline', async () => {
    const thoughtId = 'thought-123';
    const userId = 'user-456';
    const description = 'Send invoice to client';
    const deadline = new Date('2026-02-07');
    const priority = 'high';

    const todo = await repository.create({
      thoughtId,
      captureId: 'capture-789',
      userId,
      description,
      deadline,
      deadlineConfidence: 1.0,
      priority,
      priorityConfidence: 0.9,
      status: 'todo',
    });

    expect(todo.id).toBeDefined();
    expect(todo.description).toBe(description);
    expect(todo.deadline).toEqual(deadline);
    expect(todo.priority).toBe('high');
  });

  it('should create multiple Todos for same Thought', async () => {
    const thoughtId = 'thought-123';
    const todo1 = await repository.create({ thoughtId, description: 'Todo 1', ... });
    const todo2 = await repository.create({ thoughtId, description: 'Todo 2', ... });

    const todos = await repository.findByThoughtId(thoughtId);
    expect(todos).toHaveLength(2);
  });

  it('should cascade delete Todos when Thought is deleted', async () => {
    const thoughtId = 'thought-123';
    await repository.create({ thoughtId, description: 'Todo 1', ... });
    await thoughtRepository.delete(thoughtId); // Delete parent Thought

    const todos = await repository.findByThoughtId(thoughtId);
    expect(todos).toHaveLength(0); // Todos cascaded
  });
});
```

### File Structure

```
pensieve/
├── mobile/
│   └── src/
│       └── contexts/
│           └── action/
│               ├── entities/
│               │   └── Todo.ts                  # NEW - mobile entity
│               ├── services/
│               │   └── ActionService.ts         # NEW - fetch todos
│               └── hooks/
│                   └── useTodos.ts              # NEW - React Query hook
│
└── backend/
    └── src/
        └── modules/
            ├── knowledge/
            │   ├── application/
            │   │   ├── services/
            │   │   │   ├── openai.service.ts            # ENHANCE - add todos to prompt
            │   │   │   └── openai.service.spec.ts       # ENHANCE - test todos extraction
            │   │   └── consumers/
            │   │       └── digestion-job-consumer.ts    # ENHANCE - create todos
            │   ├── domain/
            │   │   ├── schemas/
            │   │   │   └── digestion-response.schema.ts # ENHANCE - add TodoSchema
            │   │   └── events/
            │   │       └── TodosExtracted.event.ts      # NEW
            │   └── infrastructure/
            │       ├── prompts/
            │       │   └── PROMPT_ENGINEERING.md        # ENHANCE - document todos
            │       └── websocket/
            │           └── knowledge-events.gateway.ts  # ENHANCE - emit TodosExtracted
            │
            └── action/                                   # NEW MODULE
                ├── domain/
                │   ├── entities/
                │   │   └── Todo.entity.ts                # NEW
                │   ├── events/
                │   │   └── TodosExtracted.event.ts       # Reference from Knowledge
                │   └── interfaces/
                │       ├── ITodoRepository.ts            # NEW
                │       └── IDeadlineParser.ts            # NEW
                ├── application/
                │   ├── repositories/
                │   │   ├── TodoRepository.ts             # NEW
                │   │   └── TodoRepository.spec.ts        # NEW
                │   ├── services/
                │   │   ├── DeadlineParserService.ts      # NEW
                │   │   ├── DeadlineParserService.spec.ts # NEW
                │   │   └── PriorityInferenceService.ts   # NEW (optional)
                │   └── controllers/
                │       ├── TodosController.ts            # NEW - API endpoints
                │       └── TodosController.spec.ts       # NEW
                ├── infrastructure/
                │   └── migrations/
                │       └── XXXXXX-CreateTodosTable.ts    # NEW
                └── action.module.ts                      # NEW
```

### Previous Story Intelligence

**From Story 4.2 (Digestion IA - Résumé et Idées Clés):**
- ✅ **Critical Integration Point:** OpenAIService already exists - EXTEND prompt, DON'T create new service
- ✅ **Pattern:** DigestionJobConsumer processes RabbitMQ jobs → calls OpenAIService → creates entities
- ✅ **Pattern:** Zod schema validation for GPT responses (DigestionResponseSchema)
- ✅ **Pattern:** ContentExtractorService extracts text/audio content from Captures
- ✅ **Pattern:** Transaction handling for atomic Thought + Ideas creation (extend to include Todos)
- ✅ **Pattern:** Domain Events published via EventBus (DigestionCompleted)
- ✅ **Pattern:** WebSocket real-time updates to mobile (KnowledgeEventsGateway)
- ✅ **Learning:** TypeORM migrations required for production (ADR from code review)
- ✅ **Learning:** CORS security critical for WebSocket (environment-based whitelist)
- ✅ **Learning:** Never log full content (PII compliance NFR12)
- ✅ **Learning:** Rate limit detection for OpenAI (429 handling with exponential backoff)
- ✅ **Learning:** Comprehensive BDD tests with jest-cucumber (40+ scenarios)
- ✅ **Tech Stack:** NestJS + TypeORM + OpenAI SDK + Zod + Socket.io + TSyringe DI

**From Story 4.1 (Queue Asynchrone):**
- ✅ **Pattern:** RabbitMQ queue management with retry logic (exponential backoff 5s → 15s → 45s)
- ✅ **Pattern:** ProgressTracker for real-time job status
- ✅ **Pattern:** Max 3 concurrent jobs (prefetch count) to prevent API rate limiting
- ✅ **Learning:** Job timeout = 60s (2x NFR3 target of 30s)

**From Epic 3 Stories:**
- ✅ **Pattern:** OP-SQLite for mobile local storage (ADR-018)
- ✅ **Pattern:** Real-time feed updates with smooth animations
- ✅ **Learning:** Performance monitoring utilities (measureAsync)

### Git Intelligence

**Recent Commits (Story 4.2):**
- commit `db70956`: Story 4.2 marked done after code review
- commit `ee0ea84`: OpenAI DI fix in submodule
- commit `2673fdb`: TypeScript fixes in submodule
- commit `f8a4baa`: Story 4.2 status updated to review
- commit `db6002e`: Task 10 BDD tests complete

**Patterns from Story 4.2:**
- OpenAIService with TSyringe DI (`@injectable()` decorator)
- Zod schema validation with detailed error messages
- TypeORM entities with proper indices and foreign keys
- TypeORM migrations for production deployment
- WebSocket gateway pattern with user-specific rooms
- Domain Events via EventBus (publish/subscribe)
- Comprehensive BDD tests (40+ scenarios)
- Unit tests with high coverage (90%+)

**Code Quality Standards:**
- TypeScript strict mode enforced
- No PII in logs (content preview max 50 chars)
- Environment-based CORS configuration
- Rate limit detection and handling
- Transaction handling for atomic operations
- Cascade delete rules properly configured

### Latest Tech Information

**chrono-node (Date Parsing):**
- Latest version: chrono-node v2.x (2026)
- Best library for natural language date parsing in Node.js
- Supports relative dates: "tomorrow", "next week", "in 3 days"
- Supports absolute dates: "Friday", "Jan 20", "2026-02-04"
- Timezone-aware parsing
- Confidence scores for ambiguous dates
- Alternative: date-fns (simpler but less powerful)

**TypeORM:**
- Version 0.3.x (latest stable)
- Migration system for production schema changes
- Transaction support: `queryRunner.startTransaction()`
- Cascade operations: `onDelete: 'CASCADE'`, `onDelete: 'SET NULL'`
- Proper indexing for foreign keys and query performance

**OpenAI SDK:**
- Version: `openai` v4.x (January 2026)
- GPT-4o-mini: Cost-effective, fast, 128k context window
- JSON mode available for structured outputs
- Rate limiting: ~500 RPM default tier
- Timeout handling: 30s recommended (NFR3), 60s max

**NestJS:**
- Version 10.x (latest stable)
- TSyringe DI pattern established (ADR-017)
- WebSocket Gateway with Socket.io
- EventBus for domain events

**Zod:**
- Version v3.x
- Runtime type validation + TypeScript inference
- Nested schema composition
- Custom error messages

### Project Structure Notes

**Alignment with Unified Structure:**
- Action Context is a NEW module under `src/modules/action/`
- Follows DDD organization (domain, application, infrastructure)
- Todo entity in `domain/entities/`
- TodoRepository in `application/repositories/`
- DeadlineParserService in `application/services/`
- TypeORM migration in `infrastructure/migrations/`

**Integration with Existing Modules:**
- Knowledge Context (Story 4.2) - Extends OpenAIService prompt and DigestionJobConsumer
- Capture Context - Todo references captureId for navigation
- Notification Context (Story 4.4) - Will consume TodosExtracted event

**Detected Conflicts:**
- None - Action Context is new, no overlaps with existing modules

### References

- [Source: _bmad-output/planning-artifacts/epics.md#Story 4.3: Extraction Automatique d'Actions]
- [Source: _bmad-output/planning-artifacts/architecture.md#Domain-Driven Design - Action Context]
- [Source: _bmad-output/planning-artifacts/architecture.md#ADR-004: Single LLM Call Strategy]
- [Source: _bmad-output/planning-artifacts/prd.md#FR14: Détection automatique actions]
- [Source: _bmad-output/planning-artifacts/prd.md#FR15: Extraction description/deadline/priorité]
- [Source: _bmad-output/implementation-artifacts/4-2-digestion-ia-resume-et-idees-cles.md#Complete Story 4.2 implementation]
- [Source: _bmad-output/implementation-artifacts/4-1-queue-asynchrone-pour-digestion-ia.md#RabbitMQ infrastructure]
- [Source: pensieve/backend/src/modules/knowledge/ (OpenAIService, DigestionJobConsumer)]
- [Source: chrono-node Documentation (Natural Language Date Parsing 2026)]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 (claude-sonnet-4-5-20250929)

### Debug Log References

### Completion Notes List

### File List

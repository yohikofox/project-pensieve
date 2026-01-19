---
stepsCompleted: [1, 2, 3, 4, 5]
inputDocuments:
  - _bmad-output/planning-artifacts/product-brief-pensine-2026-01-09.md
  - _bmad-output/planning-artifacts/prd.md
  - _bmad-output/planning-artifacts/ux-design-specification.md
  - docs/liquid-glass/site.md
workflowType: 'architecture'
project_name: 'pensine'
user_name: 'yohikofox'
date: '2026-01-12'
lastUpdated: '2026-01-13'
---

# Architecture Decision Document - Pensine

_Ce document se construit de manière collaborative à travers une découverte étape par étape. Les sections s'ajoutent au fur et à mesure que nous travaillons ensemble sur chaque décision architecturale._

**Author:** yohikofox
**Date:** 2026-01-12
**Project:** Pensine - Incubateur Personnel d'Idées Business

---

## Project Context Analysis

### Requirements Overview

**Functional Requirements (31 FRs identifiées):**

Le système doit supporter un workflow complet de gestion d'idées :

**Capture Multi-Modale (FR1-FR5):**
- Audio 1-tap depuis écran principal
- Texte rapide pour contextes publics
- Capture offline complète avec stockage en attente de sync
- Annulation possible en cours d'enregistrement

**Transcription Locale (FR6-FR8):**
- Traitement automatique post-capture via Whisper on-device
- Exécution 100% locale (pas de dépendance réseau)
- Consultation de la transcription complète par l'utilisateur

**Digestion IA Enrichie (FR9-FR13):**
- Génération automatique de résumés concis
- Extraction d'idées clés
- Support texte et audio
- Notifications de progression pour processus longs

**Todo-List & Actions (FR14-FR20):**
- Détection automatique d'actions lors de la digestion
- Extraction de : description, deadline suggérée, priorité
- Affichage inline dans le Feed par idée
- Tab Actions centralisé avec filtres (Toutes/À faire/Faites)
- Marquage completion + navigation vers idée d'origine

**Consultation (FR21-FR24):**
- Liste chronologique des captures
- Vue détail riche (audio, transcription, résumé, idées, todos)
- Consultation offline complète
- Distinction visuelle des captures en attente de digestion

**Synchronisation (FR29-FR31):**
- Sync automatique au retour réseau
- Bidirectionnelle (local ↔ cloud)
- Informations de statut pour l'utilisateur

**Gestion Compte (FR25-FR28):**
- Création compte, login/logout
- Récupération mot de passe oublié

**Non-Functional Requirements (16 NFRs identifiées):**

**Performance (NFR1-NFR5):**
- Capture audio : < 500ms après tap
- Transcription Whisper : < 2x durée audio
- Digestion IA : < 30s pour capture standard
- Chargement liste : < 1s (cache local)
- Latence perçue : feedback visuel obligatoire, jamais d'attente

**Reliability (NFR6-NFR9):**
- **Critique** : 0 capture perdue, jamais (tolérance zéro)
- Disponibilité capture offline : 100%
- Récupération après crash automatique
- Sync automatique sans intervention utilisateur

**Security (NFR10-NFR14):**
- Authentification obligatoire
- HTTPS/TLS pour toutes communications API
- Chiffrement au repos (device + cloud)
- Isolation stricte des données utilisateur
- Conformité RGPD (accès, rectification, suppression)

**Scalability (NFR15-NFR16):**
- Architecture prête pour 100+ utilisateurs sans refonte
- Pas de limite artificielle de stockage MVP

**UX-Driven Architecture Requirements:**

L'UX Spec révèle des contraintes architecturales fortes :

- **"1-Tap Liberation"** : Latence capture < 1s impose architecture réactive
- **Offline-first radical** : Toutes les features core doivent fonctionner sans réseau
- **Liquid Glass Design System** : Animations fluides, feedback haptique, transitions complexes
- **Détection concordance à chaud** : Traitement temps réel post-capture
- **Notifications intelligentes** : Queue de progression pour process IA longs
- **Métaphore "Jardin d'idées"** : Interface contemplative avec visualisation de maturité

### Scale & Complexity

**Project Scale Assessment:**

- **Primary domain:** Mobile-first Full-Stack SaaS (React Native + Backend)
- **Complexity level:** Medium
  - MVP focalisé mais techniquement riche
  - ML on-device (Whisper)
  - Architecture distribuée mobile-cloud
  - Offline-first avec sync bidirectionnelle
  - Contraintes performance strictes

- **Estimated architectural components:** 8-12 composants majeurs
  - Mobile app (React Native)
  - Transcription engine (Whisper on-device)
  - Local storage layer (SQLite/WatermelonDB)
  - Sync engine (bidirectional)
  - Backend API
  - AI digestion service
  - Authentication service
  - Todo extraction service
  - Concordance detection engine
  - Queue management (offline tasks)
  - Notification service
  - Storage management (audio retention)

**Complexity Indicators:**

- ✅ **Real-time features:** Détection concordance à chaud post-capture
- ⏳ **Multi-tenancy:** Mono-utilisateur MVP, prévoir abstraction "owner" pour future évolution
- ❌ **Regulatory compliance:** RGPD requis, pas d'autres contraintes réglementaires
- ⚠️ **Integration complexity:** ML on-device (Whisper), AI cloud (GPT), potentiellement OAuth tiers
- ✅ **User interaction complexity:** Haute - capture gestuelle, animations fluides, feedback haptique
- ⚠️ **Data complexity:** Medium - Audio brut + transcriptions + métadonnées + relations (concordances, todos)

### Technical Constraints & Dependencies

**Mandatory Technical Constraints:**

1. **Offline-First Radical**
   - 100% des features core fonctionnent sans réseau
   - Transcription locale obligatoire (Whisper on-device)
   - Cache local complet pour consultation
   - Queue persistante pour sync et digestion différée

2. **Confidentialité Audio**
   - Aucune donnée audio ne transite par des tiers non contrôlés
   - Transcription exclusivement on-device
   - Chiffrement obligatoire (transit + repos)

3. **Performance Mobile**
   - Capture < 500ms
   - Transcription < 2x durée audio (Whisper optimisé)
   - App installée < 100 Mo (sans modèle)
   - Modèle Whisper ~500 Mo (téléchargé post-install)

4. **Cross-Platform Mobile**
   - iOS 15+ et Android 10+ (API 29)
   - React Native pour développement partagé
   - APIs natives pour permissions (micro, storage, notifications)

5. **Fiabilité Absolue**
   - Aucune perte de données, tolérance zéro
   - Auto-save permanent
   - Récupération après crash

**Known Dependencies:**

- **Whisper (OpenAI)** : Modèle ML pour transcription locale
- **React Native** : Framework mobile cross-platform
- **SQLite / WatermelonDB** : Base locale offline-first
- **Backend à définir** : Node.js/Fastify pressenti (du brainstorming)
- **LLM Cloud** : GPT-4o-mini pressenti pour digestion IA
- **PostgreSQL** : Base backend pressenti (du brainstorming)

### Cross-Cutting Concerns Identified

**1. Offline-First Architecture**
- Impact : Tous les composants
- Décision requise : Pattern de sync, résolution conflits, queue management

**2. ML On-Device (Whisper)**
- Impact : Performance mobile, taille app, UX transcription
- Décision requise : Optimisation modèle, fallback si échec, gestion mémoire

**3. Security & Privacy**
- Impact : Tous les composants manipulant données sensibles
- Décision requise : Chiffrement, isolation, conformité RGPD

**4. Sync & Conflict Resolution**
- Impact : Mobile ↔ Backend
- Décision requise : Stratégie sync (optimistic/pessimistic), résolution conflits

**5. Queue Management**
- Impact : Digestion IA, Sync, Transcription retry
- Décision requise : Persistence queue, retry logic, prioritization

**6. Performance & Latency**
- Impact : UX capture, transcription, consultation
- Décision requise : Optimisations spécifiques, caching strategy, lazy loading

**7. Storage Management**
- Impact : Audio local, transcriptions, cache
- Décision requise : Rétention audio, cleanup automatique, quotas utilisateur

**8. Notification System**
- Impact : Progression IA, concordances, engagement
- Décision requise : Push vs local, timing, opt-in/out

---

## Architectural Style & Strategic Design

### Decision: Domain-Driven Design (Stratégique + Tactique)

**Rationale:**
- Domaine métier complexe avec logique riche (concordance, germination, extraction IA)
- Deux domaines parallèles identifiés : GTD/Action vs Opportunity/Germination
- Besoin de séparation claire entre contextes pour éviter couplage
- Architecture préparée pour Event Sourcing futur (pas MVP)
- CQRS non nécessaire pour MVP (peut être ajouté si divergence lecture/écriture)

**Conséquences:**
- Communication entre Bounded Contexts via Domain Events (asynchrone)
- Ubiquitous Language distinct par contexte
- Aggregates avec frontières transactionnelles claires
- Event Storming utilisé pour identifier les contextes

---

## Technology Stack Decisions

### Mobile Stack

**Framework:** React Native avec Expo (custom dev client)
- **Rationale:** Cross-platform iOS/Android, expérience utilisateur existante
- **Custom dev client:** Nécessaire pour module natif Whisper

**Local Database:** WatermelonDB
- **Rationale:** Offline-first avec sync protocol built-in, observation réactive
- **Alternative rejetée:** Realm (sync abandonné), SQLite seul (pas de sync intégré)

**Transcription:** Whisper.rn ou module custom
- **Rationale:** Transcription 100% locale (confidentialité), pas de dépendance réseau
- **Contrainte:** Modèle ~500 Mo téléchargé post-install

**Language:** TypeScript strict

---

### Backend Stack

**Framework:** NestJS (TypeScript)
- **Rationale:** Architecture modulaire, connaissance utilisateur, adapté DDD
- **Alternative rejetée:** Fastify (moins d'expérience, complexité DDD à gérer)

**Database:** PostgreSQL
- **Rationale:** Persistence données relationnelles, ACID, maturité

**Message Broker:** RabbitMQ
- **Rationale décisive:**
  - Élimination Redis SPOF (isolation pannes entre queue et cache)
  - Durabilité disk-based > Redis memory-only
  - Expérience utilisateur existante
  - Plugin delayed message exchange fonctionne bien
- **Alternative rejetée:** BullMQ (dépendance Redis = SPOF)

**Cache:** Redis (cache UNIQUEMENT, pas de queue)
- **Rationale:** Cache session, cache application, séparé de la queue

**Scheduler:** NestJS Schedule (cron jobs)
- **Rationale:** Publie vers RabbitMQ pour traitement asynchrone

**LLM:** GPT-4o-mini (pressenti)
- **Rationale:** Digestion IA (summary, tags, extraction todos/ideas)

---

### Infrastructure

**Dev/Staging:** Docker Compose selfhosted (homelab)

**Production future:** Scaleway ou AWS managed services
- **Rationale:** Migration quand customers arrivent, coût optimisé

---

## Domain-Driven Design - Strategic Design

### Event Storming - Découverte du Domaine

**Approche:** Event Storming Option A (full event storming avant bounded contexts)

**Révélations Majeures:**

#### 1. Deux Domaines Métier Parallèles (CRITIQUE)

**Flow A : GTD / Action (court, opérationnel)**
- Capture → Transcription → Digestion → **Todos extraites**
- Workflow: `EXTRACTED` → `LAUNCHED` → `IN_PROGRESS` → `COMPLETED`
- Métrique: Task completed
- Ubiquitous Language: "Tâche", "Action", "Rappel", "À faire"
- Exemple: "Pense à envoyer facture Mme Micheaux"

**Flow B : Opportunity Incubation (long, stratégique)**
- Capture → Transcription → Digestion → **Ideas extraites**
- Workflow: `Idea` → `Concordance` → `Pattern` → `Germination` → `Crystallization`
- Métrique: Idea launched
- Ubiquitous Language: "Idée", "Opportunité", "Pattern", "Germination", "Concordance", "Project"
- Exemple: "Pain point compta freelance (3ème mention)"

**❌ Erreur évitée:** Les todos ne se transforment JAMAIS en idées germées. Ce sont deux domaines séparés.

---

#### 2. Cardinalité 1-to-Many (CRITIQUE)

**Une SEULE capture** (audio 30s à plusieurs minutes) peut contenir:
- Plusieurs todos (0-N actions opérationnelles)
- Plusieurs idées (0-N insights stratégiques)
- Du contexte mélangé

**Exemple concret:**
```
Capture audio (2 min):
"Faut que je pense à envoyer la facture à Mme Micheaux avant vendredi.
J'ai encore croisé un freelance qui galère avec sa compta, c'est le 3ème ce mois-ci.
Il y a clairement un truc à faire là-dessus.
Faudrait que je creuse Pennylane et Indy.
Ah et acheter du lait en rentrant."

→ Extraction:
  Todos (3):
    - Envoyer facture Mme Micheaux (deadline: vendredi)
    - Analyser Pennylane et Indy
    - Acheter lait

  Ideas (2):
    - Pain point compta freelance (récurrence)
    - App compta simplifiée indépendants
```

**Hiérarchie:**
```
1 Capture (parent)
  ├── 0-N Todos (children)
  ├── 0-N Ideas (children)
  ├── 1 Summary
  └── 0-N Tags
```

---

#### 3. Concept de "Project" (MVP Core)

**Définition:** Matérialisation d'une concordance entre plusieurs Ideas.

**Deux chemins de création:**

**A. Bottom-Up (Émergent) - MVP**
```
1. User fait plusieurs Captures (orphelines)
2. Digestion extrait Ideas
3. Concordance Engine détecte pattern
4. ProjectSuggested event
5. User accepte → ProjectAccepted
6. Project créé, Ideas liées automatiquement
```

**B. Top-Down (Intentionnel) - Post-MVP**
```
User crée intentionnellement un Project vide
User fait des Captures dans ce Project
Enrichissement intentionnel du contexte
```

**Relations:**
- `Project ↔ Ideas`: Many-to-Many
- `Project ↔ Captures`: Many-to-Many
- Une Idea peut appartenir à plusieurs Projects
- Une Capture peut être orpheline ou liée à un/plusieurs Projects

---

#### 4. Capture Polymorphe (Strategy Pattern)

**Types de Capture:**
- Audio (Whisper transcription)
- Texte (déjà normalisé)
- Image (OCR ou Vision LLM)
- URL (Web scraper)

**Décision:** Un seul Aggregate `Capture` polymorphe avec `type` discriminant.

**Rationale:**
- Même cycle de vie métier : "capturer une pensée pour la digérer"
- Différence = transformation technique (Strategy Pattern)
- Ubiquitous Language : on parle de "Capture", pas de 4 concepts

**Conséquence:** Le "Transcription Context" devient "Normalization Context" (plus générique).

---

### Bounded Contexts Identifiés

#### 🔥 Core Domain (Différenciateurs Business)

**1. Knowledge Context**
- **Responsabilité:** Digestion IA et extraction de sens
- **Ubiquitous Language:** Thought, Digestion, Summary, Tags, Extraction
- **Aggregates:** `Thought`
- **Valeur métier:** Transformer le flux de pensée brut en insights structurés

**2. Opportunity Context**
- **Responsabilité:** Détection patterns, concordance, germination d'opportunités business
- **Ubiquitous Language:** Idea, Project, Concordance, Pattern, Germination, Maturity, BusinessCase
- **Aggregates:** `Idea`, `Project`
- **Valeur métier:** Incubateur personnel d'idées business (le cœur de Pensine)

---

#### 🔧 Supporting Domain (Nécessaires mais non différenciateurs)

**3. Capture Context**
- **Responsabilité:** Capture multi-modale (audio, texte, image, URL)
- **Ubiquitous Language:** Capture, Recording, Snapshot
- **Aggregates:** `Capture`

**4. Normalization Context**
- **Responsabilité:** Normalisation des captures en texte exploitable
- **Ubiquitous Language:** Transcription, OCR, TextExtraction, WebScraping
- **Aggregates:** Aucun (Domain Services stateless)

**5. Action Context**
- **Responsabilité:** Gestion cycle de vie des actions (GTD)
- **Ubiquitous Language:** Todo, Task, Action, Deadline, Priority, Completion
- **Aggregates:** `Todo`

---

#### ⚙️ Generic Subdomain (Génériques)

**6. Identity Context**
- **Responsabilité:** Authentification, gestion utilisateurs
- **Aggregates:** `User`

**7. Notification Context**
- **Responsabilité:** Notifications push et alertes
- **Aggregates:** `Notification`

---

#### 🏗️ Infrastructure (Non-métier)

**8. Sync (Infrastructure)**
- **Responsabilité:** Synchronisation offline-first
- **Implémentation:** WatermelonDB + backend sync endpoint
- **Note:** Pas de Bounded Context métier, infrastructure pure

---

### Aggregates & Domain Model

#### Core Aggregates

**1. Capture (Supporting - Capture Context)**
```typescript
Capture {
  id: UUID
  type: 'audio' | 'text' | 'image' | 'url'
  state: 'captured' | 'processing' | 'ready' | 'failed'
  projectId?: UUID  // Peut être null (orpheline)
  rawContent: AudioFile | string | ImageFile | URL
  normalizedText?: string
  capturedAt: DateTime
  location?: GeoPoint
  tags?: string[]
}
```

**2. Thought (Core - Knowledge Context)**
```typescript
Thought {
  id: UUID
  captureId: UUID
  summary: string
  tags: string[]
  extractedTodoIds: UUID[]
  extractedIdeaIds: UUID[]
  digestedAt: DateTime
}
```

**3. Todo (Supporting - Action Context)**
```typescript
Todo {
  id: UUID
  thoughtId: UUID
  state: 'extracted' | 'launched' | 'in_progress' | 'completed' | 'abandoned'
  description: string
  deadline?: DateTime
  priority?: 'low' | 'medium' | 'high'
}
```

**4. Idea (Core - Opportunity Context)**
```typescript
Idea {
  id: UUID
  thoughtId: UUID
  projectIds: UUID[]  // Many-to-Many
  state: 'extracted' | 'warm' | 'germinated'
  description: string
  context?: string
  concordanceScore?: number
}
```

**5. Project (Core - Opportunity Context)**
```typescript
Project {
  id: UUID
  name: string  // Auto-généré par IA
  origin: 'suggested'  // MVP = toujours suggested (top-down post-MVP)
  state: 'suggested' | 'accepted' | 'germinating' | 'crystallized' | 'rejected'

  // Relations Many-to-Many
  ideaIds: UUID[]
  captureIds: UUID[]

  // Germination
  maturityScore: number
  businessCase?: BusinessCase

  createdAt: DateTime
}
```

---

### Context Map (Relations)

```
┌─────────────────────────────────────────────────────────────┐
│                    CORE DOMAIN                               │
│                                                              │
│  ┌──────────────────┐         ┌─────────────────────────┐   │
│  │   Knowledge      │         │    Opportunity          │   │
│  │   Context        │◄────────│    Context              │   │
│  │                  │         │                         │   │
│  │ - Thought        │         │ - Idea                  │   │
│  │ - Digestion      │         │ - Project               │   │
│  └──────────────────┘         │ - Concordance           │   │
│         ▲                     └─────────────────────────┘   │
│         │                              ▲                    │
└─────────┼──────────────────────────────┼─────────────────────┘
          │                              │
          │ ThoughtDigested              │ IdeasExtracted
          │                              │
┌─────────┼──────────────────────────────┼─────────────────────┐
│         │        SUPPORTING            │                     │
│         │                              │                     │
│  ┌──────┴───────┐      ┌──────────────┴────┐                │
│  │   Capture    │      │     Action         │                │
│  │   Context    │      │     Context        │                │
│  │              │      │                    │                │
│  │ - Capture    │      │ - Todo             │                │
│  └──────┬───────┘      └────────────────────┘                │
│         │                                                    │
│         │ CaptureReady                                       │
│         ▼                                                    │
│  ┌──────────────┐                                            │
│  │Normalization │                                            │
│  │  Context     │                                            │
│  │  (Services)  │                                            │
│  └──────────────┘                                            │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                    GENERIC                                  │
│  ┌──────────────┐         ┌──────────────────┐             │
│  │  Identity    │         │  Notification    │             │
│  │  Context     │         │  Context         │             │
│  └──────────────┘         └──────────────────┘             │
└────────────────────────────────────────────────────────────┘
```

**Upstream/Downstream Relations:**

| Upstream (U) | Downstream (D) | Communication |
|--------------|----------------|---------------|
| Capture | Knowledge | `CaptureReady` event |
| Knowledge | Action | `TodosExtracted` event |
| Knowledge | Opportunity | `IdeasExtracted` event |
| Opportunity | Capture | `CaptureLinkedToProject` (user capture dans project) |
| Opportunity | Notification | `ProjectSuggested` → notification |
| Knowledge | Notification | `ThoughtDigested` → notification |

**Anti-Corruption Layers (ACL):**
- **Non nécessaires pour MVP** (contextes internes, ubiquitous language clair)
- **À prévoir pour intégrations externes futures** (Notion, Trello, API publique)

---

### Domain Events (MVP)

**Capture Phase:**
- `ThoughtCaptured`
- `NormalizationStarted`
- `CaptureReady`
- `NormalizationFailed`

**Digestion Phase:**
- `DigestionRequested`
- `DigestionStarted`
- `ThoughtDigested`
- `TodosExtracted` (0-N todos)
- `IdeasExtracted` (0-N ideas)
- `DigestionFailed`

**Project Phase (Concordance):**
- `ConcordanceDetectionRequested`
- `ConcordanceDetected`
- `PatternRecognized`
- `ProjectSuggested`
- `ProjectAccepted`
- `ProjectRejected`
- `IdeaAddedToProject`
- `CaptureLinkedToProject`

**Action Phase (Todo Lifecycle):**
- `TodoCreated`
- `TodoLaunched`
- `TodoStarted`
- `TodoCompleted`
- `TodoAbandoned`
- `TodoPostponed`
- `TodoPriorityChanged`

**Sync Phase (Infrastructure):**
- `LocalChangeDetected`
- `SyncRequested`
- `ChangesPushed`
- `ChangesPulled`
- `ConflictDetected`
- `ConflictResolved`
- `SyncCompleted`

**User/Session Phase:**
- `UserRegistered`
- `UserLoggedIn`
- `UserLoggedOut`
- `SessionExpired`

**Notification Phase:**
- `NotificationScheduled`
- `NotificationSent`
- `NotificationDelivered`
- `NotificationFailed`

---

### Policies (Réactions Automatiques)

**Policy 1: Auto-Normalization**
```
WHEN ThoughtCaptured
THEN StartNormalization
```

**Policy 2: Auto-Digestion**
```
WHEN CaptureReady
THEN RequestDigestion
```

**Policy 3: Auto-Extract Todos**
```
WHEN ThoughtDigested
THEN CreateTodos (pour chaque todo dans digest.todos[])
```

**Policy 4: Auto-Extract Ideas**
```
WHEN ThoughtDigested
THEN CreateIdeas (pour chaque idea dans digest.ideas[])
```

**Policy 5: Auto-Detect Concordance**
```
WHEN IdeasExtracted
THEN DetectConcordance
(peut être async/batched, pas immédiat)
```

**Policy 6: Auto-Suggest Project**
```
WHEN ConcordanceDetected
AND concordanceScore > threshold
THEN SuggestProject
```

**Policy 7: Auto-Link Ideas to Project**
```
WHEN ProjectAccepted
THEN AddIdeaToProject (pour chaque idea dans concordance)
```

**Policy 8: Auto-Sync on Network Return**
```
WHEN NetworkAvailable
AND hasPendingChanges
THEN RequestSync
```

**Policy 9: Notify Digestion Complete**
```
WHEN ThoughtDigested
THEN ScheduleNotification("Votre capture a été digérée")
```

**Policy 10: Notify Project Suggested**
```
WHEN ProjectSuggested
THEN ScheduleNotification("Nouveau pattern détecté : {projectName}")
```

---

## Architectural Decisions Record

### ADR-001: Aggregate Granularity - Aggregates Séparés

**Status:** ✅ ACCEPTÉ

**Context:** Décider si les phases (Capture, Thought, Idea, Project) sont un seul aggregate ou plusieurs.

**Decision:** Aggregates séparés avec communication via Domain Events.

**Rationale:**
- `Capture` → cycle de vie distinct (normalisation peut échouer indépendamment)
- `Thought` → cycle de vie distinct (digestion complète même si rien n'est extrait)
- `Idea` → cycle de vie long (concordance, germination sur plusieurs semaines/mois)
- `Project` → cycle de vie long et distinct (agrège plusieurs Ideas)
- Frontières transactionnelles naturelles

**Conséquences:**
- Loose coupling entre phases
- Communication asynchrone via Domain Events
- Résilience : échec d'une phase n'impacte pas les autres

---

### ADR-002: Normalization - Domain Service (pas Aggregate)

**Status:** ✅ ACCEPTÉ

**Context:** La normalisation (Whisper, OCR, web scraper) doit-elle être un Aggregate ou un Service ?

**Decision:** Domain Services stateless, pas d'Aggregate `Normalization`.

**Rationale:**
- Transformation technique transitoire
- État métier pertinent reste dans `Capture.state`
- Traçabilité via logs applicatifs suffisante pour MVP
- Pas de business logic complexe dans la normalisation elle-même

**Conséquences:**
- Simplicité : pas d'aggregate supplémentaire
- Pas de double état (Capture + Normalization)
- Si besoin analytics plus tard → ajouter `NormalizationAttempt` comme Value Object

---

### ADR-003: Sync - Infrastructure (pas Bounded Context)

**Status:** ✅ ACCEPTÉ

**Context:** Le sync offline-first est-il un contexte métier ou de l'infrastructure ?

**Decision:** Infrastructure pure, pas de Bounded Context métier.

**Rationale:**
- WatermelonDB gère déjà le sync protocol
- Pattern générique offline-first (pas spécifique à Pensine)
- Résolution conflits simple (last-write-wins ou timestamp)
- Pas de logique métier Pensine dans le sync

**Conséquences:**
- Sync = infrastructure technique
- Domain Events doivent être syncables (transports par le sync)
- Pas de SyncContext dans le modèle métier

---

### ADR-004: Digestion IA - Un seul appel LLM

**Status:** ✅ ACCEPTÉ

**Context:** L'extraction (summary, tags, todos, ideas) doit-elle se faire en un ou plusieurs appels LLM ?

**Decision:** Un seul appel LLM structuré.

```typescript
const digest = await llm.digest(transcription, {
  extract: ['summary', 'tags', 'todos', 'ideas']
});
```

**Rationale:**
- Coût LLM réduit (1 appel vs 3)
- Latence réduite (pas d'attente séquentielle)
- LLM voit contexte global pour extraction cohérente
- Prompt engineering moderne permet multi-extraction structurée

**Conséquences:**
- Prompt complexe (mais gérable avec structured output)
- Si LLM échoue → tout échoue (mais retry possible)
- Frugalité : coût optimisé pour MVP

---

### ADR-005: Capture Vide - Stockée

**Status:** ✅ ACCEPTÉ

**Context:** Que faire d'une Capture sans todo ni idea extraite ? (ex: "Il fait beau")

**Decision:** Stockée quand même, user décide de la conserver ou supprimer.

**Rationale:**
- Principe NFR6 : "0 capture perdue, jamais"
- Contexte futur potentiel (pensée spontanée peut devenir pertinente plus tard)
- Stockage texte minime (quelques octets)
- User autonomie : UI peut filtrer "Avec actions/idées uniquement"

**Conséquences:**
- Historique complet préservé
- Capture reste transcrite et résumée (avec tags potentiels)
- UI doit permettre filtrage pour ne pas polluer le feed

---

### ADR-006: Association Manuelle Capture ↔ Todo - Post-MVP

**Status:** ✅ ACCEPTÉ (Post-MVP)

**Context:** User peut-il lier manuellement une Capture à une Todo existante après coup ?

**Decision:** Pas MVP, mais architecture prévue via tags communs.

**Use case post-MVP:**
- Todo extraite : "Investiguer solutions compta"
- Plus tard, user capture : "Article intéressant sur pain points compta"
- Tags communs détectés → suggestion de lien

**Rationale:**
- Complexité UX non prioritaire pour MVP
- Tags générés par IA permettront d'implémenter cette fonctionnalité facilement plus tard
- Architecture events-based permet d'ajouter `TodoLinkedToCapture` event en V1.5

**Conséquences:**
- MVP : pas d'association manuelle
- Post-MVP : via tags ou UI explicite
- Event `TodoLinkedToCapture` prévu dans l'architecture

---

## V1.5 Features (Post-MVP)

**Enrichissement Post-Capture:**
- User peut ajouter audio/texte supplémentaire à une Capture existante
- Re-digestion automatique avec contexte enrichi
- Events: `EnrichmentRequested`, `ThoughtEnriched`, `ReDigestionTriggered`

**Brainstorm Guidé:**
- IA pose questions pour creuser une Idea/Project
- Génération Business Case structuré
- Events: `BrainstormSessionStarted`, `ConceptCrystallized`, `BusinessCaseGenerated`

**Partage Filtré:**
- User partage une Idea/Project (digest anonymisé)
- Collaboration invitation
- Events: `IdeaSharedRequested`, `ShareLinkCreated`, `CollaborationAccepted`

**Création Manuelle de Project (Top-Down):**
- User crée intentionnellement un Project vide
- Captures dans ce Project enrichissent contexte
- Command: `CreateProject` (manuel, pas suggested)

---

## Starter Template Decision

### ADR-007: From Scratch Approach (pas de starter full-stack)

**Status:** ✅ ACCEPTÉ

**Context:** Choisir entre utiliser des starters/boilerplates existants vs partir des CLI officiels.

**Starters Évalués:**

**Mobile:**
- [Obytes Starter](https://starter.obytes.com/) : Expo + TypeScript + MMKV + React Query + TailwindCSS
- [Fast Expo App](https://github.com/Teczer/fast-expo-app) : CLI avec Expo SDK 54 + MMKV + React Query

**Backend:**
- [NestJS Clean Architecture Boilerplate](https://github.com/rezawr/nestjs-clean-architecture-boilerplate) : DDD + Clean Architecture
- [NestJS-DDD-DevOps](https://andrea-acampora.github.io/nestjs-ddd-devops/) : DDD Modular Monolith + DevOps

**Decision:** Partir de zéro avec CLI officiels, copier uniquement configs/tooling.

**Rationale:**

**1. Architecture DDD Spécifique Déjà Définie**
- 8 Bounded Contexts identifiés via Event Storming
- Structure par contexte métier unique à Pensine
- Les starters imposent leur structure générique (user/product/order)
- Refonte complète nécessaire = perte de temps

**2. Stack Technique Particulière**
- Whisper.rn (module custom, pas dans les starters)
- WatermelonDB (peu de starters l'intègrent bien)
- Liquid Glass Design System (custom, pas TailwindCSS générique)
- RabbitMQ (rare dans starters NestJS)
- Aucun starter ne match cette stack exactement

**3. Suringénierie des Starters**
- Patterns "enterprise-grade" over-engineered (multi-tenancy, microservices ready)
- MVP mono-user = simplicité prioritaire
- Abstractions complexes à comprendre et maintenir
- Dépendances imposées difficiles à remplacer

**4. Temps Réel**
```
Starter full-stack:
- Setup: 2h
- Comprendre: 3h
- Nettoyer inutile: 5h
- Adapter architecture: 8h
- Debug conflicts: 3h
TOTAL: 21h

From Scratch:
- CLI official: 1h
- Tooling: 2h
- Structure DDD custom: 4h
- Intégrations: 6h
- Tests: 3h
TOTAL: 16h

Gain net: 5h + contrôle total
```

**5. Projet Solo**
- Pas besoin conventions d'équipe imposées
- Flexibilité pour ajuster rapidement
- Code maîtrisé à 100%

**Decision Détaillée:**

**Mobile:**
```bash
# Partir de Expo CLI officiel
npx create-expo-app@latest pensine-mobile --template blank-typescript

# Copier configs de Obytes (tooling uniquement):
- tsconfig.json strict
- .eslintrc minimal
- jest.config.js
- GitHub Actions EAS Build workflow
```

**Backend:**
```bash
# Partir de NestJS CLI officiel
npx @nestjs/cli new pensine-backend

# Structure DDD custom basée sur Event Storming:
src/
  contexts/
    knowledge/      (Core Domain)
      domain/
      application/
      infrastructure/
    opportunity/    (Core Domain)
    capture/        (Supporting)
    action/         (Supporting)
    normalization/  (Supporting)
  shared/
```

**Ce Qui Sera Réutilisé des Starters:**
- ✅ Config files (tsconfig, eslint, jest)
- ✅ Scripts package.json (lint, test, format)
- ✅ GitHub Actions workflows (CI/CD basiques)
- ✅ Documentation patterns (README structure)
- ❌ Code métier (100% custom)

**Conséquences:**
- Setup initial légèrement plus long (1-2h)
- Contrôle total sur architecture et dépendances
- Code parfaitement aligné avec Event Storming
- Pas de dette technique dès le départ
- Maîtrise complète du codebase

**Références:**
- [Obytes Starter](https://starter.obytes.com/)
- [Fast Expo App](https://github.com/Teczer/fast-expo-app)
- [NestJS Clean Architecture Boilerplate](https://github.com/rezawr/nestjs-clean-architecture-boilerplate)
- [Awesome NestJS Boilerplates](https://awesome-nestjs.com/resources/boilerplate.html)

---

### ADR-008: Anti-Corruption Layer (ACL) à la Frontière Mobile/Backend

**Status:** ✅ ACCEPTÉ

**Context:** Déterminer où les ACL sont nécessaires dans l'architecture Pensine.

**Decision:** ACL obligatoire à la frontière mobile ↔ backend (API Sync Layer), mais pas entre bounded contexts backend.

**Rationale:**

**Pas d'ACL entre Bounded Contexts Backend (Internes):**
- Contextes métier internes avec contrôle total
- Ubiquitous language cohérent et partagé
- Communication via Domain Events bien définis
- Aucun risque de corruption de modèle

**ACL OBLIGATOIRE à la frontière Mobile ↔ Backend:**

La structure des données mobile et backend PEUT diverger volontairement:

```typescript
// MOBILE (WatermelonDB - structure dénormalisée pour performance)
@model('captures')
class Capture {
  @field('type') type: string
  @field('raw_content') rawContent: string  // JSON stringifié
  @field('normalized_text') normalizedText: string
  @field('tags') tags: string  // JSON array stringifié
  @json('metadata', (json) => json) metadata: any
}

// BACKEND (DDD - structure normalisée)
class Capture {
  id: UUID
  type: CaptureType  // Value Object
  rawContent: AudioFile | TextContent | ImageFile  // Union type
  normalizedText?: NormalizedText  // Value Object
  tags: Tag[]  // Collection de Value Objects
  metadata: CaptureMetadata  // Value Object structuré
}
```

**L'API Sync Layer joue le rôle d'ACL et DOIT:**

**1. VALIDATION (Protection Domaine)**
```typescript
// Rejeter données invalides avant qu'elles n'atteignent le domaine
POST /sync/push
{
  changes: {
    captures: {
      created: [{
        id: "...",
        type: "audio",
        raw_content: "invalid_json"  // ❌ REJETÉ par ACL
      }]
    }
  }
}
→ 400 Bad Request (domaine backend jamais pollué)
```

**2. TRANSFORMATION (Structure Mobile ↔ Backend)**
```typescript
// Mobile → Backend (Pull response)
{
  raw_content: '{"audioUri": "file://..."}',  // JSON string
  tags: '["business", "idea"]'                 // JSON array string
}

// ACL transforme vers structure backend:
{
  rawContent: new AudioFile(uri),              // Value Object
  tags: [new Tag("business"), new Tag("idea")] // Collection VOs
}

// Backend → Mobile (Push)
{
  rawContent: audioFile.toJSON(),
  tags: tags.map(t => t.value)
}
→ Mobile reçoit format dénormalisé attendu
```

**3. TRADUCTION VOCABULARY**
```typescript
// Mobile utilise vocabulaire technique:
raw_content, normalized_text, audio_file_path

// Backend utilise vocabulaire métier:
rawContent, normalizedText, audioFile

// ACL traduit bidirectionnellement
```

**Avantages de cette approche:**

- **Mobile optimisé:** Structure plate, JSON strings pour performance, moins de joins
- **Backend protégé:** Validation stricte, types forts, Value Objects
- **Flexibilité:** Changer structure mobile sans impacter backend (et vice-versa)
- **Évolution indépendante:** Mobile et backend peuvent évoluer à des rythmes différents

**Conséquences:**
- API Sync doit implémenter transformations bidirectionnelles
- Tests d'intégration critiques pour valider ACL
- Documentation des mappings mobile ↔ backend obligatoire
- Coût CPU léger pour transformations (acceptable pour MVP)

---

### ADR-009: Stratégie de Synchronisation Mobile ↔ Backend

**Status:** ✅ ACCEPTÉ

**Context:** Définir comment la synchronisation offline-first fonctionne entre mobile et backend.

**Decision:** 6 décisions validées pour la stratégie de sync complète.

---

#### 9.1 - Timing de Synchronisation

**Decision:** Option C - Balanced (launch + post-action + polling 15min)

**Triggers de sync:**
1. **Au lancement app** (priorité haute)
2. **Après actions critiques** (capture, digestion completed, todo completed)
3. **Polling background 15min** (si app ouverte)
4. **Retour réseau** (après période offline)

**Rationale:**
- Balance entre réactivité et consommation batterie
- Données critiques (captures) synchronisées rapidement
- Polling 15min raisonnable pour updates non-critiques
- Pas de sync agressive permanente

**Implémentation:**
```typescript
// 1. Au lancement
useEffect(() => {
  syncService.sync({ priority: 'high' });
}, []);

// 2. Après action critique
onCaptureComplete(() => {
  syncService.sync({ priority: 'high', entity: 'captures' });
});

// 3. Polling background
setInterval(() => {
  if (appState === 'active') {
    syncService.sync({ priority: 'low' });
  }
}, 15 * 60 * 1000);

// 4. Retour réseau
NetInfo.addEventListener(state => {
  if (state.isConnected) {
    syncService.sync({ priority: 'medium' });
  }
});
```

---

#### 9.2 - Détection et Résolution de Conflits

**Decision:** Utiliser le mécanisme standard de WatermelonDB avec `lastPulledAt` + `last_modified` per-record.

**Comment ça fonctionne:**

```typescript
// 1. PULL PHASE (Backend → Mobile)
GET /sync/pull?last_pulled_at=2026-01-13T09:00:00Z

Backend retourne:
{
  changes: {
    captures: {
      updated: [
        { id: "c1", last_modified: 1736759400000, ... }  // Modifié après lastPulledAt
      ]
    }
  },
  timestamp: 1736760600000  // Nouveau lastPulledAt pour le client
}

Mobile enregistre: lastPulledAt = 1736760600000

// 2. PUSH PHASE (Mobile → Backend)
POST /sync/push
{
  last_pulled_at: 1736760600000,
  changes: {
    captures: {
      updated: [
        { id: "c1", ... }
      ]
    }
  }
}

Backend vérifie pour CHAQUE record:
IF server.last_modified > request.last_pulled_at
THEN → CONFLIT (donnée modifiée côté serveur depuis dernier pull)
ELSE → OK (accepter push)
```

**Résolution des conflits détectés:**

```typescript
// Stratégie per-column client-wins avec logique métier
class SyncConflictResolver {
  resolve(serverRecord, clientRecord, entity) {
    switch(entity) {
      case 'capture':
        // Métadonnées techniques: serveur gagne
        return {
          ...clientRecord,
          normalized_text: serverRecord.normalized_text,  // Serveur
          state: serverRecord.state,                      // Serveur

          // Données user: client gagne
          tags: clientRecord.tags,                        // Client
          projectId: clientRecord.projectId               // Client
        };

      case 'todo':
        // État métier: client gagne (user a agi localement)
        return {
          ...serverRecord,
          state: clientRecord.state,                      // Client
          completed_at: clientRecord.completed_at,        // Client

          // Métadonnées: serveur gagne
          priority: serverRecord.priority                 // Serveur (calculé par IA)
        };

      default:
        return clientRecord;  // Client-wins par défaut
    }
  }
}
```

**Gestion multi-clients:**

Chaque client a son propre `lastPulledAt`, le système fonctionne correctement:

```
Timeline:
10:00 - Record créé (last_modified = 10:00)
10:15 - MobileA pull (lastPulledAt_A = 10:15)
10:30 - MobileB pull (lastPulledAt_B = 10:30)
10:31 - MobileA modifie + push
        → Backend: last_modified = 10:31
10:32 - MobileB modifie + push
        → Backend détecte: server.last_modified (10:31) > client.lastPulledAt_B (10:30)
        → CONFLIT correctement détecté
```

**Support backoffice & API externes:**

```typescript
// Modification directe en DB (backoffice, cron job, API externe)
UPDATE captures SET normalized_text = '...', last_modified = NOW();

// Prochain pull mobile détecte automatiquement:
GET /sync/pull?last_pulled_at=2026-01-13T10:00:00Z
→ Backend retourne record avec last_modified > lastPulledAt
→ Mobile applique update correctement
```

**Rationale:**
- Mécanisme éprouvé de WatermelonDB
- `lastPulledAt` client-specific permet multi-clients
- `last_modified` server-side source de vérité
- Fonctionne avec backoffice et modifications externes
- Per-column resolution permet business logic fine

---

#### 9.3 - Gestion des Fichiers (Audio)

**Decision:** Option C - Upload Queue séparée

**Architecture:**

```typescript
// 1. Capture audio sauvegardée localement immédiatement
const capture = await captureService.create({
  type: 'audio',
  audioUri: 'file://local/audio123.m4a',  // Local file
  state: 'captured'
});

// 2. Ajout à upload queue
await uploadQueue.enqueue({
  captureId: capture.id,
  fileUri: capture.audioUri,
  priority: 'high'
});

// 3. Upload asynchrone indépendant du sync
uploadQueue.process({
  onSuccess: (captureId, remoteUrl) => {
    // Update capture avec URL remote
    db.captures.update(captureId, {
      audioRemoteUrl: remoteUrl,
      uploadState: 'completed'
    });
  },
  onError: (captureId, error) => {
    // Retry avec backoff
    uploadQueue.retry(captureId, { delay: fibonacci(attemptCount) });
  }
});

// 4. Sync metadata séparément (pas bloqué par upload)
syncService.sync();  // Sync capture metadata même si upload en cours
```

**Retry Upload Strategy:**

```typescript
class UploadQueue {
  async retry(item, options) {
    const delays = [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]; // Fibonacci en secondes
    const maxDelay = 5 * 60;  // Cap à 5 minutes

    const delay = Math.min(
      delays[item.attemptCount] * 1000,
      maxDelay * 1000
    );

    await sleep(delay);
    return this.upload(item);
  }
}
```

**Rationale:**
- Upload fichiers lents/volumineux ne bloque pas sync metadata
- Retry indépendant avec backoff
- Capture utilisable immédiatement (fichier local)
- Transcription Whisper fonctionne sur fichier local
- Upload asynchrone en arrière-plan

**Conséquences:**
- Complexité légèrement augmentée (2 systèmes)
- Gestion états: `pending`, `uploading`, `completed`, `failed`
- Cleanup fichiers locaux après upload réussi

---

#### 9.4 - Priorité de Synchronisation

**Decision:** Option B - Priority-based (Captures > Todos > Ideas > Projects)

**Ordre de sync:**

```typescript
const SYNC_PRIORITY = {
  captures: 1,     // HIGHEST - données critiques user
  todos: 2,        // HIGH - actions utilisateur
  thoughts: 3,     // MEDIUM - résultats digestion
  ideas: 4,        // LOW - concordances peuvent attendre
  projects: 5      // LOWEST - suggestions peuvent attendre
};

async function sync() {
  const entities = Object.keys(SYNC_PRIORITY)
    .sort((a, b) => SYNC_PRIORITY[a] - SYNC_PRIORITY[b]);

  for (const entity of entities) {
    await syncEntity(entity);
  }
}
```

**Sync incrémental si timeout:**

```typescript
// Si connexion lente ou timeout, sync par batch prioritaires
async function syncWithTimeout(timeoutMs = 10000) {
  const startTime = Date.now();

  for (const entity of prioritizedEntities) {
    if (Date.now() - startTime > timeoutMs) {
      console.log(`Timeout: synced up to ${entity}`);
      break;  // Reste sera synced au prochain trigger
    }

    await syncEntity(entity);
  }
}
```

**Rationale:**
- Captures = données critiques (NFR6: "0 capture perdue")
- Todos = actions user importantes
- Ideas/Projects = peuvent attendre (non-bloquant)
- Connexion lente ou timeout ne perd pas données critiques

---

#### 9.5 - Retry Logic & Error Handling

**Decision:** Result Pattern + Fibonacci Backoff (cap 5min)

**Result Pattern (pas try/catch):**

```typescript
// Enum de résultats
enum SyncResult {
  SUCCESS = 'success',
  NETWORK_ERROR = 'network_error',
  AUTH_ERROR = 'auth_error',
  CONFLICT = 'conflict',
  SERVER_ERROR = 'server_error',
  TIMEOUT = 'timeout'
}

type SyncResponse = {
  result: SyncResult;
  data?: any;
  error?: string;
  retryable: boolean;
};

// Usage
const response = await syncService.sync();

switch (response.result) {
  case SyncResult.SUCCESS:
    showToast('Sync réussie');
    break;

  case SyncResult.NETWORK_ERROR:
    if (response.retryable) {
      scheduleRetry({ delay: fibonacci(attemptCount) });
    }
    break;

  case SyncResult.AUTH_ERROR:
    redirectToLogin();  // Non retryable
    break;

  case SyncResult.CONFLICT:
    resolveConflict(response.data);
    break;
}
```

**Fibonacci Backoff:**

```typescript
const fibonacciDelays = [1, 1, 2, 3, 5, 8, 13, 21, 34, 55];  // Secondes
const maxDelay = 5 * 60;  // Cap 5 minutes

function getRetryDelay(attemptCount: number): number {
  const fibDelay = fibonacciDelays[Math.min(attemptCount, fibonacciDelays.length - 1)];
  return Math.min(fibDelay, maxDelay) * 1000;  // En millisecondes
}

// Exemple timeline:
// Attempt 1: 1s
// Attempt 2: 1s
// Attempt 3: 2s
// Attempt 4: 3s
// Attempt 5: 5s
// Attempt 6: 8s
// Attempt 7: 13s
// Attempt 8: 21s
// Attempt 9: 34s
// Attempt 10: 55s
// Attempt 11+: 5min (capped)
```

**Rationale (Fibonacci > Exponential):**

```
Fibonacci: 1, 1, 2, 3, 5, 8, 13, 21, 34, 55 → 5min cap
Exponential (base 2): 2, 4, 8, 16, 32, 64, 128, 256 → trop agressif

Avantages Fibonacci:
- Bagottage réseau temporaire: récupération rapide (1s, 1s, 2s)
- Backend down prolongé: monte progressivement sans exploser
- 5min cap évite attente infinie
- Soulage backend lors recovery (pas de stampede avec exponential)
```

**Rationale Result Pattern > try/catch:**
- Code appelant peut interpréter enum facilement
- Switch exhaustif (TypeScript vérifie tous les cas)
- Retry logic centralisée et claire
- Pas de exceptions non catchées qui crashent l'app

---

#### 9.6 - Schema Versioning

**Decision:** Simple versioning (pas de Schema Registry pour MVP)

**Approche:**

```typescript
// Mobile envoie version de schema dans headers
POST /sync/push
Headers:
  X-Schema-Version: 1.0.0

// Backend valide compatibilité
if (requestSchemaVersion !== serverSchemaVersion) {
  if (isCompatible(requestSchemaVersion, serverSchemaVersion)) {
    // OK - backward compatible
    applyMigration(requestSchemaVersion);
  } else {
    // KO - force app update
    return { error: 'APP_UPDATE_REQUIRED' };
  }
}
```

**Migration strategy:**

```typescript
// Changement de schéma (ex: ajout colonne)
// V1.0.0 → V1.1.0

// Backend supporte les 2 versions:
function handlePush(data, schemaVersion) {
  switch (schemaVersion) {
    case '1.0.0':
      // Ancienne version - remplir nouvelle colonne avec default
      return {
        ...data,
        newColumn: 'default_value'
      };

    case '1.1.0':
      // Nouvelle version - utiliser directement
      return data;
  }
}
```

**Rationale:**
- MVP mono-user: pas besoin Schema Registry complexe
- Header version suffit pour validation
- Backend peut supporter N-1 versions facilement
- Force update si breaking change critique
- Post-MVP: migrer vers Schema Registry si multi-versions critiques

**Conséquences:**
- Simple à implémenter
- Backend doit gérer migrations manuellement
- Documentation versions obligatoire
- Si scaling: ajouter Schema Registry (Kafka Schema Registry, etc.)

---

**Conséquences globales ADR-009:**
- Sync robuste et performante
- 0 perte de données garantie
- Multi-clients supporté nativement
- Backoffice compatible
- Code maintenable avec Result Pattern

---

### ADR-010: Security & Encryption Strategy

**Status:** ✅ ACCEPTÉ

**Context:** Définir la stratégie de sécurité et de chiffrement pour protéger les données sensibles des utilisateurs (audio, transcriptions, idées business).

**Decision:** 5 décisions validées pour la sécurité complète du système.

---

#### 10.1 - Stratégie d'Authentification

**Decision:** JWT Access Token (courte durée) + Refresh Token (longue durée)

**Architecture:**

```typescript
// 1. Login
POST /auth/login
{
  email: "user@example.com",
  password: "***"
}

Response:
{
  accessToken: "eyJhbGc...",      // JWT 15min
  refreshToken: "rt_abc123...",   // 30 jours
  expiresIn: 900                  // 15min en secondes
}

// 2. Access Token stocké en mémoire (React state)
// 3. Refresh Token stocké en Keychain/Keystore (sécurisé)

// 4. Refresh automatique avant expiration
GET /auth/refresh
Headers:
  Authorization: Bearer rt_abc123...

Response:
{
  accessToken: "eyJhbGc...",      // Nouveau JWT 15min
  expiresIn: 900
}

// 5. Révocation (logout, compromission)
POST /auth/revoke
Headers:
  Authorization: Bearer rt_abc123...

→ Refresh token supprimé de DB (blacklist)
```

**Caractéristiques:**

**Access Token (JWT):**
- Durée: **15 minutes**
- Stockage: **Mémoire** (React state, pas AsyncStorage)
- Format: JWT signé (HS256 ou RS256)
- Claims: `{ sub: userId, email, iat, exp }`
- Stateless: backend ne stocke pas (vérifie signature)

**Refresh Token:**
- Durée: **30 jours**
- Stockage: **Keychain (iOS) / Keystore (Android)** (chiffré OS)
- Format: Random string opaque (UUID v4)
- Stateful: stocké en DB backend avec metadata
- Révocable: peut être invalidé côté serveur

**Rationale:**
- Access token court (15min) limite window d'exploitation si volé
- Refresh token long (30 jours) évite re-login fréquent
- Stockage séparé: access en mémoire (volatil), refresh sécurisé OS
- Révocation: logout ou compromission invalide refresh token
- Balance sécurité/UX optimale pour mobile

**Rotation Refresh Token (Post-MVP):**
```typescript
// Chaque refresh génère nouveau refresh token (rotation)
POST /auth/refresh
→ Ancien refresh token invalidé
→ Nouveau refresh token émis
→ Protection contre vol de refresh token
```

---

#### 10.2 - Chiffrement au Repos (Mobile)

**Decision:** MVP = Keychain/Keystore pour tokens, Post-MVP = SQLCipher pour DB complète

**MVP (Minimal Viable Security):**

```typescript
// Tokens sensibles chiffrés par OS
import * as SecureStore from 'expo-secure-store';

// Stockage refresh token
await SecureStore.setItemAsync('refresh_token', refreshToken);

// Récupération
const refreshToken = await SecureStore.getItemAsync('refresh_token');

// Suppression (logout)
await SecureStore.deleteItemAsync('refresh_token');
```

**Données stockées en Keychain/Keystore (MVP):**
- ✅ Refresh Token (critique)
- ✅ User credentials si "Remember me" (optionnel)
- ❌ DB WatermelonDB: **non chiffrée** (SQLite standard)

**Rationale MVP:**
- Keychain/Keystore = chiffrement hardware-backed (Secure Enclave iOS, TEE Android)
- Protection tokens critique sans overhead performance
- DB non chiffrée acceptable pour MVP mono-user
- Transcriptions/ideas pas réglementées (pas HIPAA/données santé)

**Post-MVP (Production Hardening):**

```typescript
// SQLCipher pour chiffrement DB complète
import { Database } from '@nozbe/watermelondb';
import SQLiteAdapter from '@nozbe/watermelondb/adapters/sqlite';

const adapter = new SQLiteAdapter({
  dbName: 'pensine',
  // SQLCipher encryption
  encryptionKey: await getEncryptionKey()  // Dérivée du Keychain
});

const database = new Database({
  adapter,
  modelClasses: [Capture, Thought, Todo, Idea, Project]
});
```

**Clé chiffrement DB:**
- Générée au premier lancement (crypto.randomBytes(32))
- Stockée dans Keychain/Keystore
- DB inaccessible sans déverrouillage device
- Performance impact: ~15% overhead (acceptable)

**Rationale Post-MVP:**
- Protection complète données sensibles (transcriptions, idées business)
- Conformité certifications (si B2B futur)
- Pas d'impact UX (transparent)

---

#### 10.3 - Chiffrement au Repos (Backend)

**Decision:** MVP = Disk/Volume Encryption (Infrastructure), Post-MVP = TDE ou Column-level si régulé

**MVP (Infrastructure-level Encryption):**

```bash
# Docker Volume chiffré (LUKS)
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/encrypted/data \
  --opt o=bind \
  pensine-data

# PostgreSQL stocké sur volume chiffré
services:
  postgres:
    volumes:
      - pensine-data:/var/lib/postgresql/data

# Scaleway/AWS: Volumes chiffrés par défaut
# - AWS EBS: encryption at rest (AES-256)
# - Scaleway Block Storage: LUKS encryption
```

**Rationale MVP:**
- Chiffrement volume = protection vol physique disques
- Transparent pour application (pas de code)
- Gratuit sur cloud providers modernes
- Conforme exigences basiques sécurité

**Post-MVP (Database-level Encryption):**

**Option A: Transparent Data Encryption (TDE)** (PostgreSQL 15+)
```sql
-- Chiffrement automatique toutes les tables
ALTER DATABASE pensine SET encryption = 'on';
```

**Option B: Column-level Encryption** (données ultra-sensibles)
```typescript
// Chiffrement sélectif colonnes sensibles
class Capture {
  id: UUID;

  @Encrypted()  // Chiffré avec clé backend
  normalizedText: string;

  @Encrypted()
  rawContent: string;

  state: string;  // Non chiffré (ok)
}
```

**Quand utiliser Post-MVP:**
- Régulations spécifiques (HIPAA, PCI-DSS)
- Clients B2B exigeants (compliance)
- Multi-tenancy avec isolation forte
- Audit trail chiffré (traçabilité)

**Rationale:**
- MVP: infrastructure encryption suffit
- Post-MVP: database encryption si besoin métier
- Éviter sur-engineering prématuré
- Performance impact TDE: ~10%, Column: ~20%

---

#### 10.4 - Communication Sécurisée (TLS/HTTPS)

**Decision:** TLS 1.3 obligatoire, certificats Let's Encrypt, HSTS activé

**Configuration Backend:**

```typescript
// NestJS HTTPS configuration
import * as https from 'https';
import * as fs from 'fs';

const httpsOptions = {
  key: fs.readFileSync('/certs/privkey.pem'),
  cert: fs.readFileSync('/certs/fullchain.pem'),
  // TLS 1.3 minimum
  minVersion: 'TLSv1.3',
  // Ciphers modernes uniquement
  ciphers: [
    'TLS_AES_128_GCM_SHA256',
    'TLS_AES_256_GCM_SHA384',
    'TLS_CHACHA20_POLY1305_SHA256'
  ].join(':')
};

const app = await NestFactory.create(AppModule, {
  httpsOptions
});
```

**Headers Sécurité:**

```typescript
// Helmet.js (security headers)
app.use(helmet({
  hsts: {
    maxAge: 31536000,        // 1 an
    includeSubDomains: true,
    preload: true
  },
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.pensine.example.local"]
    }
  }
}));
```

**Mobile (Certificate Pinning - Post-MVP):**

```typescript
// Expo config.json
{
  "expo": {
    "ios": {
      "infoPlist": {
        "NSAppTransportSecurity": {
          "NSExceptionDomains": {
            "api.pensine.example.local": {
              "NSIncludesSubdomains": true,
              "NSExceptionRequiresForwardSecrecy": true,
              "NSExceptionMinimumTLSVersion": "TLSv1.3",
              "NSPinnedDomains": [
                "api.pensine.example.local"
              ]
            }
          }
        }
      }
    }
  }
}
```

**Rationale:**
- TLS 1.3: protection MITM, forward secrecy
- Let's Encrypt: gratuit, auto-renew, trusted
- HSTS: force HTTPS, protection downgrade attacks
- Certificate pinning (post-MVP): protection CA compromise

**Conséquences:**
- Aucune communication en clair (HTTP interdit)
- Dev local: certificats self-signed ok (pas prod)
- Monitoring expiration certificats (alerte 30j avant)

---

#### 10.5 - Conformité RGPD

**Decision:** Data export, account deletion (soft delete 30j), opt-in consent analytics

**Droits Utilisateur (RGPD Articles 15, 17, 20):**

**1. Droit d'accès (Article 15):**
```typescript
// Export complet données utilisateur
GET /user/export

Response: ZIP file containing:
  - user_data.json (profile, metadata)
  - captures.json (toutes les captures)
  - audio_files/ (tous les fichiers audio)
  - thoughts.json (digestions, résumés)
  - todos.json (actions)
  - ideas.json (idées)
  - projects.json (projets)
```

**2. Droit à l'effacement (Article 17):**
```typescript
// Suppression compte
DELETE /user/account

Process:
1. Soft delete (30 jours de grâce)
   → account.state = 'pending_deletion'
   → account.deletion_date = NOW() + 30 days
   → User peut annuler dans les 30j

2. Hard delete (après 30 jours, cron job)
   → Suppression DB: CASCADE sur toutes les tables
   → Suppression S3: audio files
   → Suppression Redis: cache
   → Anonymisation logs: remplace userId par 'deleted_user'

3. Exceptions:
   → Logs applicatifs: 90 jours (debugging)
   → Backups: 7 jours (restauration)
   → Legal hold: si procédure judiciaire en cours
```

**3. Consentement (Article 6):**
```typescript
// Onboarding - Opt-in explicite
const consents = {
  required: [
    {
      type: 'terms_of_service',
      mandatory: true,
      description: "Conditions d'utilisation Pensine"
    },
    {
      type: 'data_processing',
      mandatory: true,
      description: "Traitement données pour fonctionnement app"
    }
  ],
  optional: [
    {
      type: 'analytics',
      mandatory: false,
      description: "Statistiques usage anonymisées (amélioration produit)",
      default: false  // Opt-in, pas opt-out
    },
    {
      type: 'crash_reports',
      mandatory: false,
      description: "Rapports crash automatiques (Sentry)",
      default: true   // Recommandé mais optionnel
    }
  ]
};

// User peut modifier consentements après onboarding
PUT /user/consents
{
  analytics: false,
  crash_reports: true
}
```

**4. Portabilité (Article 20):**
```json
// Format export JSON structuré
{
  "version": "1.0.0",
  "exported_at": "2026-01-13T10:00:00Z",
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "created_at": "2026-01-01T00:00:00Z"
  },
  "captures": [
    {
      "id": "uuid",
      "type": "audio",
      "captured_at": "2026-01-12T15:30:00Z",
      "audio_file": "audio_files/capture_123.m4a",
      "transcription": "...",
      "summary": "...",
      "tags": ["business", "idea"]
    }
  ],
  "todos": [...],
  "ideas": [...]
}
```

**Rationale:**
- Conformité légale UE obligatoire
- Export JSON + audio = portabilité complète
- Soft delete 30j = protection erreur utilisateur
- Opt-in analytics = respect vie privée
- Anonymisation logs = balance sécurité/privacy

**Documentation Utilisateur:**
- Privacy Policy (français + anglais)
- Data Processing Agreement (si B2B)
- FAQ RGPD (droits, procédures)

---

**Conséquences globales ADR-010:**
- Protection données sensibles multi-niveaux
- Conformité RGPD complète
- Balance sécurité / UX / performance
- Coût infrastructure: +10% (chiffrement, certificats)
- Maintenance: monitoring certificats, audit logs, RGPD requests

---

### ADR-011: Performance Optimization Strategy

**Status:** ✅ ACCEPTÉ

**Context:** Définir les stratégies d'optimisation pour respecter les NFRs de performance critiques (capture < 500ms, transcription < 2x durée audio, digestion < 30s).

**Decision:** 3 décisions validées pour optimisation globale.

---

#### 11.1 - Redis Caching Strategy

**Decision:** Cache stratégique ciblé, pas de cache agressif

**Usage de Redis :**

```typescript
// 1. Session cache (JWT payload décodé)
const sessionKey = `session:${userId}`;
await redis.setex(sessionKey, 900, JSON.stringify(sessionData));  // 15min TTL

// 2. User profile cache (fréquemment accédé)
const userKey = `user:${userId}`;
await redis.setex(userKey, 3600, JSON.stringify(userProfile));  // 1h TTL

// 3. Digestion results cache (temps réel récent)
const digestKey = `digest:${captureId}`;
await redis.setex(digestKey, 1800, JSON.stringify(digestResult));  // 30min TTL

// 4. Concordance scores (calculs coûteux)
const concordanceKey = `concordance:${ideaId}`;
await redis.setex(concordanceKey, 3600, JSON.stringify(scores));  // 1h TTL
```

**Ce qui N'est PAS caché :**
- ❌ Captures (changent fréquemment, mobile offline-first a déjà le cache)
- ❌ Todos (état utilisateur volatile)
- ❌ Ideas/Projects (concordance dynamique)

**Invalidation :**

```typescript
// Invalidation ciblée (pas de flush global)
class CacheService {
  async invalidateUser(userId: string) {
    await redis.del(`user:${userId}`);
    await redis.del(`session:${userId}`);
  }

  async invalidateDigest(captureId: string) {
    await redis.del(`digest:${captureId}`);
  }

  // Pattern-based invalidation (attention performance)
  async invalidateConcordanceForIdea(ideaId: string) {
    const keys = await redis.keys(`concordance:*${ideaId}*`);
    if (keys.length > 0) {
      await redis.del(...keys);
    }
  }
}
```

**Rationale :**
- Cache UNIQUEMENT les hot paths (session, profile)
- TTL courts pour éviter stale data
- Mobile a déjà cache complet (WatermelonDB)
- Redis = cache serveur uniquement
- MVP mono-user = moins de pression cache

**Metrics à monitorer :**
- Cache hit rate (objectif > 80% pour session/user)
- Latence avec/sans cache
- Memory usage Redis

---

#### 11.2 - Lazy Loading Strategies

**Decision:** Lazy loading ciblé UI + data, pagination backend

**Mobile (UI Lazy Loading) :**

```typescript
// 1. Liste captures avec virtualisation
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={captures}
  renderItem={({ item }) => <CaptureCard capture={item} />}
  estimatedItemSize={120}
  // Virtualization automatique (ne render que visible)
/>

// 2. Images lazy avec blurhash
<Image
  source={{ uri: capture.thumbnailUrl }}
  placeholder={{ blurhash: capture.blurhash }}
  transition={200}
/>

// 3. Audio player lazy (chargé au tap)
const AudioPlayer = lazy(() => import('./AudioPlayer'));

<Suspense fallback={<AudioPlayerSkeleton />}>
  {isPlaying && <AudioPlayer uri={audioUri} />}
</Suspense>

// 4. Tabs lazy (pas de pré-chargement)
// Tab Actions chargé seulement quand user tape dessus
<Tab.Navigator lazy={true}>
  <Tab.Screen name="Feed" component={FeedScreen} />
  <Tab.Screen name="Actions" component={ActionsScreen} />  {/* Lazy */}
  <Tab.Screen name="Projects" component={ProjectsScreen} />  {/* Lazy */}
</Tab.Navigator>
```

**Backend (Data Pagination) :**

```typescript
// API avec cursor-based pagination
GET /captures?cursor=c123&limit=20

// Query optimisé avec index
const captures = await db.captures
  .where('user_id', userId)
  .where('created_at', '<', cursorDate)
  .orderBy('created_at', 'desc')
  .limit(20);

Response:
{
  data: [...],
  nextCursor: 'c143',
  hasMore: true
}

// Mobile incremental load
class CaptureList {
  async loadMore() {
    if (this.isLoading || !this.hasMore) return;

    const response = await api.getCaptures({
      cursor: this.nextCursor,
      limit: 20
    });

    this.captures.push(...response.data);
    this.nextCursor = response.nextCursor;
    this.hasMore = response.hasMore;
  }
}
```

**WatermelonDB Lazy Queries :**

```typescript
// Ne charge QUE ce qui est affiché (pas toute la DB)
const visibleCaptures = await database.get('captures')
  .query(
    Q.where('created_at', Q.gte(last30Days)),
    Q.sortBy('created_at', Q.desc),
    Q.take(50)  // Limite initiale
  )
  .observe();  // Réactif

// Expand si user scroll
```

**Rationale :**
- Capture < 500ms : pas de chargement lourd au lancement
- FlashList : performance lists longues (1000+ items)
- Pagination backend : évite OOM si 10k+ captures
- Tabs lazy : économie mémoire
- Blurhash : perception vitesse (placeholder immédiat)

---

#### 11.3 - Audio Compression & Formats

**Decision:** AAC-LC compression, qualité adaptative, nettoyage automatique

**Format Audio :**

```typescript
// Recording config (Expo AV)
const recordingOptions = {
  android: {
    extension: '.m4a',
    outputFormat: Audio.AndroidOutputFormat.MPEG_4,
    audioEncoder: Audio.AndroidAudioEncoder.AAC,
    sampleRate: 44100,
    numberOfChannels: 1,  // Mono (voix)
    bitRate: 64000,       // 64 kbps (balance qualité/taille)
  },
  ios: {
    extension: '.m4a',
    outputFormat: Audio.IOSOutputFormat.MPEG4AAC,
    audioQuality: Audio.IOSAudioQuality.MEDIUM,
    sampleRate: 44100,
    numberOfChannels: 1,
    bitRate: 64000,
    linearPCMBitDepth: 16,
    linearPCMIsBigEndian: false,
    linearPCMIsFloat: false,
  },
  web: {
    mimeType: 'audio/webm;codecs=opus',
    bitsPerSecond: 64000,
  }
};
```

**Tailles Attendues :**

```
Durée    | Taille (64 kbps mono)
---------|----------------------
30s      | ~240 KB
1 min    | ~480 KB
2 min    | ~960 KB
5 min    | ~2.4 MB

5 captures/jour × 1 min × 30 jours = ~72 MB/mois
```

**Compression Serveur (Post-Upload) :**

```typescript
// Optionnel : re-compression serveur pour stockage long terme
class AudioProcessor {
  async compressForStorage(audioFile: Buffer): Promise<Buffer> {
    // FFmpeg : AAC 32 kbps pour stockage (qualité suffisante)
    return ffmpeg(audioFile)
      .audioCodec('aac')
      .audioBitrate('32k')
      .audioChannels(1)
      .toBuffer();
  }
}

// Réduction: 64 kbps → 32 kbps = 50% économie stockage
```

**Nettoyage Local (Mobile) :**

```typescript
// Cleanup audio local après upload réussi + X jours
class StorageManager {
  async cleanupOldAudio() {
    const threshold = Date.now() - (30 * 24 * 60 * 60 * 1000);  // 30 jours

    const oldCaptures = await db.captures.query(
      Q.where('upload_state', 'completed'),
      Q.where('captured_at', Q.lt(threshold))
    );

    for (const capture of oldCaptures) {
      // Supprimer fichier local
      await FileSystem.deleteAsync(capture.audioLocalUri, { idempotent: true });

      // Garder metadata + URL remote
      await capture.update(c => {
        c.audioLocalUri = null;  // Indique "plus en local"
      });
    }
  }
}

// Cron: tous les jours à 3h du matin
```

**Streaming (Post-MVP) :**

```typescript
// Si fichier pas en local, stream depuis serveur
async playAudio(capture: Capture) {
  const uri = capture.audioLocalUri
    ? capture.audioLocalUri           // Local (rapide)
    : capture.audioRemoteUrl;         // Stream serveur (si cleanup)

  await Audio.Sound.createAsync(
    { uri },
    { shouldPlay: true }
  );
}
```

**Rationale :**
- AAC-LC : meilleur ratio qualité/taille pour voix
- 64 kbps mono : qualité suffisante pour transcription Whisper
- Cleanup 30j : économie stockage mobile (128 GB = limité)
- Format M4A : compatible iOS/Android/Web
- Mono channel : voix uniquement (pas besoin stéréo)

**Whisper Performance :**
- Qualité audio n'impacte pas beaucoup précision transcription
- 64 kbps AAC > suffisant pour Whisper
- Compression réduit aussi temps upload

---

**Conséquences globales ADR-011:**
- NFRs performance respectées (capture < 500ms, chargement < 1s)
- Économie bande passante : ~50% (compression audio)
- Économie stockage mobile : cleanup automatique
- UX fluide : lazy loading + virtualisation
- Cache ciblé : pas de over-caching

---

### ADR-012: Queue Management avec RabbitMQ

**Status:** ✅ ACCEPTÉ

**Context:** Gérer les tâches asynchrones (transcription, digestion IA, concordance) avec RabbitMQ pour isolation pannes et résilience.

**Decision:** 4 décisions validées pour gestion complète des queues.

---

#### 12.1 - Dead Letter Queues (DLQ)

**Decision:** DLQ systématique pour chaque queue métier

**Architecture :**

```typescript
// Configuration RabbitMQ
const queueConfig = {
  // Queue principale
  digestion: {
    name: 'digestion.queue',
    durable: true,
    arguments: {
      'x-dead-letter-exchange': 'dlx',
      'x-dead-letter-routing-key': 'digestion.dlq',
      'x-message-ttl': 300000,  // 5 min max processing
    }
  },

  // Dead Letter Queue
  digestion_dlq: {
    name: 'digestion.dlq',
    durable: true,
    // Pas de retry automatique depuis DLQ
  }
};

// Transcription queue
const transcriptionConfig = {
  name: 'transcription.queue',
  durable: true,
  arguments: {
    'x-dead-letter-exchange': 'dlx',
    'x-dead-letter-routing-key': 'transcription.dlq',
    'x-message-ttl': 600000,  // 10 min max (audio long)
  }
};

// Concordance queue
const concordanceConfig = {
  name: 'concordance.queue',
  durable: true,
  arguments: {
    'x-dead-letter-exchange': 'dlx',
    'x-dead-letter-routing-key': 'concordance.dlq',
    'x-message-ttl': 60000,  // 1 min max
  }
};
```

**Consumer avec ACK manuel :**

```typescript
@Injectable()
class DigestionConsumer {
  @RabbitSubscribe({
    exchange: 'pensine',
    routingKey: 'digestion.queue',
    queue: 'digestion.queue',
  })
  async handleDigestion(msg: DigestionMessage, context: RabbitContext) {
    const channel = context.getChannelRef();
    const originalMsg = context.getMessage();

    try {
      // Traitement
      const result = await this.digestionService.process(msg);

      // ACK success
      channel.ack(originalMsg);

      // Publier résultat
      await this.eventBus.publish(new ThoughtDigested(result));

    } catch (error) {
      // Log erreur
      this.logger.error('Digestion failed', { msg, error });

      // NACK → va en DLQ
      channel.nack(originalMsg, false, false);

      // Notifier erreur
      await this.notificationService.sendError(msg.userId, 'digestion_failed');
    }
  }
}
```

**Monitoring DLQ :**

```typescript
// Cron job : monitorer DLQ toutes les 10 minutes
@Cron('*/10 * * * *')
async monitorDeadLetters() {
  const dlqStats = await this.rabbitService.getQueueStats([
    'digestion.dlq',
    'transcription.dlq',
    'concordance.dlq'
  ]);

  for (const [queue, stats] of Object.entries(dlqStats)) {
    if (stats.messages > 0) {
      // Alerte si messages en DLQ
      await this.alertService.send({
        severity: 'warning',
        message: `${stats.messages} messages in ${queue}`,
        queue,
        count: stats.messages
      });
    }
  }
}
```

**Rationale :**
- DLQ évite perte de messages échoués
- TTL empêche blocage infini (timeout)
- NACK sans requeue → DLQ directement
- Monitoring DLQ = détection erreurs systémiques

---

#### 12.2 - Retry Logic & Exponential Backoff

**Decision:** Retry avec Fibonacci backoff, max 5 attempts

**Retry Headers (RabbitMQ) :**

```typescript
// Message avec retry count dans headers
interface MessageWithRetry {
  payload: any;
  headers: {
    'x-retry-count': number;
    'x-first-attempt': number;  // Timestamp
    'x-last-attempt': number;
  };
}

// Publisher ajoute headers
await this.rabbitService.publish('digestion.queue', {
  captureId: 'c123',
  userId: 'u456',
}, {
  headers: {
    'x-retry-count': 0,
    'x-first-attempt': Date.now(),
    'x-last-attempt': Date.now(),
  }
});
```

**Consumer avec retry logic :**

```typescript
@Injectable()
class DigestionConsumer {
  private readonly MAX_RETRIES = 5;
  private readonly FIBONACCI_DELAYS = [1, 1, 2, 3, 5, 8];  // Secondes

  async handleDigestion(msg: MessageWithRetry, context: RabbitContext) {
    const channel = context.getChannelRef();
    const originalMsg = context.getMessage();
    const retryCount = msg.headers['x-retry-count'] || 0;

    try {
      await this.digestionService.process(msg.payload);
      channel.ack(originalMsg);

    } catch (error) {
      // Vérifier si erreur retryable
      const isRetryable = this.isRetryableError(error);

      if (!isRetryable || retryCount >= this.MAX_RETRIES) {
        // Non retryable ou max retries → DLQ
        this.logger.error('Max retries reached or non-retryable', { msg, error });
        channel.nack(originalMsg, false, false);  // → DLQ
        return;
      }

      // Retry avec backoff
      const delaySeconds = this.FIBONACCI_DELAYS[retryCount];

      // Re-publier avec delay
      await this.retryWithDelay(msg, retryCount + 1, delaySeconds);

      // ACK message original (sera re-traité via republication)
      channel.ack(originalMsg);
    }
  }

  private async retryWithDelay(
    msg: MessageWithRetry,
    newRetryCount: number,
    delaySeconds: number
  ) {
    await this.rabbitService.publish(
      'digestion.queue',
      msg.payload,
      {
        headers: {
          'x-retry-count': newRetryCount,
          'x-first-attempt': msg.headers['x-first-attempt'],
          'x-last-attempt': Date.now(),
        },
        // Delayed message exchange plugin
        delay: delaySeconds * 1000,
      }
    );
  }

  private isRetryableError(error: Error): boolean {
    // Erreurs temporaires = retryable
    const retryableErrors = [
      'ECONNREFUSED',     // LLM API down
      'ETIMEDOUT',        // Timeout réseau
      'ENOTFOUND',        // DNS temporaire
      '429',              // Rate limit LLM
      '503',              // Service unavailable
    ];

    return retryableErrors.some(code =>
      error.message.includes(code) || error.name.includes(code)
    );
  }
}
```

**RabbitMQ Delayed Message Exchange :**

```bash
# Installation plugin
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# Configuration exchange
{
  "name": "delayed_exchange",
  "type": "x-delayed-message",
  "durable": true,
  "arguments": {
    "x-delayed-type": "direct"
  }
}
```

**Rationale :**
- Fibonacci backoff : progression douce (1s, 1s, 2s, 3s, 5s, 8s)
- Max 5 retries : évite boucle infinie
- Erreurs retryables vs permanentes : stratégie différenciée
- Delayed exchange : évite polling (natif RabbitMQ)

---

#### 12.3 - Queue Prioritization

**Decision:** Queues séparées avec consommateurs prioritaires

**Architecture multi-queues :**

```typescript
// Queues par priorité métier
const QUEUES = {
  // CRITICAL : impact user immédiat
  transcription: {
    name: 'transcription.queue',
    priority: 'critical',
    consumers: 3,  // 3 workers dédiés
  },

  // HIGH : expérience user
  digestion: {
    name: 'digestion.queue',
    priority: 'high',
    consumers: 2,
  },

  // MEDIUM : background
  concordance: {
    name: 'concordance.queue',
    priority: 'medium',
    consumers: 1,
  },

  // LOW : batch
  analytics: {
    name: 'analytics.queue',
    priority: 'low',
    consumers: 1,
  },
};
```

**Consumer avec scaling dynamique :**

```typescript
// NestJS worker scalable
@Module({
  imports: [
    RabbitMQModule.forRoot({
      exchanges: [{ name: 'pensine', type: 'topic' }],
      uri: process.env.RABBITMQ_URI,
      connectionInitOptions: { wait: false },
    }),
  ],
})
class WorkersModule implements OnModuleInit {
  constructor(private rabbitService: AmqpConnection) {}

  onModuleInit() {
    // Spawn consumers selon config
    for (const [name, config] of Object.entries(QUEUES)) {
      for (let i = 0; i < config.consumers; i++) {
        this.spawnConsumer(name, i);
      }
    }
  }

  private spawnConsumer(queueName: string, workerId: number) {
    this.logger.log(`Spawning consumer ${queueName}:${workerId}`);
    // Consumer s'enregistre automatiquement via @RabbitSubscribe
  }
}
```

**Metrics & Auto-scaling (Post-MVP) :**

```typescript
// Monitorer queue depth
@Cron('*/1 * * * *')  // Toutes les minutes
async monitorQueueDepth() {
  const stats = await this.rabbitService.getQueueStats('digestion.queue');

  if (stats.messages > 100) {
    // Queue saturée → augmenter consumers
    await this.scalingService.scaleUp('digestion', targetConsumers: 4);
  }

  if (stats.messages < 10 && stats.consumers > 2) {
    // Queue vide → réduire consumers
    await this.scalingService.scaleDown('digestion', targetConsumers: 2);
  }
}
```

**Rationale :**
- Queues séparées : isolation pannes (transcription down ≠ digestion bloquée)
- Consumers dédiés : garantie traitement prioritaire
- Scaling par queue : optimisation ressources
- MVP : consumers fixes, Post-MVP : auto-scaling

---

#### 12.4 - Monitoring & Alerting

**Decision:** Métriques RabbitMQ + alertes proactives

**Métriques RabbitMQ à tracker :**

```typescript
interface QueueMetrics {
  // Volume
  messages: number;           // Messages en attente
  messagesReady: number;      // Prêts à consommer
  messagesUnacked: number;    // En cours de traitement

  // Performance
  publishRate: number;        // Msgs/sec publiés
  consumeRate: number;        // Msgs/sec consommés
  ackRate: number;            // Msgs/sec acknowledgés

  // Consumers
  consumers: number;          // Consumers actifs
  consumerUtilisation: number; // % utilisation

  // Durée
  avgProcessingTime: number;  // Temps moyen traitement
}
```

**Collecte métriques (Prometheus) :**

```typescript
// NestJS Prometheus exporter
@Injectable()
class RabbitMetricsCollector {
  private readonly gauges = {
    queueDepth: new Gauge({
      name: 'rabbitmq_queue_messages',
      help: 'Messages in queue',
      labelNames: ['queue'],
    }),
    consumers: new Gauge({
      name: 'rabbitmq_queue_consumers',
      help: 'Active consumers',
      labelNames: ['queue'],
    }),
  };

  @Cron('*/30 * * * * *')  // Toutes les 30s
  async collectMetrics() {
    for (const queueName of Object.keys(QUEUES)) {
      const stats = await this.rabbitService.getQueueStats(queueName);

      this.gauges.queueDepth.set({ queue: queueName }, stats.messages);
      this.gauges.consumers.set({ queue: queueName }, stats.consumers);
    }
  }
}
```

**Alertes (critères) :**

```typescript
const ALERTS = {
  queueSaturated: {
    condition: 'queue_depth > 500',
    severity: 'warning',
    message: 'Queue saturée, scaling nécessaire',
  },

  consumerDown: {
    condition: 'consumers == 0 && queue_depth > 0',
    severity: 'critical',
    message: 'Aucun consumer actif',
  },

  dlqNotEmpty: {
    condition: 'dlq_depth > 0',
    severity: 'warning',
    message: 'Messages en DLQ nécessitent investigation',
  },

  slowProcessing: {
    condition: 'avg_processing_time > 60000',  // 60s
    severity: 'warning',
    message: 'Traitement lent détecté',
  },
};
```

**Dashboard Grafana (KPIs) :**

```
📊 RabbitMQ Dashboard

┌─────────────────────────────────────────┐
│ Queue Depth (real-time)                 │
│ ▂▃▅▇█▇▅▃▂ Digestion: 23 msgs           │
│ ▁▁▂▃▂▁▁▁▁ Transcription: 5 msgs        │
│ ▁▁▁▁▁▁▁▁▁ Concordance: 0 msgs          │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Throughput (msgs/min)                   │
│ Published: 120 msg/min                  │
│ Consumed: 115 msg/min                   │
│ DLQ: 2 msg/min ⚠️                      │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Processing Time (avg)                   │
│ Digestion: 12.3s                        │
│ Transcription: 45s                      │
│ Concordance: 3.2s                       │
└─────────────────────────────────────────┘
```

**Rationale :**
- Prometheus : métriques time-series standard
- Grafana : visualisation temps réel
- Alertes proactives : détection avant incident
- KPIs métier : digestion < 30s, transcription < 2x durée

---

**Conséquences globales ADR-012:**
- Résilience : DLQ + retry évitent perte messages
- Performance : queues prioritaires + scaling
- Observabilité : métriques + alertes proactives
- Maintenance : isolation pannes par queue
- Coût : RabbitMQ léger (< 512 MB RAM pour MVP)

---

### ADR-013: Notification System

**Status:** ✅ ACCEPTÉ

**Context:** Notifier l'utilisateur des événements importants (digestion complétée, concordance détectée, todo reminder) sans spammer.

**Decision:** Local notifications MVP, push notifications Post-MVP, opt-in/opt-out granulaire

**Architecture Notifications :**

```typescript
// Types de notifications
enum NotificationType {
  DIGESTION_COMPLETED = 'digestion_completed',
  TRANSCRIPTION_FAILED = 'transcription_failed',
  CONCORDANCE_DETECTED = 'concordance_detected',
  TODO_REMINDER = 'todo_reminder',
  PROJECT_SUGGESTED = 'project_suggested',
}

// Préférences user (opt-in/opt-out)
interface NotificationPreferences {
  digestionCompleted: boolean;    // Default: true
  concordanceDetected: boolean;   // Default: true
  todoReminders: boolean;         // Default: true
  projectSuggested: boolean;      // Default: true

  // Timing
  quietHours: {
    enabled: boolean;
    start: string;  // "22:00"
    end: string;    // "08:00"
  };
}
```

**MVP : Local Notifications Uniquement**

```typescript
// Mobile (Expo Notifications)
import * as Notifications from 'expo-notifications';

class NotificationService {
  async scheduleLocal(
    type: NotificationType,
    title: string,
    body: string,
    data?: any,
    triggerSeconds: number = 0
  ) {
    // Vérifier préférences user
    const prefs = await this.getPreferences();
    if (!this.isEnabled(type, prefs)) return;

    // Vérifier quiet hours
    if (this.isQuietHours(prefs.quietHours)) {
      // Reporter après quiet hours
      triggerSeconds = this.calculateDelayAfterQuietHours(prefs.quietHours);
    }

    await Notifications.scheduleNotificationAsync({
      content: {
        title,
        body,
        data,
        sound: true,
        badge: 1,
      },
      trigger: triggerSeconds === 0
        ? null  // Immédiat
        : { seconds: triggerSeconds }
    });
  }

  // Exemples d'usage
  async notifyDigestionCompleted(captureId: string) {
    await this.scheduleLocal(
      NotificationType.DIGESTION_COMPLETED,
      '✨ Capture digérée',
      'Votre capture a été analysée et des idées ont été extraites',
      { captureId },
      0  // Immédiat
    );
  }

  async notifyTodoReminder(todo: Todo) {
    const delay = this.calculateDelayUntilDeadline(todo.deadline);

    await this.scheduleLocal(
      NotificationType.TODO_REMINDER,
      '⏰ Rappel',
      todo.description,
      { todoId: todo.id },
      delay
    );
  }

  async notifyProjectSuggested(project: Project) {
    await this.scheduleLocal(
      NotificationType.PROJECT_SUGGESTED,
      '🌱 Nouveau pattern détecté',
      `"${project.name}" - ${project.ideaIds.length} idées connexes`,
      { projectId: project.id },
      0
    );
  }
}
```

**Post-MVP : Push Notifications (Firebase)**

```typescript
// Backend envoie push via FCM
class PushNotificationService {
  async sendPush(
    userId: string,
    type: NotificationType,
    title: string,
    body: string,
    data?: any
  ) {
    const user = await this.userService.findById(userId);

    // Récupérer FCM token
    const fcmToken = user.fcmToken;
    if (!fcmToken) return;  // User pas enregistré pour push

    // Vérifier préférences
    const prefs = await this.preferencesService.get(userId);
    if (!this.isEnabled(type, prefs)) return;

    // Envoyer via FCM
    await this.fcm.send({
      token: fcmToken,
      notification: {
        title,
        body,
      },
      data,
      android: {
        priority: 'high',
        notification: {
          sound: 'default',
          channelId: 'pensine_default',
        },
      },
      apns: {
        payload: {
          aps: {
            sound: 'default',
            badge: 1,
          },
        },
      },
    });
  }
}
```

**Rationale :**
- MVP : local notifications suffisent (mono-user, app ouverte fréquemment)
- Post-MVP : push pour engagement (concordance détectée pendant app fermée)
- Opt-in/opt-out : respect préférences user
- Quiet hours : pas de spam nocturne

---

### ADR-014: Storage Management

**Status:** ✅ ACCEPTÉ

**Context:** Gérer le stockage audio (mobile + cloud) avec retention policy et cleanup automatique.

**Decision:** 3 décisions pour optimisation stockage

---

#### 14.1 - Audio Retention Policy

**Decision:** Mobile 30 jours, Cloud permanent (avec compression)

**Lifecycle Audio :**

```
Capture → [Mobile 64kbps] → Upload → [Cloud 32kbps compressed]
   ↓                              ↓
Cleanup après 30j          Permanent (tant que compte actif)
```

**Mobile Cleanup :**

```typescript
class StorageManager {
  // Cron quotidien à 3h du matin
  @Cron('0 3 * * *')
  async cleanupOldAudio() {
    const threshold = Date.now() - (30 * 24 * 60 * 60 * 1000);  // 30 jours

    const toCleanup = await db.captures.query(
      Q.where('upload_state', 'completed'),  // Upload réussi
      Q.where('captured_at', Q.lt(threshold)),  // > 30 jours
      Q.where('audio_local_uri', Q.notEq(null))  // Fichier existe
    );

    let freedSpace = 0;

    for (const capture of toCleanup) {
      try {
        // Récupérer taille fichier avant suppression
        const fileInfo = await FileSystem.getInfoAsync(capture.audioLocalUri);
        if (fileInfo.exists) {
          freedSpace += fileInfo.size;

          // Supprimer fichier
          await FileSystem.deleteAsync(capture.audioLocalUri, { idempotent: true });

          // Mettre à jour record (garder metadata)
          await capture.update(c => {
            c.audioLocalUri = null;
          });
        }
      } catch (error) {
        this.logger.error('Cleanup failed', { captureId: capture.id, error });
      }
    }

    this.logger.info(`Cleanup freed ${(freedSpace / 1024 / 1024).toFixed(2)} MB`);
  }

  // User peut déclencher cleanup manuel
  async cleanupManual() {
    // Même logique mais avec confirmation user
  }
}
```

**Cloud Permanent avec Compression :**

```typescript
// Backend : compression post-upload
class AudioStorageService {
  async handleAudioUpload(file: Express.Multer.File, captureId: string) {
    // 1. Upload original temporaire
    const originalKey = `audio/temp/${captureId}.m4a`;
    await this.s3.upload(originalKey, file.buffer);

    // 2. Compression FFmpeg (64kbps → 32kbps)
    const compressed = await this.compressAudio(file.buffer);

    // 3. Upload compressed permanent
    const permanentKey = `audio/${captureId}.m4a`;
    await this.s3.upload(permanentKey, compressed);

    // 4. Supprimer original
    await this.s3.delete(originalKey);

    // 5. Retourner URL permanent
    return this.s3.getSignedUrl(permanentKey);
  }

  private async compressAudio(buffer: Buffer): Promise<Buffer> {
    return new Promise((resolve, reject) => {
      ffmpeg(buffer)
        .audioCodec('aac')
        .audioBitrate('32k')
        .audioChannels(1)
        .on('end', () => resolve(outputBuffer))
        .on('error', reject)
        .run();
    });
  }
}
```

**Rationale :**
- 30 jours mobile : balance stockage/re-écoute (user peut re-écouter récent)
- Cloud permanent : aucune perte de données, compliance RGPD
- Compression cloud : économie 50% stockage (64kbps → 32kbps)

---

#### 14.2 - Quotas Utilisateur

**Decision:** Pas de limite MVP, monitoring usage, quotas Post-MVP si abuse

**Monitoring Usage :**

```typescript
// Backend : tracker usage par user
interface UserStorageStats {
  userId: string;
  capturesCount: number;
  totalAudioSizeMB: number;
  averageCaptureSizeMB: number;
  oldestCapture: Date;
  newestCapture: Date;
}

class StorageAnalytics {
  @Cron('0 0 * * 0')  // Hebdomadaire (dimanche minuit)
  async computeStorageStats() {
    const users = await this.userService.findAll();

    for (const user of users) {
      const stats = await this.computeUserStats(user.id);

      // Alerter si usage anormal
      if (stats.totalAudioSizeMB > 5000) {  // > 5 GB
        await this.alertService.send({
          severity: 'info',
          message: `User ${user.id} has ${stats.totalAudioSizeMB} MB audio`,
          userId: user.id,
        });
      }

      // Stocker stats pour analytics
      await this.analyticsService.save(stats);
    }
  }
}
```

**Quotas Post-MVP (si nécessaire) :**

```typescript
const QUOTAS = {
  free: {
    maxCaptures: null,        // Illimité
    maxAudioSizeMB: 1000,     // 1 GB
    maxAudioDurationMin: null, // Illimité
  },

  premium: {
    maxCaptures: null,
    maxAudioSizeMB: 10000,    // 10 GB
    maxAudioDurationMin: null,
  },
};

class QuotaService {
  async checkQuota(userId: string, audioSizeMB: number): Promise<boolean> {
    const user = await this.userService.findById(userId);
    const quota = QUOTAS[user.plan];

    const currentUsage = await this.getCurrentUsage(userId);

    if (currentUsage.totalAudioSizeMB + audioSizeMB > quota.maxAudioSizeMB) {
      // Quota dépassé
      await this.notificationService.send(userId, {
        type: 'quota_exceeded',
        message: 'Quota stockage atteint',
      });
      return false;
    }

    return true;
  }
}
```

**Rationale MVP sans quotas :**
- Mono-user = usage limité naturellement
- 5 captures/jour × 1min × 365j × 32kbps = ~350 MB/an (acceptable)
- Monitoring suffit pour détecter abuse
- Quotas ajoutés seulement si problème identifié

---

#### 14.3 - Media Optimization

**Decision:** Thumbnails blurhash, lazy loading, adaptive quality

**Blurhash pour Perception Vitesse :**

```typescript
// Backend : générer blurhash lors digestion
class DigestionService {
  async process(capture: Capture) {
    // Si capture contient image
    if (capture.type === 'image') {
      const blurhash = await this.generateBlurhash(capture.imageFile);

      return {
        ...digest,
        blurhash,  // String compact (20-30 chars)
      };
    }
  }

  private async generateBlurhash(imageBuffer: Buffer): Promise<string> {
    const { data, info } = await sharp(imageBuffer)
      .resize(32, 32, { fit: 'inside' })
      .ensureAlpha()
      .raw()
      .toBuffer({ resolveWithObject: true });

    return encode(
      new Uint8ClampedArray(data),
      info.width,
      info.height,
      4,  // Components X
      3   // Components Y
    );
  }
}

// Mobile : afficher blurhash pendant chargement
<Image
  source={{ uri: capture.imageUrl }}
  placeholder={{ blurhash: capture.blurhash }}
  transition={200}
  style={styles.image}
/>
```

**Adaptive Quality (Post-MVP) :**

```typescript
// Servir audio qualité adaptée à connexion
class AdaptiveAudioService {
  async getAudioUrl(captureId: string, quality: 'low' | 'medium' | 'high') {
    const qualityMap = {
      low: '16kbps',     // 2G/slow 3G
      medium: '32kbps',  // 3G/4G
      high: '64kbps',    // WiFi/5G
    };

    const key = `audio/${captureId}_${qualityMap[quality]}.m4a`;
    return this.s3.getSignedUrl(key);
  }
}

// Mobile détecte connexion
const quality = netInfo.type === 'wifi' ? 'high' : 'medium';
const audioUrl = await api.getAudioUrl(captureId, quality);
```

**Rationale :**
- Blurhash : placeholder immédiat (perception vitesse)
- Lazy loading : économie bande passante
- Adaptive quality : optimisation connexion lente

---

**Conséquences globales ADR-014:**
- Stockage mobile optimisé : cleanup 30j automatique
- Cloud économique : compression 50%
- Quotas : monitoring sans limite artificielle MVP
- UX : blurhash + lazy loading = fluidité

---

### ADR-015: Observability Strategy

**Status:** ✅ ACCEPTÉ

**Context:** Monitorer performance, erreurs et usage pour détecter problèmes avant impact utilisateur.

**Decision:** Stack observability complète (Logs + Metrics + Tracing + Alertes)

---

#### 15.1 - Logging Strategy

**Decision:** Structured logging (JSON), niveaux appropriés, rotation automatique

**Niveaux de Log :**

```typescript
// NestJS Winston logger
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'pensine-backend',
    environment: process.env.NODE_ENV,
  },
  transports: [
    // Console (développement)
    new winston.transports.Console({
      format: winston.format.simple(),
    }),

    // Fichier (production)
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      maxsize: 10485760,  // 10 MB
      maxFiles: 5,
      tailable: true,
    }),

    new winston.transports.File({
      filename: 'logs/combined.log',
      maxsize: 10485760,
      maxFiles: 10,
      tailable: true,
    }),
  ],
});
```

**Structured Logging :**

```typescript
// ✅ Bon (structured)
logger.info('Digestion completed', {
  captureId: 'c123',
  userId: 'u456',
  duration: 12.3,
  todosExtracted: 3,
  ideasExtracted: 2,
});

// ❌ Mauvais (unstructured)
logger.info(`Digestion completed for capture c123 (12.3s)`);
```

**Sensitive Data Filtering :**

```typescript
// Middleware de filtrage
class SensitiveDataFilter {
  filter(log: any): any {
    const filtered = { ...log };

    // Masquer tokens
    if (filtered.token) {
      filtered.token = this.maskToken(filtered.token);
    }

    // Masquer emails
    if (filtered.email) {
      filtered.email = this.maskEmail(filtered.email);
    }

    // Masquer transcriptions (PII potentiel)
    if (filtered.transcription) {
      filtered.transcription = '[REDACTED]';
    }

    return filtered;
  }

  private maskToken(token: string): string {
    return token.substring(0, 8) + '...';
  }

  private maskEmail(email: string): string {
    const [user, domain] = email.split('@');
    return `${user.substring(0, 2)}***@${domain}`;
  }
}
```

**Rationale :**
- JSON structured : queryable, aggregable
- Rotation automatique : évite disques pleins
- PII filtering : conformité RGPD
- Niveaux appropriés : debug (dev), info (prod), error (toujours)

---

#### 15.2 - Metrics & Monitoring

**Decision:** Prometheus + Grafana, métriques RED (Rate, Errors, Duration)

**Métriques Backend (Prometheus) :**

```typescript
// NestJS Prometheus metrics
@Injectable()
class MetricsService {
  private readonly counters = {
    capturesCreated: new Counter({
      name: 'captures_created_total',
      help: 'Total captures created',
      labelNames: ['type'],  // audio, text, image
    }),

    digestionsCompleted: new Counter({
      name: 'digestions_completed_total',
      help: 'Total digestions completed',
    }),

    digestionsErrored: new Counter({
      name: 'digestions_errored_total',
      help: 'Total digestions failed',
      labelNames: ['error_type'],
    }),
  };

  private readonly gauges = {
    activeUsers: new Gauge({
      name: 'active_users',
      help: 'Currently active users',
    }),

    queueDepth: new Gauge({
      name: 'queue_depth',
      help: 'Messages in queue',
      labelNames: ['queue'],
    }),
  };

  private readonly histograms = {
    digestionDuration: new Histogram({
      name: 'digestion_duration_seconds',
      help: 'Digestion processing time',
      buckets: [1, 5, 10, 20, 30, 60],  // Secondes
    }),

    apiLatency: new Histogram({
      name: 'http_request_duration_seconds',
      help: 'HTTP request latency',
      labelNames: ['method', 'route', 'status'],
      buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
    }),
  };

  // Instrumentation
  recordCapture(type: string) {
    this.counters.capturesCreated.inc({ type });
  }

  recordDigestionDuration(seconds: number) {
    this.histograms.digestionDuration.observe(seconds);
  }

  recordApiCall(method: string, route: string, status: number, duration: number) {
    this.histograms.apiLatency.observe({ method, route, status }, duration);
  }
}
```

**Dashboard Grafana :**

```
📊 Pensine - Backend Metrics

┌─────────────────────────────────────────┐
│ Request Rate (req/min)                  │
│ ▂▃▅▇█▇▅▃▂ Current: 45 req/min          │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Error Rate (%)                          │
│ ▁▁▁▂▁▁▁▁▁ Current: 0.5%                │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ P50 / P95 / P99 Latency                 │
│ Digestion: 8s / 18s / 28s               │
│ API: 50ms / 120ms / 250ms               │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Queue Depth (messages)                  │
│ Digestion: 12                           │
│ Transcription: 3                        │
│ Concordance: 0                          │
└─────────────────────────────────────────┘
```

**Rationale :**
- RED metrics : standard SRE (Google)
- Histograms : P95/P99 latency (pas juste moyenne)
- Labels : segmentation par type/route/status

---

#### 15.3 - Error Tracking

**Decision:** Sentry pour crash reports + error aggregation

**Sentry Configuration :**

```typescript
// Backend
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,  // 10% traces (coût)

  beforeSend(event, hint) {
    // Filter PII
    if (event.request?.data) {
      event.request.data = filterSensitiveData(event.request.data);
    }
    return event;
  },

  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
    new Sentry.Integrations.Prisma({ client: prisma }),
  ],
});

// Mobile
Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  enableInExpoDevelopment: false,
  debug: false,

  beforeSend(event) {
    // User opt-out crash reports
    if (!userConsents.crashReports) {
      return null;  // Ne pas envoyer
    }
    return event;
  },
});
```

**Error Contexte Enrichi :**

```typescript
// Ajouter contexte métier aux erreurs
try {
  await this.digestionService.process(capture);
} catch (error) {
  Sentry.withScope(scope => {
    scope.setContext('capture', {
      id: capture.id,
      type: capture.type,
      userId: capture.userId,
      createdAt: capture.createdAt,
    });

    scope.setTag('operation', 'digestion');
    scope.setLevel('error');

    Sentry.captureException(error);
  });

  throw error;
}
```

**Rationale :**
- Sentry : standard industrie, gratuit < 5k events/mois
- Error aggregation : déduplication automatique
- Context enrichi : debug plus rapide
- Opt-out : respect consentement user (crash reports optionnel)

---

#### 15.4 - Performance Monitoring

**Decision:** APM léger (Sentry Performance), alertes seuils critiques

**Sentry Performance Monitoring :**

```typescript
// Tracer opérations critiques
const transaction = Sentry.startTransaction({
  op: 'digestion',
  name: 'Digest Capture',
});

try {
  // 1. Transcription span
  const transcriptionSpan = transaction.startChild({
    op: 'transcription',
    description: 'Whisper transcription',
  });
  const transcription = await this.whisper.transcribe(audio);
  transcriptionSpan.finish();

  // 2. LLM span
  const llmSpan = transaction.startChild({
    op: 'llm',
    description: 'GPT-4o-mini digestion',
  });
  const digest = await this.llm.digest(transcription);
  llmSpan.finish();

  transaction.setStatus('ok');
} catch (error) {
  transaction.setStatus('internal_error');
  throw error;
} finally {
  transaction.finish();
}
```

**Alertes Performance :**

```typescript
const PERFORMANCE_ALERTS = {
  digestionSlow: {
    condition: 'p95(digestion_duration) > 30s',
    severity: 'warning',
    message: 'Digestion P95 dépasse 30s (NFR)',
  },

  apiSlow: {
    condition: 'p95(api_latency) > 1s',
    severity: 'warning',
    message: 'API P95 dépasse 1s',
  },

  transcriptionSlow: {
    condition: 'p95(transcription_duration) > audio_duration * 2',
    severity: 'warning',
    message: 'Transcription P95 dépasse 2x durée audio (NFR)',
  },
};
```

**Rationale :**
- APM léger : Sentry Performance (pas Datadog coûteux pour MVP)
- Distributed tracing : debug problèmes cross-service
- Alertes sur NFRs : respecter contraintes performance

---

**Conséquences globales ADR-015:**
- Observabilité complète : logs + metrics + tracing + errors
- Détection proactive : alertes avant impact user
- Debug rapide : contexte enrichi sur erreurs
- Coût maîtrisé : stack gratuit/low-cost MVP (Prometheus + Grafana + Sentry free tier)

---

## Step 5 Architectural Decisions - COMPLET ✅

**Décisions finalisées :**
1. ✅ ADR-009 : Sync Patterns (6 sous-décisions)
2. ✅ ADR-010 : Security & Encryption (5 sous-décisions)
3. ✅ ADR-011 : Performance Optimization (3 sous-décisions)
4. ✅ ADR-012 : Queue Management RabbitMQ (4 sous-décisions)
5. ✅ ADR-013 : Notification System
6. ✅ ADR-014 : Storage Management (3 sous-décisions)
7. ✅ ADR-015 : Observability Strategy (4 sous-décisions)

**Total : 25 sous-décisions architecturales documentées**

---


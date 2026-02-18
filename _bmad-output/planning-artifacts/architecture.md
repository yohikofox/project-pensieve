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
- **OP-SQLite** : Base locale offline-first (SUPERSEDES WatermelonDB - see ADR-018)
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

**Local Database:** OP-SQLite ⚠️ **SUPERSEDES WatermelonDB** (see ADR-018)
- **Rationale:** JSI-native, performance supérieure (4× vs WatermelonDB), maintenance active
- **Migration reason:** WatermelonDB JSI incompatibility + maintenance abandonnée (découvert Story 2.1)
- **Trade-off:** Sync protocol manuel (Epic 6) vs déblocage technique immédiat
- **Alternatives rejetées:**
  - WatermelonDB (JSI incompatible - bloquant)
  - Realm (bundle size 4.8 MB + Atlas lock-in)
  - SQLite raw (verbeux, pas de React Native wrapper optimisé)
- **Référence:** ADR-018 pour détails migration complète

**Transcription:** Whisper.rn ou module custom
- **Rationale:** Transcription 100% locale (confidentialité), pas de dépendance réseau
- **Contrainte:** Modèle ~500 Mo téléchargé post-install

**Language:** TypeScript strict

**HTTP Client:** fetch natif + wrapper custom (ADR-025)
- **Rationale:** Bundle size (-13 KB vs axios), ownership total, Node 22 a fetch natif
- **Mobile:** `fetchWithRetry` wrapper (~100 lignes) avec retry Fibonacci + timeout
- **Backend:** fetch natif (Node 22 built-in)
- **Web:** fetch natif (Next.js optimisations auto-cache)
- **Alternative rejetée:** axios (bundle size, dépendance externe, redondant Node 22)
- **Référence:** ADR-025 pour détails complets

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
- **Implémentation:** ~~WatermelonDB~~ OP-SQLite + backend sync endpoint
- ⚠️ **UPDATE (2026-01-22):** Voir [ADR-018](./adrs/ADR-018-migration-watermelondb-opsqlite.md) - Migration WatermelonDB → OP-SQLite
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

## Architectural Decisions Record (ADR)

Toutes les décisions architecturales ont été documentées sous forme d'ADRs séparés dans le dossier [`/adrs/`](./adrs/).

**Pour l'index complet et tous les ADRs, voir : [Index ADR](./adrs/README.md)**

**Total : 25 ADRs documentés (40+ sous-décisions architecturales)**

---

### Quick Links - ADRs par Catégorie

**Domain Architecture (ADR-001 à ADR-008) :**
- [ADR-001](./adrs/ADR-001-aggregate-granularity.md) - Aggregate Granularity : Aggregates Séparés
- [ADR-002](./adrs/ADR-002-normalization-domain-service.md) - Normalization : Domain Service
- [ADR-003](./adrs/ADR-003-sync-infrastructure.md) - Sync : Infrastructure
- [ADR-004](./adrs/ADR-004-digestion-ia-single-llm-call.md) - Digestion IA : Un seul appel LLM
- [ADR-005](./adrs/ADR-005-capture-vide-stockee.md) - Capture Vide : Stockée
- [ADR-006](./adrs/ADR-006-association-manuelle-post-mvp.md) - Association Manuelle Capture ↔ Todo (Post-MVP)
- [ADR-007](./adrs/ADR-007-from-scratch-approach.md) - From Scratch Approach (pas de starter full-stack)
- [ADR-008](./adrs/ADR-008-anti-corruption-layer.md) - Anti-Corruption Layer (ACL) à la Frontière Mobile/Backend

**Infrastructure & Technical Decisions (ADR-009 à ADR-027) :**
- [ADR-009](./adrs/ADR-009-sync-patterns.md) - Sync Patterns (6 sous-décisions)
- [ADR-010](./adrs/ADR-010-security-encryption.md) - Security & Encryption (5 sous-décisions)
- [ADR-011](./adrs/ADR-011-performance-optimization.md) - Performance Optimization (3 sous-décisions)
- [ADR-012](./adrs/ADR-012-queue-management.md) - Queue Management RabbitMQ (4 sous-décisions)
- [ADR-013](./adrs/ADR-013-notification-system.md) - Notification System
- [ADR-014](./adrs/ADR-014-storage-management.md) - Storage Management (3 sous-décisions)
- [ADR-015](./adrs/ADR-015-observability-strategy.md) - Observability Strategy (4 sous-décisions)
- [ADR-016](./adrs/ADR-016-hybrid-architecture.md) - Hybrid Architecture : Cloud Auth + Homelab Storage
- [ADR-017](./adrs/ADR-017-ioc-di-strategy.md) - Dependency Injection & IoC Container Strategy (2 sous-décisions)
- [ADR-018](./adrs/ADR-018-migration-watermelondb-opsqlite.md) - Migration WatermelonDB → OP-SQLite ⚠️ **CRITIQUE** (Supersedes Technology Stack - Local Database)
- [ADR-019](./adrs/ADR-019-eventbus-domain-events.md) - EventBus Architecture - Domain Events avec RxJS
- [ADR-020](./adrs/ADR-020-background-processing-strategy.md) - Background Processing Strategy - expo-task-manager
- [ADR-021](./adrs/ADR-021-di-lifecycle-transient-first.md) - DI Lifecycle Strategy - Transient First (Révision ADR-017)
- [ADR-022](./adrs/ADR-022-state-persistence-opsqlite.md) - State Persistence Strategy - OP-SQLite for All State
- [ADR-023](./adrs/ADR-023-error-handling-strategy.md) - Stratégie Unifiée de Gestion des Erreurs - Result Pattern
- [ADR-024](./adrs/ADR-024-clean-code-standards.md) - Standards Clean Code Appliqués au Projet Pensieve
- [ADR-025](./adrs/ADR-025-http-client-strategy.md) - HTTP Client Strategy - fetch natif + wrapper custom (Supersedes Technology Stack - HTTP Client)
- [ADR-026](./adrs/ADR-026-backend-data-model-design-rules.md) - Backend Data Model Design Rules - Règles canoniques MCD
- [ADR-027](./adrs/ADR-027-unit-cache-strategy.md) - Unit Cache Strategy - Cache unitaire opt-in par héritage de repository ⚠️ **NOUVEAU**

---

### ADRs Critiques pour Implementation

**Epic 2 (Capture Audio) :**
- ⚠️ [ADR-018](./adrs/ADR-018-migration-watermelondb-opsqlite.md) - OP-SQLite (database layer)
- [ADR-017](./adrs/ADR-017-ioc-di-strategy.md) - TSyringe DI (mobile architecture)
- [ADR-011](./adrs/ADR-011-performance-optimization.md) - Performance patterns

**Epic 3 (Transcription) :**
- [ADR-001](./adrs/ADR-001-aggregate-granularity.md) - Aggregate boundaries
- [ADR-002](./adrs/ADR-002-normalization-domain-service.md) - Normalization as Domain Service

**Epic 4 (Digestion IA) :**
- [ADR-004](./adrs/ADR-004-digestion-ia-single-llm-call.md) - Single LLM call strategy
- [ADR-012](./adrs/ADR-012-queue-management.md) - RabbitMQ queue patterns

**Epic 6 (Sync) :**
- [ADR-009](./adrs/ADR-009-sync-patterns.md) - Sync protocol & conflict resolution
- [ADR-003](./adrs/ADR-003-sync-infrastructure.md) - Sync as infrastructure
- [ADR-008](./adrs/ADR-008-anti-corruption-layer.md) - ACL mobile ↔ backend
- ⚠️ [ADR-025](./adrs/ADR-025-http-client-strategy.md) - HTTP Client (fetch natif, remplace axios)

**Backend (tous epics) :**
- ⚠️ [ADR-026](./adrs/ADR-026-backend-data-model-design-rules.md) - Data Model Design Rules (PKs, référentiels, soft delete, multi-tenancy, TypeORM)
- ⚠️ [ADR-027](./adrs/ADR-027-unit-cache-strategy.md) - Unit Cache Strategy (cache unitaire opt-in, namespace Redis, invalidation event-driven)

**Security & Operations :**
- [ADR-010](./adrs/ADR-010-security-encryption.md) - Encryption at rest & in transit
- [ADR-015](./adrs/ADR-015-observability-strategy.md) - Monitoring & logging
- [ADR-016](./adrs/ADR-016-hybrid-architecture.md) - Cloud + Homelab architecture

---

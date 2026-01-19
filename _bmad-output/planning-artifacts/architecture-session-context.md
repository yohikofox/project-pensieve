# Architecture Session Context - Pensine
**Date:** 2026-01-12
**Session:** Event Storming & DDD Strategic
**Dernière mise à jour:** 2026-01-12 (après compaction + Event Storming complet)

---

## 🎯 Où on en est

### Workflow Architecture
- ✅ **Step 1 (Init)** : Complété - Documents chargés
- ✅ **Step 2 (Context Analysis)** : Complété - Sauvegardé dans architecture.md
- ✅ **Step 3 (DDD Strategic + Event Storming)** : COMPLÉTÉ - Toutes les 4 phases terminées
- ✅ **Step 4 (Starter Template)** : COMPLÉTÉ - ADR-007 (From Scratch)
- ✅ **Step 5 (Architectural Decisions)** : COMPLÉTÉ - 7 ADRs (25 sous-décisions)

### Documents Chargés
- Product Brief: `_bmad-output/planning-artifacts/product-brief-pensine-2026-01-09.md`
- PRD: `_bmad-output/planning-artifacts/prd.md`
- UX Design: `_bmad-output/planning-artifacts/ux-design-specification.md`
- Liquid Glass: `docs/liquid-glass/site.md`

---

## ✅ Décisions Techniques Prises

### Stack Technique Validé

**Mobile:**
- ✅ React Native avec Expo (custom dev client)
- ✅ WatermelonDB (offline-first local DB avec sync built-in)
- ✅ Whisper.rn ou module custom (transcription locale)
- ✅ TypeScript strict

**Backend:**
- ✅ NestJS (TypeScript)
- ✅ PostgreSQL (data persistence)
- ✅ RabbitMQ (message broker) - pas BullMQ
  - **Raison décisive:** Redis SPOF éliminé (isolation pannes)
  - Durabilité disk-based > Redis
  - Expérience existante utilisateur
  - Plugin delayed message exchange fonctionne bien
- ✅ Redis (cache UNIQUEMENT, pas de queue)
- ✅ NestJS Schedule (cron jobs) → publie vers RabbitMQ

**Architecture:**
- ✅ DDD Tactique + DDD Stratégique (pas Layered Architecture)
- ✅ Domain Events pour communication entre contexts (asynchrone)
- ✅ Event Sourcing : architecture prévue pour ajout futur, pas immédiat MVP
- ✅ Pas de CQRS pour MVP (ajouter si besoin perf lecture/écriture divergent)

**Infrastructure:**
- ✅ Dev/Staging: Docker Compose selfhosted (homelab)
- ✅ Production future: Scaleway ou AWS managed services

---

## 🎨 Event Storming - État Actuel

### Phase en cours : Event Storming Option A
**Objectif:** Identifier Bounded Contexts via Event Storming complet

### Révélations Majeures Durant Event Storming

#### 1. Deux Domaines Métier Parallèles (CRITIQUE)

**Flow A : GTD / Action (court, opérationnel)**
- Capture → Transcription → Digestion → **Todos extraites**
- Workflow: EXTRACTED → LAUNCHED → IN_PROGRESS → COMPLETED
- Métrique: Task completed
- Ubiquitous Language: "Tâche", "Action", "Rappel", "À faire"
- Exemple: "Pense à envoyer facture Mme Micheaux"

**Flow B : Opportunity Incubation (long, stratégique)**
- Capture → Transcription → Digestion → **Ideas extraites**
- Workflow: Idea → Concordance → Pattern → Germination → Crystallization
- Métrique: Idea launched
- Ubiquitous Language: "Idée", "Opportunité", "Pattern", "Germination", "Concordance"
- Exemple: "Pain point compta freelance (3ème mention)"

**❌ Erreur corrigée:** Les todos ne se transforment JAMAIS en idées germées. Ce sont deux domaines séparés.

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

### Domain Events Identifiés (Complet)

#### 1️⃣ Capture Phase
- `ThoughtCaptured` (audio + metadata)

#### 2️⃣ Transcription Phase
- `TranscriptionRequested`
- `TranscriptionStarted`
- `AudioTranscribed`
- `TranscriptionFailed`

#### 3️⃣ Digestion Phase
- `DigestionRequested`
- `DigestionStarted`
- `ThoughtDigested` (contient: summary, tags, todos[], ideas[])
- `TodosExtracted` (0-N todos extraites)
- `IdeasExtracted` (0-N ideas extraites)
- `DigestionFailed`

#### 4️⃣ Connection Phase (Concordance)
- `ConcordanceDetectionRequested`
- `ConcordanceDetected`
- `PatternRecognized`
- `MaturityScoreCalculated`
- `IdeaPromoted` (devient "chaude")

#### 5️⃣ Germination Phase
- `GerminationCriteriaEvaluated`
- `IdeaGerminated`
- `BusinessCaseGenerated`

#### 6️⃣ Action Phase (Todo Lifecycle)
- `TodoCreated` (extraction)
- `TodoLaunched` (user décide d'agir)
- `TodoStarted` (user commence)
- `TodoInProgress`
- `TodoCompleted`
- `TodoAbandoned`
- `TodoPostponed`
- `TodoPriorityChanged`

#### 7️⃣ Sync Phase
- `LocalChangeDetected`
- `SyncRequested`
- `ChangesPushed`
- `ChangesPulled`
- `ConflictDetected`
- `ConflictResolved`
- `SyncCompleted`

#### 8️⃣ User/Session Phase
- `UserRegistered`
- `UserLoggedIn`
- `UserLoggedOut`
- `SessionExpired`

#### 9️⃣ Notification Phase
- `NotificationScheduled`
- `NotificationSent`
- `NotificationDelivered`
- `NotificationFailed`

#### 🔟 Enrichissement Post-Capture (V1.5)
- `EnrichmentRequested`
- `AudioEnrichmentAdded`
- `TextEnrichmentAdded`
- `ThoughtEnriched`
- `ReDigestionTriggered`

#### 1️⃣1️⃣ Brainstorm Guidé (V1.5)
- `BrainstormSessionStarted`
- `BrainstormQuestionAsked`
- `BrainstormAnswerProvided`
- `BrainstormInsightGenerated`
- `ConceptCrystallized`
- `BusinessCaseGenerated`
- `BrainstormSessionCompleted`

#### 1️⃣2️⃣ Partage Filtré (V1.5)
- `IdeaSharedRequested`
- `ShareableDigestGenerated`
- `ShareLinkCreated`
- `IdeaShared`
- `IdeaViewed`
- `CollaborationInviteSent`
- `CollaborationAccepted`

### Bounded Contexts Candidats (à valider)

**Core Domain:**
- **Knowledge Context** : Digestion IA (résumé, extraction todos/ideas)
- **Opportunity Context** : Détection patterns, concordances, germination

**Supporting:**
- **Capture Context** : Capture audio/texte
- **Transcription Context** : Audio → Texte (Whisper)
- **Action Context** : Gestion cycle de vie todos (GTD)

**Generic:**
- **Sync Context** : Synchronisation mobile ↔ cloud
- **Identity Context** : Auth, users

---

## ✅ Event Storming COMPLET - Session du 2026-01-12

### Phase 1 : Domain Events ✅
- Tous les events identifiés (12 phases : Capture, Digestion, Project, Action, Sync, User, Notification, V1.5)
- Sauvegardé dans architecture.md

### Phase 2 : Commands & Aggregates ✅
- Mapping complet Command → Aggregate → Event
- 5 Aggregates core identifiés : Capture, Thought, Todo, Idea, Project
- Sauvegardé dans architecture.md

### Phase 3 : Policies ✅
- 10 policies automatiques MVP définies
- WHEN event THEN command pour chaque réaction auto
- Sauvegardé dans architecture.md

### Phase 4 : Bounded Contexts & Context Map ✅
- 8 contextes identifiés (2 Core, 3 Supporting, 2 Generic, 1 Infrastructure)
- Context Map avec relations upstream/downstream
- ACL : non nécessaires pour MVP (à prévoir pour intégrations externes)
- Sauvegardé dans architecture.md

### Les 6 Questions en Suspens ✅ TOUTES RÉSOLUES

**Q1 - Aggregate Granularity :** Aggregates séparés (ADR-001)
**Q2 - Normalization :** Domain Service stateless (ADR-002)
**Q3 - Sync :** Infrastructure pure (ADR-003)
**Q4 - Digestion IA :** Un seul appel LLM (ADR-004)
**Q5 - Capture vide :** Stockée quand même (ADR-005)
**Q6 - Association manuelle :** Post-MVP via tags (ADR-006)

**Toutes les décisions documentées dans architecture.md avec ADRs.**

---

---

## 📝 Prochaines Étapes

### Step 4 : Starter Template Evaluation (À FAIRE)

**React Native + Expo :**
- Rechercher templates officiels Expo avec TypeScript
- Vérifier support custom dev client
- Évaluer intégration WatermelonDB

**NestJS + DDD :**
- Rechercher starters NestJS avec DDD structure
- Évaluer boilerplates modulaires
- Vérifier intégration RabbitMQ

**Documentation :**
- Documenter choix dans architecture.md
- Justifier selections

---

### Step 5 : Architectural Decisions - ✅ COMPLÉTÉ

**✅ ADR-009 : Sync Patterns** (6 sous-décisions)
- Timing : Balanced (launch + post-action + 15min)
- Conflits : lastPulledAt standard WatermelonDB + per-column merge
- Fichiers : Upload Queue séparée avec retry
- Priority : Priority-based (Captures first)
- Retry : Result Pattern + Fibonacci backoff (cap 5min)
- Schema : Simple versioning (pas de Registry MVP)
- ACL : API Sync Layer obligatoire (mobile ↔ backend)

**✅ ADR-010 : Security & Encryption** (5 sous-décisions)
- Auth : JWT (15min) + Refresh Token (30 jours)
- Mobile encryption : Keychain/Keystore (MVP), SQLCipher (Post-MVP)
- Backend encryption : Disk/Volume (MVP), TDE (Post-MVP)
- TLS/HTTPS : TLS 1.3 obligatoire, Let's Encrypt
- RGPD : Export, soft delete 30j, opt-in consent

**✅ ADR-011 : Performance Optimization** (3 sous-décisions)
- Caching : Redis ciblé (session, profile, digestion, concordance)
- Lazy loading : FlashList, pagination, tabs lazy, blurhash
- Audio : AAC 64kbps mono, compression serveur 32kbps, cleanup 30j

**✅ ADR-012 : Queue Management RabbitMQ** (4 sous-décisions)
- DLQ : Systématique pour chaque queue, monitoring
- Retry : Fibonacci backoff, max 5 attempts, erreurs retryables
- Prioritization : Queues séparées, consumers dédiés par priorité
- Monitoring : Prometheus + Grafana, alertes proactives

**✅ ADR-013 : Notification System**
- MVP : Local notifications uniquement
- Post-MVP : Push (Firebase FCM)
- Opt-in/opt-out granulaire, quiet hours

**✅ ADR-014 : Storage Management** (3 sous-décisions)
- Retention : Mobile 30j cleanup, Cloud permanent (compression 50%)
- Quotas : Pas de limite MVP, monitoring usage
- Optimization : Blurhash, lazy loading, adaptive quality

**✅ ADR-015 : Observability** (4 sous-décisions)
- Logging : Winston structured JSON, rotation, PII filtering
- Metrics : Prometheus + Grafana, RED metrics
- Error tracking : Sentry backend + mobile, opt-out
- Performance : Sentry APM, alertes NFRs

**Total : 7 ADRs majeurs, 25 sous-décisions architecturales documentées**

---

## 🔑 Points Critiques à Se Rappeler

1. **1 Capture → N Todos + M Ideas** (cardinalité critique)
2. **Deux domaines parallèles** : GTD (Action) vs Opportunity (Germination)
3. **Pas de promotion Todo → Idea** (erreur corrigée)
4. **RabbitMQ choisi** (pas BullMQ) pour isolation pannes Redis SPOF
5. **DDD Stratégique + Tactique** (pas seulement tactique)
6. **Event Storming Option A** (full event storming avant BC)
7. **Domain Events** pour communication entre contexts (pas direct calls)

---

## 📚 Fichiers Importants

- **Architecture en cours:** `_bmad-output/planning-artifacts/architecture.md`
- **Ce contexte:** `_bmad-output/planning-artifacts/architecture-session-context.md`
- **Workflow status:** `_bmad-output/planning-artifacts/bmm-workflow-status.yaml`

---

## 🎯 Command de Reprise

**État actuel : Event Storming COMPLET ✅**

Quand on reprend :
1. ✅ Event Storming terminé (4 phases complètes)
2. ✅ Toutes les questions architecturales résolues (6 ADRs documentés)
3. ✅ Décisions sauvegardées dans `architecture.md`
4. ⏳ **Prochaine étape :** Step 4 - Starter Template Evaluation
5. ⏳ **Ensuite :** Step 5+ - Décisions restantes (Sync, Security, Performance, etc.)

**Tout le contexte est sauvegardé. Architecture DDD Strategic complète et documentée.**

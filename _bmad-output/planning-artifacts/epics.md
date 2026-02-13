---
stepsCompleted: [1, 2, 3, 4]
currentStep: completed
workflowStatus: completed
currentEpic: completed
epicsCompleted:
  - epic1: 3 stories (Story 1.1 - 1.3) [v2 Architecture]
  - epic2: 8 stories (Story 2.1 - 2.8)
  - epic3: 6 stories (Story 3.1 - 3.4 + 2 sub-stories)
  - epic4: 4 stories (Story 4.1 - 4.4)
  - epic5: 4 stories (Story 5.1 - 5.4)
epicsPending:
  - epic6: 4 stories (Story 6.1 - 6.4) [Synchronisation]
  - epic7: 3 stories (Story 7.1 - 7.3) [Support & Observability]
  - epic8: 12 stories (Story 8.1 - 8.12) [GitHub Issues Resolution]
  - epic9: 1 story (Story 9.1) [Advanced Notifications]
  - epic10: 1 story (Story 10.1) [Monetization]
totalStories: 46
totalFRsCovered: 31
githubIssuesIntegrated: 23
coveragePercentage: 100
inputDocuments:
  - "_bmad-output/planning-artifacts/prd.md"
  - "_bmad-output/planning-artifacts/architecture.md"
  - "_bmad-output/planning-artifacts/ux-design-specification.md"
  - "https://github.com/yohikofox/pensieve/issues (23 issues ouvertes)"
workflow: "create-epics-and-stories"
agent: "pm + sm"
lastUpdate: "2026-02-13"
---

# Pensine - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for Pensine, decomposing the requirements from the PRD, UX Design, and Architecture into implementable stories.

## Requirements Inventory

### Functional Requirements

**Capture de Pensées (FR1-FR5):**
- FR1: L'utilisateur peut enregistrer une pensée audio en un tap depuis l'écran principal
- FR2: L'utilisateur peut capturer une pensée texte via saisie rapide
- FR3: L'utilisateur peut annuler une capture audio en cours
- FR4: Le système peut capturer de l'audio même sans connexion réseau
- FR5: Le système peut stocker les captures en attente de synchronisation

**Transcription (FR6-FR8):**
- FR6: Le système peut transcrire automatiquement les captures audio en texte
- FR7: Le système peut effectuer la transcription localement sur l'appareil (offline)
- FR8: L'utilisateur peut consulter la transcription complète d'une capture audio

**Digestion IA (FR9-FR13):**
- FR9: Le système peut générer un résumé concis de chaque capture
- FR10: Le système peut extraire les idées clés d'une capture
- FR11: Le système peut digérer une capture texte
- FR12: Le système peut digérer une transcription audio
- FR13: L'utilisateur peut être notifié de la progression des process IA longs

**Todo-list & Actions (FR14-FR20):**
- FR14: Le système peut détecter automatiquement les actions dans une capture lors de la digestion
- FR15: Le système peut extraire pour chaque action : description de tâche, deadline suggérée, priorité
- FR16: L'utilisateur peut voir les todos générées inline avec chaque idée dans le Feed
- FR17: L'utilisateur peut accéder à une vue centralisée de toutes ses actions via le tab "Actions"
- FR18: L'utilisateur peut filtrer les actions (Toutes / À faire / Faites)
- FR19: L'utilisateur peut marquer une action comme complétée (checkbox)
- FR20: L'utilisateur peut accéder à l'idée d'origine depuis une action

**Consultation (FR21-FR24):**
- FR21: L'utilisateur peut consulter la liste de ses captures
- FR22: L'utilisateur peut voir le détail d'une capture (audio, transcription, résumé, idées)
- FR23: L'utilisateur peut consulter ses captures hors connexion
- FR24: L'utilisateur peut distinguer les captures en attente de digestion

**Gestion de Compte (FR25-FR28):**
- FR25: L'utilisateur peut créer un compte
- FR26: L'utilisateur peut se connecter à son compte
- FR27: L'utilisateur peut se déconnecter
- FR28: L'utilisateur peut récupérer l'accès à son compte (mot de passe oublié)

**Synchronisation (FR29-FR31):**
- FR29: Le système peut synchroniser les captures locales vers le cloud au retour du réseau
- FR30: Le système peut synchroniser les données cloud vers l'appareil
- FR31: L'utilisateur peut être informé du statut de synchronisation

### Non-Functional Requirements

**Performance (NFR1-NFR5):**
- NFR1: Temps de démarrage de capture audio < 500ms après tap
- NFR2: Temps de transcription (Whisper on-device) < 2x durée audio
- NFR3: Temps de digestion IA (réseau disponible) < 30s pour capture standard
- NFR4: Temps de chargement liste captures < 1s (cache local)
- NFR5: Latence perçue : L'utilisateur ne doit jamais attendre sans feedback visuel

**Reliability (NFR6-NFR9):**
- NFR6: Perte de données = 0 capture perdue, jamais (tolérance zéro)
- NFR7: Disponibilité capture offline = 100% — capture fonctionne sans réseau
- NFR8: Récupération après crash : Captures en cours sauvegardées automatiquement
- NFR9: Synchronisation au retour réseau : Automatique, sans intervention utilisateur

**Security (NFR10-NFR14):**
- NFR10: Authentification obligatoire pour accès aux données
- NFR11: Chiffrement transit : HTTPS/TLS pour toutes les communications API
- NFR12: Chiffrement stockage : Données sensibles chiffrées au repos (device + cloud)
- NFR13: Isolation données : Un utilisateur ne peut jamais accéder aux données d'un autre
- NFR14: RGPD : Droit d'accès, rectification, suppression des données personnelles

**Scalability (NFR15-NFR16):**
- NFR15: Architecture prête pour 100+ utilisateurs sans refonte
- NFR16: Pas de limite artificielle de stockage MVP, monitoring usage

### Additional Requirements

**From Architecture Document:**

**Starter Template:**
- ADR-007: From Scratch Approach (pas de starter full-stack)
- Partir des CLI officiels (create-expo-app, nest new) sans boilerplate préexistant
- Structure projet custom adaptée aux spécificités de Pensine (Whisper, WatermelonDB, RabbitMQ)

**Technology Stack:**
- Mobile: React Native + Expo (custom dev client), TypeScript strict, WatermelonDB (offline-first)
- Backend: NestJS (TypeScript), PostgreSQL, RabbitMQ (message broker), Redis (cache uniquement)
- ML: Whisper.rn ou module custom pour transcription on-device (~500 Mo modèle post-install)
- LLM: GPT-4o-mini (pressenti) pour digestion IA
- Infrastructure Dev/Staging: Docker Compose selfhosted (homelab)

**Domain-Driven Design:**
- 8 Bounded Contexts identifiés (Core: Knowledge, Opportunity | Supporting: Capture, Normalization, Action | Generic: Identity, Notification | Infrastructure: Sync)
- Aggregates: Capture, Thought, Idea, Project, Todo
- Domain Events pour communication asynchrone entre contextes
- Event Sourcing prévu (pas MVP), CQRS non nécessaire pour MVP

**Security & Encryption:**
- MVP: Disk/Volume Encryption (Infrastructure-level)
- Post-MVP: TDE (Transparent Data Encryption) ou Column-level si régulé
- Chiffrement obligatoire transit + repos
- Aucune donnée audio ne transite par des tiers non contrôlés

**Cross-Cutting Concerns:**
- Offline-First Architecture avec sync bidirectionnelle
- Queue Management persistante (retry logic, prioritization)
- Storage Management: Rétention audio, cleanup automatique
- Notification System: Push vs local, timing, opt-in/out
- Performance mobile: App < 100 Mo, modèle Whisper ~500 Mo post-install

**From UX Design Specification:**

**Liquid Glass Design System:**
- Animations fluides 60fps obligatoires
- Micro-animations pour germination et transitions
- Feedback haptique iOS/Android sur actions critiques
- Animation douce lors de sauvegarde et célébrations discrètes

**Platform-Specific Requirements:**
- iOS: Human Interface Guidelines, haptics, animations SwiftUI-like
- Android: Material Design 3, gestures, animations Jetpack
- Cross-platform: react-native-svg + custom animations + Lottie

**UX-Driven Architecture Constraints:**
- "1-Tap Liberation": Latence capture < 1s impose architecture réactive
- Offline-first radical: Toutes les features core fonctionnent sans réseau
- Détection concordance à chaud: Traitement temps réel post-capture
- Notifications intelligentes: Queue de progression pour process IA longs
- Métaphore "Jardin d'idées": Interface contemplative avec visualisation de maturité

**Device Permissions:**
- Microphone (capture audio MVP)
- Storage (cache offline, fichiers audio MVP)
- Network (sync, API calls, IA distante MVP)
- Camera (capture photo V2)
- Notifications (push process IA longs, rappels MVP)

### FR Coverage Map

**Functional Requirements Coverage:**

- FR1 → Epic 2 (Capture audio 1-tap)
- FR2 → Epic 2 (Capture texte)
- FR3 → Epic 2 (Annuler capture audio)
- FR4 → Epic 2 (Capture offline)
- FR5 → Epic 2 (Stockage en attente sync)
- FR6 → Epic 2 (Transcription automatique)
- FR7 → Epic 2 (Transcription locale)
- FR8 → Epic 2 (Consulter transcription)
- FR9 → Epic 4 (Générer résumé)
- FR10 → Epic 4 (Extraire idées clés)
- FR11 → Epic 4 (Digérer texte)
- FR12 → Epic 4 (Digérer audio)
- FR13 → Epic 4 (Notifications progression)
- FR14 → Epic 4 (Détecter actions auto)
- FR15 → Epic 4 (Extraire description/deadline/priorité)
- FR16 → Epic 5 (Todos inline dans Feed)
- FR17 → Epic 5 (Tab Actions centralisé)
- FR18 → Epic 5 (Filtrer actions)
- FR19 → Epic 5 (Marquer complétée)
- FR20 → Epic 5 (Accéder idée d'origine)
- FR21 → Epic 3 (Consulter liste captures)
- FR22 → Epic 3 (Voir détail capture)
- FR23 → Epic 3 (Consultation offline)
- FR24 → Epic 3 (Distinguer en attente digestion)
- FR25 → Epic 1 (Créer compte)
- FR26 → Epic 1 (Se connecter)
- FR27 → Epic 1 (Se déconnecter)
- FR28 → Epic 1 (Récupération mot de passe)
- FR29 → Epic 6 (Sync local → cloud)
- FR30 → Epic 6 (Sync cloud → local)
- FR31 → Epic 6 (Informer statut sync)

**Coverage Summary:** All 31 FRs are mapped to Epics 1-6.

**Additional Epics:**
- Epic 7 → Support & Observability (feature transverse, non liée à un FR spécifique du PRD)

## Epic List

### Epic 1: Foundation & Authentification

Les utilisateurs peuvent créer un compte, se connecter, et commencer à utiliser Pensine de manière sécurisée.

**FRs couverts:** FR25, FR26, FR27, FR28

**Notes d'implémentation:**
- Setup initial du projet (ADR-007 : from scratch, pas de starter)
- React Native + Expo custom dev client
- NestJS backend
- PostgreSQL + RabbitMQ + Redis
- Structure DDD avec Bounded Contexts
- Identity Context (Generic Subdomain)

---

### Epic 2: Capture & Transcription de Pensées

Les utilisateurs peuvent capturer leurs pensées (audio ou texte) et obtenir une transcription automatique, même hors connexion.

**FRs couverts:** FR1, FR2, FR3, FR4, FR5, FR6, FR7, FR8

**Notes d'implémentation:**
- Capture Context (Supporting Domain)
- Normalization Context (Supporting Domain)
- Whisper on-device (~500 Mo modèle)
- Offline-first avec WatermelonDB
- NFR1 (capture < 500ms), NFR2 (transcription < 2x durée), NFR7 (100% offline)
- Feedback haptique + animations (UX req)

---

### Epic 3: Consultation & Navigation des Captures

Les utilisateurs peuvent consulter leurs captures, voir les transcriptions et naviguer dans leur historique, même offline.

**FRs couverts:** FR21, FR22, FR23, FR24

**Notes d'implémentation:**
- Feed chronologique avec vue détail
- Cache local complet (NFR4 : chargement < 1s)
- Distinction visuelle des captures en attente
- Liquid Glass design system
- Métaphore "Jardin d'idées"

---

### Epic 4: Digestion IA & Extraction d'Insights

Les utilisateurs obtiennent automatiquement un résumé, des idées clés, et des actions détectées dans leurs captures.

**FRs couverts:** FR9, FR10, FR11, FR12, FR13, FR14, FR15

**Notes d'implémentation:**
- Knowledge Context (Core Domain)
- Digestion IA via GPT-4o-mini
- Extraction todos avec description/deadline/priorité
- Notifications progression (NFR13, NFR3 : < 30s)
- Queue RabbitMQ pour process asynchrones
- Action Context (Supporting Domain)

---

### Epic 5: Gestion des Actions (Tab Actions)

Les utilisateurs peuvent gérer toutes leurs actions extraites dans une vue centralisée avec filtres et completion tracking.

**FRs couverts:** FR16, FR17, FR18, FR19, FR20

**Notes d'implémentation:**
- Tab Actions centralisé
- Filtres (Toutes / À faire / Faites)
- Navigation vers idée d'origine
- Action Context (Supporting Domain)
- Animations completion

---

### Epic 6: Synchronisation Multi-Device

Les utilisateurs peuvent synchroniser automatiquement leurs captures et données entre leurs appareils.

**FRs couverts:** FR29, FR30, FR31

**Notes d'implémentation:**
- Sync Infrastructure (pas de Bounded Context métier)
- WatermelonDB sync protocol
- Bidirectionnel (local ↔ cloud)
- NFR9 (automatique sans intervention)
- Informations de statut

---

## Epic 1: Foundation & Authentification

Les utilisateurs peuvent créer un compte, se connecter, et commencer à utiliser Pensine de manière sécurisée.

### Story 1.1: Project Foundation & Infrastructure Setup

As a **developer**,
I want **the foundational project structure with all core infrastructure components configured**,
So that **I have a solid base to implement features following DDD architecture and ADR-007 (from scratch approach)**.

**Acceptance Criteria:**

**Given** I am setting up a new Pensine project from scratch
**When** I initialize the mobile and backend projects
**Then** the following structure is created:

**Mobile (React Native + Expo):**
- Expo custom dev client initialized with TypeScript strict mode
- WatermelonDB configured for offline-first storage
- Project structure follows DDD bounded contexts layout
- Basic navigation structure (React Navigation)

**Backend (NestJS):**
- NestJS project initialized with TypeScript strict
- PostgreSQL database connection configured
- RabbitMQ message broker connected
- Redis cache configured (cache only, not queue)
- DDD folder structure with bounded contexts (Identity, Capture, Knowledge, Opportunity, Action, Normalization, Notification, Sync infrastructure)

**And** Docker Compose file exists for local development (PostgreSQL + RabbitMQ + Redis)
**And** Environment configuration files are set up (.env templates)
**And** README with setup instructions is created
**And** No starter/boilerplate code is used (ADR-007 compliance)

### Story 1.2: User Registration

As a **new user**,
I want **to create an account with email and password**,
So that **my captures are encrypted with my credentials and securely stored**.

**Acceptance Criteria:**

**Given** I am a new user who has not registered yet
**When** I provide a valid email address and a strong password (min 8 chars, 1 uppercase, 1 number)
**Then** a new user account is created in the Identity Context
**And** my password is hashed using bcrypt before storage
**And** a unique user ID is generated
**And** the User entity is persisted in PostgreSQL
**And** an authentication token (JWT) is returned
**And** I am automatically logged in after registration

**Given** I provide an email that already exists
**When** I attempt to register
**Then** I receive an error message "Email already registered"
**And** no duplicate account is created

**Given** I provide an invalid email format or weak password
**When** I attempt to register
**Then** I receive appropriate validation error messages
**And** no account is created

**Given** registration succeeds
**When** the account is created
**Then** encryption keys are derived from my credentials for data protection (NFR12)

### Story 1.3: User Login

As a **registered user**,
I want **to log in with my email and password**,
So that **I can access my encrypted captures securely**.

**Acceptance Criteria:**

**Given** I am a registered user with valid credentials
**When** I provide my correct email and password
**Then** my credentials are validated against the hashed password in the database
**And** a JWT access token is generated and returned
**And** my user session is established
**And** I am redirected to the main app screen

**Given** I provide an incorrect password
**When** I attempt to login
**Then** I receive an error message "Invalid email or password"
**And** no session is created
**And** the failed attempt is logged for security monitoring

**Given** I provide an email that doesn't exist
**When** I attempt to login
**Then** I receive an error message "Invalid email or password" (same message to prevent email enumeration)
**And** no session is created

**Given** I successfully log in
**When** the JWT token is issued
**Then** the token has a reasonable expiration time (e.g., 7 days)
**And** the token includes my user ID for data isolation (NFR13)

### Story 1.4: User Logout

As a **logged-in user**,
I want **to log out of my account**,
So that **my session is terminated and my data is secure on shared devices**.

**Acceptance Criteria:**

**Given** I am currently logged in with an active session
**When** I trigger the logout action
**Then** my JWT token is invalidated (added to Redis blacklist)
**And** my local session data is cleared from the mobile app
**And** I am redirected to the login screen
**And** subsequent API requests with the old token are rejected

**Given** I log out
**When** I close and reopen the app
**Then** I remain logged out and must authenticate again

**Given** I log out
**When** any pending local data exists (offline captures)
**Then** the data remains securely stored locally for future sync
**And** I am warned if unsynchronized data exists (optional notification)

### Story 1.5: Password Recovery

As a **registered user who forgot my password**,
I want **to reset my password via email**,
So that **I can regain access to my account and encrypted captures**.

**Acceptance Criteria:**

**Given** I am a registered user who forgot my password
**When** I request a password reset with my email address
**Then** a unique password reset token is generated with 1-hour expiration
**And** a password reset email is sent to my registered email address
**And** the email contains a secure reset link with the token
**And** the system does not reveal whether the email exists (security best practice)

**Given** I receive the password reset email
**When** I click the reset link within 1 hour
**Then** I am directed to a password reset form
**And** the token is validated before allowing password change

**Given** I am on the password reset form with a valid token
**When** I provide a new strong password (min 8 chars, 1 uppercase, 1 number)
**Then** my password is updated in the database (bcrypt hashed)
**And** the reset token is invalidated immediately
**And** all existing sessions are terminated (security measure)
**And** I receive a confirmation that my password was changed
**And** I can log in with the new password

**Given** I attempt to use an expired or invalid reset token
**When** I try to reset my password
**Then** I receive an error message "Reset link expired or invalid"
**And** I am prompted to request a new reset link

---

## Epic 2: Capture & Transcription de Pensées

Les utilisateurs peuvent capturer leurs pensées (audio ou texte) et obtenir une transcription automatique, même hors connexion.

### Story 2.1: Capture Audio 1-Tap

As a **user**,
I want **to record an audio thought with a single tap from the main screen**,
So that **I can quickly capture my ideas without friction, even without network connectivity**.

**Acceptance Criteria:**

**Given** I am on the main screen of the app
**When** I tap the record button
**Then** audio recording starts within 500ms (NFR1 compliance)
**And** visual feedback is displayed (pulsing red indicator)
**And** haptic feedback is triggered on iOS/Android
**And** a Capture entity is created in WatermelonDB with status "recording"
**And** audio data is streamed to local storage

**Given** I am recording audio
**When** I tap the stop button
**Then** the recording stops immediately
**And** the audio file is saved to device storage
**And** the Capture entity is updated with status "captured" and file path
**And** the audio file metadata (duration, size, timestamp) is stored

**Given** I have no network connectivity
**When** I start and complete an audio recording
**Then** the capture works identically to online mode (FR4, NFR7 compliance)
**And** the Capture entity is marked for future sync
**And** no error is shown to the user

**Given** the app crashes during recording
**When** I reopen the app
**Then** the partial recording is recovered if possible (NFR8 compliance)
**And** I receive a notification about the recovered capture

**Given** microphone permission is not granted
**When** I attempt to record
**Then** I am prompted to grant microphone access
**And** recording only starts after permission is granted

### Story 2.2: Capture Texte Rapide

As a **user**,
I want **to capture a text thought via quick text input**,
So that **I can record ideas that are better suited to typing than speaking**.

**Acceptance Criteria:**

**Given** I am on the main screen of the app
**When** I tap the text capture button
**Then** a text input field appears immediately
**And** the keyboard opens automatically
**And** the cursor is focused in the text field

**Given** I have typed text in the capture field
**When** I tap the save/submit button
**Then** a Capture entity is created in WatermelonDB with type "text"
**And** the text content is stored with the Capture
**And** a timestamp is recorded
**And** the Capture status is set to "captured"
**And** the text input field is cleared for the next capture

**Given** I start typing a text capture
**When** I cancel or navigate away before saving
**Then** I am prompted to confirm discarding unsaved content
**And** the capture is not saved if I confirm discard

**Given** I have no network connectivity
**When** I create and save a text capture
**Then** the capture is saved locally (FR4 compliance)
**And** the Capture entity is marked for future sync
**And** no error is shown to the user

**Given** I submit an empty text field
**When** I attempt to save
**Then** I receive a validation message "Please enter some text"
**And** no empty Capture entity is created

**Given** I create a text capture
**When** the save is successful
**Then** subtle haptic feedback confirms the save (UX requirement)
**And** a brief animation shows the capture being added to the feed

### Story 2.3: Annuler Capture Audio

As a **user**,
I want **to cancel an audio recording in progress**,
So that **I can discard unwanted captures without cluttering my feed**.

**Acceptance Criteria:**

**Given** I am currently recording audio
**When** I tap the cancel button
**Then** the recording stops immediately
**And** the audio file is deleted from device storage
**And** the Capture entity is removed from WatermelonDB
**And** I am returned to the main screen ready for a new capture

**Given** I am recording audio
**When** I swipe down or perform the cancel gesture
**Then** a confirmation prompt appears "Discard this recording?"
**And** if I confirm, the recording is cancelled (same behavior as cancel button)
**And** if I decline, the recording continues

**Given** I cancel a recording
**When** the cancellation is processed
**Then** haptic feedback indicates the cancellation (UX requirement)
**And** a brief animation shows the capture being discarded

**Given** I accidentally tap cancel during recording
**When** the confirmation prompt appears
**Then** I can choose to continue recording
**And** the recording resumes without data loss

**Given** the app is offline
**When** I cancel a recording
**Then** the cancellation works identically to online mode (FR4 compliance)
**And** no orphaned files remain in storage

### Story 2.4: Stockage Offline des Captures

As a **user**,
I want **my captures to be stored securely on my device even without network**,
So that **I never lose a thought and can access them anytime** (NFR6: zero data loss tolerance).

**Acceptance Criteria:**

**Given** I have created captures (audio or text) while offline
**When** the captures are saved
**Then** all Capture entities are persisted in WatermelonDB
**And** audio files are stored in secure device storage
**And** each Capture has a sync status field (pending/synced)
**And** captures marked "pending" are queued for future synchronization

**Given** the app has no network connectivity
**When** I create multiple captures in succession
**Then** all captures are saved locally without errors (NFR7: 100% offline availability)
**And** storage space is monitored to prevent overflow
**And** if storage is critically low, I receive a warning before capturing

**Given** I have offline captures stored locally
**When** I reopen the app while still offline
**Then** all my captures are immediately accessible (NFR4: < 1s load time)
**And** the feed displays all captures with offline indicators
**And** no network errors are shown

**Given** the app crashes with unsynchronized captures
**When** I relaunch the app
**Then** all saved captures are recovered intact (NFR8: crash recovery)
**And** pending sync status is preserved
**And** no data is lost

**Given** I have accumulated many offline captures
**When** storage management runs
**Then** old audio files can be cleaned up based on retention policy
**And** transcriptions and metadata are always retained
**And** I am notified before any cleanup occurs (Additional Requirement: Storage Management)

**Given** captures are encrypted at rest (NFR12)
**When** captures are written to device storage
**Then** audio files and text content are encrypted using device-level encryption
**And** metadata includes encryption status flag

### Story 2.5: Transcription On-Device avec Whisper

As a **user**,
I want **my audio captures to be automatically transcribed into text on my device**,
So that **I can read my thoughts even when offline, without sending my audio to third-party services**.

**Acceptance Criteria:**

**Given** I have completed an audio capture
**When** the audio file is saved
**Then** a transcription job is automatically queued
**And** the Capture entity status is updated to "transcribing"
**And** a background process starts the Whisper transcription

**Given** transcription is running
**When** Whisper processes the audio file
**Then** transcription completes in less than 2x the audio duration (NFR2 compliance)
**And** the transcribed text is stored in the Capture entity
**And** the Capture status is updated to "transcribed"
**And** the original audio file is retained

**Given** the app is offline during transcription
**When** Whisper runs on-device
**Then** transcription works identically to online mode (FR7: local transcription)
**And** no network calls are made
**And** the Whisper model (~500 Mo) is already installed on device

**Given** the Whisper model is not yet installed on first app launch
**When** the user attempts their first audio capture
**Then** I am prompted to download the Whisper model (~500 Mo)
**And** the download progress is displayed
**And** captures can still be saved before the model is ready
**And** transcription is queued until the model is available

**Given** transcription is in progress for a long audio file
**When** processing takes more than 10 seconds
**Then** I receive visual feedback of transcription progress (NFR5: no waiting without feedback)
**And** I can continue using the app while transcription runs in background

**Given** transcription fails due to corrupted audio or technical error
**When** Whisper encounters an error
**Then** the Capture status is updated to "transcription_failed"
**And** the error is logged for debugging
**And** I am notified of the failure with option to retry
**And** the audio file is preserved for manual review or retry

**Given** multiple audio captures are pending transcription
**When** transcription jobs are queued
**Then** they are processed in FIFO order
**And** only one transcription runs at a time to preserve performance
**And** the queue status is visible in the UI

### Story 2.6: Consultation de Transcription

As a **user**,
I want **to view the complete transcription of my audio captures**,
So that **I can read my thoughts in text format and verify transcription accuracy**.

**Acceptance Criteria:**

**Given** I have an audio capture that has been transcribed
**When** I open the capture detail view
**Then** the full transcription text is displayed
**And** the original audio file is available to play
**And** the transcription timestamp is shown
**And** the audio duration is displayed

**Given** I am viewing a transcription
**When** I tap the play button
**Then** the original audio starts playing
**And** the playback controls (play/pause, progress bar) are available
**And** I can listen while reading the transcription

**Given** a capture is still being transcribed
**When** I open the capture detail view
**Then** I see a "Transcription in progress..." indicator
**And** the audio file is available to play
**And** the transcription appears when ready (live update)

**Given** transcription failed for a capture
**When** I open the capture detail view
**Then** I see a "Transcription failed" message
**And** I have the option to retry transcription
**And** the original audio remains playable

**Given** I am offline
**When** I view a transcription
**Then** the transcription is loaded from local cache (FR23 compliance)
**And** all functionality works identically to online mode
**And** no network errors are shown

**Given** I have a text capture (not audio)
**When** I open the capture detail view
**Then** only the text content is displayed (no transcription section)
**And** the capture type is clearly indicated

**Given** the transcription contains formatting or special characters
**When** displayed in the UI
**Then** the text is rendered correctly and readable
**And** line breaks and paragraphs are preserved from Whisper output

---

## Epic 3: Consultation & Navigation des Captures

Les utilisateurs peuvent consulter leurs captures, voir les transcriptions et naviguer dans leur historique, même offline.

### Story 3.1: Feed Chronologique des Captures

As a **user**,
I want **to see a chronological feed of all my captures on the main screen**,
So that **I can quickly browse my recent thoughts and access them instantly, even offline**.

**Acceptance Criteria:**

**Given** I have created multiple captures (audio and text)
**When** I open the main feed screen
**Then** all my captures are displayed in reverse chronological order (newest first)
**And** the feed loads in less than 1 second from local cache (NFR4 compliance)
**And** each capture card shows: type icon (audio/text), timestamp, preview text, status indicator

**Given** I have a mix of audio and text captures
**When** viewing the feed
**Then** audio captures show the first line of transcription as preview (if available)
**And** text captures show the first line of text content as preview
**And** captures without transcription show "Transcribing..." or audio duration

**Given** I am offline
**When** I open the feed
**Then** all locally stored captures are displayed (FR23 compliance)
**And** the feed works identically to online mode
**And** no network errors or loading spinners are shown
**And** an offline indicator is visible in the app header

**Given** I have many captures (50+)
**When** scrolling through the feed
**Then** infinite scroll or pagination is implemented for smooth performance
**And** older captures are lazy-loaded as I scroll
**And** scroll performance remains smooth (60fps, UX requirement)

**Given** I pull down to refresh the feed
**When** network is available
**Then** the feed syncs with cloud data
**And** a refresh animation is shown (Liquid Glass design)
**And** new captures appear at the top

**Given** I have no captures yet
**When** I open the feed
**Then** an empty state message is displayed with onboarding guidance
**And** the capture buttons are prominently highlighted
**And** a welcoming illustration reflects the "Jardin d'idées" metaphor

**Given** the feed is loading for the first time
**When** data is being retrieved from WatermelonDB
**Then** skeleton loading cards are shown (Liquid Glass design)
**And** the transition from skeleton to actual content is smooth (60fps animation)

### Story 3.2: Vue Détail d'une Capture

As a **user**,
I want **to open a capture and see all its details (audio, transcription, metadata)**,
So that **I can review the complete content and context of my thought**.

**Acceptance Criteria:**

**Given** I am viewing the feed
**When** I tap on a capture card
**Then** the capture detail screen opens with smooth transition animation (Liquid Glass design)
**And** the full content is displayed based on capture type

**Given** the capture is an audio capture with transcription
**When** I view the detail screen
**Then** I see: audio player with waveform visualization, full transcription text, timestamp, duration, status badges
**And** the audio player has play/pause, seek controls
**And** playback position is highlighted in the transcription (sync)

**Given** the capture is a text capture
**When** I view the detail screen
**Then** I see: full text content, timestamp, character/word count
**And** the text is formatted with proper line breaks and spacing
**And** the UI adapts to show text-specific layout (no audio player)

**Given** I am viewing a capture detail offline
**When** all content is cached locally
**Then** the detail view loads instantly (FR23 compliance)
**And** all features work identically to online mode
**And** no network-dependent features cause errors

**Given** I am viewing the detail of a capture still being transcribed
**When** transcription completes in the background
**Then** the transcription text appears automatically without refresh (live update)
**And** a subtle animation indicates the new content (germination metaphor)

**Given** I swipe left or right on the detail screen
**When** other captures exist in the feed
**Then** I can navigate to the previous/next capture
**And** the transition is smooth with horizontal swipe animation
**And** the detail content updates accordingly

**Given** I tap the back button or swipe down
**When** returning to the feed
**Then** the feed scrolls to the previously viewed capture
**And** the transition animation is smooth (Liquid Glass design)

**Given** I long-press on the detail screen
**When** the context menu appears
**Then** I can access actions: share, delete, edit (if applicable)
**And** haptic feedback confirms the long-press

### Story 3.3: Distinction Visuelle des Captures en Attente

As a **user**,
I want **to easily identify captures that are still being processed (transcription or digestion)**,
So that **I know which captures are complete and which are still in progress**.

**Acceptance Criteria:**

**Given** I have captures in different processing states
**When** I view the feed
**Then** each capture card displays a clear status indicator
**And** the status badge shows: "Captured", "Transcribing...", "Digesting...", "Ready", or "Failed"
**And** the badge color reflects the status (neutral, in-progress, success, error)

**Given** a capture is being transcribed
**When** displayed in the feed
**Then** a subtle pulsing animation appears on the status badge (Liquid Glass design)
**And** a progress indicator shows transcription advancement (if determinable)
**And** the capture card is slightly dimmed to indicate incompleteness

**Given** a capture has completed transcription but not yet digestion
**When** displayed in the feed
**Then** the status shows "Awaiting digestion" or similar
**And** the visual indicator differentiates it from fully processed captures
**And** the card is partially highlighted to show partial completion

**Given** transcription or digestion fails for a capture
**When** displayed in the feed
**Then** an error badge is prominently displayed with red/warning color
**And** tapping the card shows the error details and retry option
**And** the card stands out visually to draw attention to the issue

**Given** all captures in the feed have different statuses
**When** I scroll through the feed
**Then** I can quickly scan and identify processing states at a glance
**And** status icons are consistent and easily recognizable
**And** color coding follows accessibility guidelines (not color-only indicators)

**Given** a capture completes processing while I'm viewing the feed
**When** the status changes from "in progress" to "ready"
**Then** the status badge updates in real-time without refresh
**And** a subtle "germination" animation celebrates completion (UX metaphor)
**And** haptic feedback signals the completion (optional, can be disabled in settings)

**Given** I filter or sort captures by status
**When** I access filter options
**Then** I can view only: "All", "Processing", "Ready", "Failed"
**And** the feed updates to show only captures matching the selected status

### Story 3.4: Navigation et Interactions dans le Feed

As a **user**,
I want **fluid, intuitive interactions and animations throughout the feed**,
So that **the app feels responsive, delightful, and reflects the "Jardin d'idées" contemplative experience**.

**Acceptance Criteria:**

**Given** I am interacting with the feed
**When** I perform any gesture (tap, swipe, scroll)
**Then** all animations run at 60fps (UX requirement)
**And** haptic feedback is triggered for key actions (iOS/Android native)
**And** the Liquid Glass design system is consistently applied

**Given** I tap on a capture card
**When** the detail view opens
**Then** a smooth hero transition animation morphs the card into the detail view
**And** the animation respects iOS/Android platform conventions
**And** the transition completes in 250-350ms (feels instant)

**Given** I swipe a capture card horizontally
**When** the swipe gesture is detected
**Then** contextual actions appear (archive, delete, share)
**And** the swipe reveals options smoothly with spring physics
**And** haptic feedback confirms the action threshold

**Given** I am scrolling through the feed
**When** new captures appear during scroll
**Then** they animate in with a subtle fade and slide (germination metaphor)
**And** the animation is staggered for a natural organic feel
**And** scroll performance never drops below 60fps

**Given** I perform a long-press on a capture card
**When** the press is held for 300ms
**Then** a contextual menu appears with smooth scale animation
**And** haptic feedback signals menu activation
**And** the background subtly blurs (Liquid Glass effect)
**And** menu options are: Share, Delete, Pin, Mark as favorite

**Given** I use platform-specific gestures
**When** on iOS: I use edge swipe to go back
**When** on Android: I use back button or swipe from edge
**Then** the navigation respects platform conventions
**And** the back transition is smooth and predictable

**Given** visual elements reflect the "Jardin d'idées" metaphor
**When** I view the feed over time
**Then** captures visually "grow" or "mature" with subtle visual indicators
**And** older captures may show growth rings or maturity badges
**And** the overall aesthetic is calming and contemplative (not rushed or anxiety-inducing)

**Given** I interact with empty states or placeholders
**When** no captures exist or filters show no results
**Then** illustrations are beautiful and on-brand (garden metaphor)
**And** micro-animations add life (butterflies, gentle breeze effects)
**And** the empty state guides me to take the next action

---

## Epic 4: Digestion IA & Extraction d'Insights

Les utilisateurs obtiennent automatiquement un résumé, des idées clés, et des actions détectées dans leurs captures.

### Story 4.1: Queue Asynchrone pour Digestion IA

As a **developer**,
I want **a robust asynchronous queue system for AI digestion jobs**,
So that **AI processing runs reliably in the background without blocking the user experience**.

**Acceptance Criteria:**

**Given** the backend infrastructure is set up
**When** RabbitMQ is configured
**Then** a dedicated queue "digestion-jobs" is created
**And** a dead-letter queue "digestion-failed" is configured for retry logic
**And** queue persistence is enabled to survive server restarts
**And** connection pooling is optimized for concurrent job processing

**Given** a capture completes transcription (or text capture is created)
**When** the capture is ready for digestion
**Then** a digestion job is automatically published to the RabbitMQ queue
**And** the job payload includes: capture ID, user ID, content type, priority
**And** the Capture entity status is updated to "queued_for_digestion"
**And** the job is persisted in the queue

**Given** multiple digestion jobs are in the queue
**When** workers process jobs
**Then** jobs are processed in priority order (user-initiated > auto-background)
**And** concurrent processing is limited to prevent API rate limiting (max 3 concurrent GPT calls)
**And** each job has a timeout of 60 seconds (2x the NFR3 target of 30s)

**Given** a digestion job is being processed
**When** the worker picks up the job
**Then** the Capture status is updated to "digesting"
**And** a timestamp records when processing started
**And** progress updates are published to a real-time channel for UI notifications

**Given** a digestion job fails due to API error or timeout
**When** the failure is detected
**Then** the job is moved to the dead-letter queue
**And** retry logic attempts up to 3 retries with exponential backoff
**And** the Capture status is updated to "digestion_failed" after max retries
**And** the error details are logged for debugging

**Given** the system experiences high load
**When** many digestion jobs are queued
**Then** queue depth is monitored and alerts are triggered if backlog exceeds threshold
**And** users receive feedback about estimated processing time (NFR5: no waiting without feedback)
**And** the system gracefully degrades without crashing

**Given** a user creates captures while offline
**When** network connectivity returns
**Then** queued captures are automatically submitted for digestion
**And** jobs are prioritized by user activity recency
**And** batch processing optimizes API calls when possible

### Story 4.2: Digestion IA - Résumé et Idées Clés

As a **user**,
I want **my captures to be automatically analyzed by AI to generate a concise summary and extract key ideas**,
So that **I can quickly understand the essence of my thoughts without rereading everything**.

**Acceptance Criteria:**

**Given** a digestion job is picked up by the worker
**When** the capture content (text or transcription) is sent to GPT-4o-mini
**Then** a prompt requests: concise summary (2-3 sentences) and key ideas (bullet points)
**And** the API call completes in less than 30 seconds for standard captures (NFR3 compliance)
**And** the response is validated for completeness and format

**Given** the capture is a text capture
**When** digestion is triggered
**Then** the raw text content is sent directly to GPT-4o-mini (FR11)
**And** the AI analyzes the text for themes, insights, and actionable items
**And** the summary captures the core message accurately

**Given** the capture is an audio capture with transcription
**When** digestion is triggered
**Then** the transcribed text is sent to GPT-4o-mini (FR12)
**And** the AI considers both the literal content and implied context
**And** the summary reflects the spoken nature of the input

**Given** GPT-4o-mini returns the digestion results
**When** the response is processed
**Then** a Thought entity is created in the Knowledge Context (Core Domain)
**And** the Thought contains: summary text, array of key ideas (Ideas), reference to original Capture
**And** each Idea is stored as a separate entity with: text, timestamp, source Capture reference
**And** the Capture status is updated to "digested"
**And** the processing time is logged for performance monitoring

**Given** the AI digestion completes successfully
**When** the Thought and Ideas are persisted
**Then** the mobile app is notified via real-time channel
**And** the feed automatically updates to show the new insights
**And** a subtle germination animation celebrates the new Ideas (UX metaphor)

**Given** the capture content is too long for a single API call
**When** the content exceeds GPT token limits
**Then** the content is intelligently chunked with overlap
**And** summaries are generated for each chunk then combined
**And** the final summary remains concise and coherent

**Given** GPT-4o-mini returns an error or malformed response
**When** the response validation fails
**Then** the job is retried according to the retry policy (Story 4.1)
**And** if all retries fail, the Capture is marked "digestion_failed"
**And** the user is notified and can manually trigger re-digestion

**Given** the capture content is minimal or unclear
**When** GPT struggles to extract meaningful insights
**Then** the AI provides a best-effort summary
**And** the response indicates low confidence if appropriate
**And** the user can view both the summary and original content

### Story 4.3: Extraction Automatique d'Actions

As a **user**,
I want **the AI to automatically detect and extract actionable tasks from my captures**,
So that **I don't miss important todos buried in my thoughts**.

**Acceptance Criteria:**

**Given** the AI is digesting a capture (during Story 4.2 processing)
**When** GPT-4o-mini analyzes the content
**Then** the prompt also requests detection of actionable items/todos
**And** for each action detected, the AI extracts: task description, suggested deadline (if mentioned), suggested priority (high/medium/low)
**And** the extraction happens in the same API call as summary/ideas (no additional latency)

**Given** GPT-4o-mini detects actionable items in the content
**When** the response is processed
**Then** a Todo entity is created in the Action Context (Supporting Domain) for each action
**And** each Todo contains: description, deadline (optional), priority, status ("todo"), source Idea reference
**And** the Todo is associated with the originating Capture and Idea
**And** the Todo is persisted to the database

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

**Given** an action has priority indicators in the content
**When** GPT extracts the action
**Then** priority is inferred from keywords ("urgent", "important", "when I have time")
**And** the priority is set to high/medium/low accordingly
**And** if no priority indicators exist, default to "medium"

**Given** multiple actions are detected in a single capture
**When** the digestion completes
**Then** all actions are extracted and created as separate Todo entities
**And** each Todo maintains its link to the source Idea and Capture
**And** the Todos are displayed inline with the Ideas in the feed (FR16)

**Given** the capture contains no actionable items
**When** GPT analyzes the content
**Then** no Todo entities are created
**And** the digestion still completes successfully with summary and ideas only
**And** the feed shows the Thought without any action badges

**Given** GPT incorrectly identifies a non-action as an action (false positive)
**When** the Todo is displayed to the user
**Then** the user can dismiss or delete the incorrect action
**And** feedback is optionally collected to improve future extraction (post-MVP)

### Story 4.4: Notifications de Progression IA

As a **user**,
I want **to be notified of the progress of long-running AI processes**,
So that **I'm never left waiting without feedback and know when my insights are ready** (NFR5).

**Acceptance Criteria:**

**Given** a digestion job is queued
**When** the job enters the queue
**Then** I receive a local notification "Processing your thought..."
**And** the capture card in the feed shows a progress indicator
**And** the status badge displays "Queued" with estimated wait time if queue is backed up

**Given** digestion is actively processing
**When** the worker picks up the job
**Then** the capture status updates to "Digesting..." in real-time
**And** a progress animation is displayed (pulsing, shimmer effect)
**And** if processing exceeds 10 seconds, a notification shows "Still processing..."
**And** haptic feedback provides subtle pulse every 5 seconds (optional, can be disabled)

**Given** digestion completes successfully
**When** the Thought, Ideas, and Todos are persisted
**Then** I receive a push notification "New insights from your thought!" (if app in background)
**And** I receive a local notification if app is in foreground
**And** the feed updates in real-time with germination animation
**And** haptic feedback celebrates completion (single strong pulse)
**And** the notification includes a preview of key insights

**Given** I tap on the completion notification
**When** the notification is activated
**Then** the app opens directly to the detailed view of the digested capture
**And** the insights are highlighted with a subtle glow effect
**And** the transition is smooth and immediate

**Given** digestion fails after retries
**When** the capture is marked "digestion_failed"
**Then** I receive an error notification "Unable to process thought. Tap to retry."
**And** the capture card shows an error badge with retry option
**And** tapping the notification or retry button re-queues the job

**Given** multiple captures are being processed simultaneously
**When** I view the feed
**Then** each capture shows its individual processing status
**And** a global progress indicator shows "Processing X thoughts"
**And** I can tap to see the queue details (order, estimated times)

**Given** I have opted out of notifications in settings
**When** processing completes
**Then** no push or local notifications are sent
**And** the feed still updates in real-time with visual indicators only
**And** the notification settings are respected

**Given** the app is offline during processing
**When** digestion jobs are queued for when network returns
**Then** I see "Queued for when online" status
**And** a notification informs me when connectivity returns and processing starts
**And** the transition from offline queue to online processing is seamless

**Given** processing takes longer than expected (>30s)
**When** the timeout threshold is approached
**Then** I receive a notification "This is taking longer than usual..."
**And** I'm offered options: "Keep waiting" or "Cancel and retry later"
**And** the system logs the slow processing for monitoring

---

## Epic 5: Gestion des Actions (Tab Actions)

Les utilisateurs peuvent gérer toutes leurs actions extraites dans une vue centralisée avec filtres et completion tracking.

### Story 5.1: Affichage Inline des Todos dans le Feed

As a **user**,
I want **to see the detected actions displayed inline with each idea in the feed**,
So that **I can immediately see what needs to be done without navigating to a separate screen**.

**Acceptance Criteria:**

**Given** a capture has been digested and contains extracted actions
**When** I view the capture in the feed
**Then** each Idea that has associated Todos displays them inline below the idea text
**And** the action badges are visually distinct (icon, color, priority indicator)
**And** each Todo shows: checkbox, description, deadline (if any), priority badge

**Given** an Idea has multiple Todos
**When** displayed in the feed
**Then** all Todos are listed in priority order (high → medium → low)
**And** each Todo is presented on its own line for clarity
**And** the list is visually grouped under the parent Idea

**Given** an Idea has no Todos
**When** displayed in the feed
**Then** no action section is shown for that Idea
**And** the Idea displays normally without empty action placeholders

**Given** I view a Todo inline in the feed
**When** the Todo is displayed
**Then** I can see its complete description (truncated if very long, with "...more")
**And** the deadline is shown in human-readable format ("Today", "Tomorrow", "In 3 days")
**And** overdue deadlines are highlighted in red/warning color
**And** priority is indicated with color-coded badges (🔴 High, 🟡 Medium, 🟢 Low)

**Given** a Todo is marked as completed
**When** displayed inline
**Then** the checkbox shows as checked
**And** the text is styled with strikethrough
**And** the Todo is slightly dimmed to indicate completion
**And** completed Todos appear at the bottom of the list (below active ones)

**Given** I tap on an inline Todo
**When** the Todo is tapped
**Then** a detail popover appears with full information
**And** I can edit the description, deadline, or priority
**And** I can mark it as complete/incomplete
**And** I can navigate to the source Idea/Capture (FR20)

**Given** multiple captures in the feed have Todos
**When** scrolling through the feed
**Then** inline Todos are consistently styled across all captures
**And** the visual hierarchy makes it clear which Todos belong to which Ideas
**And** animations are smooth when expanding/collapsing Todo lists (optional)

### Story 5.2: Tab Actions Centralisé

As a **user**,
I want **to access a dedicated "Actions" tab that shows all my todos in one centralized view**,
So that **I can manage all my tasks in one place without having to scroll through the feed**.

**Acceptance Criteria:**

**Given** the app has a bottom navigation bar
**When** I look at the navigation options
**Then** I see an "Actions" tab with a distinctive icon (e.g., checkbox or list icon)
**And** a badge shows the count of active (incomplete) todos
**And** the badge updates in real-time as todos are completed or created

**Given** I tap on the "Actions" tab
**When** the tab is activated
**Then** I navigate to the Actions screen
**And** all my Todos from all captures are displayed in a unified list
**And** the screen loads in less than 1 second from local cache (NFR4)
**And** the transition animation is smooth (Liquid Glass design)

**Given** I am viewing the Actions screen
**When** the list is displayed
**Then** Todos are grouped and sorted by default: Overdue → Today → This Week → Later → No Deadline
**And** within each group, Todos are sorted by priority (High → Medium → Low)
**And** each Todo card shows: checkbox, description, deadline, priority badge, source capture preview

**Given** I have many Todos (50+)
**When** viewing the Actions screen
**Then** the list uses efficient rendering (virtualization) for smooth scrolling
**And** scroll performance remains at 60fps (UX requirement)
**And** infinite scroll or pagination is implemented for large lists

**Given** I have no active Todos
**When** I view the Actions screen
**Then** an empty state is displayed with encouraging message
**And** the illustration reflects the "Jardin d'idées" metaphor (e.g., "Your garden is peaceful today")
**And** a subtle animation adds life to the empty state

**Given** a Todo card is displayed in the Actions list
**When** I view the card
**Then** I can see a truncated preview of the source Idea/Capture
**And** the card shows how long ago the capture was created
**And** tapping the card opens the full Todo detail view

**Given** I pull down to refresh on the Actions screen
**When** the refresh gesture is triggered
**Then** the list syncs with the latest data from the database
**And** a refresh animation is shown (Liquid Glass design)
**And** new or updated Todos appear with subtle animation

**Given** I switch from Feed to Actions tab
**When** returning to the Actions tab
**Then** my previous scroll position is preserved
**And** the filter/sort state is maintained
**And** the transition feels instant and seamless

### Story 5.3: Filtres et Tri des Actions

As a **user**,
I want **to filter and sort my actions by status, priority, and deadline**,
So that **I can focus on the most relevant todos for my current context**.

**Acceptance Criteria:**

**Given** I am on the Actions screen
**When** I look at the top of the screen
**Then** I see filter tabs: "Toutes" | "À faire" | "Faites" (FR18)
**And** the current filter is highlighted
**And** a count badge shows the number of todos in each category

**Given** I tap on the "À faire" filter
**When** the filter is activated
**Then** only incomplete todos are displayed
**And** the list updates with smooth animation
**And** the "À faire" tab is highlighted as active
**And** completed todos are hidden from view

**Given** I tap on the "Faites" filter
**When** the filter is activated
**Then** only completed todos are displayed
**And** all todos show checkboxes checked and strikethrough text
**And** the list is sorted by completion date (most recent first)
**And** I can tap to uncheck and move back to "À faire"

**Given** I tap on the "Toutes" filter
**When** the filter is activated
**Then** both completed and incomplete todos are displayed
**And** incomplete todos appear first, followed by completed ones
**And** each section is visually separated

**Given** I am viewing filtered todos
**When** I access additional sort options (button or menu)
**Then** I can sort by: Default (deadline groups), Priority, Created Date, Alphabetical
**And** the sort option is applied within the current filter
**And** the selected sort option is persisted across sessions

**Given** I select "Sort by Priority"
**When** the sort is applied
**Then** todos are ordered: High Priority → Medium → Low
**And** within each priority, secondary sort is by deadline
**And** the sort changes with smooth list reordering animation

**Given** I select "Sort by Created Date"
**When** the sort is applied
**Then** todos are ordered chronologically by when the source capture was created
**And** I can see the creation timestamp on each card
**And** this helps me focus on recent or older tasks

**Given** I apply a filter and sort combination
**When** I switch tabs or navigate away
**Then** my filter and sort preferences are saved
**And** when I return to the Actions screen, the same view is restored
**And** the state persists even after closing the app

**Given** I have no todos matching the current filter
**When** the filter shows an empty result
**Then** a contextual empty state is shown (e.g., "No completed tasks yet" for "Faites")
**And** the illustration and message match the filter context
**And** I can easily switch to another filter

**Given** the filter changes (new todo created or completed)
**When** a todo status changes in real-time
**Then** the filter counts update immediately
**And** if the todo no longer matches the current filter, it smoothly animates out
**And** if I'm viewing "Toutes", it moves between sections with animation

### Story 5.4: Complétion et Navigation des Actions

As a **user**,
I want **to mark actions as completed with satisfying feedback and navigate back to the original idea**,
So that **I can track my progress and understand the context of each action**.

**Acceptance Criteria:**

**Given** I am viewing a Todo (inline in feed or in Actions tab)
**When** I tap the checkbox to mark it complete (FR19)
**Then** the checkbox animates to checked state
**And** the Todo text gets strikethrough styling
**And** a satisfying haptic feedback confirms the action (medium impact)
**And** a subtle completion animation plays (e.g., confetti, glow, or checkmark burst)
**And** the Todo status is immediately updated in the database

**Given** I mark a Todo as complete
**When** the completion is processed
**Then** the Todo entity status changes to "completed"
**And** a completion timestamp is recorded
**And** if viewing "À faire" filter, the Todo smoothly animates out
**And** the filter count badges update in real-time
**And** the change syncs to the cloud when online

**Given** I have marked a Todo as complete
**When** I tap the checkbox again
**Then** the Todo is unmarked (status returns to "todo")
**And** the strikethrough is removed with reverse animation
**And** haptic feedback confirms the un-completion
**And** the Todo reappears in the "À faire" filter if applicable

**Given** I am viewing a Todo
**When** I tap anywhere on the Todo card (except the checkbox)
**Then** a detail view opens showing full Todo information
**And** I can see the complete description, deadline, priority
**And** I can edit any of these fields inline
**And** changes are saved immediately with optimistic UI updates

**Given** I am in the Todo detail view
**When** I look for the source context
**Then** I see a "View Origin" or "Go to Idea" button/link (FR20)
**And** the button shows a preview of the source Idea text
**And** tapping it navigates me to the original Capture detail view

**Given** I tap "View Origin" on a Todo
**When** navigating to the source (FR20)
**Then** the app navigates to the Feed tab
**And** opens the detail view of the source Capture
**And** the relevant Idea and Todo are highlighted or scrolled into view
**And** the transition animation is smooth (hero animation if possible)
**And** a back button allows me to return to the Actions tab

**Given** I complete a Todo from the Feed (inline)
**When** the completion animation plays
**Then** the animation respects the "Jardin d'idées" metaphor (e.g., flower blooms, seed sprouts)
**And** the animation is subtle and celebratory, not disruptive
**And** the Todo remains visible but dimmed and moved to bottom of the list

**Given** I have completed many Todos
**When** viewing the "Faites" filter
**Then** I can bulk delete or archive old completed Todos
**And** a confirmation dialog prevents accidental deletion
**And** deleted Todos are removed from the database (or soft-deleted for analytics)

**Given** I swipe a Todo card in the Actions tab
**When** the swipe gesture is detected
**Then** quick actions appear: Complete, Delete, Edit
**And** the swipe reveals actions with smooth animation and haptic feedback
**And** tapping "Complete" immediately marks the Todo as done with celebration animation

---

## Epic 6: Synchronisation Multi-Device

Les utilisateurs peuvent synchroniser automatiquement leurs captures et données entre leurs appareils.

### Story 6.1: Infrastructure de Synchronisation WatermelonDB

As a **developer**,
I want **the WatermelonDB sync infrastructure configured on both mobile and backend**,
So that **we have a robust foundation for bidirectional data synchronization**.

**Acceptance Criteria:**

**Given** the backend NestJS application is running
**When** the sync infrastructure is configured
**Then** a dedicated sync API endpoint is created (e.g., POST /api/sync)
**And** the endpoint accepts WatermelonDB sync protocol payloads
**And** the endpoint handles authentication via JWT tokens
**And** the endpoint validates user isolation (NFR13 compliance)

**Given** the mobile app has WatermelonDB configured
**When** the sync client is initialized
**Then** the sync engine is configured with the backend sync endpoint URL
**And** sync is configured to work with all relevant tables (Captures, Thoughts, Ideas, Todos)
**And** sync timestamps are tracked per table for incremental syncs
**And** the sync client includes authentication headers in all requests

**Given** the sync infrastructure is set up
**When** data models are defined
**Then** all entities have sync-compatible fields: id, _status, _changed, last_modified_at
**And** soft deletes are implemented (_status = 'deleted') for sync consistency
**And** conflict resolution strategy is defined (last-write-wins for MVP)
**And** schema migrations are sync-aware

**Given** the backend receives a sync request
**When** processing the sync payload
**Then** changes are validated for data integrity
**And** user permissions are verified (user can only sync their own data)
**And** changes are applied to PostgreSQL database
**And** a sync response is generated with server-side changes
**And** the response follows WatermelonDB sync protocol format

**Given** sync infrastructure is operational
**When** network connectivity is unreliable
**Then** the sync client implements exponential backoff for retries
**And** partial sync batches are supported (chunking for large datasets)
**And** sync errors are logged with detailed diagnostics
**And** the system gracefully handles interrupted syncs

**Given** encryption is required (NFR12)
**When** data is synced
**Then** all data in transit uses HTTPS/TLS (NFR11 compliance)
**And** sensitive fields are encrypted at rest in both mobile and backend
**And** encryption keys are properly managed per user

**Given** the sync system needs monitoring
**When** syncs occur
**Then** metrics are logged: sync duration, data volume, success/failure rate
**And** alerts are triggered for repeated sync failures
**And** sync performance is tracked for optimization

### Story 6.2: Synchronisation Local → Cloud

As a **user**,
I want **my local captures and changes to automatically sync to the cloud when network is available**,
So that **my data is safely backed up and accessible from other devices** (FR29, NFR9).

**Acceptance Criteria:**

**Given** I have created captures while offline
**When** network connectivity returns
**Then** the sync engine automatically detects the network change
**And** a sync operation is triggered without user intervention (NFR9 compliance)
**And** all pending local changes are queued for upload
**And** the sync begins within 5 seconds of network availability

**Given** local changes are being synced to the cloud
**When** the sync executes
**Then** only changes since the last successful sync are uploaded (incremental sync)
**And** changes are batched efficiently to minimize API calls
**And** the upload includes: new Captures, Thoughts, Ideas, Todos, and modifications
**And** deleted items (_status = 'deleted') are synced to propagate deletions

**Given** I create a new capture while online
**When** the capture is saved locally
**Then** a sync is automatically triggered after a short delay (e.g., 3 seconds)
**And** the new capture is uploaded to the cloud
**And** the sync completes in the background without blocking the UI
**And** I can continue using the app normally during sync

**Given** I modify an existing Todo (mark complete, edit description)
**When** the change is saved locally
**Then** the modified Todo is marked as changed (_changed = true)
**And** the next sync operation includes this change
**And** the server receives and applies the modification
**And** the local record is updated with server confirmation

**Given** sync upload fails due to network error
**When** the failure is detected
**Then** the sync is automatically retried with exponential backoff
**And** failed changes remain in the local queue
**And** I see a sync status indicator showing "Sync pending"
**And** the app continues to function normally offline

**Given** I have large audio files to sync
**When** uploading captures with audio
**Then** audio files are uploaded separately from metadata
**And** resumable uploads are supported for large files
**And** upload progress is tracked and resumable if interrupted
**And** failed uploads are retried automatically

**Given** a sync conflict occurs (same record modified on multiple devices)
**When** the conflict is detected during upload
**Then** the last-write-wins strategy is applied (MVP approach)
**And** the server timestamp determines the winning version
**And** the local record is updated with the server's version if server wins
**And** no data is silently lost (conflict logged for future improvement)

**Given** sync completes successfully
**When** all changes are uploaded
**Then** local sync timestamps are updated
**And** the _changed flags are reset for synced records
**And** the sync queue is cleared
**And** I see a "Synced" indicator in the UI

### Story 6.3: Synchronisation Cloud → Local

As a **user**,
I want **cloud data to automatically sync to my device when I install the app on a new device or log in**,
So that **I can access all my captures from any device** (FR30, NFR9).

**Acceptance Criteria:**

**Given** I install the app on a new device and log in
**When** authentication completes
**Then** an initial full sync is automatically triggered
**And** all my cloud data (Captures, Thoughts, Ideas, Todos) is downloaded
**And** a progress indicator shows "Syncing your data..." with percentage
**And** the sync happens automatically without user intervention (NFR9 compliance)

**Given** the initial sync is downloading data
**When** downloading my captures
**Then** metadata is downloaded first (fast)
**And** audio files are downloaded in the background (priority queue)
**And** I can start browsing captures with transcriptions immediately
**And** audio files download progressively as needed (lazy loading)

**Given** I'm using the app on multiple devices
**When** I create a capture on Device A
**Then** Device B automatically receives the new capture within seconds (when online)
**And** the sync happens in the background without user action
**And** the feed on Device B updates in real-time with the new capture
**And** a subtle animation indicates the new content

**Given** cloud data has changed (new captures, completed todos)
**When** the app performs a sync check
**Then** only changes since the last sync are downloaded (incremental sync)
**And** the download is efficient and minimizes bandwidth usage
**And** changes are applied to the local WatermelonDB
**And** the UI updates reactively to show the new data

**Given** I delete a capture on one device
**When** the deletion syncs to the cloud
**Then** other devices receive the deletion in the next sync
**And** the capture is removed from local storage on all devices
**And** the deletion is processed as a soft delete (sync protocol)

**Given** sync download fails due to network interruption
**When** the failure is detected
**Then** the sync is automatically retried with exponential backoff
**And** partially downloaded data is preserved for resumption
**And** the app continues to work with locally available data
**And** I see a "Sync paused" indicator

**Given** cloud data conflicts with local changes (rare edge case)
**When** both devices modified the same record
**Then** the last-write-wins strategy is applied based on server timestamp
**And** the winning version is applied locally
**And** the losing version is logged (for post-MVP conflict UI)
**And** no data is silently lost

**Given** I open the app after being offline for days
**When** network becomes available
**Then** a background sync automatically starts
**And** all cloud changes since last sync are downloaded
**And** large downloads are chunked to avoid memory issues
**And** the app remains responsive during sync

**Given** sync download completes successfully
**When** all cloud changes are applied
**Then** local sync timestamps are updated
**And** the UI reflects all new data
**And** I see a brief "Synced" confirmation
**And** the app is ready for normal use

### Story 6.4: Indicateurs de Statut de Synchronisation

As a **user**,
I want **to see clear indicators of sync status throughout the app**,
So that **I always know if my data is synced, syncing, or pending** (FR31).

**Acceptance Criteria:**

**Given** I am using the app
**When** I look at the app header or status bar
**Then** I see a sync status indicator with one of these states:
- ✅ "Synced" (green) - all data is synced
- 🔄 "Syncing..." (blue, animated) - sync in progress
- ⏸️ "Sync pending" (orange) - waiting for network
- ❌ "Sync failed" (red) - error occurred
**And** the indicator updates in real-time as sync status changes

**Given** sync is in progress
**When** data is being uploaded or downloaded
**Then** the sync indicator shows an animated spinner or progress ring
**And** tapping the indicator shows detailed sync info (items syncing, progress %)
**And** I can see which type of data is syncing (e.g., "Syncing 3 captures...")
**And** the animation is subtle and doesn't distract from main content

**Given** I have unsync changes (offline captures)
**When** viewing the app while offline
**Then** the sync indicator shows "Sync pending" with count (e.g., "5 items pending")
**And** tapping shows a list of pending items
**And** the indicator is noticeable but not alarming
**And** I understand my data is safe locally and will sync later

**Given** sync completes successfully
**When** all changes are synced
**Then** the indicator briefly shows "Synced" with a checkmark
**And** a subtle haptic confirms successful sync (optional)
**And** the indicator fades to a neutral state after a few seconds
**And** the last sync time is displayed (e.g., "Last synced: 2 min ago")

**Given** sync fails after retries
**When** an error prevents sync
**Then** the indicator shows "Sync failed" in red
**And** tapping shows error details and retry option
**And** a manual "Retry sync" button is available
**And** the error message is user-friendly (not technical jargon)

**Given** I'm viewing a specific capture
**When** the capture has unsynchronized changes
**Then** a small sync badge appears on the capture card
**And** the badge indicates "Not synced" or "Syncing"
**And** once synced, the badge disappears with animation

**Given** I pull down to refresh on the Feed or Actions tab
**When** the refresh gesture is triggered
**Then** a manual sync is initiated
**And** the refresh indicator shows sync progress
**And** the sync completes even if I navigate away
**And** I receive a brief "Synced" confirmation when done

**Given** I'm on a metered network or low bandwidth
**When** sync would consume significant data
**Then** I can optionally pause sync in settings
**And** a warning appears if large audio files will be synced
**And** I can choose to sync only on Wi-Fi (settings option)

**Given** multiple sync operations are queued
**When** viewing sync details
**Then** I see a prioritized list: user-initiated changes first, then background
**And** estimated time remaining is shown (if calculable)
**And** I can cancel pending background syncs if needed

**Given** I haven't synced in a long time (days)
**When** opening the app
**Then** a gentle reminder appears "You have unsynced changes. Connect to sync."
**And** the reminder is dismissible
**And** it doesn't block app usage

---

## Epic 7: Support & Observability

Les utilisateurs bénéficient d'outils de support et de débogage contrôlés par des permissions backend, permettant un support technique rapide et flexible sans nécessiter de nouvelles publications d'application.

**FRs couverts:** Aucun FR du PRD initial (feature transverse de support)

**Notes d'implémentation:**
- Système de permissions backend extensible (feature flags)
- Double niveau de contrôle : permission d'accès (backend) + activation locale (settings)
- Cache des permissions avec expiration à minuit
- Interface admin pour gérer les permissions utilisateur
- Support pour debugging en production sans republication app

---

### Story 7.1: Support Mode avec Permissions Backend

As a **user ayant besoin de support technique**,
I want **un système de permissions backend qui contrôle l'accès au mode debug de l'app**,
So that **je puisse activer/désactiver le mode debug sans republier l'app, permettant un support rapide et flexible pour les power users et early adopters**.

**Context:** Cette story permet de gérer l'accès au mode debug depuis l'interface admin backend, avec un système à double niveau (permission d'accès + activation locale). Cas d'usage principal : déboguer des problèmes utilisateurs en production (ex: logs locaux, retry transcription, rapport d'erreur) sans attendre de publication App Store/Play Store.

**Voir fichier de story complet:** `_bmad-output/implementation-artifacts/7-1-support-mode-avec-permissions-backend.md`

**Acceptance Criteria Summary:**
- AC1: Backend API endpoint pour récupération des permissions utilisateur
- AC2: Interface admin avec toggle pour activer/désactiver permission debug
- AC3: Mobile fetch des permissions au démarrage
- AC4: Gestion du cache offline jusqu'à minuit
- AC5: Refresh manuel des permissions sans reconnexion
- AC6: Affichage conditionnel du switch debug dans settings
- AC7: Masquage du switch si permission refusée
- AC8: Vérification permission dans settingsStore avant activation
- AC9: Intégration avec features debug existantes
- AC10: Persistance et synchronisation de l'état
- AC11: Scénario de support end-to-end validé

**Tasks Summary:**
- Backend: Endpoint API + interface admin + tests
- Mobile: UserFeaturesService + cache + intégration settingsStore + UI
- Testing: Tests unitaires + E2E + validation scénario support

**Definition of Done:**
- Tous les AC implémentés et testés
- Tests backend et mobile à 100%
- Interface admin opérationnelle
- Documentation support créée
- Code review approuvé

---

## Epic 8: GitHub Issues Resolution

Les développeurs peuvent corriger les bugs et implémenter les fonctionnalités identifiées dans les issues GitHub, assurant la qualité du code et l'amélioration continue du produit.

**FRs couverts:** Aucun FR du PRD initial (corrections techniques et améliorations qualité)

**Notes d'implémentation:**
- Résolution des bugs TypeScript critiques bloquant la compilation
- Standardisation des patterns de code pour améliorer la maintenabilité
- Corrections de bugs UX identifiés par les utilisateurs
- Extensions fonctionnelles basées sur les retours terrain

**Source:** Issues GitHub du repository yohikofox/pensieve

**Référence issues fermées:** `_bmad-output/implementation-artifacts/github-issues-closed-reference.md`

---

### Story 8.1: Fix TypeScript Compilation Errors

As a **developer**,
I want **to fix all critical TypeScript compilation errors in the codebase**,
So that **the application compiles without errors and type safety is guaranteed**.

**GitHub Issues:** #22, #21, #20, #19

**Context:** Plusieurs erreurs TypeScript bloquent la compilation et réduisent la fiabilité du code. Ces erreurs concernent principalement les types de retour des hooks React Query et les événements du système de capture.

**Acceptance Criteria:**

**AC1: Fix CaptureRepository Event Type Mismatches (#22)**

**Given** the CaptureRepository publishes events
**When** constructing event payloads for CaptureRecordedEvent and CaptureDeletedEvent
**Then** the `captureType` field uses type assertion `as 'audio' | 'text'` (lines 154, 281, 420)
**And** the `audioDuration` field uses nullish coalescing `?? undefined` to convert null to undefined (lines 157, 284)
**And** all event payloads compile without TypeScript errors
**And** runtime validation ensures only 'audio' or 'text' captures trigger events

**AC2: Fix useUpdateTodo Return Type Mismatch (#21)**

**Given** the useUpdateTodo hook is defined
**When** declaring the hook's return type
**Then** the return type is `UseMutationResult<boolean, Error, UpdateTodoParams>` (not void)
**And** the boolean return value from todoRepository.update is preserved
**And** consumers can check if the update was actually applied
**And** the hook compiles without TypeScript errors

**AC3: Fix useToggleTodoStatus QueryFilters Syntax (#20)**

**Given** the useToggleTodoStatus hook handles errors
**When** rolling back optimistic updates in the onError callback
**Then** `setQueriesData` is called with `{ queryKey: ["todos"] }` (QueryFilters object)
**And** the rollback syntax matches the optimistic update pattern (line 48)
**And** the error handler compiles without TypeScript errors
**And** rollback functionality works correctly when mutations fail

**AC4: Fix NPUDetectionService Platform.constants Access (#19)**

**Given** NPUDetectionService needs to access device model information
**When** accessing Platform.constants properties
**Then** a type assertion pattern is used: `Platform.constants as Record<string, unknown>`
**And** the deviceModel is safely extracted with fallback: `(platformConsts?.Model || 'iPhone') as string`
**And** the pattern is consistent with existing code (line 160)
**And** the service compiles without TS2339 errors

**AC5: Verification and Testing**

**Given** all TypeScript fixes are applied
**When** running `tsc --noEmit`
**Then** zero TypeScript compilation errors are reported
**And** all affected features continue to work correctly
**And** existing tests pass without regression
**And** code review confirms type safety improvements

**Tasks:**
1. Fix CaptureRepository event payload construction (3 locations)
2. Update useUpdateTodo return type declaration
3. Fix useToggleTodoStatus setQueriesData syntax
4. Add type assertion for Platform.constants access
5. Run full TypeScript compilation check
6. Verify runtime behavior unchanged
7. Update tests if needed

**Definition of Done:**
- All 4 TypeScript errors fixed
- `tsc --noEmit` passes without errors
- All existing tests pass
- Code review approved
- No runtime regressions

---

### Story 8.2: Standardize Type Constants Pattern

As a **developer**,
I want **all type constants to follow the `as const` object pattern consistently**,
So that **the codebase has uniform type definitions with autocomplete and refactor safety**.

**GitHub Issue:** #24

**Context:** Le projet utilise actuellement trois patterns différents pour les constantes de types, créant de l'inconsistance. Le pattern `as const` object est déjà utilisé pour `ANALYSIS_TYPES` et `METADATA_KEYS`, mais `captureType` utilise encore des unions littérales inline.

**Acceptance Criteria:**

**AC1: Create CAPTURE_TYPES Constant**

**Given** I am defining capture type constants
**When** adding the constant to Capture.model.ts
**Then** a constant `CAPTURE_TYPES` is defined as:
```typescript
export const CAPTURE_TYPES = {
  AUDIO: "audio",
  TEXT: "text",
  IMAGE: "image",
  URL: "url",
} as const;

export type CaptureType = (typeof CAPTURE_TYPES)[keyof typeof CAPTURE_TYPES];
```
**And** the constant is exported for use across the codebase

**AC2: Update Capture Interface**

**Given** the CAPTURE_TYPES constant is defined
**When** defining the Capture interface
**Then** the `type` field uses `CaptureType` instead of `string`
**And** TypeScript enforces only valid capture types
**And** the interface remains backward compatible

**AC3: Update CaptureEvents**

**Given** the CaptureType is exported
**When** defining event interfaces in CaptureEvents.ts
**Then** `captureType` properties use the `CaptureType` type (not inline unions)
**And** the import statement includes: `import { CAPTURE_TYPES, CaptureType } from '../domain/Capture.model'`
**And** all event interfaces compile without errors

**AC4: Replace Hardcoded Strings in Repository**

**Given** I am using capture types in the repository
**When** assigning or comparing capture types
**Then** hardcoded strings ('audio', 'text', 'image', 'url') are replaced with `CAPTURE_TYPES.AUDIO`, etc.
**And** IDE autocomplete suggests available types
**And** refactoring a constant updates all usages

**AC5: Update Tests**

**Given** tests use capture type literals
**When** updating test files
**Then** all string literals are replaced with constants from `CAPTURE_TYPES`
**And** tests remain readable and maintainable
**And** all tests pass after the change

**AC6: Consistency Verification**

**Given** the pattern is applied
**When** reviewing the codebase
**Then** all type constants use the `as const` object pattern
**And** no inline literal unions remain for capture types
**And** the pattern is consistent with `ANALYSIS_TYPES` and `METADATA_KEYS`

**Tasks:**
1. Define CAPTURE_TYPES constant in Capture.model.ts
2. Update Capture interface to use CaptureType
3. Update CaptureEvents.ts imports and types
4. Search and replace hardcoded strings in CaptureRepository
5. Update all test files to use constants
6. Verify TypeScript compilation
7. Document pattern in coding guidelines

**Definition of Done:**
- CAPTURE_TYPES constant defined and exported
- All interfaces use CaptureType
- No hardcoded capture type strings remain
- All tests pass
- Pattern consistent across codebase
- Code review approved

---

### Story 8.3: Fix Audio Trim Truncating Transcriptions

As a **user**,
I want **my audio recordings to preserve the complete content without aggressive silence trimming**,
So that **transcriptions include all my spoken words, especially at the end of recordings**.

**GitHub Issue:** #16

**Context:** Les transcriptions audio perdent les derniers mots/phrases car le traitement post-enregistrement trim les silences de manière trop agressive. Cette perte de contenu dégrade l'expérience utilisateur et la qualité des transcriptions.

**Acceptance Criteria:**

**AC1: Disable Audio Trim by Default**

**Given** I am recording audio
**When** the recording completes and is saved
**Then** no automatic silence trimming is applied to the audio file
**And** the complete audio content is preserved as recorded
**And** transcriptions include all spoken content from start to end

**AC2: Add Settings Toggle**

**Given** I am in the app settings
**When** I navigate to audio recording settings
**Then** I see a toggle option "Supprimer les silences automatiquement"
**And** the toggle is OFF by default
**And** the setting is clearly labeled and explained

**AC3: Configurable Trim Behavior**

**Given** I have enabled "Supprimer les silences automatiquement" in settings
**When** I record audio
**Then** silence trimming is applied after recording
**And** the trimming level is conservative (preserves more content than before)
**And** an optional aggressiveness level can be configured (if feasible)

**AC4: Settings Persistence**

**Given** I change the silence trimming setting
**When** I restart the app
**Then** my preference is preserved
**And** future recordings respect my chosen setting
**And** the setting is synced if multi-device sync is enabled

**AC5: Validate Transcription Completeness**

**Given** I have trim disabled (default)
**When** I record audio with content near the end
**Then** the transcription includes the final words/phrases
**And** no content is lost compared to the original recording
**And** previously truncated transcriptions are now complete

**AC6: File Size Impact Communication**

**Given** trim is disabled
**When** viewing recording details
**Then** I am informed that files may be slightly larger
**And** the trade-off (completeness vs. size) is explained
**And** storage usage remains acceptable

**Tasks:**
1. Identify audio trim logic in post-recording processing
2. Disable trim by default in recording pipeline
3. Add "Supprimer les silences automatiquement" toggle in settings UI
4. Implement settings persistence (AsyncStorage/Secure Store)
5. Apply conditional trim logic based on user preference
6. Test with recordings of various lengths
7. Validate transcription completeness before/after
8. Update storage monitoring if needed

**Definition of Done:**
- Audio trim disabled by default
- Settings toggle implemented and functional
- User preference persisted across sessions
- Transcriptions include full content (especially endings)
- Tests validate completeness
- Storage impact acceptable
- Code review approved

---
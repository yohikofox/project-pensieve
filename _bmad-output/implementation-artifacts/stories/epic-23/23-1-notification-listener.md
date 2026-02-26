# Story 23.1 — Notification Listener (Android)

Status: backlog

## Story

As a **user**,
I want **Pensine to passively detect action requests in my incoming notifications**,
So that **Todos and ExecutionJobs are created automatically without me opening the app**.

## Context

Inverse du flux habituel : au lieu de capturer activement, Pensine surveille les notifications
entrantes et détecte les intents actionnables. Exemple : ton frère t'envoie un WhatsApp
"pense à m'envoyer le doc" → Pensine crée un Todo automatiquement.

**Android uniquement** : `NotificationListenerService` — permission spéciale requise
(pas root, mais doit être accordée manuellement dans les paramètres système).
**iOS** : impossible — Apple n'autorise pas l'accès aux notifications d'autres apps.

## Acceptance Criteria

### AC1 — Permission NotificationListenerService
**Given** l'app Pensine installée sur Android
**When** l'utilisateur active la feature dans les Settings
**Then** Pensine redirige vers `Settings > Notification Access` pour accorder la permission

### AC2 — Détection d'intents actionnables dans les notifications
**Given** la permission accordée
**When** une notification entrante contient un texte actionnable (ex: "pense à X", "n'oublie pas de Y")
**Then** le LLM local analyse le texte et crée un Todo ou ExecutionJob selon `isExecutable`

### AC3 — Filtrage par source
**Given** la feature activée
**When** une notification arrive
**Then** seules les notifications des apps whitelistées par l'utilisateur sont analysées
(ex: WhatsApp, SMS, Telegram — pas les notifs système, publicités, etc.)

### AC4 — Contrôle utilisateur
**Given** un Todo créé depuis une notification
**When** l'utilisateur consulte le Todo
**Then** la source est indiquée ("Créé depuis notification WhatsApp — Paul Durand")
**And** l'utilisateur peut désactiver l'analyse par app source

## References
- ADR-020 (Background Processing — expo-task-manager)
- ADR-034 (modèle sémantique Action/Todo/ExecutionJob)
- Epic 19 (Delegation BC — pipeline de création ExecutionJob)

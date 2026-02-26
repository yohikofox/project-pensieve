# Story 23.16 — Email Action Extraction (Gmail)

Status: backlog

## Story

As a **user receiving emails with implicit action requests**,
I want **Pensine to detect and extract actions from my important emails**,
So that **"Paul m'a demandé d'envoyer le devis avant vendredi" becomes a Todo automatically**.

## Context

Parser les emails Gmail entrants → détecter les actions qui incombent à l'utilisateur
→ créer des Todos / ExecutionJobs. Scope `gmail.readonly` — lecture seule.

Réutilise le pipeline LLM existant (même prompt `delegation_items` / `action_items`).

## Acceptance Criteria

### AC1 — OAuth Gmail avec scope readonly
**Given** l'OAuth Google existant (story 19-7)
**When** l'utilisateur active Email Action Extraction
**Then** le scope `gmail.readonly` est ajouté (pas de re-auth complète)

### AC2 — Analyse des emails entrants
**Given** la feature activée
**When** un email est reçu d'un expéditeur whitelisté (contacts Google uniquement par défaut)
**Then** le LLM analyse le corps de l'email et extrait les actions attendues de l'utilisateur

### AC3 — Création de Todos / ExecutionJobs
**Given** des actions extraites
**When** Pensine les présente à l'utilisateur
**Then** il peut les accepter (→ Todo ou ExecutionJob) ou les rejeter en 1 tap

### AC4 — Filtrage anti-spam
**Given** la feature activée
**When** un email arrive
**Then** seuls les emails d'expéditeurs dans Google Contacts sont analysés
**And** les emails marketing/newsletters sont exclus (détection d'en-tête List-Unsubscribe)

## References
- Story 19-7 (OAuth Google — scope partagé)
- Story 19-13 (ContactResolutionPipeline — whitelist expéditeurs)
- ADR-004 (Single LLM call)

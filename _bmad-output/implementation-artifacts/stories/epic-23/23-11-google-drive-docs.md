# Story 23.11 — Google Drive & Docs Integration

Status: backlog

## Story

As a **user**,
I want **Pensine to create Google Docs or save files to Drive from a capture**,
So that **"crée un Google Doc pour le projet X" or "envoie le fichier à Paul" work automatically**.

## Context

Intégration naturelle dans l'écosystème Google (Google Only golden path).
Scope `drive.file` — minimal et approuvable facilement par Google (crée/modifie
uniquement les fichiers créés par l'app, pas accès à tous les Drive).

**Capabilities** :
- `DriveDocCreate` : créer un Google Doc avec titre et contenu initial
- `DriveFileShare` : partager un fichier existant avec un contact

## Acceptance Criteria

### AC1 — OAuth Google avec scope drive.file
**Given** l'OAuth Google déjà configuré (story 19-7)
**When** l'utilisateur active l'intégration Drive
**Then** le scope `drive.file` est ajouté à la demande d'autorisation existante (pas de re-auth complète)

### AC2 — Capability DriveDocCreate
**Given** une capture contenant "crée un doc / document pour [titre]"
**When** le LLM classifie l'intent
**Then** un `ExecutionJob DriveDocCreate` est créé avec :
- `params.title` : titre du document
- `params.content_hint` : contenu initial suggéré par le LLM (optionnel)

**And** à l'exécution, un Google Doc est créé via l'API Drive et le lien est partagé
en notification à l'utilisateur

### AC3 — Capability DriveFileShare
**Given** une capture contenant "envoie le fichier [nom] à [contact]"
**When** le LLM classifie l'intent et le ContactResolver résout le contact
**Then** un `ExecutionJob DriveFileShare` est créé
**And** à l'exécution, le fichier est partagé avec l'email résolu

### AC4 — Lien retourné en notification
**Given** un `ExecutionJob DriveDocCreate` exécuté
**When** le Doc est créé
**Then** une notification est envoyée avec le lien direct vers le Doc

## Dev Notes
- Google Drive API v3 : `POST https://www.googleapis.com/drive/v3/files`
- Scope `drive.file` : accès limité aux fichiers créés par l'app (non sensible)
- Réutilise OAuth Google story 19-7

## References
- Story 19-7 (OAuth2 Google scope partagé)
- Story 19-13 (ContactResolutionPipeline — pour DriveFileShare)
- ADR-033 (Delegation BC — capabilities)

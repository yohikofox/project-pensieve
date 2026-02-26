# Story 23.21 — Speaker Diarization

Status: backlog

## Story

As a **user recording meetings or conversations**,
I want **Pensine to distinguish between what I said and what others said**,
So that **"ce que Paul s'est engagé à faire" and "ce qu'on m'a demandé" are attributed correctly**.

## Context

Whisper supporte la diarization basique via pyannote ou via le paramètre `diarize`
de certains wrappers. Sur device, c'est une feature avancée qui nécessite
une validation de faisabilité (spike).

Différencie radicalement Pensine des apps de note vocale — attribution des actions
par speaker transforme une réunion en compte-rendu automatique.

## Acceptance Criteria

### AC1 — Spike : faisabilité diarization on-device
**Given** whisper.rn (bibliothèque actuelle)
**When** le spike est réalisé
**Then** la faisabilité de la diarization on-device est documentée (latence, qualité, modèle requis)

### AC2 — Mode "Réunion" (si spike validé)
**Given** la diarization activée
**When** l'utilisateur démarre une capture en mode "Réunion"
**Then** l'audio est transcrit avec labels de speaker (Speaker 1, Speaker 2...)

### AC3 — Attribution des actions par speaker
**Given** une transcription multi-speaker
**When** le LLM analyse la transcription
**Then** chaque action extraite est associée au speaker concerné :
- `assigned_to: "me"` → Todo pour l'utilisateur
- `assigned_to: "Speaker 2"` → Todo avec note "à suivre avec [contact]"

### AC4 — Résolution du speaker → contact
**Given** Speaker 2 identifié
**When** le ContactResolutionPipeline tourne
**Then** si un seul autre participant est dans Google Contacts, la résolution est automatique

## Dev Notes
- Whisper.rn : vérifier support diarization dans la version actuelle
- Alternative : pyannote.audio — modèle séparé, plus lourd

## References
- Story 19-13 (ContactResolutionPipeline)
- ADR-036 (local-first — diarization on-device)

# Story 23.18 — PDF & Document Analysis

Status: backlog

## Story

As a **user receiving contracts, quotes and invoices**,
I want **to share a PDF to Pensine and extract key dates, amounts and actions**,
So that **"devis Paul — deadline vendredi 500€" is automatically captured from a document**.

## Context

Partager un PDF depuis Files, Gmail ou WhatsApp → Pensine extrait le texte (PDFKit ou
ML Kit Document Scanner) → pipeline LLM identique au pipeline audio.

Cas d'usage prioritaires : devis, contrats, factures, documents administratifs.

## Acceptance Criteria

### AC1 — Pensine comme cible de partage pour les PDF
**Given** un PDF dans Files / Gmail / WhatsApp
**When** l'utilisateur appuie sur "Partager"
**Then** Pensine apparaît comme cible et accepte les fichiers PDF

### AC2 — Extraction de texte on-device
**Given** un PDF partagé
**When** Pensine le reçoit
**Then** le texte est extrait localement (ML Kit Document Scanner ou react-native-pdf)
**And** le pipeline LLM génère : résumé, dates clés, montants, contacts, actions

### AC3 — Extraction structurée pour documents financiers
**Given** une facture ou un devis
**When** le LLM analyse le texte extrait
**Then** les champs structurés sont extraits si présents :
- Montant total, date d'échéance, émetteur, destinataire
- Action automatique proposée : `ReminderCreate` si deadline détectée

### AC4 — Document stocké chiffré
**Given** le PDF partagé
**When** Pensine le traite
**Then** le fichier original est stocké chiffré localement (conforme ADR-017)
**And** seuls les métadonnées extraites sont synchées vers le backend

## References
- ADR-017 (chiffrement données locales)
- ADR-036 (local-first — extraction on-device)
- Story 23-2 (Camera + ML Kit — infrastructure similaire)

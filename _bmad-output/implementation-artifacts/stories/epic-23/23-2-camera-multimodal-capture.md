# Story 23.2 — Capture Multi-Modale : Camera + ML Kit (Android)

Status: backlog

## Story

As a **user**,
I want **to capture actions from photos** (business cards, whiteboards, receipts),
So that **Pensine can extract structured data without voice or typing**.

## Context

Étend la capture au-delà de la voix. ML Kit (Google) tourne 100% on-device —
zero cloud, zero coût, cohérent avec local-first (ADR-036).

**Cas d'usage :**
- Carte de visite → `ContactCreate` direct
- Tableau blanc / post-it de réunion → extraction d'actions multiples
- Devis / facture → extraction montant, date, fournisseur (future capability `ExpenseCreate`)

## Acceptance Criteria

### AC1 — Capture photo depuis le feed
**Given** l'écran principal de Pensine
**When** l'utilisateur appuie sur le bouton capture (long press ou icône dédié)
**Then** il peut choisir entre capture audio, texte, ou photo

### AC2 — OCR on-device via ML Kit
**Given** une photo capturée
**When** ML Kit Text Recognition traite l'image
**Then** le texte extrait est passé au pipeline LLM local (identique au pipeline audio)

### AC3 — Détection carte de visite → ContactCreate
**Given** une photo de carte de visite
**When** le LLM détecte un nom + email ou téléphone
**Then** un ExecutionJob `ContactCreate` est créé avec les params extraits

### AC4 — Whiteboard → extraction d'actions multiples
**Given** une photo de tableau blanc avec des notes
**When** le LLM analyse le texte OCR
**Then** plusieurs Todos / ExecutionJobs peuvent être créés depuis une seule photo

## Dev Notes
- **ML Kit Text Recognition v2** : `com.google.android.gms:play-services-mlkit-text-recognition`
- Android uniquement pour ML Kit natif ; iOS : Vision framework (même story, implémentation différente)
- Pas de nouvelle dépendance cloud

## References
- ADR-036 (local-first)
- ADR-034 (modèle sémantique)
- Story 19-13 (ContactResolutionPipeline — ContactCreate capability)

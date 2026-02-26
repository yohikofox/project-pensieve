# Story 23.7 — WhatsApp Integration

Status: backlog

## Story

As a **user**,
I want **Pensine to send WhatsApp messages on my behalf**,
So that **"envoie un WhatsApp à mon frère" works as naturally as Telegram**.

## Context

WhatsApp est l'app de messagerie la plus utilisée — plus naturelle que Telegram
pour la communication familiale et professionnelle.

**Deux mécanismes possibles :**
1. **Deep link** (`https://wa.me/{phone}?text={message}`) — ouvre WhatsApp, pas d'envoi automatique
2. **WhatsApp Business API** (cloud, payant) — envoi automatique mais frais + approbation Meta

Pour la Delegation BC, le deep link suffit pour MVP (Trust Level 1/2 avec validation) ;
le Business API est réservé aux power users ou aux usages professionnels (futur).

**Capability** : `WhatsAppMessage` (parallèle à `TelegramMessage`).

## Acceptance Criteria

### AC1 — Capability WhatsAppMessage dans Delegation BC
**Given** une Action classifiée `isExecutable: true` avec intent "WhatsApp"
**When** le LLM détecte "envoie un WhatsApp / message WhatsApp à [contact]"
**Then** un `ExecutionJob` avec `capability: "WhatsAppMessage"` est créé

### AC2 — Exécution via deep link (Trust Level 1/2)
**Given** un `ExecutionJob WhatsAppMessage` validé
**When** Pensine l'exécute
**Then** WhatsApp s'ouvre avec le contact et le message pré-rempli pour envoi manuel de confirmation

### AC3 — Contact resolution
**Given** `target: "mon frère"` dans l'ExecutionJob
**When** le ContactResolutionPipeline (story 19-13) résout le contact
**Then** le numéro WhatsApp est résolu depuis Google Contacts

## Dev Notes
- Deep link format : `intent://send?phone=+33XXXXXXXXX&text=Message#Intent;scheme=whatsapp;package=com.whatsapp;end`
- WhatsApp Business API : à évaluer post-monétisation si besoin d'envoi fully automated

## References
- Story 19-13 (ContactResolutionPipeline)
- Story 21-1 (TelegramMessage — pattern similaire)
- ADR-033 (Delegation BC — capabilities)

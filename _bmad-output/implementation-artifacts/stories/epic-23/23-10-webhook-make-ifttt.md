# Story 23.10 — WebhookSend : Make / IFTTT / Zapier

Status: backlog

## Story

As a **user wanting to automate anything**,
I want **Pensine to trigger a webhook when an intent is captured**,
So that **Make, IFTTT or Zapier can route it to any app without me building each integration**.

## Context

Un seul connecteur → accès à 1000+ apps. C'est l'extension point universel pour tout
ce que Pensine ne veut pas développer nativement (Trello, Linear, Slack, Airtable,
Google Sheets, Jira, HubSpot...).

**Capability** : `WebhookSend` dans Delegation BC.

**Cas d'usage :**
- "Crée une carte Trello pour [projet]" → webhook → Make → Trello
- "Ajoute une ligne dans mon Google Sheet budget" → webhook → Make → Sheets
- "Crée un ticket Linear pour [bug]" → webhook → Make → Linear

## Acceptance Criteria

### AC1 — Configuration webhooks dans les Settings
**Given** Settings > Intégrations > Webhooks
**When** l'utilisateur configure un webhook
**Then** il peut définir :
- URL de destination
- Méthode (POST par défaut)
- Headers custom (ex: Authorization)
- Mapping des champs Pensine → payload webhook

### AC2 — Capability WebhookSend dans Delegation BC
**Given** un `ExecutionJob` avec `capability: "WebhookSend"`
**When** Pensine l'exécute
**Then** un POST est envoyé vers l'URL configurée avec le payload :
```json
{
  "title": "...",
  "capability_hint": "...",
  "target": "...",
  "deadline_date": "...",
  "params": { ... },
  "capture_id": "...",
  "captured_at": "..."
}
```

### AC3 — Retry sur échec
**Given** un webhook qui échoue (timeout, 5xx)
**When** l'erreur est détectée
**Then** l'`ExecutionJob` passe dans la queue de retry (conforme NFR20 — offline queue story 19-11)

### AC4 — Webhook templates prédéfinis
**Given** la page de configuration
**When** l'utilisateur veut créer un webhook
**Then** des templates sont disponibles : Make, IFTTT, Zapier, n8n (URL + payload pré-configurés)

## References
- ADR-033 (Delegation BC — capabilities extensibles)
- Story 19-11 (offline queue + retry)
- ADR-025 (HTTP client — fetch natif)

# ADR-033 — Nouveau Bounded Context : Delegation

**Date:** 2026-02-26
**Statut:** Accepté
**Auteurs:** yohikofox, Winston (Architect)
**Session:** Architecture Mode Délégation

---

## Contexte

Le Mode Délégation (FR32-FR49) introduit un domaine métier distinct des todos humaines :
- Vocabulaire propre : ExecutionJob, Intent, TrustLevel, ExecutionCapability, DelayedSend
- Cycle de vie incompatible avec Todo : `pending_validation → ambiguous → validated → queued → executed`
- Expert métier distinct : pipeline d'automation, intégrations API tierces, gestion de confiance
- Changements indépendants : modifier la logique de validation n'impacte pas les todos

Les 4 tests DDD (vocabulaire, expert, changement, cycle de vie) concluent unanimement à un nouveau BC.

## Décision

**Créer le Bounded Context `Delegation`**, séparé de `Task BC`.

**Responsabilités de Delegation BC :**
- Recevoir les Actions exécutables depuis l'événement `CaptureDigested`
- Créer et persister les `ExecutionJob` en local (OP-SQLite)
- Implémenter le trust model (niveaux 1/2/3)
- Exécuter les actions via les intégrations tierces (Google Calendar, Gmail, Apple Calendar, Apple Reminders, Telegram)
- Gérer l'offline queue (persistance + retry à la reconnexion)
- Gérer le Delayed Send email (exception universelle — même niveau 3)

**Hors périmètre de Delegation BC :**
- La création de Todos (→ Task BC)
- La digestion et l'extraction d'intents (→ Knowledge BC)
- La gestion des triggers de déclenchement (→ mobile infrastructure)

## Frontières et Communication

```
Knowledge BC
  → émet CaptureDigested { extractedActions: Action[] }
        ↓
  Task BC (subscribe) → crée Todo pour chaque Action { isExecutable: false }
  Delegation BC (subscribe) → crée ExecutionJob pour chaque Action { isExecutable: true }
```

Pas de communication directe entre Task BC et Delegation BC. Un Todo ne devient pas un ExecutionJob automatiquement — l'utilisateur peut "déléguer" manuellement (future feature).

## Aggregate Principal

```typescript
ExecutionJob {
  id: UUID
  captureId: UUID
  action: Action
  capability: 'CalendarCreate' | 'EmailSend' | 'ReminderCreate' | 'TelegramMessage'
  state: 'pending_validation' | 'ambiguous' | 'validated' | 'rejected'
        | 'queued' | 'executed' | 'failed' | 'cancelled'
  trustLevel: 1 | 2 | 3
  params: Record<string, unknown>
  delayedSendExpiry?: DateTime
  createdAt: DateTime
  executedAt?: DateTime
}
```

## Règle fondamentale

> **Intégrité avant commodité.** Aucune action ne s'exécute sans intent confirmé (NFR19). En cas d'ambiguïté, l'ExecutionJob passe en état `ambiguous` — jamais en exécution.

## Conséquences

**Positives :**
- Isolation complète de la complexité d'exécution (OAuth, retry, offline, trust model)
- Task BC reste simple (CRUD todos)
- Chaque BC peut évoluer indépendamment

**Négatives :**
- Nouveau BC à implémenter de zéro (Epics 19-21)
- Surface de test plus large

## Références

- Epics 19, 20, 21 (implémentation)
- ADR-034 (modèle sémantique Action/Todo/ExecutionJob)
- ADR-035 (chorégraphie event-driven)
- FR37-FR49, NFR19-NFR21

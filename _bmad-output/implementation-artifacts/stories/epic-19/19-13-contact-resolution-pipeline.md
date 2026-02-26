# Story 19.13 — Contact Resolution Pipeline

Status: backlog

## Story

As a **user dictating a capture**,
I want **Pensine to automatically resolve "Paul" to Paul Durand's actual email/details**,
So that **ExecutionJobs can be executed without me having to manually fill in contact details**.

## Context

Le LLM extrait un `target` flou depuis la capture (ex: `"Paul"`). Sans résolution,
les `ExecutionJob` de type `EmailSend`, `CalendarCreate` ou `TelegramMessage` ne peuvent
pas s'exécuter — les params sont incomplets.

Ce pipeline résout le `target` **post-LLM**, avant la transition vers `pending_validation`.
La séparation est intentionnelle : le LLM extrait l'intent, le resolver enrichit les params.

**Golden path validé :** Google Contacts uniquement (Gmail + Google Contacts),
utilisateurs Free/Plus/Pro (pas Enterprise/Edu/Team).

**Réutilise :** OAuth2 Google de story 19-7 (scope `contacts.readonly` ajouté).

## Acceptance Criteria

### AC1 — Interface IContactResolver et chaîne de résolution

**Given** un `target: string` extrait par le LLM
**When** `ContactResolutionService.resolve(target)` est appelé
**Then** la chaîne suivante est exécutée dans l'ordre :
1. `GoogleContactsResolver` (si permission + token OAuth valide)
2. `LocalContactsResolver` (expo-contacts, si permission)
3. `ambiguous` si aucune chaîne n'aboutit

```typescript
interface IContactResolver {
  resolve(target: string): Promise<ContactResolution>
}

type ContactResolution =
  | { status: 'resolved'; contact: ResolvedContact }
  | { status: 'ambiguous'; candidates: ResolvedContact[] }  // 2+ matches
  | { status: 'not_found' }

type ResolvedContact = {
  display_name: string
  email?: string
  phone?: string
  contact_id?: string   // gcid_xxx pour Google, local ID pour expo-contacts
  source: 'google' | 'local'
}
```

### AC2 — GoogleContactsResolver : cas 1 (match unique)

**Given** un token OAuth Google valide avec scope `contacts.readonly`
**When** `GoogleContactsResolver.resolve("Paul")` trouve exactement 1 contact
**Then** :
- `ContactResolution.status = 'resolved'`
- `params` de l'`ExecutionJob` sont enrichis avec les données du contact
- `ExecutionJob.state` transite vers `pending_validation`

**Enrichissement params par capability :**

| Capability | Champ enrichi |
|-----------|--------------|
| `EmailSend` | `params.to = contact.email` |
| `CalendarCreate` | `params.attendee = contact.email` |
| `TelegramMessage` | `params.to = contact.phone` |
| `ReminderCreate` | Pas de résolution requise (target optionnel) |

### AC3 — Cas 2 : ambiguïté (0 match ou 2+ matches)

**Given** `resolve("Paul")` retourne 0 ou 2+ résultats
**When** la résolution échoue
**Then** :
- `ExecutionJob.state = 'ambiguous'`
- Une notification push est envoyée via le `NotificationService` existant :
  - **0 match** : `"Pour [capability], impossible de trouver « Paul ». Fournissez son [email / Telegram / téléphone]."`
  - **2+ matches** : `"Quel Paul ? [Paul Durand] [Paul Martin] [Autre]"`
- L'`ExecutionJob` reste en `ambiguous` jusqu'à résolution manuelle

### AC4 — Résolution manuelle depuis la notification

**Given** un `ExecutionJob` en état `ambiguous`
**When** l'utilisateur répond à la notification (sélectionne un contact, saisit un email ou joint une vCard)
**Then** :
- Les `params` de l'`ExecutionJob` sont enrichis
- `ExecutionJob.state` transite vers `pending_validation`
- La notification est dismissée

**Actions disponibles dans la notification selon la capability :**

| Capability | Options proposées |
|-----------|-----------------|
| `EmailSend` | "Choisir dans contacts" · "Saisir un email" · "Joindre une vCard" |
| `CalendarCreate` | "Choisir dans contacts" · "Saisir un email" · "Ignorer l'invité" |
| `TelegramMessage` | "Choisir dans contacts" · "Saisir un username/numéro" |

### AC5 — Extension point IContactResolver

**Given** l'interface `IContactResolver`
**When** un futur resolver tiers (CRM, HubSpot, etc.) est implémenté
**Then** il peut être ajouté à la chaîne de résolution sans modifier le `ContactResolutionService`

```typescript
// DI token
const CONTACT_RESOLVERS = Token<IContactResolver[]>('IContactResolver[]')

// Enregistrement (ordre = priorité)
container.registerInstance(CONTACT_RESOLVERS, [
  new GoogleContactsResolver(googleOAuthService),
  new LocalContactsResolver(expoContacts),
  // Futur : new HubSpotResolver(hubspotClient),
])
```

### AC6 — Pas de résolution pour ReminderCreate

**Given** un `ExecutionJob` de type `ReminderCreate`
**When** le pipeline de résolution est invoqué
**Then** la résolution de contact est **skippée** — `target` est optionnel pour les rappels personnels

## Dev Notes

### Dépendances

- **Story 19-4** : `ExecutionJob` + machine d'états (état `ambiguous` requis)
- **Story 19-5** : Trust model N1 (la résolution précède la validation)
- **Story 19-7** : OAuth2 Google partagé — ajouter scope `contacts.readonly`
- **NotificationService** existant (Epic 13/14) pour les notifications push

### Google People API

```typescript
// Endpoint de recherche
GET https://people.googleapis.com/v1/people:searchContacts
  ?query=Paul
  &readMask=names,emailAddresses,phoneNumbers
  &Authorization: Bearer {access_token}
```

Résultat → parser `connections[]` → filtrer par score de pertinence.

### Positionnement dans le pipeline ExecutionJob

```
CaptureDigested
  → Delegation BC crée ExecutionJob (target: "Paul", state: created)
  → ContactResolutionService.resolve("Paul")
    → resolved  → params enrichis → state: pending_validation
    → ambiguous → notification    → state: ambiguous (attend user input)
```

## Tasks

- [ ] Définir `IContactResolver`, `ContactResolution`, `ResolvedContact` dans `delegation/domain/`
- [ ] Implémenter `GoogleContactsResolver` (People API, fuzzy match)
- [ ] Implémenter `LocalContactsResolver` (expo-contacts fallback)
- [ ] Implémenter `ContactResolutionService` (chaîne ordonnée)
- [ ] Intégrer la résolution dans le pipeline de création d'`ExecutionJob`
- [ ] Implémenter les notifications d'ambiguïté (NotificationService)
- [ ] Implémenter le handler de réponse utilisateur (enrichissement params)
- [ ] Ajouter scope `contacts.readonly` à l'OAuth2 Google (story 19-7)
- [ ] Tests acceptance : résolution automatique, ambiguïté, réponse utilisateur

## References

- ADR-033 — Delegation BC (`ambiguous` dans la machine d'états)
- ADR-034 — Modèle sémantique Action/Todo/ExecutionJob
- ADR-019 — EventBus RxJS (events de résolution)
- Story 19-4 — ExecutionJob + machine d'états
- Story 19-7 — OAuth2 Google (scope partagé)

# Story 27.2: PAT Mobile — Écran Gestion des Personal Access Tokens

Status: ready-for-dev

## Story

En tant qu'**utilisateur de Pensine**,
je veux **gérer mes Personal Access Tokens directement depuis l'application mobile**,
afin de **contrôler en autonomie l'accès de mes outils externes (MCP, scripts) à mes données**.

## Context & Motivation

Dépend de Story 27.1 (backend PAT). L'écran est accessible depuis les Settings. Le token ne s'affiche qu'une seule fois à la création — impératif UX de sécurité.

## Acceptance Criteria

### AC1 : Accès depuis les Settings
**Given** l'utilisateur est sur l'écran Settings
**When** il navigue vers "Accès API" (ou "Personal Access Tokens")
**Then** la liste de ses PATs est affichée (nom, prefix, scopes, expiry, last_used_at)
**And** les PATs révoqués ou expirés sont clairement distingués visuellement

### AC2 : Création d'un PAT
**Given** l'utilisateur est sur l'écran PAT
**When** il appuie sur "Créer un token"
**Then** un formulaire s'affiche avec : nom (obligatoire), scopes (checkboxes), durée (7j / 30j / 90j / 1an)
**When** il valide
**Then** le token complet (`pns_...`) est affiché dans une modale dédiée avec bouton "Copier"
**And** un message d'avertissement indique que le token ne sera plus visible après fermeture
**And** la modale ne peut être fermée qu'après avoir appuyé sur "J'ai copié mon token"

### AC3 : Copie du token
**Given** la modale de création est affichée
**When** l'utilisateur appuie sur "Copier"
**Then** le token est copié dans le presse-papier
**And** un feedback visuel confirme la copie (icône check, toast)

### AC4 : Modification d'un PAT (nom et scopes)
**Given** un PAT actif dans la liste
**When** l'utilisateur appuie sur "Modifier"
**Then** un formulaire pré-rempli s'affiche (nom, scopes)
**And** il peut modifier le nom et/ou les scopes
**And** la durée d'expiry ne peut pas être modifiée (immutable après création)

### AC5 : Renew d'un PAT
**Given** un PAT actif dans la liste
**When** l'utilisateur appuie sur "Renouveler"
**Then** une confirmation s'affiche : "Renouveler ce token révoquera l'ancien immédiatement. Votre client devra être reconfiguré."
**When** il confirme
**Then** le nouveau token est affiché dans la modale (même flow que AC2)
**And** l'ancien token est immédiatement invalidé

### AC6 : Révocation d'un PAT
**Given** un PAT actif dans la liste
**When** l'utilisateur appuie sur "Révoquer" et confirme
**Then** le PAT passe à l'état "révoqué" visuellement dans la liste
**And** toute requête avec l'ancien token retourne 401

### AC7 : États visuels des PATs
- **Actif** : badge vert, last_used_at affiché si disponible
- **Expiré** : badge gris, mention "Expiré le [date]"
- **Révoqué** : badge rouge, mention "Révoqué le [date]"
- Les PATs révoqués et expirés sont affichés en bas de liste, collapsables

### AC8 : Gestion offline
**Given** l'utilisateur n'a pas de connexion réseau
**When** il accède à l'écran PAT
**Then** la liste est affichée depuis le cache local (lecture seule)
**And** les actions de création/modification/renew/révocation sont désactivées avec message "Connexion requise"

## Technical Specification

### Navigation

```
Settings
└── Accès API (PersonalAccessTokensScreen)
    ├── Liste des PATs
    ├── → PATCreateScreen (formulaire)
    └── → PATTokenDisplayModal (affichage unique du token)
```

### Composants

```
mobile/src/screens/settings/pat/
├── PersonalAccessTokensScreen.tsx    # liste + actions
├── PATCreateScreen.tsx               # formulaire création/modification
├── PATTokenDisplayModal.tsx          # affichage unique + copie
└── components/
    ├── PATListItem.tsx               # item de liste avec badge état
    └── PATScopeSelector.tsx          # checkboxes scopes
```

### Store Zustand / React Query

- `usePatList()` — React Query, `GET /api/auth/pat`
- `useCreatePat()` — mutation, retourne token en clair pour affichage
- `useUpdatePat()` — mutation PATCH
- `useRenewPat()` — mutation POST /renew, retourne nouveau token
- `useRevokePat()` — mutation DELETE

## Tasks / Subtasks

### Task 1 : Navigation et structure (AC1)

- [ ] Subtask 1.1 : Ajouter entrée "Accès API" dans le menu Settings
- [ ] Subtask 1.2 : Créer `PersonalAccessTokensScreen` avec liste React Query
- [ ] Subtask 1.3 : Enregistrer la route dans `registry.ts`

### Task 2 : Hooks API (AC1-AC6)

- [ ] Subtask 2.1 : `usePatList()` — fetch + cache React Query
- [ ] Subtask 2.2 : `useCreatePat()` — mutation création, retourne `{ token, pat }`
- [ ] Subtask 2.3 : `useUpdatePat()` — mutation PATCH
- [ ] Subtask 2.4 : `useRenewPat()` — mutation renew, retourne `{ token, pat }`
- [ ] Subtask 2.5 : `useRevokePat()` — mutation DELETE + invalidation cache

### Task 3 : PATListItem + états visuels (AC1, AC7)

- [ ] Subtask 3.1 : Composant `PATListItem` avec badge état (actif/expiré/révoqué)
- [ ] Subtask 3.2 : Affichage prefix, scopes, expiry, last_used_at
- [ ] Subtask 3.3 : Actions contextuelles (modifier, renouveler, révoquer) selon état

### Task 4 : Formulaire création/modification (AC2, AC4)

- [ ] Subtask 4.1 : `PATCreateScreen` — champ nom, `PATScopeSelector`, durée (picker)
- [ ] Subtask 4.2 : Validation formulaire (nom requis, min 1 scope)
- [ ] Subtask 4.3 : Mode édition pré-rempli (modifier)

### Task 5 : PATTokenDisplayModal — affichage unique (AC2, AC3)

- [ ] Subtask 5.1 : Modale non-dismissable avant confirmation "J'ai copié mon token"
- [ ] Subtask 5.2 : Bouton copier → clipboard + feedback visuel
- [ ] Subtask 5.3 : Message d'avertissement sécurité bien visible

### Task 6 : Confirm dialogs — renew et révocation (AC5, AC6)

- [ ] Subtask 6.1 : Dialog confirmation renew avec explication impact (client à reconfigurer)
- [ ] Subtask 6.2 : Dialog confirmation révocation

### Task 7 : Gestion offline (AC8)

- [ ] Subtask 7.1 : Désactiver les actions mutation si `!isConnected`
- [ ] Subtask 7.2 : Message informatif "Connexion requise"

### Task 8 : Tests

- [ ] Subtask 8.1 : Tests unitaires composants (PATListItem, PATTokenDisplayModal)
- [ ] Subtask 8.2 : Tests BDD — scénarios création, copie, renew, révocation
- [ ] Subtask 8.3 : `npm run test:acceptance` — zéro régression

## Dev Agent Record

### Debug Log References

### Completion Notes List

### File List

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-03-10 | Story créée | yohikofox |

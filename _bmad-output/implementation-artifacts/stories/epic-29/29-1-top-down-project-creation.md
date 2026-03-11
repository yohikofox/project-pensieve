# Story 29.1 — Top-Down Project Creation

Status: backlog

## Story

As a **user with a clear project in mind**,
I want **to create a Project intentionally from the app**,
So that **I can organize my captures and future GitHub issues around a named initiative**.

## Context

Actuellement, les Projects n'existent que via la voie Bottom-Up (Story 23-12 : clustering
sémantique émergent). Cette story introduit la voie **Top-Down** : l'utilisateur crée un
Project manuellement.

Le `Project` est un agrégat du BC **Opportunity**. Cette story l'enrichit avec les champs
nécessaires à l'intégration Git (ajoutés pour l'Epic 29), sans implémenter ces intégrations.

**Modèle de données enrichi :**
```
Project {
  id:               string (uuidv7)
  name:             string
  description?:     string
  status:           'active' | 'archived'
  repositoryLayout: 'monorepo' | 'polyrepo' | 'unset'  ← hint outillage futur
  createdAt:        timestamptz
  updatedAt:        timestamptz
  deletedAt?:       timestamptz  ← soft delete (ADR-026 R4)
}
```

**Permission requise :** `project:create` (RBAC, backend authorization module).

**Feature flag :** `projects` (déjà existant depuis Story 8-22) — le bouton de création
n'est visible que si le flag est actif.

## Acceptance Criteria

### AC1 — Accès à la création
**Given** l'utilisateur a le feature flag `projects` actif
**When** il navigue vers l'onglet Projets
**Then** un bouton "Nouveau projet" est visible

### AC2 — Formulaire de création
**Given** l'utilisateur tape sur "Nouveau projet"
**When** le formulaire s'ouvre
**Then** il peut renseigner :
- Nom (obligatoire, max 100 caractères)
- Description (optionnel, max 500 caractères)
- Layout du repo (`monorepo` / `polyrepo` / non défini — optionnel, avec tooltip explicatif)

### AC3 — Validation et persistance
**Given** le formulaire est valide (nom non vide)
**When** l'utilisateur confirme
**Then** :
- Le Project est créé en local (OP-SQLite) avec `status: 'active'`
- Il apparaît dans la liste des projets
- Un event `ProjectCreated` est émis sur l'EventBus

### AC4 — Permission RBAC
**Given** l'utilisateur n'a pas la permission `project:create`
**When** l'API backend est appelée
**Then** une erreur 403 est retournée et un message est affiché

### AC5 — Feature flag caché
**Given** l'utilisateur n'a pas le feature flag `projects` actif
**When** il navigue dans l'app
**Then** l'onglet Projets n'est pas visible (comportement inchangé depuis Story 8-22)

## Notes techniques

- L'onglet Projets est déjà gaté par le feature flag `projects` (Story 8-22) — pas de
  changement à ce gating.
- La liste des projets peut être vide initialement (état "empty state" avec CTA de création).
- Le champ `repositoryLayout` est un hint pour des outils futurs (Claude Code, scripts de
  scaffold) : `monorepo` → créer un domaine dans le même repo ; `polyrepo` → vérifier si
  le domaine existe ailleurs avant de créer.
- Sync backend : le Project est créé localement d'abord, puis répliqué via le pipeline
  de sync existant (Epic 6).

## References

- Story 8-22 (feature flag `projects`)
- Story 23-12 (voie Bottom-Up — complémentaire)
- ADR-026 (BaseEntity + soft delete)
- ADR-023 (Result Pattern)
- Epic 6 (sync infrastructure)

---
adr: ADR-026
title: "Backend Data Model Design Rules - Règles de Conception du Modèle de Données"
date: 2026-02-17
status: "✅ Accepted"
context: "Phase 4 - Implementation - Tous les Epics Backend"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-026: Backend Data Model Design Rules

**Date:** 2026-02-17
**Status:** ✅ Accepté
**Context:** Tous les bounded contexts backend (NestJS + PostgreSQL)
**Decision Makers:** yohikofox (Product Owner), Winston (Architect)

---

## Context & Problem

Au fil de l'implémentation, des incohérences ont émergé dans les choix de modélisation de données entre les différents bounded contexts du backend. Des conceptions incorrectes ont été détectées (colonnes texte pour les statuts, PKs importées du client mobile, absence de soft delete) nécessitant des corrections.

Ce document établit les règles canoniques applicables à **tous les contextes backend** pour prévenir toute régression future et servir de référence unique pour les agents d'implémentation.

---

## Decisions

### Règle 1 — Génération des Clés Primaires (PKs)

**Décision :** Les PKs sont générées par la **couche domaine (application)**, jamais par la base de données ni par un client externe.

**Rationale DDD :**
En DDD pur, une Entity possède une identité dès sa création. Un objet sans ID avant persistance n'est pas encore une Entity au sens DDD — c'est un problème conceptuel. Déléguer la génération d'ID à PostgreSQL (`gen_random_uuid()`) crée une dépendance d'infrastructure dans la couche domaine.

**Règle stricte :**
- ✅ UUID généré dans le constructeur ou la factory de l'entité domaine (`crypto.randomUUID()`)
- ✅ TypeORM : `@PrimaryColumn('uuid')` (stockage uniquement — la valeur vient du domaine)
- ❌ `@PrimaryGeneratedColumn('uuid')` — interdit (délègue la génération à la DB)
- ❌ Utiliser ou accepter une PK provenant d'un client mobile ou d'une source externe

```typescript
// ✅ Correct — l'entité génère son propre ID
export class Capture {
  readonly id: string;

  constructor(props: CaptureProps) {
    this.id = props.id ?? crypto.randomUUID();
    // ...
  }
}

// ✅ Correct — TypeORM stocke l'UUID fourni par le domaine
@Entity()
export class CaptureEntity {
  @PrimaryColumn('uuid')
  id: string;
}

// ❌ Interdit — génération déléguée à PostgreSQL
@PrimaryGeneratedColumn('uuid')
id: string;
```

---

### Règle 2 — Types et Statuts : Tables Référentielles Obligatoires

**Décision :** Aucune colonne de type `text` ou `varchar` ne doit stocker un type ou un statut métier. Toute valeur catégorielle doit être représentée par une **table référentielle** avec une **FK**.

**Pourquoi pas les ENUMs PostgreSQL :**
- `ALTER TYPE` pour ajouter/renommer une valeur = opération DDL avec lock potentiel en production
- Supprimer une valeur d'un enum est quasi-impossible sans recréer le type entier
- Aucune métadonnée possible (libellé, ordre, actif/inactif, description)
- Migrations fragiles et destructives

**Les tables référentielles sont supérieures :**
- Évolution sans migration destructive
- Métadonnées extensibles (libellé i18n, ordre d'affichage, actif/inactif)
- FK assure l'intégrité référentielle
- Lisible et auto-documenté

**Règle stricte :**
- ✅ Table dédiée `capture_statuses`, `action_types`, `notification_types`, etc.
- ✅ FK vers la table référentielle dans l'entité
- ❌ `status VARCHAR(50)` ou `type TEXT` directement dans la table principale
- ❌ Enums PostgreSQL (`CREATE TYPE ... AS ENUM`)

```sql
-- ✅ Correct — table référentielle
CREATE TABLE capture_statuses (
  id UUID PRIMARY KEY,
  code VARCHAR(50) NOT NULL UNIQUE,  -- ex: 'pending', 'transcribed', 'digested'
  label VARCHAR(100) NOT NULL,
  display_order INT NOT NULL DEFAULT 0,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE captures (
  id UUID PRIMARY KEY,
  status_id UUID NOT NULL REFERENCES capture_statuses(id),
  -- ...
);

-- ❌ Interdit
CREATE TABLE captures (
  id UUID PRIMARY KEY,
  status TEXT NOT NULL,  -- 'pending', 'transcribed'... sans contrainte réelle
);
```

**Exception :** Les tables référentielles elles-mêmes peuvent utiliser `VARCHAR(50)` pour le champ `code` (valeur technique, stable, jamais affichée directement).

---

### Règle 3 — Cascades : Interdites par Défaut

**Décision :** Les cascades (`ON DELETE CASCADE`, `ON UPDATE CASCADE`) sont **interdites par défaut**. Chaque exception doit faire l'objet d'une décision explicite documentée.

**Rationale :**
- Les cascades cachent des suppressions involontaires de données
- Difficiles à auditer et à déboguer en production
- Incompatibles avec le soft delete (une suppression en cascade contourne la logique applicative)
- La suppression doit être un acte intentionnel et traçable

**Règle stricte :**
- ✅ FK simples sans cascade
- ✅ La suppression logique passe par le soft delete (voir Règle 4)
- ✅ Si une dépendance doit être nettoyée, le faire explicitement dans la couche applicative
- ❌ `ON DELETE CASCADE` sauf exception documentée et approuvée par yohikofox

---

### Règle 4 — Soft Delete

**Décision :** Toutes les tables **non référentielles** implémentent le soft delete via une colonne `deleted_at TIMESTAMPTZ`.

**Règle stricte :**
- ✅ Colonne `deleted_at TIMESTAMPTZ NULL` sur toutes les tables de données
- ✅ TypeORM : `@DeleteDateColumn({ type: 'timestamptz' })` — active le soft delete natif
- ✅ Index sur `deleted_at` pour les requêtes filtrant les enregistrements actifs
- ❌ Suppression physique des données utilisateur (sauf obligation légale RGPD explicite)
- ❌ Soft delete sur les tables référentielles (elles ont un champ `is_active` à la place)

```typescript
// ✅ Correct — TypeORM soft delete natif
@Entity()
export class CaptureEntity extends BaseEntity {
  @DeleteDateColumn({ type: 'timestamptz' })
  deletedAt: Date | null;
}

// Usage : soft delete automatique
await repository.softRemove(capture);

// Récupérer les supprimés (si besoin)
await repository.find({ withDeleted: true });
```

---

### Règle 5 — Types de Données Canoniques

**Décision :** Types PostgreSQL standardisés pour tout le backend.

| Usage | Type PostgreSQL | TypeORM Décorateur | Interdit |
|---|---|---|---|
| Clé primaire | `UUID` | `@PrimaryColumn('uuid')` | `BIGINT`, `SERIAL` |
| Clé étrangère | `UUID` | `@Column({ type: 'uuid' })` | `BIGINT`, `INT` |
| Date + heure | `TIMESTAMPTZ` | `@CreateDateColumn({ type: 'timestamptz' })` | `TIMESTAMP` (sans tz) |
| Date seule | `DATE` | `@Column({ type: 'date' })` | `VARCHAR` |
| Texte court | `VARCHAR(n)` | `@Column({ type: 'varchar', length: n })` | — |
| Texte long | `TEXT` | `@Column({ type: 'text' })` | — |
| Booléen | `BOOLEAN` | `@Column({ type: 'boolean' })` | `INT` (0/1) |
| Montant/nombre | `NUMERIC(p,s)` | `@Column({ type: 'numeric', precision: p, scale: s })` | `FLOAT` |

**Règles spécifiques :**
- **UUID** : toujours pour les PKs et FKs — jamais `bigint` ou `serial`
- **TIMESTAMPTZ** : toujours pour les dates — stockage exclusivement en UTC avec information timezone
- **Jamais** de `TIMESTAMP` sans timezone (perte de l'information de fuseau)
- La connexion PostgreSQL doit être configurée avec `timezone: 'UTC'`

---

### Règle 6 — Entité de Base Commune (Mapped Superclass)

**Décision :** Toutes les entités TypeORM héritent d'une `BaseEntity` abstraite factorisant les champs communs.

```typescript
// src/common/entities/base.entity.ts
export abstract class BaseEntity {
  @PrimaryColumn('uuid')
  id: string;

  @CreateDateColumn({ type: 'timestamptz' })
  createdAt: Date;

  @UpdateDateColumn({ type: 'timestamptz' })
  updatedAt: Date;

  @DeleteDateColumn({ type: 'timestamptz' })
  deletedAt: Date | null;
}

// Usage
@Entity('captures')
export class CaptureEntity extends BaseEntity {
  // Pas besoin de redéclarer id, createdAt, updatedAt, deletedAt
  @Column({ type: 'varchar', length: 255 })
  title: string;
}
```

**Règle stricte :**
- ✅ Pattern **Data Mapper** exclusivement (pas Active Record — anti-pattern DDD)
- ✅ Toutes les entités héritent de `BaseEntity`
- ✅ Les tables référentielles héritent d'une `BaseReferentialEntity` sans `deletedAt` mais avec `is_active`
- ❌ Duplication des champs `id`, `createdAt`, `updatedAt` dans chaque entité

---

### Règle 7 — Multi-tenancy et Isolation des Données

**Décision :** Architecture multi-tenancy en 3 phases, conçue dès maintenant pour absorber l'évolution.

#### Phase 1 — Ownership Simple (actuel)

Chaque ressource possède un `owner_id` (UUID, FK → `users`). Toutes les requêtes filtrent sur `owner_id` via un middleware NestJS injectant le contexte utilisateur.

```sql
CREATE TABLE captures (
  id UUID PRIMARY KEY,
  owner_id UUID NOT NULL REFERENCES users(id),
  -- ...
);
```

#### Phase 2 — Partage et Délégation de Droits (futur)

Table `resource_shares` pour la délégation d'accès sans modifier les ressources elles-mêmes.

```sql
CREATE TABLE resource_shares (
  id UUID PRIMARY KEY,
  resource_type_id UUID NOT NULL REFERENCES resource_types(id),  -- table référentielle
  resource_id UUID NOT NULL,
  granted_by UUID NOT NULL REFERENCES users(id),
  granted_to_user UUID REFERENCES users(id),         -- nullable si org
  granted_to_org UUID REFERENCES organizations(id),  -- nullable si user
  permission_id UUID NOT NULL REFERENCES permissions(id),  -- table référentielle
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ NULL  -- soft delete = révocation
);
```

#### Phase 3 — Organisations (futur)

```sql
CREATE TABLE organizations (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(100) NOT NULL UNIQUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ NULL
);

CREATE TABLE organization_members (
  id UUID PRIMARY KEY,
  organization_id UUID NOT NULL REFERENCES organizations(id),
  user_id UUID NOT NULL REFERENCES users(id),
  role_id UUID NOT NULL REFERENCES organization_roles(id),  -- table référentielle
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ NULL,
  UNIQUE (organization_id, user_id)
);
```

**Règle stricte :**
- ✅ `owner_id` sur **toutes** les tables de données dès maintenant
- ✅ L'accès élargi passe **exclusivement** par `resource_shares`
- ✅ Aucune logique d'accès dans les colonnes de la ressource elle-même
- ❌ Partager une ressource en modifiant la ligne de la ressource (ex: colonne `shared: boolean`)

---

## Consequences

### Positives

✅ **Cohérence** : Règles uniformes sur tous les bounded contexts — moins de surprises inter-agents
✅ **Intégrité DDD** : Les entités possèdent leur identité dès la création
✅ **Évolutivité** : Tables référentielles évoluent sans migrations destructives
✅ **Traçabilité** : Soft delete conserve l'historique — 0 perte de données (NFR6)
✅ **Sécurité données** : `owner_id` systématique + isolation stricte via `resource_shares`
✅ **Multi-tenancy ready** : Architecture conçue pour absorber phases 2 et 3 sans refactoring majeur

### Négatives (et mitigation)

⚠️ **Tables référentielles à initialiser** : Les seeds de données de référence doivent être maintenus
→ Mitigation : Fichiers de migration dédiés aux seeds référentiels, versionnés

⚠️ **Correction des conceptions existantes** : Des stories déjà implémentées peuvent ne pas respecter ces règles
→ Mitigation : Audit et corrections à planifier par yohikofox — pas de dette silencieuse

⚠️ **Verbosité du schéma** : Plus de tables qu'une approche avec enums/text
→ Acceptable : La cohérence et l'évolutivité l'emportent

---

## Implementation Checklist

Pour chaque nouvelle entité backend :

- [ ] ID généré dans le constructeur/factory domaine (`crypto.randomUUID()`)
- [ ] `@PrimaryColumn('uuid')` dans l'entité TypeORM (pas `@PrimaryGeneratedColumn`)
- [ ] Hérite de `BaseEntity` (ou `BaseReferentialEntity` pour les référentiels)
- [ ] `owner_id` présent si la ressource appartient à un utilisateur
- [ ] Tous les types/statuts référencent une table référentielle via FK
- [ ] Aucun `text` ou `varchar` pour stocker une valeur catégorielle
- [ ] Aucun enum PostgreSQL
- [ ] Aucune cascade sans décision explicite documentée
- [ ] Toutes les dates en `TIMESTAMPTZ`

---

## References

- [ADR-001](./ADR-001-aggregate-granularity.md) - Aggregate Granularity (boundaries DDD)
- [ADR-017](./ADR-017-ioc-di-strategy.md) - DI Strategy Backend (NestJS native)
- TypeORM - Soft Delete: https://typeorm.io/delete-query-builder#soft-delete
- TypeORM - Entity Inheritance: https://typeorm.io/entity-inheritance
- Martin Fowler - Data Mapper Pattern: https://martinfowler.com/eaaCatalog/dataMapper.html
- PostgreSQL TIMESTAMPTZ: https://www.postgresql.org/docs/current/datatype-datetime.html

---

## Decision Log

**2026-02-17** — Discussion yohikofox, Winston

→ Déclencheur : Incohérences détectées dans des stories existantes (colonnes texte pour statuts, PKs importées du mobile)
→ Périmètre : Tous les bounded contexts backend, applicable immédiatement
→ Décisions : 7 règles canoniques validées par yohikofox
→ Multi-tenancy : Architecture 3 phases — owner_id maintenant, resource_shares phase 2, organizations phase 3
→ TypeORM : Data Mapper + Mapped Superclass + soft delete natif

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

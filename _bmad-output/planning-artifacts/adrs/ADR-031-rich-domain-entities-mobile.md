---
adr: ADR-031
title: "Rich Domain Entities — Classes avec Invariants pour le Mobile"
date: 2026-02-19
status: "✅ Accepted"
context: "Phase 4 - Implementation - Mobile (React Native)"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
supersedes: "N/A - Complément à ADR-024 (Clean Code Standards)"
---

# ADR-031: Rich Domain Entities — Classes avec Invariants pour le Mobile

**Date:** 2026-02-19
**Status:** Accepted
**Context:** Mobile — Tous les bounded contexts (capture, knowledge, action, identity)
**Decision Makers:** yohikofox (Product Owner), Winston (Architect)

---

## Context & Problem

**Problème identifié :**

Les modèles de domaine du package mobile sont définis comme des **interfaces TypeScript** — des structures de données pures sans comportement ni protection. Cela produit un **modèle de domaine anémique** (anti-pattern DDD reconnu).

**Symptômes observés :**

1. **Aucune protection d'invariant** : `Capture.state` peut être modifié arbitrairement depuis n'importe quel service, sans validation de la transition
2. **Validation absente à la construction** : on peut créer un objet `Capture` dans un état incohérent (ex: `state: 'ready'` sans `normalizedText`)
3. **Logique métier dispersée** : `isCaptureProcessing()` est une fonction libre dans `Capture.model.ts` au lieu d'être une méthode de l'entité
4. **Pas d'encapsulation** : tous les champs sont publics et mutables

**Entités concernées :**

| Contexte | Entité | Fichier |
|----------|--------|---------|
| Capture | `Capture` | `contexts/capture/domain/Capture.model.ts` |
| Capture | `CaptureAnalysis` | `contexts/capture/domain/CaptureAnalysis.model.ts` |
| Capture | `CaptureMetadata` | `contexts/capture/domain/CaptureMetadata.model.ts` |
| Knowledge | `Thought` | `contexts/knowledge/domain/Thought.model.ts` |
| Knowledge | `Idea` | `contexts/knowledge/domain/Idea.model.ts` |
| Action | `Todo` | `contexts/action/domain/Todo.model.ts` |
| Identity | `UserFeatures` | `contexts/identity/domain/user-features.model.ts` |

**Exception existante qui valide le pattern :** `FilePath` (Value Object dans Normalization) est déjà une classe avec constructeur privé, factory methods, et `equals()`. Il prouve que le pattern fonctionne dans le projet.

**Contrainte :** Le backend NestJS utilise des classes TypeORM avec décorateurs `@Entity` — cette ADR concerne **exclusivement le mobile**.

---

## Decision

### Règle 1 — Les entités de domaine DOIVENT être des classes

**Décision :** Chaque entité de domaine (aggregate root ou entité enfant) dans `contexts/**/domain/` est une **classe TypeScript**, pas une interface.

```typescript
// ❌ INTERDIT — interface = aucune protection
export interface Capture {
  id: string;
  state: CaptureState;
  rawContent: string;
}

// ✅ REQUIS — classe = invariants protégés
export class Capture {
  private constructor(
    readonly id: string,
    private _state: CaptureState,
    private _rawContent: string,
  ) {}
}
```

---

### Règle 2 — Constructeur privé + `create()` et `reconstitute()` comme points d'entrée

Le constructeur est **privé**. Deux factories statiques servent de points d'entrée :

- **`create()`** — création métier avec validation des invariants, retourne `Result<T>`
- **`reconstitute()`** — désérialisation depuis un `Snapshot`, sans validation métier

```typescript
export class Capture {
  private constructor(
    readonly id: string,
    private _state: CaptureState,
    private _rawContent: string,
  ) {}

  static create(props: CreateCaptureProps): RepositoryResult<Capture> {
    if (!props.rawContent && props.type !== CAPTURE_TYPES.AUDIO) {
      return validationError('rawContent is required for non-audio captures');
    }
    return success(new Capture(
      props.id,
      CAPTURE_STATES.CAPTURED,
      props.rawContent,
    ));
  }

  static reconstitute(snapshot: CaptureSnapshot): Capture {
    return new Capture(
      snapshot.id,
      snapshot.state,
      snapshot.rawContent,
    );
  }
}
```

**Séparation des responsabilités — sérialisation vs mapping :**

L'entité gère sa propre **sérialisation/désérialisation** (Snapshot ↔ Entity). Le repository gère le **mapping** (DB row ↔ Snapshot).

```
DB row → Repository (mapping) → Snapshot → reconstitute() → Entity
Entity → toSnapshot() → Snapshot → Repository (mapping) → DB row
```

| Responsable | Action | Exemple |
|-------------|--------|---------|
| **Entité** | Sérialisation | `entity.toSnapshot()` → `CaptureSnapshot` |
| **Entité** | Désérialisation | `Capture.reconstitute(snapshot)` → `Capture` |
| **Repository** | Mapping lecture | DB row (`snake_case`, types bruts) → `CaptureSnapshot` |
| **Repository** | Mapping écriture | `CaptureSnapshot` → SQL INSERT/UPDATE |

---

### Règle 3 — Propriétés exposées en lecture seule via getters

Les champs internes sont **privés**. L'accès externe passe par des getters `readonly` ou des propriétés `readonly` dans le constructeur.

```typescript
export class Capture {
  private constructor(
    readonly id: string,           // immutable, exposé directement
    private _state: CaptureState,  // mutable en interne, getter en lecture
  ) {}

  get state(): CaptureState { return this._state; }
}
```

**Convention de nommage :**
- Propriétés immutables : `readonly` dans le constructeur (pas de préfixe)
- Propriétés mutables en interne : préfixe `_` + getter public sans préfixe

---

### Règle 4 — Transitions d'état via méthodes métier retournant `Result<void>`

Les changements d'état passent par des **méthodes nommées** qui valident la transition et retournent un `Result<void>`.

```typescript
markAsProcessing(): RepositoryResult<void> {
  if (this._state !== CAPTURE_STATES.CAPTURED) {
    return businessError(
      `Invalid transition: cannot go from '${this._state}' to 'processing'`
    );
  }
  this._state = CAPTURE_STATES.PROCESSING;
  return success(undefined);
}

markAsReady(normalizedText: string): RepositoryResult<void> {
  if (this._state !== CAPTURE_STATES.PROCESSING) {
    return businessError(
      `Invalid transition: cannot go from '${this._state}' to 'ready'`
    );
  }
  this._normalizedText = normalizedText;
  this._state = CAPTURE_STATES.READY;
  return success(undefined);
}
```

**Jamais d'assignation directe :** `capture.state = 'ready'` est impossible (propriété privée).

---

### Règle 5 — Méthode `toSnapshot()` pour la sérialisation

Chaque entité expose une méthode `toSnapshot()` qui retourne un objet plain (interface) pour la persistence et la sérialisation.

```typescript
export interface CaptureSnapshot {
  id: string;
  state: CaptureState;
  rawContent: string;
  // ... tous les champs
}

export class Capture {
  toSnapshot(): CaptureSnapshot {
    return {
      id: this.id,
      state: this._state,
      rawContent: this._rawContent,
    };
  }
}
```

**Flux de données :**
```
DB (row) → Repository (mapping) → new Capture(...) → Use Case (logique métier)
                                                          ↓
                                   DB (row) ← Repository ← capture.toSnapshot()
```

- **Le repository** lit la DB, mappe les champs, et instancie l'entité via le constructeur
- **L'entité** expose `toSnapshot()` pour que le repository puisse la persister
- **Les use cases** manipulent des entités (classes), jamais des snapshots directement

---

### Règle 6 — Méthodes de requête métier sur l'entité

Les fonctions utilitaires libres (comme `isCaptureProcessing()`) doivent migrer en **méthodes d'instance**.

```typescript
// ❌ AVANT — fonction libre
export function isCaptureProcessing(capture: { state: CaptureState }): boolean {
  return capture.state === CAPTURE_STATES.PROCESSING;
}

// ✅ APRÈS — méthode d'instance
export class Capture {
  isProcessing(): boolean {
    return this._state === CAPTURE_STATES.PROCESSING;
  }

  isTerminal(): boolean {
    return this._state === CAPTURE_STATES.READY
        || this._state === CAPTURE_STATES.FAILED;
  }
}
```

---

### Règle 7 — Les Value Objects restent des classes (pattern existant maintenu)

`FilePath` et tout futur Value Object suivent le même pattern (classe, immutable, `equals()` basé sur la valeur). Cette ADR ne change rien pour les VOs — elle **étend** le pattern aux entités.

---

### Règle 8 — Les interfaces de snapshot servent de contrat pour les repositories

Le type `Snapshot` (ex: `CaptureSnapshot`) remplace l'ancienne `interface Capture` comme contrat de sérialisation. Le repository est responsable du **mapping** entre DB row et Snapshot.

```typescript
// Repository interface (couche domain) — manipule des entités
export interface ICaptureRepository {
  findById(id: string): Promise<RepositoryResult<Capture>>;  // retourne l'entité
  save(capture: Capture): Promise<RepositoryResult<void>>;    // accepte l'entité
}

// Repository implementation (couche data) — responsable du mapping DB ↔ Snapshot
class CaptureRepository implements ICaptureRepository {
  async findById(id: string): Promise<RepositoryResult<Capture>> {
    const row = db.execute('SELECT * FROM captures WHERE id = ?', [id]);
    if (!row) return notFound();
    // 1. Mapping DB row → Snapshot (responsabilité du repository)
    const snapshot: CaptureSnapshot = {
      id: row.id,
      state: row.state,
      rawContent: row.raw_content,  // snake_case → camelCase
    };
    // 2. Désérialisation Snapshot → Entity (responsabilité de l'entité)
    return success(Capture.reconstitute(snapshot));
  }

  async save(capture: Capture): Promise<RepositoryResult<void>> {
    // 1. Sérialisation Entity → Snapshot (responsabilité de l'entité)
    const s = capture.toSnapshot();
    // 2. Mapping Snapshot → DB row (responsabilité du repository)
    db.execute('INSERT OR REPLACE INTO captures ...', [s.id, s.state, ...]);
    return success(undefined);
  }
}
```

---

## Options Evaluated

| Critère | Interface (actuel) | Classe riche (choisi) | Classe + Branded Types |
|---------|--------------------|-----------------------|------------------------|
| Protection invariants | Aucune | Factory + transitions | Factory + transitions |
| Complexité | Minimale | Modérée | Élevée |
| Testabilité | Facile (plain objects) | Bonne (factory + reconstitute) | Complexe |
| Compatibilité DDD | Anti-pattern (anémique) | Conforme | Sur-engineered |
| Migration progressive | N/A | Entité par entité | Tout ou rien |
| Performance mobile | Marginale | Marginale | Impact mesurable |
| **Score** | **3/10** | **9/10** | **6/10** |

---

## Consequences

### Positives

1. **Invariants protégés** : impossible de créer une entité dans un état incohérent
2. **Transitions validées** : les machines à états sont formalisées et testables
3. **Logique co-localisée** : le comportement métier vit avec les données
4. **Découvrabilité** : `capture.` + autocomplétion montre toutes les actions possibles
5. **Tests plus expressifs** : `expect(result.type).toBe('BUSINESS_ERROR')` plutôt que vérifier manuellement les états
6. **Conformité DDD** : alignement avec les tactical patterns (Entity, Aggregate Root, Value Object)

### Trade-offs acceptés

1. **Migration progressive** : les entités existantes doivent être converties une par une — chaque conversion impacte les repositories et use cases associés
2. **Verbosité accrue** : `Capture.create({...})` au lieu de `{ id, state, ... }` directement
3. **Snapshot overhead** : `toSnapshot()` / `reconstitute()` ajoutent une indirection (mais facilitent le découplage persistence)
4. **Tests existants** : les tests BDD qui construisent des entités comme plain objects devront être adaptés pour utiliser `create()` ou `reconstitute()`

### Impact sur l'architecture existante

- Les **interfaces** `Capture`, `Thought`, `Todo`, etc. deviennent des types **Snapshot** (renommage)
- Les **repositories** gèrent le mapping DB ↔ Snapshot ; les entités gèrent la sérialisation/désérialisation Snapshot ↔ Entity
- Les **use cases/services** passent de la manipulation de données brutes à l'appel de méthodes métier
- Le **container DI** n'est pas impacté (les entités ne sont pas injectées)

---

## Implementation

### Migration progressive (story par story)

**Ordre recommandé** (par complexité croissante) :

| Priorité | Entité | Raison |
|----------|--------|--------|
| 1 | `Todo` | Machine à états simple (3 états), peu de dépendances |
| 2 | `CaptureMetadata` | Entité simple, clé-valeur |
| 3 | `CaptureAnalysis` | Peu de logique, mais liée à Capture |
| 4 | `Thought` | Entité Knowledge, modérée |
| 5 | `Idea` | Entité Knowledge enfant |
| 6 | `Capture` | Aggregate root complexe, machine à états riche, à faire en dernier |
| 7 | `UserFeatures` | Identity context, impact limité |

### Checklist par entité migrée

- [ ] Créer la classe avec constructeur privé, `create()` et `reconstitute()`
- [ ] Définir le type `Snapshot` (ancien interface renommé)
- [ ] Implémenter `toSnapshot()` sur l'entité
- [ ] Migrer les transitions d'état en méthodes
- [ ] Migrer les fonctions utilitaires libres en méthodes d'instance
- [ ] Adapter le repository (mapping DB row ↔ Snapshot, reconstitute/toSnapshot pour Entity ↔ Snapshot)
- [ ] Adapter les use cases/services
- [ ] Adapter les tests (BDD + unit)
- [ ] Vérifier le test d'architecture passe

### Files impactés (à terme)

```
mobile/src/contexts/capture/domain/Capture.model.ts
mobile/src/contexts/capture/domain/CaptureAnalysis.model.ts
mobile/src/contexts/capture/domain/CaptureMetadata.model.ts
mobile/src/contexts/capture/data/CaptureRepository.ts
mobile/src/contexts/capture/data/CaptureAnalysisRepository.ts
mobile/src/contexts/capture/data/CaptureMetadataRepository.ts
mobile/src/contexts/knowledge/domain/Thought.model.ts
mobile/src/contexts/knowledge/domain/Idea.model.ts
mobile/src/contexts/knowledge/data/ThoughtRepository.ts
mobile/src/contexts/knowledge/data/IdeaRepository.ts
mobile/src/contexts/action/domain/Todo.model.ts
mobile/src/contexts/action/data/TodoRepository.ts
mobile/src/contexts/identity/domain/user-features.model.ts
mobile/_patterns/08-domain-entity.ts
mobile/_patterns/README.md
mobile/tests/architecture/architecture.test.ts
```

---

## Validation Criteria

ADR considéré succès SI :

- [ ] Pattern `08-domain-entity.ts` créé et documenté
- [ ] Test d'architecture ajouté et passant
- [ ] Au moins une entité migrée (Todo recommandé comme pilote)
- [ ] Tests BDD existants adaptés et passants après migration pilote
- [ ] Aucune régression sur `npm run test:acceptance` après chaque migration

**Review Date :** 2026-04 (après migration des 3 premières entités)

---

## References

- ADR-023 — Error Handling Strategy (Result Pattern — utilisé par les factories)
- ADR-024 — Clean Code Standards (conventions générales)
- ADR-028 — TypeScript Type Safety Policy (pas de `any`)
- ADR-001 — Aggregate Granularity (séparation des aggregates)
- Martin Fowler — [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html)
- Eric Evans — Domain-Driven Design (Entity, Aggregate, Value Object tactical patterns)
- `mobile/_patterns/08-domain-entity.ts` — Pattern de référence

---

## Decision Log

**2026-02-19** — Discussion yohikofox + Winston

> yohikofox signale que les modèles de domaine mobile sont tous des interfaces — intuition qu'il manque de vraies entités.
> Winston confirme : modèle anémique, aucune protection d'invariant, logique dispersée dans les services.
> Seul `FilePath` (Value Object, Normalization) est une classe — prouve que le pattern fonctionne.
> Décision : migrer progressivement les interfaces vers des classes riches, commencer par `Todo` (le plus simple).
> Livraison immédiate : ADR-031 + pattern de référence + test d'architecture.

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

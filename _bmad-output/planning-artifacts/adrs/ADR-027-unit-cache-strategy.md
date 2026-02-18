---
adr: ADR-027
title: "Unit Cache Strategy — Cache Unitaire Opt-in par Héritage de Repository"
date: 2026-02-18
status: "✅ Accepted"
context: "Phase 4 - Implementation - Backend (NestJS + Redis)"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-027: Unit Cache Strategy

**Date:** 2026-02-18
**Status:** ✅ Accepté
**Context:** Backend — tout repository nécessitant un accès optimisé aux données
**Decision Makers:** yohikofox (Product Owner), Winston (Architect)

---

## Context & Problem

Au fil de l'implémentation, certains repositories accèdent fréquemment à des données stables ou coûteuses à requêter. Pour contourner ce coût, un antipattern a émergé : hardcoder des identifiants de base de données directement dans le code source (enums TypeScript, constantes, switch-cases).

```typescript
// ❌ Antipattern — UUID hardcodé dans le code
enum CaptureStatusId {
  PENDING     = 'a1b2c3d4-...',
  TRANSCRIBED = 'e5f6g7h8-...',
}
```

Ce couplage rend le code fragile (une migration de données = un déploiement), rend les tests dépendants de fixtures avec des IDs fixes, et contredit le principe DDD selon lequel le domaine ne connaît pas l'infrastructure.

La cause racine est légitime : éviter des lookups répétés vers des données peu changeantes. Il faut une solution qui **supprime ce besoin de hardcoder** tout en **évitant les lookups inutiles**.

---

## Decisions

### Règle 1 — Principe général : le cache est une décision infrastructure, opt-in

**Décision :** Le comportement de cache est encapsulé dans une classe abstraite `CacheableRepository<T>`. Un repository accède au cache en **héritant** de cette classe. L'entité domaine reste totalement ignorante du cache.

Ce pattern est applicable à **tout repository** dont les données justifient une mise en cache — qu'il s'agisse de données référentielles, de résultats de calculs coûteux, ou de tout autre entité à accès fréquent.

L'opt-in est **explicite** : un repository qui n'hérite pas de `CacheableRepository` ne bénéficie pas du cache. Aucun comportement de cache implicite n'est introduit.

---

### Règle 2 — Structure de la clé de cache et invalidation par namespace

**Décision :** Chaque entrée de cache est identifiée par une clé composite incluant un **namespace versionné** :

```
{namespace}:{id}:{ns_value}

Exemples :
  capture-status:a1b2c3d4-...:1708123456789
  concordance-score:e5f6g7h8-...:1708000000000
```

Le `ns_value` est la valeur courante du namespace, stockée séparément dans Redis :

```
ns:capture-status → "1708123456789"
ns:concordance-score → "1708000000000"
```

**Avantage :** Invalider toutes les entrées d'un namespace se fait en O(1), sans itérer sur les clés, en changeant simplement la valeur de `ns:{namespace}`.

---

### Règle 3 — Algorithme de retrieval : batch par IDs, missing résolu en DB

**Décision :** Le point d'entrée unique est `findByIds(ids[])`. L'algorithme est :

1. Construire les clés de cache pour tous les IDs demandés
2. Récupérer en cache (`mget`) — certains sont présents, d'autres manquants
3. Pour les IDs manquants uniquement : query DB par liste (`IN` / `ANY`) — jamais un loop
4. Stocker les résultats DB en cache
5. Retourner l'union des hits et des résultats DB

Ce pattern s'applique y compris pour un seul item : `findById(id)` passe par `findByIds([id])`.

Si un accès par clé naturelle (ex: `code`, `slug`) est nécessaire, une méthode dédiée résout d'abord l'ID (query légère indexée), puis délègue à `findByIds`.

```typescript
// infrastructure/cache/cacheable.repository.ts

export abstract class CacheableRepository<T extends { id: string }> {

  constructor(
    protected readonly cache: ICacheClient,
    protected readonly namespace: string,
  ) {}

  async findByIds(ids: string[]): Promise<T[]> {
    const ns = await this.resolveNamespace();
    const cacheKeys = ids.map(id => `${this.namespace}:${id}:${ns}`);

    const cached = await this.cache.mget<T>(cacheKeys);

    const hits       = cached.filter((v): v is T => v !== null);
    const missingIds = ids.filter((_, i) => cached[i] === null);

    if (missingIds.length === 0) return hits;

    const fromDb = await this.queryByIds(missingIds);
    await this.populateCache(fromDb, ns);

    return [...hits, ...fromDb];
  }

  async findByNaturalKey(key: string): Promise<T | null> {
    const id = await this.resolveIdByNaturalKey(key);
    if (!id) return null;
    const results = await this.findByIds([id]);
    return results[0] ?? null;
  }

  async invalidateAll(): Promise<void> {
    await this.cache.set(`ns:${this.namespace}`, Date.now().toString());
  }

  async invalidateOne(id: string): Promise<void> {
    const ns = await this.resolveNamespace();
    await this.cache.del(`${this.namespace}:${id}:${ns}`);
  }

  // ─── Primitives que l'enfant DOIT implémenter ──────────────────────────

  protected abstract queryByIds(ids: string[]): Promise<T[]>;

  // Optionnel — implémenter uniquement si findByNaturalKey est utilisé
  protected resolveIdByNaturalKey(_key: string): Promise<string | null> {
    throw new Error(`resolveIdByNaturalKey not implemented in ${this.namespace}`);
  }

  // ─── Interne ───────────────────────────────────────────────────────────

  private async resolveNamespace(): Promise<string> {
    return await this.cache.get(`ns:${this.namespace}`) ?? '0';
  }

  private async populateCache(entities: T[], ns: string): Promise<void> {
    const entries = entities.map(e => ({
      key: `${this.namespace}:${e.id}:${ns}`,
      value: e,
    }));
    await this.cache.mset(entries);
  }
}
```

**Ce que l'enfant ne fait jamais :**
- Pas de construction de clé de cache
- Pas d'appel direct à `this.cache`
- Pas de logique de hit/miss
- Pas d'invalidation dans le repository (responsabilité du service)

---

### Règle 4 — Pas de TTL : invalidation par événement uniquement

**Décision :** Les entrées de cache n'ont **pas de TTL**. Une entrée est valide jusqu'à invalidation explicite.

**Rationale :**
- Un TTL arbitraire crée des expirations non liées à un événement métier réel
- L'invalidation doit être causale : une donnée est invalidée parce qu'elle a changé, pas parce qu'un timer a expiré
- La pression de cache (LRU/mémoire Redis) est le seul mécanisme d'éviction passive acceptable

**Déclencheurs d'invalidation :**

| Événement | Action |
|-----------|--------|
| Création d'une nouvelle entité | Aucune — pas encore en cache |
| Modification d'une entité | `invalidateOne(id)` dans le service |
| Suppression d'une entité | `invalidateOne(id)` dans le service |
| Migration ou réindexage batch | `invalidateAll()` dans le script de migration |

---

### Règle 5 — Périmètre actuel

**Décision :** Ce pattern s'applique **backend uniquement** (NestJS + Redis) pour le moment.

- ✅ Backend : `CacheableRepository<T>` + Redis
- ⏳ Mobile : couvert par OP-SQLite (base locale = cache complet) — pas de couche supplémentaire nécessaire à ce stade

**Premier cas d'usage :** Les données référentielles (statuts, types, catégories) définies par ADR-026. L'application à d'autres entités est décidée au cas par cas selon le besoin observé.

---

## Consequences

### Positives

✅ **Opt-in explicite** : Seuls les repositories qui en ont besoin héritent du cache
✅ **Entité domaine pure** : Aucune notion de cache dans la couche domaine
✅ **Invalidation O(1)** : Le namespace élimine toute itération de clés
✅ **Performance** : Lookups DB uniquement sur cache miss, batch par liste d'IDs
✅ **Pression de cache naturelle** : LRU Redis évince les entrées peu consultées
✅ **Pattern unique** : Comportement prévisible et reviewable sur tous les repositories cachés

### Négatives (et mitigation)

⚠️ **Complexité infra** : Une couche supplémentaire entre service et DB
→ Mitigation : Encapsulée dans le parent — l'enfant reste simple (1 méthode abstraite obligatoire)

⚠️ **Invalidation manuelle** : Le développeur doit appeler `invalidateOne`/`invalidateAll` dans le service
→ Mitigation : Convention documentée en checklist story ; vérifiable en code review

⚠️ **Décision d'opt-in** : Savoir quand utiliser `CacheableRepository` requiert du jugement
→ Mitigation : Critères clairs — données stables, accès fréquents, coût de lookup significatif

---

## Implementation Checklist

Pour tout repository héritant de `CacheableRepository` :

- [ ] L'entité domaine est **sans aucune notion de cache**
- [ ] Le `namespace` est unique et descriptif (`'capture-status'`, `'concordance-score'`)
- [ ] `queryByIds` implémente une query `IN` ou `ANY` — jamais un loop de `findOne`
- [ ] `resolveIdByNaturalKey` implémente un `SELECT id WHERE key = $1` si utilisé
- [ ] Index sur la clé naturelle présent si `findByNaturalKey` est utilisé
- [ ] `invalidateOne(id)` appelé dans le service après toute modification
- [ ] `invalidateAll()` appelé dans les scripts de migration touchant l'entité
- [ ] Aucun UUID ni identifiant DB hardcodé dans le code source (vérifiable par grep)

---

## References

- [ADR-026](./ADR-026-backend-data-model-design-rules.md) — Tables référentielles (premier cas d'usage)
- [ADR-011](./ADR-011-performance-optimization.md) — Redis : cache serveur uniquement, pas de queue
- [ADR-022](./ADR-022-state-persistence-opsqlite.md) — OP-SQLite comme cache local mobile
- Martin Fowler — Template Method Pattern: https://refactoring.guru/design-patterns/template-method

---

## Decision Log

**2026-02-18** — Discussion yohikofox, Winston

→ Déclencheur : Antipattern détecté — identifiants DB hardcodés dans le code pour éviter des lookups
→ Principe : Cache unitaire opt-in, entité domaine pure, infrastructure seulement
→ Clé : `{namespace}:{id}:{ns_value}` — namespace Redis pour invalidation bulk O(1)
→ Retrieval : Batch par IDs — `findByNaturalKey` résout en ID puis délègue
→ Invalidation : Event-driven, pas de TTL — pression LRU Redis pour éviction passive
→ Périmètre : Backend uniquement — mobile couvert par OP-SQLite
→ Premier usage : Données référentielles ADR-026, extensible à tout repository au besoin
→ Toutes les décisions validées par yohikofox

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

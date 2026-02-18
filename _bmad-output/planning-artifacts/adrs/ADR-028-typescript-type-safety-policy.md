---
adr: ADR-028
title: "TypeScript Type Safety Policy — Interdiction de `any`, Hiérarchie de Typage"
date: 2026-02-18
status: "✅ Accepted"
context: "Phase 4 - Implementation - Tous packages (mobile, backend, web)"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
supersedes: "N/A - Complément à ADR-024 (Clean Code Standards)"
---

# ADR-028: TypeScript Type Safety Policy

**Date:** 2026-02-18
**Status:** ✅ Accepté
**Context:** Tous packages — mobile (React Native), backend (NestJS), web (Next.js)
**Decision Makers:** yohikofox (Product Owner), Winston (Architect)

---

## Context & Problem

Le projet Pensieve tourne en TypeScript strict mode (`"strict": true`) sur tous ses packages. Pourtant, l'audit du codebase révèle une contamination par `any` dans plusieurs couches :

- Des assignments non typés (`Unsafe assignment of an any value`)
- Des accès de propriétés sur `any` (`Unsafe member access`)
- Des arguments `any` passés à des fonctions typées

Ces occurrences désactivent silencieusement le compilateur. Le typage strict est une contrainte choisie du projet (ADR-024) — `any` la contourne sans alerte visible, créant une fausse sécurité.

**Cause racine observée :** Absence de règle explicite. Les développeurs (humains et agents IA) utilisent `any` par commodité ou par habitude, sans ligne directrice de référence.

---

## Decisions

### Règle 1 — `any` est interdit par défaut

**Décision :** `any` est interdit dans tout code de production Pensieve. La règle ESLint `@typescript-eslint/no-explicit-any` est configurée en `"error"` sur tous les packages.

```typescript
// ❌ Interdit
function process(data: any): any { ... }
const result: any = await fetch(url);

// ✅ Correct
function process(data: unknown): Result<ProcessedData> { ... }
const result: RawApiResponse = await fetch(url).then(r => r.json() as RawApiResponse);
```

---

### Règle 2 — Hiérarchie de typage : ordre de préférence

Quand le type exact n'est pas immédiatement connu, appliquer dans cet ordre :

| Priorité | Type | Cas d'usage |
|----------|------|-------------|
| 1 | **Type précis** | Toujours en premier — interfaces, classes, types nommés |
| 2 | **Générique `<T>`** | Logique réutilisable sans couplage (ex: `CacheableRepository<T>`) |
| 3 | **Union de types** | Valeurs discrètes connues (`'active' \| 'archived'`, `Success \| Error`) |
| 4 | **`unknown`** | Données dont la forme est inconnue à la compilation — force le narrowing |
| 5 | **`never`** | Branche exhaustive théoriquement impossible |
| 6 | **`any`** | Uniquement dans les cas explicitement listés en Règle 3 |

**`unknown` vs `any` :**
- `unknown` : le compilateur force une vérification avant usage → sûr
- `any` : le compilateur se tait complètement → dangereux

```typescript
// ❌ any — aucune vérification
function parse(raw: any) {
  return raw.id; // Pas d'erreur même si raw est undefined
}

// ✅ unknown — narrowing obligatoire
function parse(raw: unknown): string {
  if (typeof raw === 'object' && raw !== null && 'id' in raw) {
    return String((raw as { id: unknown }).id);
  }
  throw new Error('Invalid payload');
}
```

---

### Règle 3 — Cas légitimes de `any` (liste exhaustive)

`any` est toléré **uniquement** dans les contextes suivants, avec commentaire obligatoire :

#### 3a. Mocks dans les fichiers de tests (`*.spec.ts`, `*.test.ts`)

```typescript
// ✅ Acceptable en test — mock partiel d'une dépendance
const mockService = { findById: jest.fn() };
new MyRepository(mockService as any); // mock partiel intentionnel
```

#### 3b. Librairies tierces sans `@types/` disponibles

```typescript
// ✅ Acceptable — lib sans types officiels
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const legacyLib = require('legacy-lib') as any;
```

Le commentaire `eslint-disable-next-line` est obligatoire et documente l'intention.

#### 3c. Migration progressive JS → TS

Sur les fichiers en cours de migration uniquement, avec un commentaire `// TODO: typer proprement — Story X.Y`.

---

### Règle 4 — Boundaries externes : `unknown` + Zod ou assertion explicite

Les données dont la forme n'est pas garantie à la compilation (réponses API, payloads RabbitMQ, body HTTP non typé) doivent être traitées comme `unknown` et validées avant usage.

**Pattern recommandé :**

```typescript
// Boundary externe → unknown → validation → type précis
async function handleWebhook(raw: unknown): Promise<ProcessedEvent> {
  // Option A : assertion de type avec garde
  if (!isValidEvent(raw)) throw new BadRequestException('Invalid payload');
  return raw; // narrowé vers ProcessedEvent

  // Option B : schema Zod (si la lib est déjà présente dans le package)
  return EventSchema.parse(raw);
}
```

---

### Règle 5 — Propagation : `any` ne doit pas "fuir" hors de son contexte

Quand `any` est temporairement nécessaire (cas Règle 3), il ne doit pas contaminer les types du code appelant.

```typescript
// ❌ any fuit vers le retour de fonction
function getConfig(): any {
  return externalLib.getConfig();
}
const port: number = getConfig().port; // plus de sécurité

// ✅ Contenu localement + assertion typée au retour
function getConfig(): AppConfig {
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  const raw = externalLib.getConfig() as any;
  return { port: Number(raw.port), host: String(raw.host) };
}
```

---

### Règle 6 — Configuration ESLint de référence

Les règles suivantes sont **obligatoires** sur tous les packages :

```javascript
// eslint.config.mjs (ou .eslintrc)
{
  '@typescript-eslint/no-explicit-any': 'error',
  '@typescript-eslint/no-unsafe-assignment': 'error',
  '@typescript-eslint/no-unsafe-member-access': 'error',
  '@typescript-eslint/no-unsafe-argument': 'error',
  '@typescript-eslint/no-unsafe-return': 'error',
  '@typescript-eslint/no-unsafe-call': 'error',
}
```

Les occurrences actuelles dans le codebase qui violent ces règles sont du **code legacy** à corriger au fil des stories, non à pérenniser.

---

## Consequences

### Positives

- **Fiabilité** : Les erreurs de runtime liées à un type inattendu sont détectées à la compilation
- **Refactoring sûr** : Le compilateur guide les changements de type à travers toute la chaîne
- **Lisibilité** : Un type précis est une documentation inline qui ne devient pas obsolète
- **Onboarding** : Un développeur junior qui voit `unknown` sait qu'il doit valider — `any` ne dit rien

### Négatives / Contraintes

- **Coût initial** : Les violations ESLint existantes doivent être résolues progressivement
- **Verbosité ponctuelle** : Le narrowing de `unknown` demande quelques lignes de plus
- **Libs tierces** : Certaines libs mal typées nécessitent des contournements documentés

### Neutrales

- Cette règle **complète ADR-024** sans le remplacer — ADR-024 reste la référence Clean Code générale
- Le strict mode TypeScript déjà en place (`tsconfig.json`) est un prérequis satisfait

---

## Références

- ADR-024 — Clean Code Standards (règles générales de qualité)
- ADR-023 — Error Handling Strategy (Result Pattern — alternative typée aux `any` dans les erreurs)
- [TypeScript Handbook — `unknown` vs `any`](https://www.typescriptlang.org/docs/handbook/2/functions.html)
- [`@typescript-eslint/no-explicit-any`](https://typescript-eslint.io/rules/no-explicit-any)

---
adr: ADR-007
title: "From Scratch Approach (pas de starter full-stack)"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Starter Template Decision"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-007: From Scratch Approach (pas de starter full-stack)

**Status:** ✅ ACCEPTÉ

**Context:** Choisir entre utiliser des starters/boilerplates existants vs partir des CLI officiels.

---

## Starters Évalués

**Mobile:**
- [Obytes Starter](https://starter.obytes.com/) : Expo + TypeScript + MMKV + React Query + TailwindCSS
- [Fast Expo App](https://github.com/Teczer/fast-expo-app) : CLI avec Expo SDK 54 + MMKV + React Query

**Backend:**
- [NestJS Clean Architecture Boilerplate](https://github.com/rezawr/nestjs-clean-architecture-boilerplate) : DDD + Clean Architecture
- [NestJS-DDD-DevOps](https://andrea-acampora.github.io/nestjs-ddd-devops/) : DDD Modular Monolith + DevOps

---

## Decision

Partir de zéro avec CLI officiels, copier uniquement configs/tooling.

---

## Rationale

### 1. Architecture DDD Spécifique Déjà Définie

- 8 Bounded Contexts identifiés via Event Storming
- Structure par contexte métier unique à Pensine
- Les starters imposent leur structure générique (user/product/order)
- Refonte complète nécessaire = perte de temps

### 2. Stack Technique Particulière

- Whisper.rn (module custom, pas dans les starters)
- ~~WatermelonDB~~ OP-SQLite (peu de starters l'intègrent bien)
- Liquid Glass Design System (custom, pas TailwindCSS générique)
- RabbitMQ (rare dans starters NestJS)
- Aucun starter ne match cette stack exactement

### 3. Suringénierie des Starters

- Patterns "enterprise-grade" over-engineered (multi-tenancy, microservices ready)
- MVP mono-user = simplicité prioritaire
- Abstractions complexes à comprendre et maintenir
- Dépendances imposées difficiles à remplacer

### 4. Temps Réel

```
Starter full-stack:
- Setup: 2h
- Comprendre: 3h
- Nettoyer inutile: 5h
- Adapter architecture: 8h
- Debug conflicts: 3h
TOTAL: 21h

From Scratch:
- CLI official: 1h
- Tooling: 2h
- Structure DDD custom: 4h
- Intégrations: 6h
- Tests: 3h
TOTAL: 16h

Gain net: 5h + contrôle total
```

### 5. Projet Solo

- Pas besoin conventions d'équipe imposées
- Flexibilité pour ajuster rapidement
- Code maîtrisé à 100%

---

## Implementation

### Mobile

```bash
# Partir de Expo CLI officiel
npx create-expo-app@latest pensine-mobile --template blank-typescript

# Copier configs de Obytes (tooling uniquement):
- tsconfig.json strict
- .eslintrc minimal
- jest.config.js
- GitHub Actions EAS Build workflow
```

### Backend

```bash
# Partir de NestJS CLI officiel
npx @nestjs/cli new pensine-backend

# Structure DDD custom basée sur Event Storming:
src/
  contexts/
    knowledge/      (Core Domain)
      domain/
      application/
      infrastructure/
    opportunity/    (Core Domain)
    capture/        (Supporting)
    action/         (Supporting)
    normalization/  (Supporting)
  shared/
```

---

## Ce Qui Sera Réutilisé des Starters

- ✅ Config files (tsconfig, eslint, jest)
- ✅ Scripts package.json (lint, test, format)
- ✅ GitHub Actions workflows (CI/CD basiques)
- ✅ Documentation patterns (README structure)
- ❌ Code métier (100% custom)

---

## Consequences

**Bénéfices:**
- Contrôle total sur architecture et dépendances
- Code parfaitement aligné avec Event Storming
- Pas de dette technique dès le départ
- Maîtrise complète du codebase

**Trade-offs acceptés:**
- Setup initial légèrement plus long (1-2h)
- Pas de patterns "enterprise-grade" pré-implémentés (mais pas nécessaires pour MVP)

---

## References

- [Obytes Starter](https://starter.obytes.com/)
- [Fast Expo App](https://github.com/Teczer/fast-expo-app)
- [NestJS Clean Architecture Boilerplate](https://github.com/rezawr/nestjs-clean-architecture-boilerplate)
- [Awesome NestJS Boilerplates](https://awesome-nestjs.com/resources/boilerplate.html)

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

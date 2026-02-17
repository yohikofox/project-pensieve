# Story 11.1: Setup Clean Code Enforcement Tooling

Status: backlog

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **developer**,
I want **automated tooling to enforce Clean Code standards (ADR-024)**,
So that **code quality is validated automatically and code reviews focus on logic rather than style**.

**Context:** ADR-024 définit les standards Clean Code du projet. Cette story implémente l'outillage automatisé pour enforcer ces règles via ESLint, pre-commit hooks, et CI/CD.

## Acceptance Criteria

### AC1: ESLint Strict Configuration

**Given** je suis un développeur travaillant sur le projet
**When** je lance ESLint sur le code
**Then** les règles strictes suivantes sont enforced :
- Noms révélateurs d'intention (no single-letter variables sauf i, j, k en loops)
- No magic numbers (constantes nommées obligatoires)
- Max 3 function parameters (sinon options object requis)
- Single Responsibility Principle checks
- No dead code
- No commented code
**And** les règles TypeScript strict mode sont activées
**And** Prettier integration fonctionne sans conflit
**And** tous les fichiers TypeScript passent linting

### AC2: Pre-commit Hook - TODO Check

**Given** je commite du code
**When** le code contient `// TODO` sans ticket référence
**Then** le commit est bloqué avec message explicite
**And** le format accepté est: `// TODO(TICKET-ID): description`
**And** le hook fonctionne sur mobile/, backend/, web/, admin/
**And** la documentation explique comment bypasser si nécessaire (cas exceptionnels)

### AC3: CI/CD Linting Enforcement

**Given** je pousse du code sur une branche
**When** la CI/CD pipeline s'exécute
**Then** ESLint est exécuté sur tous les packages (mobile, backend, web, admin)
**And** le build échoue si violations ESLint détectées
**And** le rapport de violations est clair et actionnable
**And** les développeurs reçoivent feedback immédiat

### AC4: Documentation & Onboarding

**Given** je suis un nouveau développeur
**When** je lis CONTRIBUTING.md
**Then** je trouve une section "Clean Code Standards"
**And** les règles non-négociables sont listées clairement
**And** des exemples good/bad sont fournis
**And** le lien vers ADR-024 est présent
**And** les instructions pour setup ESLint localement sont claires

### AC5: Migration Guide

**Given** je dois migrer du code existant
**When** je consulte la documentation
**Then** je trouve un guide "Migration Opportuniste Clean Code"
**And** le guide explique la Rule of Three (pas d'abstraction prématurée)
**And** les priorités de migration sont définies (NON-NÉGOCIABLES vs RECOMMANDÉS)
**And** des commandes pour détecter violations sont fournies (npx eslint, ts-prune)

## Tasks

1. Créer `.eslintrc.strict.js` avec règles ADR-024
2. Configurer Prettier compatibility
3. Créer `.git/hooks/pre-commit` script TODO check
4. Setup CI/CD linting step (GitHub Actions ou équivalent)
5. Créer `docs/clean-code-migration-guide.md`
6. Update `CONTRIBUTING.md` avec section Clean Code
7. Tester hooks sur commits avec/sans TODO
8. Vérifier CI/CD fail sur violations
9. Code review validé

## Definition of Done

- [ ] ESLint strict rules configurées et passent sur codebase
- [ ] Pre-commit hook bloque TODO sans ticket
- [ ] CI/CD échoue si violations ESLint
- [ ] CONTRIBUTING.md et docs à jour
- [ ] Migration guide disponible
- [ ] Tests validés (hook + CI/CD)
- [ ] Code review approved

## Dev Notes

_Notes techniques et décisions prises pendant l'implémentation seront ajoutées ici._

## Files Changed

_Liste des fichiers modifiés sera générée pendant l'implémentation._

## Test Results

_Résultats des tests seront documentés ici._

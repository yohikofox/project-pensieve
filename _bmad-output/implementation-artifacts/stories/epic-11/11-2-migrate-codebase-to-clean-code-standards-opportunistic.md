# Story 11.2: Migrate Codebase to Clean Code Standards (Opportunistic)

Status: backlog

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **developer**,
I want **to progressively migrate existing code to Clean Code standards**,
So that **the entire codebase benefits from improved maintainability without a disruptive big-bang rewrite**.

**Context:** Migration opportuniste signifie : toucher un fichier → appliquer règles ADR-024. Pas de rewrite complet, mais amélioration continue pendant 2-4 semaines.

## Acceptance Criteria

### AC1: Migration Opportuniste Strategy

**Given** je modifie un fichier existant pour une feature/bugfix
**When** je sauvegarde mes changements
**Then** j'applique les règles Clean Code ADR-024 au fichier entier
**And** je vérifie que ESLint passe sur ce fichier
**And** je documente les changements migration dans le commit message
**And** le scope reste raisonnable (refacto < 30 min par fichier)

### AC2: Priority Migration - NON-NÉGOCIABLES

**Given** je migre du code existant
**When** j'applique les standards Clean Code
**Then** je priorise les règles NON-NÉGOCIABLES :
- Magic numbers → constantes nommées
- Dead code supprimé
- Code commenté supprimé
- Single Responsibility violations fixées
- Functions > 50 lignes découpées
**And** je crée des tickets séparés si refacto > 30 min

### AC3: Definition of Done Extension

**Given** je marque une story comme "done"
**When** je valide la checklist Definition of Done
**Then** la checklist inclut maintenant :
- ✅ Code respecte standards Clean Code ADR-024
- ✅ ESLint strict passe sans warnings
- ✅ No magic numbers, dead code, ou code commenté
- ✅ Functions < 30 lignes (ou justification documentée)
**And** code review vérifie compliance stricte

### AC4: Migration Progress Tracking

**Given** nous progressons dans la migration
**When** je consulte le suivi de migration
**Then** je peux voir :
- % de fichiers conformes Clean Code
- Violations restantes par catégorie
- Fichiers nécessitant migration prioritaire
**And** l'équipe peut célébrer progrès (gamification optionnelle)

### AC5: Code Review Enforcement

**Given** je soumets une PR
**When** un reviewer examine mon code
**Then** le reviewer vérifie compliance ADR-024 strictement
**And** les violations bloquent la PR
**And** le reviewer utilise checklist Clean Code standardisée
**And** les discussions focalisent sur logic, pas style (ESLint gère style)

### AC6: Migration Complete Criteria

**Given** 2-4 semaines se sont écoulées
**When** nous évaluons l'état de migration
**Then** > 90% des fichiers touchés sont conformes
**And** < 10% violations NON-NÉGOCIABLES restantes
**And** dette technique mesurée et contrôlée
**And** onboarding time < 3 semaines (vs 6-8 avant)

## Tasks

1. Créer fichier `docs/clean-code-checklist.md` pour reviews
2. Update Definition of Done dans project-context.md
3. Créer script audit : `npx eslint --report-unused-disable-directives`
4. Documenter pattern migration opportuniste
5. Identifier top 10 fichiers violations prioritaires
6. Migrer opportuniste pendant features/bugfixes (continu)
7. Weekly review progression migration (team sync)
8. Célébrer milestones (75%, 90%, 100%)

## Definition of Done

- [ ] Definition of Done mise à jour avec Clean Code checks
- [ ] Checklist code review standardisée créée
- [ ] Script audit violations disponible
- [ ] > 90% fichiers touchés conformes après 2-4 semaines
- [ ] Onboarding time < 3 semaines mesuré
- [ ] Team adopte pratiques Clean Code
- [ ] Code review valide compliance stricte

## Dev Notes

_Notes techniques et décisions prises pendant l'implémentation seront ajoutées ici._

## Files Changed

_Liste des fichiers modifiés sera générée pendant l'implémentation._

## Test Results

_Résultats des tests seront documentés ici._

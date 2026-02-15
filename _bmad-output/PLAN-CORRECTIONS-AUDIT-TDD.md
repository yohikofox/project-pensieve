# 📋 PLAN DE CORRECTIONS AUDIT - MODE TDD
**Date:** 2026-02-15
**Projet:** Pensieve
**Basé sur:** AUDIT-CODE-COMPLET-2026-02-15.md

---

## 🎯 OBJECTIF

Corriger les **34 problèmes** identifiés dans l'audit adversarial en suivant le cycle **RED-GREEN-REFACTOR** du TDD.

Chaque tâche permet de:
- ✅ Travailler progressivement
- ✅ Faire des pauses et reprendre
- ✅ Valider chaque correction avec des tests
- ✅ Suivre la progression via TaskList

---

## 📊 VUE D'ENSEMBLE

### Métriques Globales

| Catégorie | Tasks | Effort Total | Problèmes Résolus |
|-----------|-------|--------------|-------------------|
| 🔴 URGENT | 6 | 12h | 15 CRITICAL |
| 🟠 HIGH | 6 | 37.5h | 8 HIGH |
| 🟡 MEDIUM | 1 | 20h | 11 MEDIUM |
| **TOTAL** | **13** | **69.5h** | **34 issues** |

### Distribution Effort

```
URGENT (12h):        ████████░░░░░░░░░░░░ 17%
HIGH   (37.5h):      ███████████████████░ 54%
MEDIUM (20h):        ██████░░░░░░░░░░░░░░ 29%
```

---

## 🔴 PHASE 1: URGENT (12h - Sprint N)

**Objectif:** Corriger les violations CRITIQUES bloquantes

### Task #1: Supprimer imports legacy expo-file-system ⚠️
**ID:** 1
**Effort:** 2h
**Priorité:** CRITICAL - Interdiction absolue

**Problème:**
- 3 fichiers importent `expo-file-system/legacy` (BANNI)
- Violation project-context.md lines 1000-1009

**Fichiers:**
- `CaptureDevTools.tsx:23`
- `SettingsScreen.tsx:17`
- `SettingsScreen.test.tsx:6`

**TDD Steps:**
1. 🔴 RED: Créer test vérifiant API moderne
2. 🟢 GREEN: Remplacer imports `/legacy` → API moderne
3. 🔵 REFACTOR: Valider aucun legacy subsiste

**Commandes:**
```bash
# Démarrer
npx jest src/components/dev/__tests__/CaptureDevTools.migration.test.ts

# Valider
grep -r "expo-file-system/legacy" src/ || echo "✅ Clean"
```

---

### Task #2: Remplacer tests placeholder Story 5.4 ⚠️
**ID:** 2
**Effort:** 4h
**Priorité:** CRITICAL - Faux positifs tests

**Problème:**
- 13 tests avec `expect(true).toBe(true)`
- Story marquée "done" mais ACs non validés

**TDD Steps:**
1. 🔴 RED: Identifier 13 placeholders
2. 🟢 GREEN: Remplacer par vraies assertions
3. 🔵 REFACTOR: Exécuter tests Story 5.4

**Commandes:**
```bash
# Lister placeholders
grep -n "expect(true).toBe(true)" tests/acceptance/story-5-4.test.ts

# Exécuter après corrections
npx jest --config jest.config.acceptance.js story-5-4.test.ts
```

---

### Task #3: Remplacer tests placeholder autres stories ⚠️
**ID:** 3
**Effort:** 2h
**Priorité:** CRITICAL

**Problème:**
- 10 tests placeholder dans 6 fichiers
- Stories 3.1, 1.2, 2.3-2.6

**Dépend de:** Task #2 (pour pattern cohérent)

---

### Task #4: Fix secret JWT hardcodé 🔒
**ID:** 4
**Effort:** 30min
**Priorité:** CRITICAL - Sécurité

**Problème:**
- Fallback `'admin-secret-key-change-in-production'`
- Risque production si JWT_SECRET manquant

**Fichier:** `admin-auth.module.ts:19-21`

**TDD Steps:**
1. 🔴 RED: Test qui vérifie throw si JWT_SECRET manquant
2. 🟢 GREEN: Remplacer fallback par throw Error
3. 🔵 REFACTOR: Documenter .env.example

---

### Task #5: Fix exposition error messages 🔒
**ID:** 5
**Effort:** 30min
**Priorité:** HIGH - Information Disclosure

**Problème:**
- `error.message` exposé au client
- Leak: DB names, file paths, stack traces

**Fichier:** `rgpd.controller.ts:45-50`

---

### Task #6: Créer feature files BDD Stories 3.3 et 7.1 ⚠️
**ID:** 6
**Effort:** 3h
**Priorité:** CRITICAL

**Problème:**
- 2 stories "done" SANS tests BDD
- ACs jamais vérifiés

**Livrables:**
- `story-3-3-visual-distinction.feature`
- `story-7-1-support-mode.feature`
- Step definitions avec vraies assertions

---

## 🟠 PHASE 2: HIGH PRIORITY (37.5h - Sprint N+1)

### Task #7: Refactor throw → Result pattern (ADR-023) 📐
**ID:** 7
**Effort:** 8h
**Priorité:** HIGH - Architecture

**Problème:**
- 11 fichiers violent ADR-023
- `throw new Error()` au lieu de `Result<T>`

**Fichiers prioritaires:**
1. SyncQueueService.ts
2. FileStorageService.ts
3. LLMModelService.ts
4. TodoRepository.ts
5. user-features.repository.ts
6. useUserFeatures.ts
7-11. Backend services (5 fichiers)

**TDD Steps:**
1. 🔴 RED: Écrire tests attendant Result<T>
2. 🟢 GREEN: Refactor throw → return Result
3. 🔵 REFACTOR: Adapter tous les callers

**Impact:** Conformité ADR-023, error handling monadic

---

### Task #8: Créer tests module Authorization (0% → 60%) 🧪
**ID:** 8
**Effort:** 16h
**Priorité:** CRITICAL - Sécurité

**Problème:**
- 25+ fichiers critiques SANS tests
- Système permissions non validé

**Scope:**
- 3 services (20+ tests chacun)
- 8 repositories (6-8 tests chacun)
- 3 guards (8-10 tests chacun)

**Métriques cible:**
- ~130 tests créés
- Coverage >= 60%

---

### Task #9: Fix CORS configuration 🔒
**ID:** 9
**Effort:** 1h
**Priorité:** HIGH - Sécurité

**Problème:**
- Localhost autorisé en production
- Regex trop larges
- credentials: true avec origins larges

**Solution:** Configuration basée sur NODE_ENV

---

### Task #10: Ajouter ValidationPipe global 🔒
**ID:** 10
**Effort:** 30min
**Priorité:** MEDIUM - Validation

**Problème:**
- Pas de ValidationPipe global
- DTOs peuvent être bypassés

**Solution:**
```typescript
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
}));
```

---

### Task #11: Fix exceptions NestJS 🔒
**ID:** 11
**Effort:** 1h
**Priorité:** MEDIUM

**Problème:**
- `throw new Error()` au lieu d'exceptions NestJS
- HTTP status codes incorrects

**Fichiers:**
- admin-auth.controller.ts → ForbiddenException
- sync.controller.ts → UnauthorizedException

---

### Task #12: Validation DTOs query params 🔒
**ID:** 12
**Effort:** 2h
**Priorité:** MEDIUM

**Problème:**
- Query params non validés
- Pas de type checking, injection possible

**Solution:** Créer DTOs avec class-validator

---

## 🟡 PHASE 3: MEDIUM PRIORITY (20h - Sprint N+2)

### Task #13: Augmenter coverage mobile (32% → 60%) 🧪
**ID:** 13
**Effort:** 20h
**Priorité:** MEDIUM - Qualité

**Problème:**
- 66 tests / 206 fichiers = 32%
- Refactoring risqué

**Plan:**
- Phase 1: Capture context (+15 tests, 8h)
- Phase 2: Knowledge context (+10 tests, 4h)
- Phase 3: Action context (+8 tests, 3h)
- Phase 4: Identity context (+6 tests, 2h)
- Phase 5: Normalization context (+10 tests, 3h)

**Total:** +49 tests pour atteindre 60% coverage

---

## 📈 PROGRESSION & TRACKING

### Utilisation des Tasks

**Voir la liste:**
```bash
/tasks
```

**Démarrer une task:**
```bash
# Marquer task en cours
TaskUpdate taskId=1 status=in_progress
```

**Compléter une task:**
```bash
# Marquer terminée
TaskUpdate taskId=1 status=completed
```

**Trouver la prochaine task:**
```bash
TaskList
# Prendre la première task "pending" sans blocage
```

---

## 🔄 WORKFLOW TDD STANDARD

Chaque tâche suit ce pattern:

### 1. 🔴 RED - Écrire tests qui échouent
```bash
# Créer test file
touch path/to/__tests__/file.spec.ts

# Écrire tests
# describe(), it(), expect()

# Exécuter (devrait échouer - RED)
npx jest path/to/__tests__/file.spec.ts
```

### 2. 🟢 GREEN - Corriger code
```bash
# Modifier le code source
# Corriger pour faire passer les tests

# Exécuter tests (devrait passer - GREEN)
npx jest path/to/__tests__/file.spec.ts
```

### 3. 🔵 REFACTOR - Améliorer & valider
```bash
# Refactoriser si nécessaire
# Exécuter TOUS les tests affectés
npm run test
npm run test:acceptance
npm run test:e2e

# Vérifier métriques
npm run test:coverage
```

---

## 🎯 CHECKPOINTS CRITIQUES

### Après URGENT (12h)

**Vérifications:**
- [ ] Aucun import `/legacy`
- [ ] Aucun test placeholder
- [ ] Secret JWT validé
- [ ] Error messages génériques
- [ ] 2 feature files BDD créés

**Commandes validation:**
```bash
grep -r "legacy" pensieve/mobile/src/ | wc -l  # = 0
grep -r "expect(true).toBe(true)" pensieve/mobile/tests/ | wc -l  # = 0
grep "JWT_SECRET.*||" pensieve/backend/src/ | wc -l  # = 0
```

---

### Après HIGH (50h cumulés)

**Vérifications:**
- [ ] ADR-023: Result pattern dans fichiers prioritaires
- [ ] Authorization: >= 60% coverage
- [ ] CORS: environment-based
- [ ] ValidationPipe: global activé
- [ ] Exceptions: NestJS typées

**Métriques:**
```bash
npx jest --coverage src/modules/authorization
# Statements: >= 60%
```

---

### Après MEDIUM (70h cumulés)

**Vérifications:**
- [ ] Mobile: >= 60% coverage
- [ ] Tests: >= 130 nouveaux tests
- [ ] Issues: 34/34 résolues

**Métriques finales:**
```bash
cd pensieve/mobile && npm run test:coverage
# Coverage: >= 60%

cd ../backend && npx jest --coverage
# Coverage authorization: >= 60%
```

---

## 🚀 DÉMARRAGE RAPIDE

### 1. Voir toutes les tâches
```bash
/tasks
```

### 2. Démarrer la première tâche URGENT
```bash
# Task #1: Imports legacy
TaskUpdate taskId=1 status=in_progress

cd pensieve/mobile
# Suivre les steps TDD dans la task description
```

### 3. Compléter et passer à la suivante
```bash
# Marquer terminée
TaskUpdate taskId=1 status=completed

# Voir prochaine task
TaskList
```

---

## 📝 NOTES IMPORTANTES

### Dépendances Tasks

- Task #3 dépend de Task #2 (pattern cohérent)
- Toutes les autres tasks sont indépendantes
- Peuvent être faites en parallèle si souhaité

### Pause & Reprise

**Le système de tasks permet:**
- ✅ De mettre une task en pause (status reste in_progress)
- ✅ De voir où on en était (description complète)
- ✅ De reprendre exactement où on s'est arrêté
- ✅ De tracker la progression globale

**Exemple pause:**
```bash
# Vous êtes sur Task #7 (Refactor Result pattern)
# Vous avez fait 3/11 fichiers

# Pas besoin de faire quoi que ce soit
# La task reste in_progress

# À la reprise:
TaskList
# Vous voyez Task #7 in_progress
# Relire description pour voir où vous en étiez
```

### Commits Git Recommandés

**Pattern:**
```bash
git commit -m "fix(mobile): remove legacy expo-file-system imports

- Replace 3 legacy imports with modern API
- Update CaptureDevTools, SettingsScreen
- Add migration test
- Closes Task #1

Refs: AUDIT-CODE-COMPLET-2026-02-15.md"
```

---

## 📚 RESSOURCES

### Documents Référence
- `/Users/yoannlorho/ws/pensine/_bmad-output/AUDIT-CODE-COMPLET-2026-02-15.md`
- `/Users/yoannlorho/ws/pensine/_bmad-output/project-context.md`
- ADR-023: Error Handling Strategy

### Helpers Existants
- `tests/acceptance/support/test-context.ts` (mocks)
- `src/types/result.types.ts` (Result pattern)
- `.env.example` (variables)

### Commandes Utiles

**Tests:**
```bash
# Mobile
npm run test:unit
npm run test:acceptance
npm run test:e2e
npm run test:coverage

# Backend
npm run test
npm run test:acceptance
npm run test:e2e
npm run test:cov
```

**Recherche:**
```bash
# Trouver tous les throw
grep -rn "throw new Error" src/

# Trouver placeholders
grep -rn "expect(true).toBe(true)" tests/

# Trouver imports legacy
grep -rn "legacy" src/
```

---

**Créé:** 2026-02-15
**Auteur:** Senior Developer (Mode Adversarial)
**Version:** 1.0
**Total Tasks:** 13
**Total Effort:** 69.5 heures
**Issues Résolues:** 34

---

_Bon courage pour les corrections ! Suivez le TDD, prenez des pauses, et validez à chaque étape._ ✨

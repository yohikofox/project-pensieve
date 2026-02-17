# Issues GitHub Fermées - Référence

**Date de trace** : 2026-02-13
**Objectif** : Éviter de se reposer la question sur ces issues déjà résolues

---

## Issues Closes (4)

### #27 - fix(tests): TS1109 placeholder in text-capture-service.test.ts
- **État** : CLOSED (2026-02-12)
- **Problème** : Erreur TypeScript TS1109 - commentaire placeholder utilisé comme valeur
- **Fichier** : `mobile/tests/acceptance/capture/text-capture-service.test.ts:25`
- **Résolution** : Placeholder remplacé par adapter SQLite valide ou `null as any`
- **Contexte** : Story 2.2 - Tests d'intégration TextCaptureService

### #26 - fix(tests): TS1109 placeholder in capture-model.test.ts
- **État** : CLOSED (2026-02-12)
- **Problème** : Même erreur TS1109 que #27
- **Fichier** : `mobile/tests/acceptance/capture/capture-model.test.ts:21`
- **Résolution** : Placeholder remplacé par adapter SQLite valide
- **Contexte** : Story 2.1 - Tests d'intégration Capture Model

### #25 - [Architecture] DTOs and mapping functions pollute domain layer
- **État** : CLOSED (2026-02-12)
- **Problème** : Violation Clean Architecture - DTOs dans la couche domaine
- **Fichiers affectés** :
  - `Capture.model.ts`
  - `CaptureAnalysis.model.ts`
  - `CaptureMetadata.model.ts`
- **Résolution** : DTOs et fonctions de mapping déplacés dans la couche infrastructure (repositories)
- **Impact** : Respect Dependency Rule - Domain → Infrastructure (correct)

### #1 - feat: Bouton de retry pour les transcriptions en échec
- **État** : CLOSED (2026-01-31)
- **Feature** : Bouton "Réessayer" sur cartes de capture en échec
- **Implémentation** :
  - Limite 3 tentatives / 20 minutes
  - Feedback utilisateur avec indicateur progression
  - Mode debug pour messages d'erreur détaillés
- **Contexte** : Amélioration UX pour gestion des erreurs de transcription

---

## Notes

Ces issues sont **résolues et fermées**. Ne pas les inclure dans les futurs plannings sprint ou analyses d'issues ouvertes.

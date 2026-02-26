# ADR-032 — Renommage Action BC → Task BC

**Date:** 2026-02-26
**Statut:** Accepté
**Auteurs:** yohikofox, Winston (Architect)
**Session:** Architecture Mode Délégation

---

## Contexte

Le Bounded Context nommé "Action" contenait les todos (tâches à faire par l'humain). Ce nom créait une ambiguïté croissante avec l'introduction du Mode Délégation, où le terme "Action" est naturellement utilisé pour désigner un intent détecté par le LLM dans une capture.

Deux concepts distincts portaient le même mot :
- "Action" = todo dans l'onglet Actions (usage existant)
- "Action" = intent détecté par Knowledge BC (nouveau concept Delegation)

## Décision

**Renommer le Bounded Context "Action" en "Task".**

- Le BC s'appelle désormais `Task Context` (ou `TaskBC` dans le code)
- L'entité principale reste `Todo` (inchangée)
- Le terme "Action" est libéré pour désigner exclusivement un intent détecté par Knowledge BC

## Conséquences

**Positives :**
- Vocabulaire DDD précis et non ambigu
- "Action" = concept amont (détection par LLM), "Todo" = concept aval (matérialisation humaine)
- Foundation propre pour le Delegation BC

**Négatives :**
- Refactoring technique dans la codebase (renommage fichiers, classes, imports)
- Couvert par Story 19.1

## Références

- Story 19.1 (implémentation du renommage)
- ADR-033 (Delegation BC — utilise "Action" comme concept amont)
- ADR-034 (modèle sémantique complet)

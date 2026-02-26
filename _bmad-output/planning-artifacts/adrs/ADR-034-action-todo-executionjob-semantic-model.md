# ADR-034 — Modèle Sémantique : Action / Todo / ExecutionJob

**Date:** 2026-02-26
**Statut:** Accepté
**Auteurs:** yohikofox, Winston (Architect)
**Session:** Architecture Mode Délégation

---

## Contexte

Avant cet ADR, le terme "action" était utilisé de façon floue pour désigner à la fois :
1. Ce que le LLM détecte dans une capture ("il faut faire X")
2. La tâche concrète affichée dans l'onglet Actions (todo)

Avec l'introduction du Mode Délégation, un troisième concept émerge : l'action que Pensine peut exécuter de façon autonome. Trois concepts distincts nécessitent trois noms distincts.

## Décision

**Modèle sémantique à trois niveaux :**

| Concept | Qui | Définition | BC |
|---------|-----|-----------|-----|
| **Action** | Knowledge BC | Intent détecté dans une capture par le LLM | Knowledge (output d'événement) |
| **Todo** | Task BC | Matérialisation d'une Action assignée à l'humain | Task BC |
| **ExecutionJob** | Delegation BC | Matérialisation d'une Action déléguée au système | Delegation BC |

**Le critère de distinction :**

> **Qui est l'exécuteur ?**
> - "Je dois faire X" → **Todo** (l'humain exécute)
> - "Fais X pour moi" → **ExecutionJob** (Pensine exécute)

**Pipeline complet :**

```
Capture audio/texte
  ↓ [Whisper — transcription locale]
  ↓ [LLM local (gemma3n-2b) — digestion]
  ↓
Knowledge BC émet CaptureDigested {
  extractedActions: Action[]  // classifiés par le LLM
}
  ↓                              ↓
Task BC (isExecutable=false)  Delegation BC (isExecutable=true)
  → Todo créé                    → ExecutionJob créé
  → onglet Actions               → section "En attente de validation"
  → humain l'exécute             → Pensine l'exécute (avec trust model)
```

**Règle de classification LLM :**

Le LLM classifie lors d'une **seule passe** (pas de double traitement). En cas de doute ou d'ambiguïté :
- `isExecutable = false` → fallback Todo (jamais de risque d'exécution non voulue)
- Ce choix est conservateur et préserve NFR19

**Cas particulier — Todo manuellement déléguée :**

L'utilisateur peut "déléguer" une Todo existante au système (future feature). Cela crée un ExecutionJob depuis la Todo. La Todo reste en état `delegated` jusqu'à confirmation d'exécution.

## Conséquences

**Positives :**
- Vocabulaire non ambigu dans tout le codebase
- Séparation claire des responsabilités BC
- Fallback conservateur protège NFR19

**Négatives :**
- Rupture sémantique avec l'existant ("Action BC" → "Task BC")
- Couvert par ADR-032 et Story 19.1

## Références

- ADR-032 (renommage BC)
- ADR-033 (Delegation BC)
- ADR-035 (chorégraphie événementielle)
- Stories 19.2, 19.3 (implémentation)

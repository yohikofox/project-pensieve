# ADR-035 — Pattern d'Intégration Delegation : Chorégraphie (Option A)

**Date:** 2026-02-26
**Statut:** Accepté
**Auteurs:** yohikofox, Winston (Architect)
**Session:** Architecture Mode Délégation

---

## Contexte

Trois options ont été évaluées pour connecter Knowledge BC (producteur d'Actions) à Task BC et Delegation BC (consommateurs) :

**Option A — Chorégraphie (event-driven)**
```
Knowledge émet CaptureDigested
  Task BC subscribe → crée Todos
  Delegation BC subscribe → crée ExecutionJobs
```

**Option B — Orchestration (coordinateur central)**
```
Knowledge → DigestOrchestrator → appelle Task BC + Delegation BC
```

**Option C — Anti-Corruption Layer dans Delegation BC**
```
Knowledge émet des Actions génériques
Delegation BC filtre selon ses capacités
```

## Décision

**Option A — Chorégraphie via événement `CaptureDigested`.**

L'événement `CaptureDigested` contient `extractedActions: Action[]` avec `isExecutable` pré-classifié par le LLM.

## Rationale

**Pourquoi Option A vs Option B (orchestration) :**
- Couplage plus faible : Knowledge ne connaît ni Task BC ni Delegation BC
- Cohérent avec l'architecture événementielle existante (ADR-019 EventBus RxJS)
- Plus simple à implémenter pour un solo dev

**Pourquoi Option A vs Option C (ACL dans Delegation) :**
- Option C nécessiterait deux passes LLM (Knowledge extrait générique, Delegation re-classifie)
- Sur modèles locaux, deux passes = latence doublée + qualité incertaine
- Le LLM a tout le contexte lors de la première passe — c'est le moment optimal de classification
- La règle de fallback (ambiguïté → `isExecutable=false`) est plus simple et plus sûre

## Implémentation

```typescript
// Knowledge BC — émission
eventBus.emit('CaptureDigested', {
  captureId,
  summary,
  extractedActions: Action[],  // classifiés avec isExecutable
  extractedIdeas: Idea[]
})

// Task BC — subscribe
eventBus.on('CaptureDigested', ({ extractedActions }) => {
  extractedActions
    .filter(a => !a.isExecutable)
    .forEach(a => todoRepository.create(Todo.fromAction(a)))
})

// Delegation BC — subscribe
eventBus.on('CaptureDigested', ({ extractedActions, captureId }) => {
  extractedActions
    .filter(a => a.isExecutable)
    .forEach(a => executionJobRepository.create(ExecutionJob.fromAction(a, captureId)))
})
```

## Conséquences

**Positives :**
- Couplage minimal entre BCs
- Cohérent avec EventBus RxJS existant (ADR-019)
- Knowledge BC ne sait pas qui consomme ses événements

**Négatives :**
- Résilience : si un subscriber plante, l'autre n'est pas impacté (acceptable)
- Debug légèrement plus complexe qu'une orchestration (compenser par logging structuré)

## Références

- ADR-019 (EventBus RxJS — architecture événementielle existante)
- ADR-034 (modèle sémantique — définition de l'Action dans l'événement)
- Stories 19.2, 19.3 (implémentation subscribers)

---
adr: ADR-004
title: "Digestion IA - Un seul appel LLM"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-004: Digestion IA - Un seul appel LLM

**Status:** ✅ ACCEPTÉ

**Context:** L'extraction (summary, tags, todos, ideas) doit-elle se faire en un ou plusieurs appels LLM ?

**Decision:** Un seul appel LLM structuré.

---

## Implementation

```typescript
const digest = await llm.digest(transcription, {
  extract: ['summary', 'tags', 'todos', 'ideas']
});
```

---

## Rationale

- Coût LLM réduit (1 appel vs 3)
- Latence réduite (pas d'attente séquentielle)
- LLM voit contexte global pour extraction cohérente
- Prompt engineering moderne permet multi-extraction structurée

---

## Consequences

**Bénéfices:**
- Frugalité : coût optimisé pour MVP
- Performance : latence réduite
- Cohérence : extraction contextuelle

**Trade-offs acceptés:**
- Prompt complexe (mais gérable avec structured output)
- Si LLM échoue → tout échoue (mais retry possible)

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

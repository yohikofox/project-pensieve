# Issues hors-sujet détectés durant l'implémentation de Story 8.6

> Problèmes existants dans le codebase **non causés par la story 8.6**.
> Chaque item indique le fichier concerné, la nature du problème et l'ADR ou la règle violée.

---

## 1. Violations ADR-021 — Container TSyringe : registerSingleton au lieu de registerTransient

**Fichier :** `mobile/src/infrastructure/di/container.ts` (lignes ~258, ~263)
**Symptôme :** `npm run test:architecture` → `Lignes sans justification ADR-021: [ 258, 263 ]`
**Description :** ADR-021 prescrit "Transient First" pour le cycle de vie DI. Deux enregistrements utilisent `registerSingleton` sans commentaire de justification (`// ADR-021: singleton justified because...`).
**Priorité :** Moyenne

---

## 2. Violation ADR-023 — throw dans un repository

**Fichier :** `mobile/src/contexts/identity/data/user-features.repository.ts`
**Symptôme :** `npm run test:architecture` → `Violations ADR-023 (throw dans repository): contexts/identity/data/user-features.repository.ts`
**Description :** ADR-023 impose l'usage du Result Pattern (`Result<T, E>`) dans les repositories. Ce fichier lève une exception (`throw`) au lieu de retourner un `Failure` ou `Error` via le pattern.
**Priorité :** Haute (cohérence avec le pattern établi dans shared/domain/Result.ts)

---

## 3. Violations ADR-031 — Interfaces au lieu de Classes dans les fichiers `*.model.ts`

**Fichiers :**
- `mobile/src/contexts/action/domain/Todo.model.ts:13` — `export interface Todo {`
- `mobile/src/contexts/capture/domain/Capture.model.ts:38` — `export interface Capture {`
- `mobile/src/contexts/capture/domain/CaptureAnalysis.model.ts:25` — `export interface CaptureAnalysis {`
- `mobile/src/contexts/capture/domain/CaptureMetadata.model.ts:45` — `export interface CaptureMetadata {`
- `mobile/src/contexts/knowledge/domain/Idea.model.ts:9` — `export interface Idea {`
- `mobile/src/contexts/knowledge/domain/Thought.model.ts:9` — `export interface Thought {`

**Symptôme :** `npm run test:architecture` → `Expected length: 0, Received length: 6`
**Description :** ADR-031 prescrit que les fichiers `*.model.ts` dans `domain/` doivent exporter des **classes** (non des interfaces) comme entité principale, pour permettre l'instanciation et la sérialisation/désérialisation.
**Priorité :** Moyenne (dette technique accumulée)

---

## 4. Propriété non-readonly dans un Domain Event

**Fichier :** `mobile/src/contexts/capture/events/CaptureEvents.ts:84`
**Symbole :** `event: DomainEvent,` (doit être `readonly event: DomainEvent,`)
**Symptôme :** `npm run test:architecture` → `Propriétés non-readonly dans les events`
**Description :** Les Domain Events doivent être immutables. La propriété `event` dans ce fichier est modifiable alors qu'elle devrait être `readonly`.
**Priorité :** Faible (cosmétique mais viole l'immutabilité des events)

---

## 5. `settingsStore.transcription.language` inexistant

**Contexte :** Story 8.6 Dev Notes mentionnaient :
```typescript
language: settingsStore.transcription.language ?? 'fr-FR',
```
**Constat :** Cette propriété n'existe pas dans le `settingsStore`. Le store expose `audioTrimEnabled` mais pas de sous-objet `transcription` avec `language`.
**Impact Story 8.6 :** Contourné par hardcoding de `'fr-FR'` dans `useLiveTranscription.ts`.
**Impact projet :** La langue de transcription live n'est pas configurable via les paramètres utilisateur. Une story dédiée pourrait exposer `transcriptionLanguage` dans le store.
**Priorité :** Basse (fonctionnel mais non configurable)

---

## Résumé par priorité

| Priorité | Issue |
|----------|-------|
| Haute    | ADR-023 : throw dans `user-features.repository.ts` |
| Moyenne  | ADR-021 : 2 registerSingleton non justifiés dans `container.ts` |
| Moyenne  | ADR-031 : 6 fichiers `*.model.ts` avec `interface` au lieu de `class` |
| Basse    | CaptureEvents.ts : propriété `event` non-readonly |
| Basse    | settingsStore : `transcription.language` manquant |

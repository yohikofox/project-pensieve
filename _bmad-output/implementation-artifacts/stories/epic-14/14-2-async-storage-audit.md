# Story 14.2: Audit AsyncStorage — Vérifier l'Absence de Données Critiques

Status: review

## Story

As a **developer**,
I want **to verify that AsyncStorage is only used for non-critical UI preferences, not for sensitive or critical data**,
So that **we comply with ADR-022 which mandates OP-SQLite for all critical data persistence**.

## Context

Audit ADR-022 (2026-02-17) révèle :
- `@react-native-async-storage/async-storage: ^2.2.0` est présent dans `package.json`
- L'ADR-022 mandate OP-SQLite pour toutes les données critiques
- AsyncStorage est acceptable UNIQUEMENT pour les préférences UI non critiques (thème, langue...)
- **Si** AsyncStorage est utilisé pour des tokens, captures, thoughts ou données de sync → violation

## Acceptance Criteria

### AC1: Inventaire exhaustif des usages d'AsyncStorage
**Given** AsyncStorage is present in the codebase
**When** I search for all AsyncStorage usages
**Then** I produce a complete list of all files using `AsyncStorage.getItem`, `AsyncStorage.setItem`, `AsyncStorage.removeItem`
**And** each usage is classified as: `UI_PREF` (acceptable) or `CRITICAL_DATA` (violation)

### AC2: Aucune donnée critique dans AsyncStorage
**Given** the audit is complete
**When** I analyze each usage
**Then** AsyncStorage is NOT used for:
- Authentication tokens (use `expo-secure-store`)
- Audio captures or transcriptions (use OP-SQLite)
- Todos, thoughts, ideas (use OP-SQLite)
- Sync metadata (lastPulledAt, sync queue) (use OP-SQLite)
- Any user PII

### AC3: Correction des violations si trouvées
**Given** any critical data is found in AsyncStorage
**When** I fix the violations
**Then** critical data is migrated to the appropriate store:
- Sensitive tokens → `expo-secure-store`
- Domain data (captures, todos...) → OP-SQLite via existing repositories
- Sync metadata → OP-SQLite `SyncStorage`
**And** AsyncStorage references for critical data are removed

### AC4: Documentation des usages autorisés
**Given** remaining AsyncStorage usages are UI preferences only
**When** I document the policy
**Then** a comment is added near each AsyncStorage usage identifying it as UI_PREF:
```typescript
// ASYNC_STORAGE_OK: UI preference only — not critical data (ADR-022)
await AsyncStorage.setItem('theme', 'dark');
```

### AC5: Tests de vérification
**Given** the audit and fixes are done
**When** I run the test suite
**Then** all existing tests pass (no regression)
**And** if violations were fixed, new unit tests validate data is in the correct store

## Tech Notes

- **Commande de recherche** :
  ```bash
  grep -rn "AsyncStorage" pensieve/mobile/src/ --include="*.ts" --include="*.tsx"
  ```
- **Stores autorisés par type de donnée** :
  - Tokens sensibles → `expo-secure-store` (chiffré, KeyChain iOS)
  - Données domaine → OP-SQLite (via repositories existants)
  - Préférences UI simples → AsyncStorage (acceptable)
- **Rapport** : Produire un rapport `async-storage-audit-report.md` dans `_bmad-output/implementation-artifacts/` listant tous les usages trouvés et leur classification

## Related

- ADR-022: State Persistence — OP-SQLite First
- ADR-010: Sécurité (expo-secure-store pour tokens)
- `@react-native-async-storage/async-storage` dans `package.json`

## Tasks/Subtasks

- [x] Task 1: Grep complet sur `AsyncStorage` dans `pensieve/mobile/src/`
  - [x] 1.1 Exécuter grep sur tous les fichiers .ts/.tsx
  - [x] 1.2 Classifier chaque usage (UI_PREF / CRITICAL_DATA)
- [x] Task 2: Produire le rapport `async-storage-audit-report.md`
- [x] Task 3: Corriger la violation supabase.ts (auth tokens → expo-secure-store)
  - [x] 3.1 Créer `src/lib/large-secure-store.ts` (adapter chunked SecureStore)
  - [x] 3.2 Mettre à jour `src/lib/supabase.ts`
- [x] Task 4: Corriger les violations SyncStorage.ts + InitialSyncService.ts (sync metadata → OP-SQLite)
  - [x] 4.1 Ajouter `CREATE_SYNC_METADATA_TABLE` dans `schema.ts`
  - [x] 4.2 Incrémenter `SCHEMA_VERSION` → 24
  - [x] 4.3 Ajouter migration v24 dans `migrations.ts`
  - [x] 4.4 Réécrire `SyncStorage.ts` pour utiliser OP-SQLite
  - [x] 4.5 Mettre à jour `InitialSyncService.ts`
- [x] Task 5: Commenter les usages UI_PREF restants avec `// ASYNC_STORAGE_OK`
- [x] Task 6: Vérifier zéro régression (tests)

## Definition of Done

- [x] Grep complet sur `AsyncStorage` dans `pensieve/mobile/src/`
- [x] Rapport `async-storage-audit-report.md` produit avec classification de chaque usage
- [x] Zéro violation (données critiques dans AsyncStorage)
- [x] Si violations : données migrées vers OP-SQLite ou expo-secure-store
- [x] Usages AsyncStorage restants commentés avec `// ASYNC_STORAGE_OK: UI preference only`
- [x] Tests : zero régression

## Dev Agent Record

### Implementation Plan

**Violations identifiées (2 critiques) :**
1. `supabase.ts` — Supabase auth utilise AsyncStorage pour stocker les JWT/refresh tokens → violation ADR-010 + ADR-022. Fix: LargeSecureStore adapter (chunking nécessaire car Keychain iOS = 2KB max)
2. `SyncStorage.ts` + `InitialSyncService.ts` — métadonnées de sync (`lastPulledAt`, etc.) dans AsyncStorage → violation ADR-022 explicite. Fix: Migration OP-SQLite v24 + table `sync_metadata`

**Usages UI_PREF (acceptables) :**
- `settingsStore.ts` — thème, color scheme, audio player, debug mode, haptic, LLM settings
- `LLMModelService.ts` — sélection modèle, flags postprocessing, état téléchargement
- `TranscriptionModelService.ts` — sélection modèle Whisper, vocabulaire custom
- `TranscriptionEngineService.ts` — type moteur transcription
- `useFilterState.ts` — filtre/tri Tab Actions
- `RetentionPolicyService.ts` — config rétention, dernière date cleanup
- `CorrectionLearningService.ts` — historique corrections transcription (learning data)
- `user-features.repository.ts` — cache fonctionnalités utilisateur (TTL, non autoritatif)

### Completion Notes

Audit complet effectué sur 100% des usages AsyncStorage dans `pensieve/mobile/src/`. 2 violations critiques identifiées et corrigées. 8 usages UI_PREF documentés avec commentaire ADR-022. Migration OP-SQLite v24 créée pour la table `sync_metadata`. Adapter LargeSecureStore créé pour contourner la limite 2KB du Keychain iOS.

### Debug Log

*(Vide — implémentation sans blocages)*

## File List

### New Files
- `pensieve/mobile/src/lib/large-secure-store.ts` — LargeSecureStore adapter (SecureStore chunking)
- `pensieve/mobile/src/infrastructure/sync/__tests__/SyncStorage.sqlite.test.ts` — Tests OP-SQLite SyncStorage
- `_bmad-output/implementation-artifacts/async-storage-audit-report.md` — Rapport d'audit

### Modified Files
- `pensieve/mobile/src/lib/supabase.ts` — Remplacer AsyncStorage par LargeSecureStore
- `pensieve/mobile/src/database/schema.ts` — Ajouter `CREATE_SYNC_METADATA_TABLE` + SCHEMA_VERSION → 24
- `pensieve/mobile/src/database/migrations.ts` — Ajouter migration v24 (`sync_metadata` table)
- `pensieve/mobile/src/infrastructure/sync/SyncStorage.ts` — Réécrire avec OP-SQLite
- `pensieve/mobile/src/infrastructure/sync/InitialSyncService.ts` — Utiliser SyncStorage refactorisé
- `pensieve/mobile/src/stores/settingsStore.ts` — Ajouter commentaire ASYNC_STORAGE_OK
- `pensieve/mobile/src/contexts/Normalization/services/LLMModelService.ts` — Ajouter commentaire ASYNC_STORAGE_OK
- `pensieve/mobile/src/contexts/Normalization/services/TranscriptionModelService.ts` — Ajouter commentaire ASYNC_STORAGE_OK
- `pensieve/mobile/src/contexts/Normalization/services/TranscriptionEngineService.ts` — Ajouter commentaire ASYNC_STORAGE_OK
- `pensieve/mobile/src/contexts/action/hooks/useFilterState.ts` — Ajouter commentaire ASYNC_STORAGE_OK
- `pensieve/mobile/src/contexts/capture/services/RetentionPolicyService.ts` — Ajouter commentaire ASYNC_STORAGE_OK
- `pensieve/mobile/src/contexts/Normalization/services/CorrectionLearningService.ts` — Ajouter commentaire ASYNC_STORAGE_OK
- `pensieve/mobile/src/contexts/identity/data/user-features.repository.ts` — Ajouter commentaire ASYNC_STORAGE_OK
- `_bmad-output/implementation-artifacts/stories/epic-14/14-2-async-storage-audit.md` — Ce fichier

## Change Log

- 2026-02-18: Story 14.2 implémentée — Audit AsyncStorage complet, 2 violations corrigées, 8 usages UI_PREF documentés, migration OP-SQLite v24 créée, LargeSecureStore adapter créé

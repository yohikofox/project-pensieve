# Story 14.2: Audit AsyncStorage — Vérifier l'Absence de Données Critiques

Status: ready-for-dev

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

## Definition of Done

- [ ] Grep complet sur `AsyncStorage` dans `pensieve/mobile/src/`
- [ ] Rapport `async-storage-audit-report.md` produit avec classification de chaque usage
- [ ] Zéro violation (données critiques dans AsyncStorage)
- [ ] Si violations : données migrées vers OP-SQLite ou expo-secure-store
- [ ] Usages AsyncStorage restants commentés avec `// ASYNC_STORAGE_OK: UI preference only`
- [ ] Tests : zero régression

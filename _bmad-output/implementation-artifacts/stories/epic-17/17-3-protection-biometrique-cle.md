# Story 17.3: Protection biométrique de l'accès à la clé maître

Status: pending

## Story

En tant qu'**utilisateur**,
je veux **que l'accès à mes données chiffrées soit conditionné à une authentification biométrique (Face ID / Touch ID) ou par code PIN**,
afin de **protéger mes données même si quelqu'un déverrouille mon téléphone**.

## Context

**Dépend de Story 17.1** — `IEncryptionService` doit exister dans le conteneur DI.

Actuellement, la clé maître est accessible dès que l'application démarre (juste après l'auth Supabase). La protection biométrique ajoute une couche supplémentaire : même avec accès au téléphone déverrouillé, l'utilisateur doit confirmer son identité pour que la clé soit déverrouillée.

**`expo-local-authentication`** n'est pas encore installé → à ajouter en première tâche.

**Comportements attendus** :
- La biométrie est **optionnelle** (désactivable dans les settings)
- Si biométrie non disponible sur le device : fallback vers code PIN du device
- Si désactivée par l'utilisateur : la clé est accessible directement (comportement actuel — Story 17.1)
- La confirmation biométrique est demandée au lancement de l'app (pas à chaque opération de chiffrement)

## Acceptance Criteria

### AC1 — Installation expo-local-authentication

**Given** le projet mobile nécessite la biométrie
**When** `expo-local-authentication` est installé
**Then** il est présent dans `package.json` avec une version compatible Expo SDK 54
**And** les permissions nécessaires sont ajoutées dans `app.config.js` (iOS NSFaceIDUsageDescription, Android USE_BIOMETRIC)

### AC2 — Service IBiometricAuthService créé et enregistré

**Given** le bounded context `identity/` (ou `security/`)
**When** un développeur veut utiliser la biométrie
**Then** une interface `IBiometricAuthService` est disponible avec : `isAvailable(): Result<boolean>`, `authenticate(reason: string): Result<boolean>`, `isEnabled(): boolean`
**And** elle est enregistrée dans `container.ts`

### AC3 — Déverrouillage biométrique au lancement (si activé)

**Given** l'utilisateur a activé la protection biométrique dans les settings
**When** l'application est lancée ou revenue au premier plan après 5 minutes d'inactivité
**Then** une demande d'authentification biométrique est présentée (prompt natif Face ID / Touch ID)
**And** la clé maître est accessible uniquement après succès biométrique
**And** si l'authentification échoue : la clé n'est pas déverrouillée et un message d'erreur est affiché

### AC4 — Fallback PIN/passcode

**Given** la biométrie est activée mais indisponible (Face ID bloqué, capteur non reconnu)
**When** la vérification biométrique échoue
**Then** le système propose le fallback PIN/passcode du device
**And** `expo-local-authentication` gère ce fallback nativement

### AC5 — Option dans les Settings

**Given** un utilisateur accède aux paramètres de sécurité
**When** il navigue vers "Sécurité" dans les Settings
**Then** il peut activer/désactiver la protection biométrique (toggle)
**And** si la biométrie n'est pas disponible sur le device : l'option est grisée avec un message explicatif

### AC6 — Désactivation biométrie = accès direct (réversible)

**Given** l'utilisateur désactive la protection biométrique dans les settings
**When** l'application est relancée
**Then** la clé maître est accessible directement sans confirmation biométrique
**And** les données restent chiffrées (seule la protection d'accès change, pas le chiffrement)

### AC7 — Préférence biométrique persistée

**Given** l'utilisateur active ou désactive la protection biométrique
**When** l'application est fermée et relancée
**Then** la préférence est maintenue
**And** la préférence est stockée dans AsyncStorage (donnée non critique, UI preference)

## Tasks / Subtasks

- [ ] **Task 1 — Installer expo-local-authentication** (AC: #1)
  - [ ] 1.1 — `npx expo install expo-local-authentication`
  - [ ] 1.2 — Vérifier compatibilité avec Expo SDK 54 (doc officielle)
  - [ ] 1.3 — Ajouter `NSFaceIDUsageDescription` dans `app.config.js` (iOS)
  - [ ] 1.4 — Vérifier permissions Android dans `app.config.js` (`USE_BIOMETRIC`, `USE_FINGERPRINT`)

- [ ] **Task 2 — Créer IBiometricAuthService et son implémentation** (AC: #2)
  - [ ] 2.1 — Créer `src/contexts/identity/domain/IBiometricAuthService.ts`
  - [ ] 2.2 — Implémenter `BiometricAuthService` dans `infrastructure/security/`
  - [ ] 2.3 — Ajouter token `TOKENS.IBiometricAuthService` dans `tokens.ts`
  - [ ] 2.4 — Enregistrer dans `container.ts`

- [ ] **Task 3 — Intégrer le déverrouillage au lancement** (AC: #3, #4)
  - [ ] 3.1 — Modifier `bootstrap.ts` ou le hook d'auth pour déclencher l'auth biométrique après login
  - [ ] 3.2 — Condition : biométrie activée (`IBiometricAuthService.isEnabled()`)
  - [ ] 3.3 — Si biométrie réussie → appeler `IEncryptionService.unlockKey()` (méthode à ajouter)
  - [ ] 3.4 — Si biométrie échoue → clé non disponible, afficher écran de blocage

- [ ] **Task 4 — Ecran/composant de blocage biométrique** (AC: #3)
  - [ ] 4.1 — Créer un écran `BiometricLockScreen` affiché quand la clé est verrouillée
  - [ ] 4.2 — Bouton "Déverrouiller" qui retrigger `IBiometricAuthService.authenticate()`
  - [ ] 4.3 — Message d'erreur si échec répété

- [ ] **Task 5 — Settings biométrie** (AC: #5, #6, #7)
  - [ ] 5.1 — Ajouter une section "Sécurité" dans l'écran Settings existant
  - [ ] 5.2 — Toggle "Protéger avec Face ID/Touch ID"
  - [ ] 5.3 — Griser le toggle si `IBiometricAuthService.isAvailable()` retourne false
  - [ ] 5.4 — Persister la préférence dans AsyncStorage (`// ASYNC_STORAGE_OK: UI security preference`)

- [ ] **Task 6 — Tests** (AC: tous)
  - [ ] 6.1 — Mocker `expo-local-authentication` dans `test-context.ts`
  - [ ] 6.2 — Tests unitaires `BiometricAuthService.test.ts`
  - [ ] 6.3 — Écrire `story-17-3.feature` avec scénarios Gherkin
  - [ ] 6.4 — Écrire les step definitions `story-17-3.test.ts`

## Dev Notes

### Fichiers concernés

| Fichier | Changement |
|---------|-----------|
| `mobile/package.json` | Ajouter `expo-local-authentication` |
| `mobile/app.config.js` | Permissions biométrie iOS/Android |
| `mobile/src/contexts/identity/domain/IBiometricAuthService.ts` | Nouveau — interface |
| `mobile/src/infrastructure/security/BiometricAuthService.ts` | Nouveau — implémentation |
| `mobile/src/infrastructure/di/tokens.ts` | Ajouter `IBiometricAuthService` |
| `mobile/src/infrastructure/di/container.ts` | Enregistrer le service |
| `mobile/src/config/bootstrap.ts` | Hook déverrouillage au lancement |
| `mobile/src/screens/settings/SettingsScreen.tsx` | Section Sécurité |
| `mobile/src/screens/security/BiometricLockScreen.tsx` | Nouveau — écran blocage |
| `mobile/tests/acceptance/features/story-17-3.feature` | Nouveau — Gherkin |
| `mobile/tests/acceptance/story-17-3.test.ts` | Nouveau — Step definitions |
| `mobile/tests/acceptance/support/test-context.ts` | Mock expo-local-authentication |

### Compatibilité expo-local-authentication avec Expo SDK 54

Vérifier la version compatible : `npx expo install expo-local-authentication` choisit automatiquement la bonne version. Si incompatibilité : **NE PAS utiliser une version dépréciée** — consulter l'architecte (règle `.claude/CLAUDE.md`).

### Préférence AsyncStorage (justification)

La préférence biométrique est une **donnée UI non critique** (pas de token, pas de capture) → usage AsyncStorage conforme à ADR-022. Annoter avec `// ASYNC_STORAGE_OK: biometric preference UI only`.

### Diagramme de séquence lancement avec biométrie activée

```
App launch → isAuthenticated? → oui
  → BiometricAuthService.isEnabled()? → oui
    → LocalAuthentication.authenticateAsync()
      → succès → EncryptionService.unlockKey()
      → échec → BiometricLockScreen (retry disponible)
  → non → EncryptionService.unlockKey() direct
```

### Inactivité 5 minutes (AC3)

Implémenter via `AppState` (React Native) : quand l'app passe en background > 5 min, verrouiller la clé en mémoire. Utiliser `AppState.addEventListener('change', ...)`.

## Dev Agent Record

### Agent Model Used

TBD

### Completion Notes List

TBD

### File List

TBD

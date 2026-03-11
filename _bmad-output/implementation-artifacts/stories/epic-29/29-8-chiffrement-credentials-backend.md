# Story 29.8 — Chiffrement des credentials backend (PAT GitHub)

Status: backlog

## Story

As a **platform operator**,
I want **GitHub PATs stored in the database to be encrypted at rest**,
So that **a database breach does not expose user credentials in plaintext**.

## Context

En MVP (Stories 29-2 à 29-7), les PAT GitHub sont stockés **en clair** dans la table
`code_repositories`. Cette story chiffre ces données sensibles.

**⚠️ Prérequis : Stories 29-2 à 29-6 doivent être terminées avant d'implémenter cette story.**

L'approche retenue est le **chiffrement applicatif** (application-level encryption) :
le chiffrement/déchiffrement est fait dans le service backend, pas au niveau de la DB.
La clé de chiffrement est injectée via variable d'environnement.

**Algorithme :** AES-256-GCM (authentifié, recommandé OWASP pour secrets au repos).

**Ce qui est chiffré :**
- `code_repositories.pat_value` (le PAT brut)

**Ce qui n'est PAS chiffré (hors scope) :**
- Les manifests de repo (`manifest_cache`) — pas des secrets
- Les `ProjectIssue` — pas des secrets
- Les données côté mobile (OP-SQLite) — le mobile ne stocke jamais le PAT

## Acceptance Criteria

### AC1 — Migration des PATs existants
**Given** des PATs stockés en clair dans `code_repositories`
**When** la migration est exécutée
**Then** :
- Tous les PATs existants sont chiffrés avec AES-256-GCM
- La clé de chiffrement est lue depuis `ENCRYPTION_KEY` (variable d'environnement, 32 bytes hex)
- La migration est réversible (rollback possible)

### AC2 — Écriture chiffrée
**Given** l'utilisateur ajoute un nouveau CodeRepository (Story 29-2)
**When** le backend persiste le PAT
**Then** le PAT est chiffré avant insertion — jamais stocké en clair

### AC3 — Déchiffrement transparent
**Given** le backend doit appeler l'API GitHub (création d'issue, sync status, indexation repo)
**When** il récupère le PAT depuis la DB
**Then** le PAT est déchiffré en mémoire au moment de l'utilisation — jamais loggué

### AC4 — Rotation de clé
**Given** la variable d'environnement `ENCRYPTION_KEY` est changée
**When** un script de re-chiffrement est exécuté (`npm run encrypt:rotate`)
**Then** tous les PATs sont re-chiffrés avec la nouvelle clé sans downtime

### AC5 — Clé manquante = fail fast
**Given** `ENCRYPTION_KEY` n'est pas défini au démarrage du backend
**When** le service démarre
**Then** une erreur explicite est levée et le démarrage est interrompu :
`FATAL: ENCRYPTION_KEY env variable is required`

### AC6 — Tests de non-régression
**Given** la story est implémentée
**When** les tests d'acceptation des Stories 29-2, 29-6, 29-7 sont réexécutés
**Then** tous les tests passent sans modification (chiffrement transparent)

## Notes techniques

**Implémentation recommandée (Node.js crypto natif) :**
```typescript
// CredentialEncryptionService
encrypt(plaintext: string): string {
  const iv = crypto.randomBytes(12)
  const cipher = crypto.createCipheriv('aes-256-gcm', key, iv)
  const encrypted = Buffer.concat([cipher.update(plaintext), cipher.final()])
  const tag = cipher.getAuthTag()
  return `${iv.hex}:${tag.hex}:${encrypted.hex}`
}

decrypt(stored: string): string {
  const [iv, tag, encrypted] = stored.split(':').map(Buffer.from)
  const decipher = crypto.createDecipheriv('aes-256-gcm', key, iv)
  decipher.setAuthTag(tag)
  return Buffer.concat([decipher.update(encrypted), decipher.final()]).toString()
}
```

- Pas de librairie externe nécessaire — `crypto` natif Node.js.
- La colonne `pat_value` devient `pat_encrypted` (migration avec rename).
- Le `CredentialEncryptionService` est injectable via TSyringe (ADR-021).
- Ne pas logger le plaintext PAT — configurer le logger pour masquer les champs sensibles
  (pattern déjà établi en Story 14-3 avec Sentry PII filter).

## References

- Story 29-2 (CodeRepository — source des PATs à chiffrer)
- Story 14-3 (Sentry + Pino — PII filtering — pattern à réutiliser)
- ADR-021 (DI lifecycle — `CredentialEncryptionService` en Singleton)
- OWASP Cryptographic Storage Cheat Sheet

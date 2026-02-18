# Story 8.18: Admin Reset Password via Supabase Admin API

Status: ready-for-dev

## Story

En tant qu'administrateur,
je veux pouvoir forcer la réinitialisation du mot de passe d'un utilisateur depuis le panel admin,
afin de débloquer un utilisateur sans passer par le flow email (rate limit ou SMTP non configuré).

## Contexte

Supabase est la source de vérité pour les données d'authentification. Un admin ne doit **jamais** modifier le mot de passe directement en base PostgreSQL backend — uniquement via l'API admin Supabase (`supabase.auth.admin.updateUserById()`).

`SupabaseAdminService` existe déjà dans `backend/src/modules/rgpd/application/services/supabase-admin.service.ts` avec le `SERVICE_ROLE_KEY`. Il suffit de l'étendre.

## Acceptance Criteria

1. **AC1 — Endpoint backend** : `POST /api/admin/users/:id/reset-password` (body: `{ newPassword: string }`) appelle `supabase.auth.admin.updateUserById(id, { password: newPassword })`. Protégé par `AdminAuthGuard` + permission `admin.users.write`.

2. **AC2 — Validation mot de passe** : Le mot de passe est validé (min 8 caractères, 1 majuscule, 1 chiffre) avant l'appel Supabase. Retourne 400 si invalide.

3. **AC3 — Erreur Supabase propagée** : Si Supabase retourne une erreur, le backend retourne 422 avec le message Supabase.

4. **AC4 — Admin UI** : La page `/users` du panel admin affiche un bouton "Reset password" dans la colonne actions. Ouvre une modale avec un champ nouveau mot de passe + confirmation.

5. **AC5 — Feedback visuel** : Toast de succès "Password reset successfully" ou d'erreur. Le champ est masqué (`type="password"`).

6. **AC6 — Audit log** : L'action est loguée dans `audit_logs` avec `action: 'ADMIN_PASSWORD_RESET'`, `user_id` de l'user ciblé, `metadata: { performed_by: adminId }`.

7. **AC7 — Tests unitaires** : 3 tests couvrent le service (succès, mot de passe invalide, erreur Supabase).

## Tasks / Subtasks

- [ ] T1 — Backend : Étendre `SupabaseAdminService` (AC1, AC3)
  - [ ] T1.1 — Ajouter `resetUserPassword(userId: string, newPassword: string): Promise<void>` dans `supabase-admin.service.ts`
  - [ ] T1.2 — Utiliser `this.supabase.auth.admin.updateUserById(userId, { password: newPassword })`

- [ ] T2 — Backend : Endpoint admin (AC1, AC2, AC6)
  - [ ] T2.1 — Créer `POST /api/admin/users/:id/reset-password` dans le controller admin existant (ou `identity/infrastructure/controllers/`)
  - [ ] T2.2 — DTO `ResetPasswordDto` avec validation class-validator (min 8 chars, regex uppercase + digit)
  - [ ] T2.3 — Enregistrer audit log `ADMIN_PASSWORD_RESET` après succès

- [ ] T3 — Admin UI : Bouton + modale (AC4, AC5)
  - [ ] T3.1 — Ajouter `resetUserPassword(userId, data)` dans `admin/lib/api-client.ts`
  - [ ] T3.2 — Ajouter colonne "Actions" dans la table users (`/app/(dashboard)/users/page.tsx`)
  - [ ] T3.3 — Créer modale avec 2 champs (nouveau mot de passe + confirmation) + bouton Submit
  - [ ] T3.4 — Afficher toast succès/erreur

- [ ] T4 — Tests unitaires (AC7)
  - [ ] T4.1 — Mock `supabase.auth.admin.updateUserById` dans les tests
  - [ ] T4.2 — Test succès (mock résout)
  - [ ] T4.3 — Test validation DTO (password trop court)
  - [ ] T4.4 — Test erreur Supabase (mock rejette → 422)

## Dev Notes

### Pattern à suivre

`SupabaseAdminService` est dans `rgpd` module mais contient des responsabilités partagées. Pour cette story, **étendre directement le service existant** sans déplacer le fichier (refactor hors scope).

```typescript
// supabase-admin.service.ts — méthode à ajouter
async resetUserPassword(userId: string, newPassword: string): Promise<void> {
  const { error } = await this.supabase.auth.admin.updateUserById(userId, {
    password: newPassword,
  });
  if (error) throw new UnprocessableEntityException(error.message);
}
```

### DTO Validation

```typescript
export class ResetPasswordDto {
  @IsString()
  @MinLength(8)
  @Matches(/[A-Z]/, { message: 'Must contain at least one uppercase letter' })
  @Matches(/[0-9]/, { message: 'Must contain at least one number' })
  newPassword: string;
}
```

### Audit log

Pattern existant dans `AuditLog` entity — ajouter `'ADMIN_PASSWORD_RESET'` à l'union de types.
[Source: `backend/src/modules/shared/infrastructure/persistence/typeorm/entities/audit-log.entity.ts`]

### Admin UI — pattern existant

Suivre le pattern de la page roles (`/app/(dashboard)/roles/page.tsx`) pour la modale Dialog shadcn/ui.
`apiClient` est dans `admin/lib/api-client.ts` — ajouter :
```typescript
resetUserPassword: (userId: string, data: { newPassword: string }) =>
  fetch(`/api/admin/users/${userId}/reset-password`, { method: 'POST', body: JSON.stringify(data), ... })
```

### Conformité

- Source of truth auth = Supabase UNIQUEMENT (ADR-016)
- `SERVICE_ROLE_KEY` déjà dans `backend/.env` sous `SUPABASE_SERVICE_ROLE_KEY`
- Jamais modifier le champ password directement en PostgreSQL

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

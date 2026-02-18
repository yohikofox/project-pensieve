---
adr: ADR-030
title: "Transactional Email Provider — Resend pour emails d'authentification"
date: 2026-02-18
status: "✅ Accepted"
context: "Phase 4 - Implementation - Conséquence de ADR-029 (Better Auth)"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-030: Transactional Email Provider — Resend

**Status:** ✅ ACCEPTÉ
**Date:** 2026-02-18
**Dépend de :** ADR-029 (Better Auth — le server auth a besoin d'envoyer des emails)

---

## Context & Problem

**Problème :**

ADR-029 remplace Supabase Auth par Better Auth self-hosted. Supabase gérait le SMTP transactionnel pour :
- Password reset
- Email de vérification (optionnel)

Better Auth n'inclut pas de provider email — il expose un hook `sendResetPassword` à implémenter. Il faut choisir un provider externe.

**Contraintes :**
- Budget : €0 (pas d'abonnement payant pour ≥500 users)
- Volume réel estimé pour 500 users : 5-15 emails/jour maximum (reset password + vérification)
- Risque "paywall inattendu" : même préoccupation que pour Supabase Auth — éviter les services qui peuvent changer leur pricing agressivement
- Délivrabilité : les emails doivent arriver en boîte de réception (pas en spam)

**Pourquoi ne pas auto-héberger le SMTP ?**

Un serveur SMTP self-hosted (Postfix, Mailcow) est techniquement possible sur le homelab mais :
- Réputation IP : une IP résidentielle ou dynamique = spam systématique
- SPF/DKIM/DMARC : configuration complexe, maintenance continue
- Blacklistage : risque permanent
- Pour 5-15 emails/jour, le coût opérationnel est disproportionné

Auto-héberger SMTP pour des emails d'auth à ce volume = over-engineering contre-productif.

---

## Decision

**Resend comme provider d'emails transactionnels.**

Resend est un service d'email transactionnel avec une API TypeScript-first.

### Free Tier Resend

| Métrique | Valeur |
|----------|--------|
| Emails/mois | 3 000 |
| Emails/jour | 100 |
| Domaines | 1 (limité free) |
| Prix | €0 |

**Calcul pour 500 users :**
- Reset password : 500 users × 0,5% actifs/jour = ~2-3 resets/jour
- Vérification email : ponctuel (onboarding uniquement)
- Pic estimé : 15 emails/jour (scénario pessimiste)
- **Marge free tier : 100/jour → utilisation <15% du quota**

On n'atteindra jamais le paywall à ce volume.

### Intégration Better Auth

```typescript
// backend/src/email/email.service.ts
import { Resend } from 'resend'

@Injectable()
export class EmailService {
  private resend = new Resend(process.env.RESEND_API_KEY)

  async sendResetPassword(email: string, resetUrl: string): Promise<void> {
    await this.resend.emails.send({
      from: 'Pensine <noreply@pensine.example.com>',
      to: email,
      subject: 'Réinitialisation de votre mot de passe',
      html: `
        <p>Cliquez sur ce lien pour réinitialiser votre mot de passe :</p>
        <a href="${resetUrl}">Réinitialiser mon mot de passe</a>
        <p>Ce lien expire dans 1 heure.</p>
      `
    })
  }

  async sendEmailVerification(email: string, verifyUrl: string): Promise<void> {
    await this.resend.emails.send({
      from: 'Pensine <noreply@pensine.example.com>',
      to: email,
      subject: 'Vérifiez votre adresse email',
      html: `
        <p>Cliquez sur ce lien pour vérifier votre adresse email :</p>
        <a href="${verifyUrl}">Vérifier mon email</a>
      `
    })
  }
}
```

```typescript
// Branchement dans auth.config.ts (ADR-029)
export const auth = betterAuth({
  emailAndPassword: {
    enabled: true,
    sendResetPassword: async ({ user, url }) => {
      await emailService.sendResetPassword(user.email, url)
    },
    sendVerificationEmail: async ({ user, url }) => {
      await emailService.sendEmailVerification(user.email, url)
    }
  }
})
```

---

## Options Considérées

| Critère | **Resend** (choisi) | Brevo (ex-Sendinblue) | Postmark | SMTP self-hosted |
|---------|---------------------|-----------------------|----------|-----------------|
| **Free tier emails/jour** | **100** | 300 | ~3/jour | Illimité |
| **Free tier emails/mois** | **3 000** | 9 000 | 100 | Illimité |
| **TypeScript SDK** | **✅ natif** | ⚠️ générique | ✅ | N/A |
| **Délivrabilité** | **✅ excellente** | ✅ bonne | ✅ excellente | ❌ risque spam |
| **Risque paywall** | **⚠️ faible** | ⚠️ faible | ⚠️ faible | **✅ aucun** |
| **Complexité setup** | **Simple** | Simple | Simple | ❌ Élevée |
| **Migration si besoin** | **2h** | 2h | 2h | 2 jours |
| **Score** | **9/10** | 8/10 | 7/10 | 5/10 |

**Pourquoi Resend plutôt que Brevo ?**

Brevo a un free tier plus généreux (300/jour vs 100). Mais Resend a un SDK TypeScript natif conçu pour NestJS/Next.js, et une DX supérieure. Pour le volume concerné (< 15 emails/jour), les deux sont équivalents en pratique. Resend est le choix optimal pour notre stack TypeScript.

**Pourquoi pas auto-héberger ?**

Vu en contexte ci-dessus. Délivrabilité ingérable sur IP homelab. Disproportionné pour 5-15 emails/jour.

---

## Consequences

### ✅ Bénéfices

1. **Délivrabilité garantie** : Infrastructure Resend avec réputation IP établie — pas de spam
2. **SDK TypeScript natif** : Intégration NestJS propre, typée
3. **Volume confortable** : 100/jour pour <15 emails/jour réels = marge ×6
4. **Migration facile** : Si Resend pose problème, migration vers Brevo = changer 3 lignes de code + clé API. Pas de lock-in réel

### ⚠️ Trade-offs acceptés

1. **Dépendance externe subsiste** : On garde une dépendance SaaS pour l'email. C'est un choix délibéré — la délivrabilité email ne se self-host pas sans coût opérationnel disproportionné.
2. **Risque paywall théorique** : Si Resend change son pricing, migration vers Brevo en 2h. Pas de lock-in architectural.

### 🔄 Impact sur architecture existante

- Nouveau service `EmailService` dans NestJS (injectable via TSyringe)
- Variable d'environnement : `RESEND_API_KEY`
- Aucun impact sur mobile, web, admin

---

## Implementation

### Étapes

1. Créer compte Resend + configurer domaine (DNS SPF/DKIM)
2. `npm install resend` dans backend
3. Implémenter `EmailService` NestJS
4. Brancher dans `auth.config.ts` (Better Auth hooks)
5. Ajouter `RESEND_API_KEY` dans `.env` et secrets homelab

### Variables d'environnement

```bash
# backend/.env
RESEND_API_KEY=re_xxxxxxxxxxxx
EMAIL_FROM=noreply@pensine.example.com
```

### Fichiers

```
pensieve/backend/src/email/
├── email.module.ts
├── email.service.ts
└── templates/
    ├── reset-password.html
    └── verify-email.html
```

---

## Validation Criteria

ADR considéré succès SI :
- ✅ Email reset password reçu en boîte de réception (pas spam)
- ✅ Email envoyé en < 5 secondes
- ✅ Aucun email perdu sur 30 jours de test
- ✅ Quota free tier jamais atteint (< 100/jour)

**Review Date :** 2026-05 (3 mois post-implémentation)

---

## References

- [Resend Documentation](https://resend.com/docs)
- [Resend NestJS Guide](https://resend.com/docs/send-with-nestjs)
- ADR-029 — Authentication Provider Better Auth (dépendance directe)

---

## Decision Log

**2026-02-18** — Discussion yohikofox / Winston

→ Better Auth (ADR-029) nécessite un provider email pour reset password
→ Contrainte : €0 en prod, délivrabilité garantie
→ Volume estimé : 5-15 emails/jour pour 500 users
→ Auto-hébergement SMTP : rejeté (délivrabilité ingérable, coût opérationnel disproportionné)
→ Resend vs Brevo : Resend choisi pour SDK TypeScript natif + DX supérieure
→ Migration si besoin : 2h de travail, pas de lock-in architectural
→ Décision finale : Resend free tier ✅

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

---

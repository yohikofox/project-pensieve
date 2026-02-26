# Exploration : Web Layer & Architecture Privacy-First

**Date :** 2026-02-26
**Statut :** Réflexion ouverte — décision différée
**Participants :** yohikofox, Winston (Architect)

---

## Contexte de la discussion

Partant du constat que les fonctionnalités d'Epic 23 (synthèse, clustering, semantic search)
sont naturellement mieux adaptées à une surface web qu'à une app mobile, la question s'est
posée : **quelle architecture pour un éventuel web layer Pensine, en particulier si on veut
y ajouter des fonctionnalités business (CRM, facturation) ?**

---

## Observation stratégique

La donnée Pensine est **naturellement business-ready** sans effort de transformation :

| Donnée Pensine | Usage business potentiel |
|----------------|--------------------------|
| Contacts résolus | CRM, historique relationnel |
| Captures liées à des contacts | Pipeline commercial enrichi |
| Actions / Todos | Suivi client, ticketing |
| Ideas extraites | Roadmap produit, backlog |

Un CRM construit sur Pensine serait unique : enrichi par l'historique vocal et le contexte AI,
pas un CRM de saisie manuelle — un **CRM de contexte**.

**Conclusion :** le backend NestJS actuel est déjà le bon fondement. Il n'y a pas de refonte
à prévoir — juste de nouveaux Bounded Contexts (`Billing`, `CRM`) quand le moment viendra.

---

## Le pattern QR Code + Clé Dérivée

### Idée explorée

L'utilisateur souhaite que **le backend ne voit jamais les données clients en clair**.
Idée : transmettre une clé dérivée au browser via QR code, tout le déchiffrement se
faisant côté client (browser).

### Ce pattern existe déjà

C'est exactement le modèle de **WhatsApp Web et Telegram Web**.

```
Mobile génère une session key dérivée de la master key (ADR-017)
  → encode en QR code (one-time, expire en 60s)

Browser scanne le QR
  → reçoit la clé via canal sécurisé (WebSocket chiffré ou fragment URL #)
  → la clé ne transite JAMAIS par le backend en clair

Browser possède la clé
  → récupère les blobs chiffrés du backend
  → déchiffre localement via Web Crypto API (AES-256-GCM natif)
  → peut appeler OpenAI directement depuis le browser (BYOK)

Backend = aveugle (zéro connaissance des données en clair)
```

**Web Crypto API** est disponible nativement dans tous les browsers modernes — pas de
dépendance tierce requise.

---

## Le mur du Zero-Knowledge : la queryabilité

Le problème fondamental du zero-knowledge :

> Si tout est chiffré côté backend, `GET /captures?contact=Paul` est impossible
> côté serveur. Le browser doit tout télécharger et filtrer localement.

**Ça tient** avec ~500 captures.
**Ça ne tient plus** avec 50 000 captures ou des features de recherche avancée.

WhatsApp Web contourne ce problème parce que les messages sont structurés par conversation
(unité de téléchargement naturelle). Pensine a un modèle plus complexe : recherche sémantique,
filtres croisés, clustering — qui nécessitent des opérations serveur.

**Conséquence :** le zero-knowledge pur oblige à sacrifier la queryabilité ou à implémenter
des structures cryptographiques avancées (searchable encryption) — très complexe en pratique.

---

## Les deux chemins

### Chemin A — Zero Knowledge pur

| Point | Détail |
|-------|--------|
| QR code + clé dérivée | ✅ Techniquement faisable |
| Chiffrement E2E complet | ✅ |
| AI cloud depuis le browser | ✅ Direct OpenAI (BYOK) — backend non impliqué |
| Recherche / filtres | ⚠️ Client-side uniquement — performance dégradée à l'échelle |
| Fonctionnalités business (CRM) | ⚠️ Très complexe à implémenter correctement |
| Complexité globale | 🔴 Élevée |
| Argument marketing | Fort pour segment ultra privacy (médecins, avocats, journalistes) |

### Chemin B — Privacy by Commitment

| Point | Détail |
|-------|--------|
| Chiffrement | En transit (TLS) + at rest (AES standard côté serveur) |
| Accès données | Serveur peut lire — mais engagement contractuel strict |
| Engagement | *"Je ne lis pas vos données sans approbation explicite"* |
| Audit logs | Tout accès aux données tracé et auditable |
| RGPD | Complet — droit d'accès, d'effacement, portabilité |
| Recherche / AI | ✅ Tout fonctionne normalement |
| CRM / Facturation | ✅ Pas de contrainte architecturale |
| Complexité | 🟢 Standard |
| Argument business | Plus rassurant pour les entreprises que la cryptographie |

### Chemin C — Compromis pragmatique (recommandé)

Chiffrement E2E **uniquement sur les données vraiment sensibles** :

| Donnée | Modèle |
|--------|--------|
| Audio brut | E2E chiffré — jamais lu par le serveur |
| Transcriptions | E2E chiffré |
| Métadonnées structurées (dates, contacts, tags) | En clair — opérations serveur possibles |
| Données business (CRM, facturation) | Chiffrement standard + accès contrôlé |

**Avantages :** garde la promesse privacy sur ce qui compte vraiment (le contenu vocal),
sans bloquer la queryabilité ni les features business.

---

## La tension à anticiper : deux tiers de données

Pour un futur web layer business, il faut distinguer :

| Tier | Exemples | Modèle adapté |
|------|----------|---------------|
| **Données personnelles** | Captures, pensées privées, audio | E2E chiffré, mobile-only |
| **Données business** | CRM, factures, projets clients | Chiffrement serveur standard + web accessible |

Si tout est mis dans le même modèle E2E sans prévoir cette distinction,
le mur technique sera difficile à franchir a posteriori.

---

## Ce que ça implique pour l'architecture actuelle

**Rien à changer maintenant.** Points d'attention pour plus tard :

1. **Backend** : garder les BCs propres et indépendants — `Billing` et `CRM` s'emboîteront naturellement dans NestJS
2. **Web app** (`pensieve/web`) : actuellement minimal dashboard — c'est la surface à enrichir, pas une 3ème app mobile
3. **ADR à écrire** quand la décision est prise : Privacy Model for Web Layer
4. **Monorepo** : même repo `pensieve` suffit largement pour ajouter des modules NestJS business

---

## Décision différée

La décision sur le modèle privacy du web layer est **volontairement différée** :
- Pas de besoin immédiat (focus mobile)
- Les contraintes réelles (volume de données, types d'utilisateurs, RGPD sectoriel) ne sont pas encore connues
- Le Chemin B (privacy by commitment) reste le fallback pragmatique par défaut

**Déclencheur de décision :** premier client B2B intéressé par le web layer.

---

## Références

- ADR-010 (Security & Encryption — chiffrement actuel)
- ADR-017 (Génération clé maître — master key mobile)
- ADR-022 (State Persistence — OP-SQLite)
- ADR-036 (LLM Local-First — BYOK)
- Epic 23.19 (Semantic Search — impacté par ce choix)
- Epic 23.26 (Multi-Capture Synthesis — impacté par ce choix)

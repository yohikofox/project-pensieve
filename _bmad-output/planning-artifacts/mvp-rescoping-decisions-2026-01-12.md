# MVP Rescoping Decisions - Pensine

**Date:** 2026-01-12
**Participants:** yohikofox + Mary (Business Analyst)
**Context:** Post-UX Design completion, pre-Architecture phase

---

## Executive Summary

Suite à des interviews utilisateurs (notamment avec Bastien, le frère de Yoann), nous avons réévalué le scope MVP pour mieux aligner le produit avec les besoins réels des early adopters. Les changements principaux :

- **Retrait** : Capture URL (déplacée en V1.5)
- **Ajout** : Todo-list générée automatiquement par IA (approche hybride)
- **Ajout** : Tab "Actions" pour vue centralisée des todos
- **Enrichissement** : Digestion IA inclut maintenant la détection d'actions

**Impact net :** Scope MVP similaire en complexité, mais mieux aligné avec les 2 personas identifiés.

---

## 1. Contexte & Feedback Utilisateur

### Interview Early Adopter (Bastien)

**Profil :** Frère de Yoann, utilisateur cible identifié dans le PRD original.

**Feedback positif :**
- ✅ **Très enthousiaste** sur le concept général
- ✅ **Fan absolu** de la capture rapide sans friction
- ✅ Le côté "je cours → je parle → c'est noté → je passe à autre chose" est LE différenciateur pour lui
- ✅ Use case principal : Capturer des business ideas en mouvement (pas nécessairement pour créer des SaaS)

**Feedback neutre/négatif :**
- ❌ **Pas intéressé** par les regroupements thématiques d'idées
- ❌ **Pas intéressé** par la découverte de patterns/connexions entre idées
- ⚠️ **Faible intérêt** pour recevoir des suggestions basées sur ses captures
- ⚠️ Le digest IA n'est **pas un besoin exprimé** pour lui

**Intérêt potentiel futur :**
- 💡 **Automatisations simples** : "Ah mince j'ai oublié d'envoyer la facture à machin" → automatisation possible
- 💡 **Assistanat actionnable** plutôt que digest stratégique

**Citation clé :** *"C'est vraiment le côté dictaphone, mais dictaphone intelligent quoi. C'est ça qui l'intéresse là dedans."*

---

## 2. Personas Identifiés

### Persona 1 : "Quick Capturer" (Bastien)

**Job-to-be-done :** "Je veux externaliser mes pensées instantanément sans me prendre la tête"

**Caractéristiques :**
- Veut capturer rapidement
- Ne veut PAS passer du temps à organiser/analyser
- Valorise la simplicité extrême
- Use case : Business ideas, notes perso, rappels

**Ce qui l'intéresse :**
- ✅ Capture vocale ultra-rapide
- ✅ Retrouver facilement ce qu'il a noté
- ✅ Actions/suggestions simples basées sur captures
- ❌ Pas de thèmes/patterns/connexions complexes

**Mantra :** *"Je cours → je parle → c'est noté → je passe à autre chose"*

---

### Persona 2 : "Strategic Builder" (Yoann)

**Job-to-be-done :** "Je veux transformer mes pensées éparses en projets structurés"

**Caractéristiques :**
- Veut capturer ET analyser
- Valorise la découverte de patterns
- Veut l'aide de l'IA pour structurer sa pensée
- Use case : Idées de SaaS, décisions stratégiques, incubation projets

**Ce qui l'intéresse :**
- ✅ Capture vocale rapide (baseline)
- ✅ Digest IA pour structurer/synthétiser
- ✅ Découverte de patterns et connexions
- ✅ Enrichissement et recommandations stratégiques

**Mantra :** *"Je capture → l'IA digère → je découvre des insights → je lance"*

---

## 3. Hypothèse de Progression Validée

**Question posée (Q1) :** Progression naturelle ou bifurcation ?

**Réponse validée :** **Option A - Progression naturelle**

**Hypothèse :**
- Tout le monde commence en mode "Quick Capturer"
- Certains utilisateurs évoluent naturellement vers "Strategic Builder" au fil du temps
- La découverte du digest se fait progressivement, pas poussée agressivement dès le départ

**Implication produit :**
- Le MVP doit avoir les deux capacités (capture + digest)
- **MAIS** l'onboarding et le messaging mettent l'accent sur la capture rapide
- Le digest est découvert naturellement par l'usage, pas imposé

**Stratégie de communication MVP :**
- Pitch principal : *"Dictaphone intelligent qui capture tes pensées en 2 secondes"*
- Pitch secondaire (découverte) : *"Et en plus, l'IA organise et structure tes idées automatiquement"*

---

## 4. Décisions de Rescoping

### Q1 : Capture URL - MVP ou V1.5 ?

**Décision :** **Pousser en V1.5**

**Rationale :**
- Moins critique pour le persona "Quick Capturer" (dictaphone intelligent)
- Simplifie le MVP en se concentrant sur audio + texte
- Reste aligné avec la vision long-terme
- Permet de valider la valeur core plus rapidement

**Impact :**
- MVP : Capture audio + texte uniquement
- V1.5 : Ajout de Capture URL avec extraction de contenu

---

### Q2 : Todo-List Générée - MVP ou V1.5 ?

**Décision :** **Intégrer au MVP (Option A)**

**Rationale :**
- C'est un **pont naturel** entre capture simple et digest stratégique
- Donne une **valeur actionnable immédiate** (même pour persona 1)
- Plus simple techniquement qu'un dashboard de scoring
- Valide que l'IA comprend les intentions des captures
- Répond à l'intérêt exprimé pour les "automatisations simples"

**Exemple concret :**
```
Capture audio : "Penser à appeler Jean lundi pour le devis"

App génère automatiquement :
📋 Action détectée :
  - "Appeler Jean pour le devis"
  - Échéance suggérée : Lundi
  - [Marquer comme fait] [Snooze] [Supprimer]
```

**Impact :**
- MVP : Digestion IA enrichie avec détection d'actions
- Nouveau feature : Todo-list générée automatiquement
- Valeur différenciante vs concurrents (Voicenotes, AudioPen)

---

### Q3 : Architecture Todo-List - Approche ?

**Décision :** **Approche C - Hybride (inline + centralisée)**

**Options évaluées :**
- A : Todo-list intégrée dans chaque idée uniquement
- B : Todo-list centralisée uniquement
- C : Hybride (les deux vues)

**Rationale pour Approche C :**
- **Best of both worlds**
- Permet de voir les todos dans leur contexte (vue idée)
- Permet aussi une vue actionnable "What should I do today?" (vue centralisée)
- Flexible selon le besoin utilisateur
- Répond aux deux personas

**Architecture Navigation :**
```
Bottom Tab Navigation (3 tabs) :

[Feed] - Feed des idées capturées
  - Liste chronologique
  - Chaque card montre : titre, résumé, todos inline
  - FAB pour capture rapide

[Actions] - Todos centralisées ✨ NEW
  - Vue "What should I do today?"
  - Toutes les actions détectées de toutes les idées
  - Filtres : Toutes / Faites / À faire
  - Tri : Date / Priorité (si détectée par IA)
  - Link vers l'idée d'origine

[Profil] - Settings & Account
  - Paramètres utilisateur
  - Stats basiques
  - Logout
```

---

## 5. Nouveau Scope MVP Finalisé

### MVP Phase 1 - "Dictaphone Intelligent"

| Feature | Description | Priorité |
|---------|-------------|----------|
| **Capture audio 1-tap** | Enregistrement vocal instantané, fire-and-forget | MVP Core ✅ |
| **Capture texte** | Saisie rapide pour contextes publics | MVP Core ✅ |
| **Transcription auto** | Audio → Texte via Whisper | MVP Core ✅ |
| **Digestion IA enrichie** | Résumé + extraction idées + **détection d'actions** | MVP Core ✅ |
| **Todo-list générée** | Actions extraites automatiquement des captures | MVP Core ✨ NEW |
| **Consultation Feed** | Liste chronologique + vue détail + todos inline | MVP Core ✅ |
| **Tab Actions** | Vue centralisée de toutes les todos avec filtres/tri | MVP Core ✨ NEW |
| **Notifications** | Process IA terminé, rappels todos (optionnel) | MVP Core ✅ |

**Navigation :** 3 tabs (Feed + Actions + Profil)

---

### Post-MVP (V1.5) - Growth Features

| Feature | Phase | Rationale |
|---------|-------|-----------|
| **Capture URL** | V1.5 | Déplacé du MVP, extraction de contenu d'articles |
| **Dashboard idées chaudes/froides** | V1.5 | Visualisation scoring, unlock persona Strategic Builder |
| **Toggle "Lancé" + feedback loop** | V1.5 | Mesure passage à l'action |
| **Brainstorm guidé** | V1.5 | Mode solo IA pour creuser une idée |
| **Enrichissement post-capture** | V1.5 | Ajouter contexte après la capture initiale |

---

### Vision (V2-V3+)

| Feature | Phase | Horizon |
|---------|-------|---------|
| **Capture photo** | V2 | Multi-modal complet |
| **Détection récurrence** | V2 | Nécessite historique suffisant |
| **Export/Partage** | V2+ | Viralité post-validation |
| **Automatisations avancées** | V2+ | "Envoyer facture à machin" |
| **Scoring avancé** | V3+ | Croisement d'idées, germination hybride |
| **Clone cognitif** | V3+ | LLM fine-tuné sur patterns utilisateur |

---

## 6. Comparaison Avant/Après

### PRD Original MVP
```
✅ Capture audio 1-tap
✅ Capture texte
✅ Capture URL ← RETIRÉ
✅ Transcription auto
✅ Digestion IA (résumé + idées)
✅ Consultation (feed)

Navigation : 2 tabs (Feed + Profil)
Total features : 6
```

### Nouveau MVP (Post-Rescoping)
```
✅ Capture audio 1-tap
✅ Capture texte
✅ Transcription auto
✅ Digestion IA (résumé + idées + actions) ← ENRICHI
✨ Todo-list générée (hybride) ← NOUVEAU
✅ Consultation (feed + détail + todos inline) ← ENRICHI
✨ Tab Actions centralisé ← NOUVEAU

Navigation : 3 tabs (Feed + Actions + Profil)
Total features : 7 (mais -1 de complexité avec retrait URL)
```

**Net Changes :**
- **-1** Capture URL (moved to V1.5)
- **+1** Détection d'actions IA
- **+1** Vue centralisée Actions
- **+1** Tab navigation

**Complexité Dev :** Similaire (trade-off équilibré)
**Alignement personas :** Beaucoup mieux aligné

---

## 7. Spécifications Techniques Ajoutées

### Détection d'Actions par l'IA

**Prompt enrichi pour digestion :**
```
Analyze this capture and extract:
1. Summary (2-3 sentences)
2. Key ideas (bullet points)
3. Actionable tasks (if any) with:
   - Task description (clear, actionable)
   - Suggested deadline (if mentioned in capture)
   - Priority (high/medium/low if inferable from context)

Output format:
{
  "summary": "...",
  "ideas": [...],
  "actions": [
    {
      "task": "Appeler Jean pour le devis",
      "deadline": "Monday",
      "priority": "medium"
    }
  ]
}
```

### Modèle de Données

**Table `ideas` (existante + ajout) :**
```sql
ideas:
  - id (uuid)
  - user_id (uuid FK)
  - audio_url (text nullable)
  - transcription (text)
  - summary (text)
  - ideas (jsonb)
  - actions (jsonb) ← NEW
  - created_at (timestamp)
  - updated_at (timestamp)
```

**Table `actions` (nouvelle) :**
```sql
actions:
  - id (uuid)
  - idea_id (uuid FK)
  - task (text NOT NULL)
  - deadline (timestamp nullable)
  - priority (enum: high/medium/low nullable)
  - completed (boolean default false)
  - completed_at (timestamp nullable)
  - created_at (timestamp)
  - updated_at (timestamp)

Indexes:
  - idea_id
  - completed
  - deadline (where not null)
```

### User Flow Exemple

**Scenario : Bastien (Quick Capturer) en train de courir**

1. **Capture (2 secondes)**
   ```
   User : *Tape FAB* → *Parle en courant*
   "Penser à appeler Jean lundi matin pour le devis
    et envoyer la facture à Marie avant vendredi"
   ```

2. **Processing Background**
   ```
   - Audio sauvegardé localement (offline-first)
   - Upload background quand réseau disponible
   - Transcription via Whisper
   - Digestion IA via GPT-4o-mini
   - Détection de 2 actions
   ```

3. **Notification**
   ```
   🔔 "Idée capturée et analysée"
   ```

4. **User ouvre app → Tab [Actions]**
   ```
   📋 2 nouvelles actions détectées :

   [ ] Appeler Jean pour le devis
       📅 Suggéré : Lundi matin
       ↳ De : Idée #42 (il y a 5 min)
       [Marquer fait] [Snooze] [Supprimer]

   [ ] Envoyer facture à Marie
       ⚠️ Avant vendredi
       ↳ De : Idée #42 (il y a 5 min)
       [Marquer fait] [Snooze] [Supprimer]
   ```

5. **User coche action**
   ```
   ✅ "Appeler Jean" marquée comme faite
   - Action disparaît de "À faire"
   - Reste visible dans "Toutes" (strikethrough)
   - Dans Feed, todo de l'idée #42 cochée aussi
   ```

6. **Plus tard, user va dans [Feed]**
   ```
   Idée #42 : "Appel Jean + facture Marie"

   📝 Transcription : [...]
   💡 Résumé : Contact commercial Jean + admin Marie

   ✅ Actions (1/2 terminées) :
     ✓ Appeler Jean pour le devis (fait il y a 2h)
     [ ] Envoyer facture à Marie (avant vendredi)
   ```

---

## 8. User Journeys Impactés

### Journey 1 : Yoann - Happy Path (Inchangé globalement)
```
Capture → Transcription → Digestion → Consultation
```

**Ajout :** Après digestion, todos générées automatiquement et consultables dans tab Actions

### Journey 2 : Yoann - Edge Case (Inchangé)
```
Capture texte → Digestion (enrichissement = V1.5)
```

**Ajout :** Détection d'actions fonctionne aussi sur captures texte

### Journey 3 : Bastien - Quick Capturer (Nouveau/Validé)
```
Capture rapide → [Ignore digestion] → Consulte Actions → Coche todos
```

**Value prop pour lui :** Dictaphone intelligent qui génère une todo-list automatiquement

---

## 9. Différenciation Concurrentielle Renforcée

### Concurrents (Voicenotes, AudioPen, Cleft)

**Ce qu'ils offrent :**
- Capture vocale
- Transcription
- Organisation basique (tags, folders)

**Ce qu'ils N'offrent PAS :**
- ❌ Détection automatique d'actions
- ❌ Todo-list générée par IA
- ❌ Digestion intelligente avec résumé + idées + actions
- ❌ Vue actionnable centralisée

### Pensine MVP (Post-Rescoping)

**Ce qu'on offre EN PLUS :**
- ✅ Détection automatique d'actions dans les captures
- ✅ Todo-list générée intelligemment (pas juste transcription)
- ✅ Vue hybride : contexte (feed) + actionnable (actions tab)
- ✅ IA qui comprend l'intention et suggère deadlines/priorités
- ✅ Bridge entre "just capture" et "strategic planning"

**Positioning :**
- **Baseline :** "Le dictaphone intelligent le plus rapide"
- **Différenciateur :** "Qui transforme tes pensées en actions automatiquement"
- **Vision :** "Ton incubateur personnel d'idées"

---

## 10. Risques & Mitigation

### Risque 1 : Complexité MVP augmentée

**Description :** Ajouter todo-list + tab Actions augmente le scope

**Mitigation :**
- Retrait Capture URL compense en complexité
- Todo-list utilise la même IA que digestion (pas de nouveau service)
- Tab Actions est une vue différente sur données existantes (pas nouveau backend)
- Net complexity : similaire au MVP original

### Risque 2 : Persona "Quick Capturer" trouve ça trop complexe

**Description :** Bastien veut juste capturer, pourrait être overwhelmed par les features

**Mitigation :**
- Onboarding met l'accent sur capture rapide (FAB prominente)
- Tab Actions est optionnel (pas imposé dans le flow principal)
- Notifications todos peuvent être désactivées
- Progressive disclosure : découverte naturelle, pas push agressif

### Risque 3 : Détection d'actions IA pas fiable

**Description :** L'IA pourrait mal interpréter ou manquer des actions

**Mitigation :**
- MVP : Accepter imparfait (mieux vaut suggestions imparfaites que rien)
- User peut toujours supprimer fausses détections
- Feedback loop : "Cette action n'était pas pertinente" → améliore modèle
- V1.5 : Affiner le prompt et ajouter few-shot examples

### Risque 4 : Users ne comprennent pas la valeur des todos générées

**Description :** Pourquoi c'est mieux qu'une todo-list manuelle ?

**Mitigation :**
- Onboarding explique : "L'app détecte automatiquement tes actions"
- Example concret dans tutorial
- Empty state suggestif : "Capture une idée avec une action, l'app la détectera"
- Analytics : mesurer % d'idées avec actions détectées vs actions manuelles ajoutées

---

## 11. Métriques de Succès MVP (Ajustées)

### Métriques Originales (Conservées)
- Confiance fire-and-forget : ≥ 5 captures/semaine
- Redécouverte de valeur : > 30% idées revisitées après 7+ jours
- Moment "aha!" : ≥ 1 idée identifiée comme prometteuse/mois
- Passage à l'action : ≥ 1 idée marquée "Lancé"/trimestre

### Nouvelles Métriques (Actions)

| Métrique | Indicateur | Cible MVP |
|----------|------------|-----------|
| **Détection d'actions** | % de captures qui génèrent ≥1 action | > 30% |
| **Utilisation tab Actions** | % users qui ouvrent tab Actions ≥1/semaine | > 50% |
| **Completion rate** | % d'actions générées qui sont cochées | > 40% |
| **Valeur perçue Actions** | User feedback "Les actions générées sont utiles" | > 60% positif |

### Critères de Succès Personas

**Quick Capturer (Bastien) :**
- ✅ Utilise l'app ≥ 5x/semaine pour capturer
- ✅ Consulte tab Actions régulièrement (≥2x/semaine)
- ✅ Coche des todos générées (completion rate > 30%)
- ⚠️ N'utilise PAS nécessairement le digest stratégique (ok)

**Strategic Builder (Yoann) :**
- ✅ Utilise l'app ≥ 5x/semaine pour capturer
- ✅ Consulte Feed pour revoir résumés/idées (≥3x/semaine)
- ✅ Utilise tab Actions pour actionner (≥2x/semaine)
- ✅ Enrichit idées et découvre patterns (V1.5+)

---

## 12. Prochaines Étapes

### Immédiat (Aujourd'hui)
1. ✅ **Document de décisions créé** (ce fichier)
2. ⏳ **Mettre à jour le PRD** avec le nouveau scope
   - Modifier section "Product Scope → MVP"
   - Modifier section "Growth Features"
   - Ajuster "User Journeys"
   - Mettre à jour "MVP Feature Set"
   - Ajouter FRs pour détection d'actions et tab Actions

### Court terme (Cette semaine)
3. ⏳ **Workflow Architecture** - Définir stack technique avec todo-list en scope
4. ⏳ **Epics & Stories** - Découper MVP en user stories avec actions feature

### Moyen terme (Ce mois)
5. ⏳ **Test Design** - Stratégie de test pour détection d'actions IA
6. ⏳ **Implementation Readiness** - Valider readiness avant sprint planning

---

## 13. Questions Ouvertes (Pour Architecture)

Ces questions devront être adressées lors de la phase Architecture :

1. **Détection d'actions :**
   - Quel prompt exactement pour GPT-4o-mini ?
   - Fallback si API IA down ?
   - Combien coûte la détection par capture ?

2. **Stockage actions :**
   - Relation ideas ↔ actions (1-to-many)
   - Besoin d'un event sourcing pour actions ?
   - Gestion des actions multi-captures (récurrence) ?

3. **Sync offline :**
   - Actions créées offline, comment sync ?
   - Conflits si action cochée sur device A et modifiée sur device B ?

4. **Performance :**
   - Tab Actions : pagination ou infinite scroll ?
   - Filtres actions : côté client ou serveur ?

5. **Notifications :**
   - Rappels pour actions avec deadline ?
   - Fréquence/stratégie pour pas spammer ?

---

## 14. Changelog vs PRD Original

### Ajouts
- ✨ **FR-XX** : Détection automatique d'actions dans digestion IA
- ✨ **FR-XX** : Todo-list générée avec task, deadline, priority
- ✨ **FR-XX** : Tab "Actions" avec vue centralisée
- ✨ **FR-XX** : Vue hybride actions (inline dans Feed + centralisée)
- ✨ **FR-XX** : Filtres/tri dans tab Actions (Toutes/Faites/À faire)
- ✨ **FR-XX** : Link action → idée d'origine
- ✨ **FR-XX** : Completion tracking (checkbox actions)

### Modifications
- 🔄 **FR Digestion IA** : Enrichi pour inclure détection d'actions
- 🔄 **Navigation** : 2 tabs → 3 tabs (Feed + Actions + Profil)
- 🔄 **Consultation Feed** : Affiche todos inline dans chaque idée

### Retraits (Déplacés en V1.5)
- ➖ **FR Capture URL** : Moved from MVP to V1.5
- ➖ **FR Extraction contenu articles** : Moved from MVP to V1.5

---

## 15. Sign-off

**Décisions validées par :**
- yohikofox (Product Owner)

**Analysé et documenté par :**
- Mary (Business Analyst Agent)

**Date de validation :** 2026-01-12

**Statut :** ✅ Approuvé pour implémentation

**Prochaine action :** Mettre à jour le PRD officiel avec ces décisions

---

## Annexe A : Verbatim Interview

### Feedback Bastien (Persona Quick Capturer)

> "Globalement, je pense qu'ils sont fans, je l'ai interviewé plusieurs. Normalement, je pense qu'ils sont chauds de utiliser mon application."

> "Le premier truc que mon frère à qui j'ai parlé principalement vous voudrez, c'est effectivement le côté où il est en train de courir, il prend son idée, il la note, vient un vocal, il ne se prend pas la tête, il reviendra dessus plus tard."

> "Il a un petit côté là qui le haïe beaucoup, il est très porté sur cette idée-là et j'en suis content."

> "Pour lui effectivement, le caractère capture et pas ça autre chose, c'est vraiment ça le truc ultra différenciant. Il n'a pas envie de se prendre la tête, c'est vraiment le côté dictaphone, mais dictaphone intelligent quoi."

> "Le digest IA ne doit pas être un bonus, je dis juste que mon frère n'a pas émis, au moins c'est un besoin pour moi déjà."

> "Je pense que pour lui, pour son persona lui particulièrement, ce sera pas dans les VIP je pense, mais on pourra ouvrir des choses à base d'assistanat."

> "C'est en gros peut-être que lui quand il va pouvoir faire 'ah mince j'ai oublié d'envoyer la facture à machin', et bien ça ça pourrait être une automatisation qu'on pourrait lui permettre de faire."

### Position Yoann (Product Owner)

> "Moi j'aimerais bien qu'on puisse faire les trucs de résumé tout ça et j'aimerais bien aussi qu'on propose des choses derrière cette digestion de data."

> "Réponses C, sinon les deux sont indissociables pour moi." (Capture + Digest)

> "Je pense que j'aurais besoin de faire de l'automatisation post MVP, mais dans un premier temps, peut être une simple todo liste générée par les captures + IA."

---

**Fin du document**

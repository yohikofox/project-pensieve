---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]
inputDocuments:
  - _bmad-output/planning-artifacts/product-brief-pensine-2026-01-09.md
  - _bmad-output/analysis/pensine-brainstorming-2026-01-09.md
workflowType: 'prd'
lastStep: 2
documentCounts:
  brief: 1
  research: 0
  brainstorming: 2
  projectDocs: 0
projectType: greenfield
---

# Product Requirements Document - Pensine

**Author:** yohikofox
**Date:** 2026-01-09

## Executive Summary

### Vision Produit

**Pensine** est un incubateur personnel qui transforme les pensées capturées en opportunités business concrètes.

L'application permet un usage "second brain" classique (capture, transcription, consultation) sans friction, mais sa vraie valeur ajoutée est d'aller plus loin : révéler les patterns, scorer les idées, et accompagner ceux qui veulent transformer leurs pensées en business.

**Philosophie :** L'utilisateur choisit jusqu'où il va — Pensine ne force rien, il propose plus. Chaque capture — même anodine — nourrit le moteur qui révélera les futures opportunités.

### Ce qui rend Pensine spécial

| Aspect | Concurrents (Voicenotes, AudioPen...) | Pensine |
|--------|---------------------------------------|---------|
| **Usage de base** | Capture + transcription + notes | ✅ Identique — compatible "second brain" |
| **Valeur ajoutée** | S'arrête là | Digestion IA, scoring, incubation business |
| **Finalité** | "Retrouve tes pensées" | "Découvre ce que tu construis sans le savoir" |
| **Données** | Stock passif | Carburant pour connexions et insights |

**Insight unique :** Plus tu captures, plus le moteur de connexions devient puissant. Valeur exponentielle, pas linéaire.

### Vision Long-terme (V3+)

- **Croisement d'idées** : Tags + matching → germination d'idées hybrides entre captures
- **Clone cognitif personnel** : LLM fine-tuné sur tes patterns de réflexion, débloqué après ~1 an d'usage intensif
- **Flywheel** : Plus tu captures → Plus de données → Plus de connexions → Plus de valeur → Plus tu captures

---

## Project Classification

| Critère | Valeur |
|---------|--------|
| **Type technique** | Mobile App (React Native) + SaaS Backend |
| **Domaine** | Productivité personnelle / Outils créateurs |
| **Complexité** | Low-Medium |
| **Contexte** | Greenfield — nouveau projet |

**Stack envisagée (du brainstorming) :**
- Client : React Native
- Backend : Node.js (Fastify)
- BDD : PostgreSQL
- Pattern : CQRS + Event Sourcing + Ports & Adapters
- IA : Whisper (transcription) + GPT-4o-mini (digestion)

---

## Success Criteria

### User Success

| Critère | Indicateur | Cible |
|---------|------------|-------|
| **Confiance fire-and-forget** | L'utilisateur capture sans hésitation | ≥ 5 captures/semaine |
| **Redécouverte de valeur** | Idées revisitées après 7+ jours | > 30% |
| **Moment "aha!"** | Idée identifiée comme "chaude" ou prometteuse | ≥ 1/mois/utilisateur actif |
| **Passage à l'action** | Idée marquée "Lancé" par l'utilisateur | ≥ 1/trimestre/utilisateur |

**Moment de succès utilisateur :** L'utilisateur réalise qu'une idée qu'il avait oubliée revient souvent et mérite d'être creusée. Il se dit : "Sans Pensine, j'aurais laissé passer ça."

### Business Success

| Horizon | Objectif | Indicateur |
|---------|----------|------------|
| **3 mois** | Validation early adopters | 5 utilisateurs actifs (3 cœur de cible + 2 exploratoires) |
| **6 mois** | Traction organique | Premiers utilisateurs acquis par bouche-à-oreille |
| **12 mois** | Croissance saine | Ratio CAC/LTV viable, churn < 10%/mois |

**Modèle économique :** Freemium. L'acquisition repose sur un accès gratuit qui permet de découvrir la valeur avant engagement payant.

**Note :** Le taux de conversion Free → Payant (cible indicative : 5%) est un indicateur de santé, pas un critère de succès MVP.

### Technical Success

| Critère | Définition |
|---------|------------|
| **Latence perçue** | La transcription et la digestion doivent sembler instantanées pour l'utilisateur |
| **Fiabilité totale** | Aucune capture ne doit jamais être perdue — confiance absolue requise |
| **Disponibilité** | L'app doit être accessible quand l'idée surgit (offline-first si nécessaire) |

*Les métriques techniques précises (latence en ms, SLA, architecture) seront définies lors de la phase Architecture.*

### Measurable Outcomes

**North Star Metric :**
> Nombre d'idées ayant mené à une action concrète par utilisateur

**Mécanisme de mesure :**
- Toggle "Lancé" sur chaque idée (optionnel, déclaratif)
- Notification in-app après X semaines sur les idées actives : "Tu l'as lancé ? Qu'est-ce qui te freine ?"
- Feedback loop intégré pour capturer le signal action

---

## Product Scope

### MVP - Minimum Viable Product

| Feature | Description | Priorité |
|---------|-------------|----------|
| **Capture audio 1-tap** | Enregistrement vocal instantané, fire-and-forget | MVP Core |
| **Capture texte** | Saisie rapide pour contextes publics | MVP Core |
| **Transcription automatique** | Audio → Texte via Whisper | MVP Core |
| **Digestion IA enrichie** | Résumé + extraction d'idées + détection d'actions automatiques | MVP Core |
| **Todo-list générée** | Actions extraites automatiquement des captures par IA | MVP Core |
| **Consultation Feed** | Liste chronologique + vue détail + todos inline | MVP Core |
| **Tab Actions** | Vue centralisée de toutes les todos avec filtres/tri | MVP Core |

**Critères de succès MVP :**
1. **Dogfooding** — Le créateur utilise Pensine quotidiennement
2. **5 early adopters actifs** — ≥ 5 captures/semaine chacun
3. **Rétention naturelle** — Ils reviennent sans relance

### Growth Features (Post-MVP)

| Feature | Phase | Rationale |
|---------|-------|-----------|
| **Capture URL avec extraction** | V1.5 | Sauvegarde d'articles, déplacé du MVP pour focus capture audio/texte |
| **Dashboard idées chaudes/froides** | V1.5 | Visualisation de la valeur, aide à "vendre" le pilotage |
| **Toggle "Lancé" + feedback loop** | V1.5 | Mesure du passage à l'action |
| **Brainstorm guidé (mode solo)** | V1.5 | Cristallisation d'idée avec IA |
| **Enrichissement post-capture** | V1.5 | Ajout de contexte après capture initiale |
| **Capture photo** | V2 | Multi-modal complet |
| **Détection de récurrence** | V2-V3 | Nécessite historique suffisant |
| **Export / Partage** | V2+ | Viralité post-validation |

### Vision (Future)

| Phase | Horizon | Objectif |
|-------|---------|----------|
| **V3+** | 12-18 mois | Scoring avancé, croisement d'idées, germination hybride |
| **Clone cognitif** | 2-3 ans | LLM fine-tuné sur les patterns de réflexion de l'utilisateur |
| **Plateforme** | 3+ ans | Communauté, matching co-fondateurs, écosystème incubateur |

---

## User Journeys

### Journey 1 : Yoann — De l'idée fugace au projet lancé

Yoann est un développeur indie hacker qui jongle entre son travail et sa quête permanente de la prochaine opportunité produit. Les idées lui viennent partout : en réunion quand un collègue râle sur un outil, dans le métro en observant les gens galérer avec une app, sous la douche quand son cerveau fait des connexions inattendues.

Son problème ? Ces idées disparaissent aussi vite qu'elles arrivent. Il a essayé les notes vocales, les post-its, les apps de notes — mais tout finit dans un cimetière numérique qu'il ne consulte jamais. Pire : il a le sentiment récurrent d'avoir déjà eu "cette super idée" mais de l'avoir perdue.

Un matin, coincé dans les transports, il entend un entrepreneur se plaindre de sa compta freelance. "Tiens, ça fait 3 fois que j'entends ce problème..." Il sort son téléphone, ouvre Pensine, et en 10 secondes il capture sa pensée vocalement : *"Compta freelance — encore quelqu'un qui galère. Regarder ce qui existe, pourquoi c'est nul."*

Il range son téléphone et oublie. Pensine s'occupe du reste : transcription, résumé, extraction de l'idée clé.

Deux semaines plus tard, en consultant son fil Pensine pendant une pause café, il découvre que cette idée de compta freelance revient pour la 4ème fois ce mois-ci, sous différentes formes. Le système lui montre les connexions. "Merde, je tourne autour de ce truc depuis des semaines sans m'en rendre compte."

Il creuse, fait quelques recherches, et décide : cette idée mérite un week-end d'exploration. Il marque l'idée "Lancé" dans Pensine. Six mois plus tard, son side-project de compta simplifiée a ses premiers utilisateurs payants.

### Journey 2 : Yoann — La capture qui échoue (edge case)

Yoann est en pleine conversation avec un client potentiel qui lui décrit un problème business juteux. Impossible de sortir son téléphone pour enregistrer — ce serait malpoli. Dès que la conversation se termine, il ouvre Pensine et tape rapidement quelques mots-clés : *"Client PME — gestion stock manuel — perd 2h/jour — frustré Excel"*.

La capture texte est instantanée. Mais le soir, en consultant Pensine, il réalise que ses notes sont trop cryptiques. La digestion IA a fait ce qu'elle pouvait, mais le contexte manque.

Il utilise la fonction "Enrichir" pour ajouter une note vocale de contexte : *"C'était le gérant d'une boîte de 15 personnes, ils utilisent Excel pour tout et ça les tue..."*

Le lendemain, la digestion combinée texte + audio lui donne un résumé exploitable. Leçon apprise : la capture texte c'est bien pour le contexte social, mais enrichir après c'est mieux que rien.

### Journey 3 : Bastien — Le destinataire d'une idée filtrée

Bastien est designer produit et ami de Yoann depuis des années. Il reçoit régulièrement des messages de Yoann du style "J'ai une idée, t'en penses quoi ?" — mais 90% du temps c'est du brainstorming brut, pas mûr, qui lui prend du temps à évaluer.

Un jour, Yoann lui envoie quelque chose de différent : un lien Pensine vers une idée qui a été capturée 6 fois en 2 mois, avec un score "chaud", des extraits de contexte, et une synthèse de pourquoi ça revient.

Bastien est impressionné. Pour une fois, ce n'est pas "j'ai eu une idée sous la douche", c'est "voici une opportunité que mon cerveau a validée inconsciemment sur 2 mois". L'idée mérite 30 minutes de son attention.

Il répond avec des questions design pertinentes. La conversation est productive parce qu'elle part d'une base solide, pas d'une intuition floue.

"Comment tu fais pour ne m'envoyer que des trucs pertinents maintenant ?"
"Pensine. Ça filtre avant que je t'embête."

### Journey Requirements Summary

| Journey | Capacités requises |
|---------|-------------------|
| **Yoann - Happy path** | Capture audio 1-tap, transcription auto, digestion IA enrichie, todo-list générée, consultation fil + actions, détection récurrence (V2), marquage "Lancé" (V1.5) |
| **Yoann - Edge case** | Capture texte rapide, todo-list générée, enrichissement post-capture (V1.5), digestion multi-source |
| **Bastien - Destinataire** | Export/partage d'idée (V2+), vue "idée filtrée" avec contexte et score |

---

## Innovation & Novel Patterns

### Detected Innovation Areas

**Innovation clé : Brainstorm guidé pour cristallisation de concept**

Pensine ne se contente pas de capturer et organiser les pensées. Sa vraie innovation est de proposer une **session de brainstorm interactive** qui transforme une pensée brute en concept projet structuré.

| Niveau | Fonction | Différenciation |
|--------|----------|-----------------|
| Base | Capture + Transcription + Digestion | Parité avec le marché (Voicenotes, AudioPen) |
| **Innovation** | Brainstorm guidé par IA | Unique — équivalent d'un analyste business à la demande |

**Mécanisme :** L'utilisateur sélectionne une idée digérée et lance une session de brainstorm. L'IA pose des questions structurées (à la manière d'un analyste produit) pour extraire :
- Le problème racine
- La cible utilisateur
- L'ébauche de solution
- Les hypothèses à valider

**Output :** Un concept projet actionnable, pas juste une note bien rangée.

### Extension : Party Mode — Facilitation de brainstorm adaptative

**Deux modes de brainstorm guidé :**

| Mode | Fonctionnement |
|------|----------------|
| **Solo** | L'IA simule les différentes perspectives expertes (PM, dev, designer, etc.) — équivalent d'avoir un analyste à disposition |
| **Collaboratif** | L'IA facilite une session avec les participants réellement présents |

**En mode collaboratif :**

1. **Déclaration des présents** — "Qui est autour de la table ?" → L'utilisateur identifie les profils
2. **Analyse de la diversité** — L'IA évalue si les profils couvrent suffisamment d'angles
3. **Conseil si déséquilibre** — "Tu n'as que des profils tech. Pour cet exercice, une perspective business/marché serait précieuse. Je te conseille d'inviter [profils recommandés]."
4. **Choix de l'utilisateur** :
   - Reporter et inviter les bons profils
   - Continuer en connaissance de cause (couverture partielle)
   - Demander à l'IA de simuler les perspectives manquantes
5. **Facilitation adaptée** — Questions orientées selon les expertises disponibles

**Philosophie :** Pensine conseille pour maximiser la valeur du brainstorm, mais ne bloque jamais. L'utilisateur reste maître de sa décision.

### Validation Approach

- **MVP :** La digestion automatique fonctionne et produit des résumés exploitables
- **V1.5+ :** Le brainstorm guidé est testé avec les early adopters — mesure : % d'idées qui passent en session brainstorm, satisfaction post-session
- **Succès :** Un utilisateur transforme une pensée capturée en projet lancé grâce au brainstorm guidé

### Risk Mitigation

**Risque principal :** Le brainstorm guidé ne produit pas assez de valeur pour justifier l'effort utilisateur.

**Approche :** Pas de fallback artificiel. Si ça ne fonctionne pas, c'est un échec du concept — et c'est acceptable. L'apprentissage vaudra l'investissement.

---

## Mobile App + SaaS Specific Requirements

### Project-Type Overview

Pensine est une **application mobile cross-platform (React Native)** avec un **backend SaaS**. L'architecture doit supporter un usage mobile-first avec des contraintes offline, tout en préparant une évolution vers des fonctionnalités collaboratives.

### Platform Requirements

| Plateforme | Support | Notes |
|------------|---------|-------|
| **iOS** | MVP | App Store dès V1 |
| **Android** | MVP | Play Store dès V1 |
| **Web** | Non prévu | Mobile-first uniquement |

### Device Permissions

| Permission | Usage | Phase |
|------------|-------|-------|
| **Microphone** | Capture audio | MVP |
| **Storage** | Cache offline, fichiers audio | MVP |
| **Network** | Sync, API calls, IA distante | MVP |
| **Camera** | Capture photo | V2 |
| **Notifications** | Push pour process IA longs, rappels | MVP |

### Offline Strategy

| Fonctionnalité | Offline Support | Sync Strategy |
|----------------|-----------------|---------------|
| **Capture audio** | ✅ Obligatoire | Sync au retour réseau |
| **Transcription** | ✅ Locale (Whisper on-device) | N/A — traitement local |
| **Digestion IA** | ❌ Requiert réseau | Queue + retry |
| **Consultation** | ✅ Cache local | Sync incrémental |

**Notifications de progression :** Pour les process IA longs (digestion, brainstorm), notifications d'avancement en temps réel.

### Multi-Tenancy Model

| Aspect | MVP | Évolution possible |
|--------|-----|-------------------|
| **Isolation** | User-level (1 user = 1 espace) | Workspace-level si mode team |
| **Permissions** | Mono-utilisateur, pas de RBAC | RBAC léger si collaboration |
| **Party Mode** | Même device, licence unique | Multi-device avec invitations (V3+) |

**Décision architecture :** Prévoir une abstraction "owner" sur les entités pour faciliter une migration future vers des workspaces, sans construire le système de permissions maintenant.

### Integration Strategy

**Philosophie :** Garder les options ouvertes sans décider prématurément.

| Option | Description | Prérequis architecture |
|--------|-------------|----------------------|
| **A. SaaS complet** | Pensine gère idéation → pilotage → reporting | Données structurées (projets, tâches, statuts) |
| **B. Export tiers** | Pensine = incubateur, délègue l'exécution | API d'export propre |

**Décision :** L'architecture doit permettre les deux chemins. Décision finale en V2-V3 selon traction et feedback.

### Store Compliance

- **App Store (iOS)** : Respect guidelines Apple, review process anticipé
- **Play Store (Android)** : Conformité policies Google
- **Permissions justifiées** : Micro, caméra, notifications — usages clairement expliqués

---

## Project Scoping & Phased Development

### MVP Strategy & Philosophy

**Approche MVP :** Problem-Solving MVP — résoudre le problème cœur (capture fire-and-forget + digestion) avec le minimum de features.

**Équipe MVP :** Solo developer (yohikofox) — contrainte assumée qui impose un scope ultra-focalisé.

### MVP Feature Set (Phase 1)

**User Journeys supportés :**
- Journey 1 (Yoann - happy path) : Capture → Transcription → Digestion enrichie → Todo-list générée → Consultation Feed + Actions
- Journey 2 (Yoann - edge case) : Capture texte → Digestion → Todo-list générée (enrichissement post-capture = V1.5)

**Capacités Must-Have :**

| Feature | Description |
|---------|-------------|
| Capture audio 1-tap | Enregistrement vocal instantané |
| Capture texte | Saisie rapide pour contextes publics |
| Transcription automatique | Audio → Texte via Whisper on-device |
| Digestion IA enrichie | Résumé + extraction d'idées + détection d'actions automatiques |
| Todo-list générée | Actions extraites des captures avec deadline/priorité suggérées |
| Consultation Feed | Liste chronologique + vue détail + todos inline par idée |
| Tab Actions | Vue centralisée toutes todos avec filtres (Toutes/Faites/À faire) |
| Notifications progression | Feedback sur les process IA longs |

### Post-MVP Features

**Phase 2 — Growth (V1.5) :**
- Capture URL avec extraction de contenu *(déplacé du MVP suite au rescoping 2026-01-12)*
- Dashboard idées chaudes/froides
- Toggle "Lancé" + feedback loop
- Brainstorm guidé (mode solo IA)
- Enrichissement post-capture (audio sur texte existant)

**Phase 3 — Expansion (V2-V3) :**
- Capture photo
- Détection de récurrence automatique
- Export/Partage d'idées filtrées
- Brainstorm collaboratif (Party Mode)
- Scoring avancé + croisement d'idées

**Phase 4 — Vision (V3+) :**
- Clone cognitif personnel (LLM fine-tuné)
- Plateforme communautaire
- Matching co-fondateurs

### Risk Mitigation Strategy

**Risques techniques :**
- Whisper on-device → Tests early sur devices variés, fallback cloud prévu
- Qualité digestion IA → Itération prompts avec early adopters

**Risques marché :**
- Perception "encore une app de notes" → Messaging différenciant (incubateur ≠ stockage)
- Friction adoption → Dogfooding + feedback loop serrée

**Risques ressources :**
- Solo dev = contrainte forte → Scope MVP minimal, discipline anti-creep

---

## Functional Requirements

### Capture de Pensées

- FR1: L'utilisateur peut enregistrer une pensée audio en un tap depuis l'écran principal
- FR2: L'utilisateur peut capturer une pensée texte via saisie rapide
- FR3: L'utilisateur peut annuler une capture audio en cours
- FR4: Le système peut capturer de l'audio même sans connexion réseau
- FR5: Le système peut stocker les captures en attente de synchronisation

### Transcription

- FR6: Le système peut transcrire automatiquement les captures audio en texte
- FR7: Le système peut effectuer la transcription localement sur l'appareil (offline)
- FR8: L'utilisateur peut consulter la transcription complète d'une capture audio

### Digestion IA

- FR9: Le système peut générer un résumé concis de chaque capture
- FR10: Le système peut extraire les idées clés d'une capture
- FR11: Le système peut digérer une capture texte
- FR12: Le système peut digérer une transcription audio
- FR13: L'utilisateur peut être notifié de la progression des process IA longs

### Todo-list & Actions

- FR14: Le système peut détecter automatiquement les actions dans une capture lors de la digestion
- FR15: Le système peut extraire pour chaque action : description de tâche, deadline suggérée, priorité
- FR16: L'utilisateur peut voir les todos générées inline avec chaque idée dans le Feed
- FR17: L'utilisateur peut accéder à une vue centralisée de toutes ses actions via le tab "Actions"
- FR18: L'utilisateur peut filtrer les actions (Toutes / À faire / Faites)
- FR19: L'utilisateur peut marquer une action comme complétée (checkbox)
- FR20: L'utilisateur peut accéder à l'idée d'origine depuis une action

### Consultation

- FR21: L'utilisateur peut consulter la liste de ses captures
- FR22: L'utilisateur peut voir le détail d'une capture (audio, transcription, résumé, idées)
- FR23: L'utilisateur peut consulter ses captures hors connexion
- FR24: L'utilisateur peut distinguer les captures en attente de digestion

### Gestion de Compte

- FR25: L'utilisateur peut créer un compte
- FR26: L'utilisateur peut se connecter à son compte
- FR27: L'utilisateur peut se déconnecter
- FR28: L'utilisateur peut récupérer l'accès à son compte (mot de passe oublié)

### Synchronisation

- FR29: Le système peut synchroniser les captures locales vers le cloud au retour du réseau
- FR30: Le système peut synchroniser les données cloud vers l'appareil
- FR31: L'utilisateur peut être informé du statut de synchronisation

---

## Non-Functional Requirements

### Performance

| NFR | Critère | Cible |
|-----|---------|-------|
| NFR1 | Temps de démarrage de capture audio | < 500ms après tap |
| NFR2 | Temps de transcription (Whisper on-device) | < 2x durée audio |
| NFR3 | Temps de digestion IA (réseau disponible) | < 30s pour capture standard |
| NFR4 | Temps de chargement liste captures | < 1s (cache local) |
| NFR5 | Latence perçue | L'utilisateur ne doit jamais attendre sans feedback visuel |

### Reliability

| NFR | Critère | Cible |
|-----|---------|-------|
| NFR6 | Perte de données | 0 capture perdue, jamais |
| NFR7 | Disponibilité capture offline | 100% — capture fonctionne sans réseau |
| NFR8 | Récupération après crash | Captures en cours sauvegardées automatiquement |
| NFR9 | Synchronisation au retour réseau | Automatique, sans intervention utilisateur |

### Security

| NFR | Critère | Cible |
|-----|---------|-------|
| NFR10 | Authentification | Obligatoire pour accès aux données |
| NFR11 | Chiffrement transit | HTTPS/TLS pour toutes les communications API |
| NFR12 | Chiffrement stockage | Données sensibles chiffrées au repos (device + cloud) |
| NFR13 | Isolation données | Un utilisateur ne peut jamais accéder aux données d'un autre |
| NFR14 | RGPD | Droit d'accès, rectification, suppression des données personnelles |

### Scalability (Préparation)

| NFR | Critère | Cible MVP |
|-----|---------|-----------|
| NFR15 | Capacité utilisateurs | Architecture prête pour 100+ utilisateurs sans refonte |
| NFR16 | Stockage par utilisateur | Pas de limite artificielle MVP, monitoring usage

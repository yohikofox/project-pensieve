# Synthèse Brainstorming Pensine

**Date :** 9 janvier 2026
**Facilitateur :** Mary (Business Analyst)
**Participant :** yohikofox

---

## 1. Validation du Problème

### Le problème identifié

> Les idées naissent dans le chaos de la vie (conversations, rue, moments inopportuns), l'écrit est une friction cognitive qui tue le momentum, et même quand on note, on se retrouve avec un "cimetière de post-it" inexploité.

### Le vrai problème racine

> Le ratio effort/valeur est inversé. Capturer, organiser et traiter demande trop d'énergie AVANT de savoir si l'idée vaut quelque chose.

### Insights clés

- **"Fire and forget"** = le mode naturel de capture
- **L'oral capture tout** (contexte, intention, émotion) vs l'écrit (3 mots cryptiques)
- **1-2h/jour pour traiter les post-it** = inacceptable → l'IA doit faire ce travail
- **La récurrence** prouve la valeur mais n'empêche pas la perte
- Tu as **plusieurs fois la même idée** sans jamais la concrétiser

### Techniques utilisées

1. **Five Whys** — Descendre à la racine du problème
2. **Assumption Reversal** — Challenger les hypothèses fondamentales
3. **Shadow Work Mining** — Révéler les angles morts et risques cachés

### Hypothèses validées

| Hypothèse | Statut | Évolution |
|-----------|--------|-----------|
| Les gens ont des idées précieuses | ✅ Validée | Pour la cible (créatifs, entrepreneurs) |
| Les idées se perdent | ✅ Renforcée | Même les bonnes, même les récurrentes |
| Idée perdue = opportunité perdue | 🔄 Transformée | On ne sait pas → Pensine permet de mesurer |
| La voix est idéale | 🔄 Ajustée | Voix = privilégiée, mais multimodal nécessaire |
| L'IA distingue signal/bruit | 🔄 Recadrée | L'IA pré-mâche, l'humain tranche |
| Les gens veulent revoir | ✅ Validée avec condition | Si coût quasi-nul (20 sec vs 2h) |
| Récurrence = importance | 🔄 Recadrée | UN signal parmi d'autres, pas absolu |

**Statut : ✅ PROBLÈME VALIDÉ**

---

## 2. Définition du MVP

### Hypothèse centrale

> "Si je peux capturer mes idées sans friction (voix) et les retrouver pré-digérées (IA), alors je vais réellement les exploiter au lieu de les oublier."

### Features MVP

| # | Feature | Priorité |
|---|---------|----------|
| 1 | Capture audio en 1 tap | MVP |
| 2 | Transcription automatique | MVP |
| 3 | Résumé IA | MVP |
| 4 | Extraction d'idées / highlights | MVP |
| 5 | Capture texte | MVP |
| 6 | Capture URL | MVP |
| 7 | Dashboard idées chaudes/froides | V2 |
| 8 | Capture photo | V2 |
| 9 | Détection de récurrence | V2 |
| 10 | Enrichissement a posteriori | V2 |
| 11 | Retour aux sources (audio) | LATER |
| 12 | Tags / catégories manuelles | LATER |
| 13 | Notifications / rappels | LATER |
| 14 | Export | LATER |

### Critères de succès

| Niveau | Critère |
|--------|---------|
| 🥉 Base | J'arrête les post-it |
| 🥈 Cœur | Plus de sentiment "merde c'était quoi déjà" |
| 🥇 Cerise | 1-2 idées actionnables par mois |

---

## 3. Modèle Économique

### Modèle choisi : Freemium

### Grille tarifaire

| | 🆓 Free | 🌱 Starter (10€) | 🚀 Pro (20€) | 👑 Ultimate (100€) |
|--|---------|------------------|--------------|-------------------|
| **Captures/jour** | 3 | 10 | 30 | Illimité |
| **Audio max** | 1 min | 2 min | 5 min | 10 min |
| **Texte max** | 3000 car. | 5000 car. | 10 000 car. | Illimité |
| **Digestions/mois** | 10 | 50 | 200 | Illimité |
| **Capture URL** | ❌ | ❌ | ✅ | ✅ |
| **Historique** | 30 jours | 6 mois | Illimité | Illimité |
| **Transcription** | On-device | On-device | Cloud | Cloud (prioritaire) |
| **Détection récurrence** | ❌ | ❌ | ✅ | ✅ |
| **Plan d'action** | ❌ | ❌ | ✅ | ✅ |
| **Export** | ❌ | ❌ | ✅ | ✅ |
| **API access** | ❌ | ❌ | ❌ | ✅ |
| **Support** | Ticketing | Ticketing | Prioritaire | Prioritaire |

### Plan Custom (sur devis)

- Support dédié
- Consulting IA (élaboration business plan)
- Mise en relation investisseurs
- Volume custom / On-premise

### Triggers de conversion Free → Premium

| Limite | Douleur ressentie |
|--------|-------------------|
| 3 captures/jour | "J'ai encore des idées et je peux plus capturer !" |
| 10 digestions/mois | "J'ai des captures non digérées" |
| Pas d'URL | "Je veux sauver cet article mais je peux pas" |
| Historique 30j | "Mes vieilles idées vont disparaître !" |

---

## 4. Architecture Technique

### Stack technique

| Composant | Choix |
|-----------|-------|
| **Client Mobile** | React Native |
| **Backend API** | Node.js (Fastify) |
| **BDD** | PostgreSQL |
| **Pattern** | CQRS + Event Sourcing + Ports & Adapters |
| **Message Broker** | Redis Streams (abstrait via interface) |
| **Transcription Free** | Whisper on-device ou self-hosted CPU |
| **Transcription Payant** | Whisper API OpenAI |
| **IA Digestion** | OpenAI GPT-4o-mini |
| **Conteneurisation** | Docker / Docker Compose |
| **Orchestration cloud** | Kubernetes (Scaleway Kapsule) |
| **IaC** | Terraform |

### Architecture CQRS + Event Sourcing

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                 │
│   📱 Mobile App    🖥️ Web Dashboard    🔌 API (Ultimate)        │
└──────────────────────────┬──────────────────────────────────────┘
                           │
          ┌────────────────┴────────────────┐
          ▼                                 ▼
┌─────────────────────┐          ┌─────────────────────┐
│     COMMANDS        │          │      QUERIES        │
│   (Write Side)      │          │    (Read Side)      │
└──────────┬──────────┘          └──────────┬──────────┘
           │                                ▲
           ▼                                │
┌─────────────────────┐                     │
│    EVENT STORE      │──── Projections ────┘
│   (Source of Truth) │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐          ┌─────────────────────┐
│   EVENT STORE DB    │          │   READ MODEL DB     │
│   (Append-only)     │ ──proj─▶ │   (Optimisé query)  │
│   PostgreSQL        │          │   PostgreSQL        │
└─────────────────────┘          └─────────────────────┘
```

### Principes architecturaux

- **Event Sourcing obligatoire** — RGPD / données personnelles (droit à l'oubli, droit d'accès)
- **CQRS** — Isoler les reads pour l'API Ultimate, protéger le core
- **Ports & Adapters dès le MVP** — Switch Redis ↔ RabbitMQ via variable d'environnement
- **Transcription on-device pour Free** — Coût serveur = 0
- **Code métier découplé** — Aucune dépendance directe à Redis/RabbitMQ dans le code

### Infra par phase

| Phase | Infra | Coût |
|-------|-------|------|
| **MVP (perso)** | Homelab Docker | ~0€ |
| **Premium (cloud)** | Scaleway Kapsule (Terraform) | ~20-40€/mois |
| **Scale** | Nodes additionnels K8s | Variable |

### Structure de dossiers

```
src/
├── domain/                    # Logique métier pure
│   ├── events/
│   ├── commands/
│   └── models/
├── ports/                     # Interfaces (contrats)
│   ├── EventPublisher.ts
│   ├── EventSubscriber.ts
│   ├── EventStore.ts
│   └── Repositories.ts
├── adapters/                  # Implémentations
│   ├── messaging/
│   │   ├── RedisStreamsAdapter.ts
│   │   ├── RabbitMQAdapter.ts
│   │   └── InMemoryAdapter.ts
│   ├── persistence/
│   └── external/
├── application/               # Use cases / Command & Query handlers
│   ├── commands/
│   └── queries/
├── infrastructure/            # Config, DI, Bootstrap
└── api/                       # Controllers / Routes
```

---

## 5. Expérience Utilisateur

### Parcours Capture

```
┌─────────────────────────────────────┐
│         ÉCRAN D'ACCUEIL             │
│                                     │
│     ┌─────────────────────┐         │
│     │    Dernières        │         │
│     │    captures         │         │
│     └─────────────────────┘         │
│                                     │
│            ┌───────┐                │
│            │  🎤   │  ← GROS bouton │
│            └───────┘                │
│                                     │
│     [Aa]              [🔗]          │
│     Texte             URL           │
└─────────────────────────────────────┘
```

### Écran d'enregistrement

```
┌─────────────────────────────────────┐
│         ENREGISTREMENT              │
│                                     │
│              🔴 REC                 │
│                                     │
│         ◉ 00:34 / 1:00              │
│                                     │
│     ∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿∿             │
│     (waveform live)                 │
│                                     │
│            ┌───────┐                │
│            │  ⏹️   │                │
│            └───────┘                │
│          Tap = stop                 │
│                                     │
│     (auto-stop à la limite)         │
└─────────────────────────────────────┘
```

### Vue Liste (Cards)

```
┌─────────────────────────────────────┐
│  ┌─────────────────────────────────┐│
│  │ 📅 Aujourd'hui, 14:32 • 🎤 47s ││
│  │                                 ││
│  │ "Idée de pricing freemium pour ││
│  │  une app SaaS avec limite..."  ││
│  │                                 ││
│  │ 💡 Freemium • Limite capture   ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
```

### Vue Capture individuelle

Ordre d'affichage :
1. 📅 Date / Contexte
2. 📝 Transcript complet
3. ✨ Résumé IA
4. 💡 Idées extraites / Highlights
5. 🎧 Audio original

### Card non digérée (Free)

- Card visuellement grisée/locked
- Transcript visible
- Pas de résumé ni d'idées
- CTA : "🔒 Digérer cette capture" + compteur restant

---

## 6. Stratégie Go-to-market

### Approche : Stealth avec jalon de confiance

### Philosophie

> "Je veux avoir une longueur d'avance qui fait que même quand je commence à en parler, je suis le seul à pouvoir avancer aussi vite."

### Jalon de confiance pour montrer

- ✅ J'appuie, ça enregistre
- ✅ L'audio est transcrit
- ✅ L'IA sort un résumé + idées
- ✅ Je peux revoir ma capture

**Ce n'est PAS un PoC jetable** — c'est le Jalon 4 du MVP incrémental.

### Séquence de lancement

| Phase | Action |
|-------|--------|
| **Build (stealth)** | Dev incrémental, seul |
| **Jalon 4 atteint** | Capture → Transcript → Digest → Consult |
| **Montrer** | Réseau proche (5-10 personnes) |
| **Itérer** | Feedback → amélioration |
| **Ouvrir** | Beta privée élargie |
| **Launch** | Product Hunt / communautés |

### Jalons de développement

| Jalon | Features | Montrable ? |
|-------|----------|-------------|
| J1 | Capture audio + stockage local | ❌ |
| J2 | + Transcription | ❌ |
| J3 | + Digestion IA (résumé + idées) | ⚠️ Presque |
| **J4** | + Consultation (liste + détail) | ✅ **OUI** |
| J5 | + Capture texte | ✅ |
| J6 | + Capture URL (Premium) | ✅ |
| J7 | + Auth + Multi-user | ✅ |

---

## 7. Analyse Concurrentielle

**Statut : ✅ RÉALISÉE**

### Concurrents directs (Voice-to-Idea)

#### Voicenotes (concurrent principal)
- **Concept :** Brain dump vocal → transcription → AI qui connecte les idées
- **Features :** Transcription, Ask AI (chat avec notes), auto-tagging, résumés
- **Pricing :** ~$14.99/mois ou $99.99/an
- **Forces :** UX simple, "second brain" vocal
- **Faiblesses :** Nécessite internet, pas d'offline, pas de focus "produit"

#### AudioPen
- **Concept :** Pensées floues → texte clair et résumé
- **Features :** Transcription, nettoyage filler words, Super Summary
- **Pricing :** Free (3 min, 10 notes) → $75/an ou $29 lifetime
- **Forces :** Simple, rapide
- **Faiblesses :** Pas de connexion d'idées, pas de second brain

#### TalkNotes
- **Concept :** Capture vocale → formats variés (meeting notes, to-do, brainstorm)
- **Features :** Styles prédéfinis, multi-format output
- **Faiblesses :** Moins d'intelligence "émergente"

#### Cleft Notes
- **Concept :** Transcription locale → notes structurées → sync Obsidian
- **Features :** On-device transcription, export .md
- **Forces :** Privacy (local), intégration Obsidian
- **Faiblesses :** Moins d'AI "intelligence"

### Concurrents indirects (Second Brain / Note-taking)

#### Mem.ai
- **Concept :** Notes + AI qui organise et connecte automatiquement
- **Pricing :** Free (25 notes/mois) → $12/mois Pro
- **Faiblesses :** Plus orienté "notes" que "idées émergentes"

#### Rabrain
- **Concept :** Bookmarks + voice memos → second brain
- **Forces :** Combine URL + Voice (similaire à Pensine)
- **Faiblesses :** Moins connu, moins mature

#### Otter.ai / Notion AI Meeting Notes
- **Concept :** Transcription réunions, collaboration équipe
- **Faiblesses :** Pas pour l'idéation rapide, orienté meetings uniquement

### Grille comparative

| App | Capture rapide | Digestion IA | Idées émergentes | Récurrence | On-device | Focus produit |
|-----|----------------|--------------|------------------|------------|-----------|---------------|
| **Pensine** | ✅ | ✅ | ✅ | ✅ (V2) | ✅ | ✅ **UNIQUE** |
| Voicenotes | ✅ | ✅ | ⚠️ Ask AI | ⚠️ Partiel | ❌ | ❌ |
| AudioPen | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Mem.ai | ⚠️ | ✅ | ⚠️ | ❌ | ❌ | ❌ |
| Otter.ai | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |

---

## 8. Différenciation Clé de Pensine

### Le positionnement unique

> **Pensine n'est pas une app de notes. C'est un incubateur personnel qui transforme tes frustrations quotidiennes en opportunités business.**

### Pensine vs Voicenotes — La différence fondamentale

| | **Voicenotes** | **Pensine** |
|--|----------------|-------------|
| **Ce qu'ils font** | Capture → Transcription → Notes organisées | Capture → Transcription → **Genèse produit** |
| **Leur finalité** | "Retrouve tes pensées" | "Trouve ton prochain business" |
| **Leur métaphore** | Second brain (mémoire) | **Incubateur d'idées** |
| **Leur user** | Quelqu'un qui veut se souvenir | **Quelqu'un qui veut créer** |
| **Leur output** | Notes bien rangées | **Opportunités SaaS identifiées** |

### La vision Pensine

```
VOICENOTES:
  Pensée → Transcription → Tags → Notes organisées → FIN

PENSINE:
  Pensée → Transcription → Analyse → Patterns → Récurrence
     → "Tu te plains souvent de X"
     → "C'est peut-être un problème à résoudre"
     → "Voici une ébauche de business model"
     → "Voici un plan d'action pour valider"
     → DÉBUT D'UN PRODUIT
```

### La roadmap qui différencie (V2+)

| Feature | Voicenotes | Pensine |
|---------|------------|---------|
| Transcription | ✅ | ✅ |
| Résumé | ✅ | ✅ |
| Tags | ✅ | ✅ |
| Détection de récurrence | ⚠️ Partiel | ✅ **"Tu parles de ça souvent"** |
| Qualification d'idée | ❌ | ✅ **"Idée chaude / froide"** |
| Analyse problème/solution | ❌ | ✅ **"C'est un pain point récurrent"** |
| Ébauche business model | ❌ | ✅ **IA Analyst (style "Mary")** |
| Brainstorming guidé | ❌ | ✅ **Workflows de validation** |
| Mise en relation investisseurs | ❌ | ✅ **Plan Custom** |

### Scoring d'idées — Critères

| Critère | Signal | Poids |
|---------|--------|-------|
| **Récurrence** | Tu en parles X fois en Y jours | Élevé |
| **Émotion** | Frustration, enthousiasme détecté (ton/mots) | Élevé |
| **Mots-clés business** | "il faudrait", "pourquoi personne", "je paierais pour" | Élevé |
| **Contexte géoloc** | Toujours au même endroit ? (bureau, trajet, client) | Moyen |
| **Lien URL associé** | Tu as sauvé des articles sur le sujet | Moyen |
| **Heure/Moment** | Pattern temporel (matin, après réunion) | Faible |

**Output scoring :**
- 🔥 **Idée chaude** — Récurrence + Émotion forte + Mots-clés business
- 🌡️ **Idée tiède** — Quelques signaux mais pas assez
- ❄️ **Idée froide** — Mention unique, pas de pattern

### Messages marketing

| Voicenotes dit | Pensine dit |
|----------------|-------------|
| "Capture tes pensées" | "Découvre tes futures idées de business" |
| "Retrouve tes notes" | "Révèle ce que tu essaies déjà de résoudre" |
| "Organise ta mémoire" | "Fais germer des produits" |

### Taglines proposées

- **"Pensine — From thoughts to products"**
- **"Pensine — Your ideas, productized"**
- **"Pensine — Stop losing business ideas"**
- **"L'incubateur personnel qui transforme tes pensées en produits"**

---

## Actions identifiées

| Priorité | Action | Statut |
|----------|--------|--------|
| ✅ Fait | Analyse concurrentielle | Réalisée |
| 🟠 Moyenne | Définir les jalons de dev détaillés | À faire |
| 🟢 Prochaine | Commencer le dev (Jalon 1 : Capture audio) | À faire |

---

## Annexes

### Cible utilisateur

| Qui | Caractéristiques |
|-----|------------------|
| Créatifs | Idées fréquentes, besoin de capturer vite |
| Entrepreneurs / Indie Hackers | Cherchent des opportunités produit |
| Consultants / Leaders | Pensée réflexive, externaliser la mémoire |
| Chercheurs / Designers | Connexions d'idées dispersées |

**PAS la cible :** Madame Michu et ses carottes 🥕

### Tagline

> **Pensine**
> *Your thoughts, digested. Your ideas, revealed.*

### Références

- Nom inspiré de la Pensine de Dumbledore (Harry Potter)
- Concept : mémoire augmentée qui ne stocke pas seulement, mais digère et révèle

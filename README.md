dev claude --resume ef474739-dbce-48cb-8f56-f0db2bf87980  
admin claude --resume 82c8e258-a544-4cd2-9114-4995271694ca                                   
# Pensine - Documentation & Planning

[![CI](https://github.com/yohikofox/pensine/actions/workflows/ci.yml/badge.svg)](https://github.com/yohikofox/pensine/actions/workflows/ci.yml)
[![Backend Coverage](https://codecov.io/gh/yohikofox/pensine/branch/main/graph/badge.svg?flag=backend)](https://codecov.io/gh/yohikofox/pensine)

> **Pensine** : Votre jardin d'idées personnel. Capturez, transcrivez et digérez vos pensées avec l'aide de l'IA.

---

## 📖 À Propos

Ce repository contient **toute la documentation de planification et de conception** du projet Pensine, une application mobile offline-first permettant de capturer des pensées audio/texte, les transcrire automatiquement, et les enrichir avec de l'IA pour extraire des insights et actions.

### 🔗 Repositories Liés

- **📚 Documentation (ce repo)** : [project-pensieve](https://github.com/yohikofox/project-pensieve)
- **💻 Code Applicatif** : [pensieve](https://github.com/yohikofox/pensieve) *(à venir)*

---

## 🎯 Vision du Projet

Pensine est une application mobile qui transforme vos pensées fugaces en insights actionnables :

- **🎤 Capture 1-Tap** : Enregistrez vos idées en un seul geste
- **📝 Transcription Locale** : Whisper on-device pour transcription offline
- **🤖 Digestion IA** : GPT-4o-mini extrait résumés, idées clés et actions
- **✅ Gestion d'Actions** : Todos automatiquement détectées et organisées
- **🔄 Offline-First** : 100% fonctionnel sans réseau, sync automatique
- **🌱 Métaphore "Jardin d'Idées"** : Interface contemplative avec Liquid Glass design

---

## 📁 Structure du Repository

```
_bmad-output/
├── planning-artifacts/          # Artefacts de planification
│   ├── prd.md                  # Product Requirements Document (31 FRs, 16 NFRs)
│   ├── architecture.md         # Architecture technique (DDD, 8 Bounded Contexts)
│   ├── ux-design-specification.md  # Spécifications UX/UI complètes
│   ├── epics.md                # Breakdown en 6 epics, 27 stories
│   ├── implementation-readiness-report-2026-01-19.md  # Rapport IR (93% confiance)
│   └── bmm-workflow-status.yaml    # Suivi progression méthodologie
│
├── implementation-artifacts/    # Artefacts d'implémentation
│   └── sprint-status.yaml      # Suivi progression sprint (6 epics, 27 stories)
│
└── analysis/                    # Recherche et brainstorming
    ├── brainstorming-session-2026-01-09.md
    └── product-brief-pensine-2026-01-09.md
```

---

## 🧭 Navigation de la Documentation

### 1️⃣ **Découverte du Projet**
Commencez par :
- [`product-brief-pensine-2026-01-09.md`](_bmad-output/analysis/product-brief-pensine-2026-01-09.md) - Vision et contexte
- [`prd.md`](_bmad-output/planning-artifacts/prd.md) - Exigences détaillées

### 2️⃣ **Spécifications Techniques**
Explorez :
- [`architecture.md`](_bmad-output/planning-artifacts/architecture.md) - Architecture système (DDD, tech stack)
- [`ux-design-specification.md`](_bmad-output/planning-artifacts/ux-design-specification.md) - Design UX/UI complet

### 3️⃣ **Implémentation**
Prêt à développer :
- [`epics.md`](_bmad-output/planning-artifacts/epics.md) - 27 stories détaillées avec acceptance criteria
- [`sprint-status.yaml`](_bmad-output/implementation-artifacts/sprint-status.yaml) - Suivi de progression

### 4️⃣ **Validation**
Vérification de qualité :
- [`implementation-readiness-report-2026-01-19.md`](_bmad-output/planning-artifacts/implementation-readiness-report-2026-01-19.md) - Audit complet

---

## 🛠️ Stack Technique (Résumé)

### Mobile
- **React Native** + Expo (custom dev client)
- **TypeScript** (strict mode)
- **@op-engineering/op-sqlite** (offline-first, synchronous queries)
- **Whisper.rn** (transcription on-device ~500 Mo)

### Backend
- **NestJS** (TypeScript)
- **PostgreSQL** (base de données)
- **RabbitMQ** (message broker)
- **Redis** (cache uniquement)
- **GPT-4o-mini** (digestion IA)

### Infrastructure
- Docker Compose pour dev local (selfhosted homelab)

---

## 📊 État d'Avancement

| Phase                 | Statut      | Fichiers Clés                     |
|-----------------------|-------------|-----------------------------------|
| **1. Analysis**       | ✅ Complété  | Product Brief, Brainstorming      |
| **2. Planning**       | ✅ Complété  | PRD, UX Design                    |
| **3. Solutioning**    | ✅ Complété  | Architecture, Epics, IR Report    |
| **4. Implementation** | 🚧 En cours | Sprint Status (0/27 stories done) |

**Verdict Implementation Readiness :** ✅ **READY FOR IMPLEMENTATION** (93% confiance)

---

## 🎨 Architecture Highlights

### Domain-Driven Design (DDD)
8 Bounded Contexts identifiés :

**Core Domains :**
- 🧠 **Knowledge** : Thoughts, Ideas, insights
- 🎯 **Opportunity** : Discovery, concordance

**Supporting Domains :**
- 🎤 **Capture** : Audio/text recording
- 📝 **Normalization** : Transcription (Whisper)
- ✅ **Action** : Todos, tasks

**Generic Subdomains :**
- 👤 **Identity** : Users, auth
- 🔔 **Notification** : Push, local

**Infrastructure :**
- 🔄 **Sync** : OP-SQLite sync protocol

---

## 🧪 Épics & Stories

| Epic       | Stories | Description                          |
|------------|---------|--------------------------------------|
| **Epic 1** | 5       | Foundation & Authentification        |
| **Epic 2** | 6       | Capture & Transcription              |
| **Epic 3** | 4       | Consultation & Navigation            |
| **Epic 4** | 4       | Digestion IA & Extraction d'Insights |
| **Epic 5** | 4       | Gestion des Actions (Tab Actions)    |
| **Epic 6** | 4       | Synchronisation Multi-Device         |
| **Total**  | **27**  | MVP complet                          |

---

## 🚀 Méthodologie

Ce projet suit la **méthodologie BMAD (BMad Agile Development)** :

### Phases Complétées :
1. ✅ **Analysis** (optionnel) : Brainstorming, product brief
2. ✅ **Planning** : PRD avec FRs/NFRs, UX Design
3. ✅ **Solutioning** : Architecture (DDD), Epics & Stories, Implementation Readiness

### Phase En Cours :
4. 🚧 **Implementation** : Sprint-based development avec stories ready-for-dev

### Outils BMAD Utilisés :
- Product Manager (PM) pour PRD et Epics
- Architect pour Architecture et IR
- UX Designer pour specifications UX/UI
- Scrum Master (SM) pour Sprint Planning

---

## 📝 Prochaines Étapes

1. **Story 1.1** : Project Foundation & Infrastructure Setup
2. Initialiser le repository [`pensieve`](https://github.com/yohikofox/pensieve) avec la structure technique
3. Développer les stories Epic 1 (Foundation) séquentiellement

---

## 🤝 Contribution

Ce repository est la **source de vérité** pour toutes les décisions de conception et de planification. Toute modification des specs doit être documentée ici.

---

## 📄 Licence

*(À définir)*

---

**Créé avec ❤️ en suivant la méthodologie BMAD**

# noms des apps

- Pensine capture
- Omnio révèle et connecte
- Aloho débloque et exécute

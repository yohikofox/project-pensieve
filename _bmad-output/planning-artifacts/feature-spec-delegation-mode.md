# Feature Spec — Pensine Mode Délégation (Agentic Layer)

> **Statut** : Exploration complète — prêt pour transformation en Epic BMAD
> **Auteur** : Mary (Business Analyst) — session du 2026-02-25
> **Input** : Session de brainstorming avec yohikofox

---

## 1. Problème Utilisateur

Les utilisateurs de Pensine ont des pensées actionnables dans des contextes où interagir avec leur téléphone est **impossible ou dangereux** (conduite, sport, mains occupées). Aujourd'hui, Pensine capture et valorise les pensées — mais elle ne peut pas **agir en leur nom**.

### Gap identifié

```
État actuel :  Pensée → Capture → Transcription → Idées/Todos (passif)
État cible  :  Pensée → Capture → Transcription → Idées/Todos + Actions exécutées (actif)
```

---

## 2. Persona Primaire : Le Nomade Cognitif

**Profil** : Utilisateur à haute mobilité physique, forte activité cognitive, interaction écran contrainte.

**Contextes typiques** :
- En camion (mains occupées, regard sur la route — légalement interdit de toucher le téléphone)
- En course à pied (effort physique, ne veut pas s'arrêter)
- En réunion (discret, ne peut pas sortir son téléphone)
- En cuisine, bricolage, sport

**Besoins** :
- Capturer une intention sans friction
- Déléguer l'exécution à Pensine
- Valider ou rejeter plus tard, au moment opportun
- Ne pas être interrompu dans son activité

**Device représentatif** : Google Pixel 9 Pro / Pixel 10 Pro

---

## 3. Vision Produit

### 3.1 Mode Délégation

Pensine devient un **agent de confiance** capable d'exécuter des actions au nom de l'utilisateur, avec un modèle de confiance progressif.

**Actions ciblées** (révisées) :
- Calendrier : créer/modifier un rendez-vous
- Email : rédiger et envoyer (avec contrainte rollback spécifique)
- Rappels : créer un rappel
- Todos internes : ajouter à la liste Pensine (pas d'outil externe)
- Messaging : envoyer un message via Telegram bot

### 3.2 Modèle de Confiance Progressive

La relation entre l'utilisateur et Pensine évolue dans le temps :

```
Niveau 1 (défaut) — Validation obligatoire
  Toute action → stockée en BDD → notification → validation humaine → exécution
  Aucune action ne part sans validation explicite

Niveau 2 — Validation allégée
  Actions de type connu et validées régulièrement → délai de grâce (annulation possible)

Niveau 3 — Fire & Forget (opt-in explicite)
  Actions approuvées → exécution directe → notification post-exécution
  Exception : email (voir section 5)
```

**Règle fondamentale** : le niveau 1 est le défaut. La progression vers les niveaux supérieurs est **toujours à l'initiative de l'utilisateur**, jamais automatique.

**Principe d'intégrité** : il vaut mieux notifier une ambiguïté ou un échec que de laisser croire que tout s'est bien passé. La valeur première de Pensine — la confiance — prime sur la commodité.

### 3.3 Toute interaction reste une Capture

**Décision architecturale clé** : quel que soit le mode de déclenchement, toute délégation passe par le pipeline de capture Pensine.

```
Déclencheur (voix/geste/widget) → Audio capturé → Transcription (Whisper on-device)
→ IA digestion → Intent extrait → Action créée → Exécution (avec/sans validation)
```

Pensine reste **souveraine** sur la capture. Aucun tiers ne parse l'audio à la place de Pensine (sauf Option 1 — intégration assistant système).

### 3.4 Todos — Stratégie Interne

**Décision produit** : les todos restent **internes à Pensine**, sans intégration vers des outils tiers (Todoist, Jira, Linear, etc.).

Raisonnement : les todos sont un mécanisme de rétention. Exporter vers un outil externe crée une sortie de l'app et réduit l'engagement. Si une demande massive émerge, la question sera réexaminée avec des données.

```
Capture → Action extraite → Todo Pensine → Retour quotidien → Nouvelle capture
```

---

## 4. Architecture de Déclenchement — Les 4 Options

L'utilisateur choisit son mode de déclenchement dans les **Settings**. Toutes les options sont cumulables.

### Option 1 — Intégration Assistant Système ✅ Validée

**Principe** : L'assistant natif de l'OS (Siri / Google Assistant / Gemini) reçoit la commande et la transmet à Pensine en arrière-plan.

| | iOS (Siri) | Android (App Actions + Gemini) |
|-|-----------|-------------------------------|
| Formulation | *"Hey Siri, dis à Pensine de..."* | *"Hey Google, dis à Pensine de..."* |
| API | App Intents Framework | App Actions + `shortcuts.xml` |
| Ouverture app | Non | Non |
| NLP | Intents prédéfinis (strict) | Gemini sur Pixel = langage naturel riche |
| Batterie | 🟢 Nulle | 🟢 Nulle |
| Souveraineté | ⚠️ Partielle (audio transite par Apple/Google) | ⚠️ Partielle |
| Complexité dev | Moyenne | Moyenne |

**Contrainte** : L'utilisateur doit nommer explicitement l'app dans sa commande.

---

### Option 2 — Déclencheur Physique ✅ Validée

**Principe** : Un geste ou bouton physique active Pensine en mode capture sans ouvrir l'app.

#### iOS — Champions universels

| Déclencheur | Disponibilité | Friction |
|-------------|--------------|---------|
| Back Tap (2x / 3x) | iOS 14+, **tous iPhones** | Quasi nulle |
| Control Center shortcut | iOS 16+, **tous iPhones** | Très faible |
| Action Button | iPhone 15+ / 16 seulement | Quasi nulle |

**Recommandation iOS** : Back Tap (universel) + Action Button (iPhone 15+ en bonus)

#### Android — Stratégie par phases

**Phase 1 — Cible prioritaire : Pixel 9 Pro / Pixel 10 Pro**

| Déclencheur | Disponibilité | Notes |
|-------------|--------------|-------|
| Quick Tap (dos du téléphone ×2) | Pixel 5+ ✅ | Équivalent Back Tap iOS, natif Pixel |
| Quick Settings tile | Android 7+, tous ✅ | Swipe down + tap |

**Phase 2 — Autres Android (selon demande utilisateurs)**

| Déclencheur | Disponibilité |
|-------------|--------------|
| Double-press bouton power | Samsung One UI, Xiaomi, etc. |
| Lock screen shortcut | Android 14+, variable launcher |
| Boutons constructeur (Bixby, etc.) | Trop fragmenté — à cibler cas par cas |

**Batterie** : 🟢 Nulle — déclenchement ponctuel, aucun service permanent.

**Risque** : Back Tap / Quick Tap peut se déclencher accidentellement (en poche, posé sur table). Sensibilité configurable dans les settings.

---

### Option 3 — Widget ✅ Validée

**Principe** : Un widget sur l'écran d'accueil ou verrouillé expose un bouton micro et/ou un dashboard d'actions en attente.

#### Maquette widget

```
┌─────────────────────────────┐
│  🎤  Pensine                │
│  ─────────────────────────  │
│  [●  Capturer]              │
│                             │
│  ⏳ 3 actions en attente    │
│  💡 2 idées non traitées    │
└─────────────────────────────┘
```

| | iOS | Android |
|-|-----|---------|
| Tap widget | Ouvre app en mode capture (inévitable Apple) | Déclenche capture en arrière-plan ✅ |
| Mic dans widget | Non supporté nativement | ✅ Supporté |
| Tailles | S / M | S / M / L |
| Lock screen | iOS 16+ | Android 14+ |
| Dashboard actions | ✅ Via widget dynamique | ✅ Plus riche |

**Avantage fort** : le widget est aussi un **rappel passif** — l'utilisateur voit sans ouvrir l'app combien d'actions l'attendent.

**Batterie** : 🟢 Nulle.

**Complexité dev** : Faible à moyenne. Les widgets ont leur propre cycle de vie natif — surface à maintenir séparément.

---

### Option 4 — Wake Word "Hey Pensine" ✅ Validée (implémentation custom)

**Principe** : Détection permanente du wake word en basse consommation via DSP dédié.

#### Architecture DSP (seule option viable)

```
❌ Architecture naïve : Micro → CPU continu → 30-50% batterie/heure
✅ Architecture DSP   : Micro → DSP basse conso → wake word → CPU réveillé → Pensine
```

#### Stratégie d'implémentation — Solution Propriétaire

**Décision** : Picovoice Porcupine est **écarté** — le modèle de licence commerciale est incompatible avec la stratégie produit.

**Alternative retenue** : implémentation custom, inspirée des bonnes pratiques de Porcupine et des librairies de référence. Un fork de Porcupine a été étudié comme base d'inspiration.

| Critère | Valeur |
|---------|--------|
| Traitement | 100% on-device |
| Wake word custom | ✅ "Hey Pensine" |
| DSP Android | ✅ À exploiter via AlwaysOnHotwordDetectionService (Android 12+) |
| Licence | Propriétaire — zéro dépendance tierce commerciale |
| Compatibilité Expo | ⚠️ Nécessite Expo Development Build (native module) |

**Argument privacy fort** : l'audio ne quitte jamais le device avant détection du wake word. Différenciateur vs Siri/Google.

```
Hey Google  → audio → serveurs Google → traitement
Hey Pensine → audio → DSP local → détection → Pensine locale → 0 fuite
```

#### Impact batterie par plateforme

| Plateforme | Architecture | Consommation estimée |
|-----------|-------------|---------------------|
| Android Pixel 9/10 Pro (Tensor G4 DSP) | DSP natif | 🟢 ~1-2% / heure |
| Android autres | DSP standard | 🟢 ~2-3% / heure |
| iOS (solution custom sur CPU) | CPU, pas DSP accessible | 🟡 ~4-6% / heure |
| iOS naïf background audio | CPU | 🔴 ~30-50% / heure |

**iOS — Contrainte fondamentale** : Apple ne donne aucun accès public au DSP pour des wake words tiers. La solution custom tourne sur CPU en opt-in explicite avec avertissement batterie.

#### Parcours utilisateur idéal (Pixel 9 Pro, en camion)

```
[Mains sur le volant, téléphone en poche]
→ "Hey Pensine"
→ DSP Tensor G4 détecte → CPU réveillé → bip confirmation
→ "Cale un RDV avec Paul demain à 10h"
→ Capture créée, action extraite
→ Bip + vibration courte
→ [Fire and forget — validation plus tard]
```

---

## 5. Modèle de Validation des Actions

### Principe fondamental : intégrité avant commodité

> En cas d'ambiguïté (intent flou, référence incertaine), Pensine **ne tente jamais d'agir**. Elle met l'action en attente et notifie l'utilisateur. Faire croire que tout est OK quand ce n'est pas le cas trahit la valeur première de l'app.

### Flow par défaut — Niveau 1

```
Action capturée
    ↓
Stockée en BDD (état : en attente)
    ↓
Notification : "Action en attente : [résumé]"
    ↓
Tap → Layer de validation
    ↓
[✅ Valider] ou [❌ Rejeter] ou [✏️ Modifier]
    ↓
Exécution ou abandon
```

### Gestion d'ambiguïté

```
Intent ambigu détecté (ex: "Pierre" = Pierre Dupont ou Pierre Martin ?)
    ↓
Action stockée en BDD (état : ambiguë)
    ↓
Notification : "Pensine a besoin de précisions : [description de l'ambiguïté]"
    ↓
Utilisateur clarifie → action résolue → flow de validation normal
```

### Rollback selon niveau de confiance

| Niveau | Rollback | Mécanique |
|--------|----------|-----------|
| 1 — Validation obligatoire | N/A | Action ne part jamais sans validation |
| 2 — Délai de grâce | ✅ Pendant le délai | Notification + bouton "Annuler" countdown |
| 3 — Fire & Forget | Dépend de l'action | Voir tableau ci-dessous |

### Rollback par type d'action

| Action | Rollback possible ? | Mécanique |
|--------|-------------------|-----------|
| Créer un RDV calendrier | ✅ | Supprimer l'événement |
| Créer un rappel | ✅ | Annuler |
| Todo Pensine | ✅ | Supprimer la tâche |
| Email **brouillon** | ✅ | Supprimer le brouillon |
| Email **envoyé** | ⚠️ Fenêtre limitée | Delayed Send obligatoire (voir ci-dessous) |
| Message Telegram bot | N/A | Usage bot uniquement — pas de rollback nécessaire |

### Cas spécial Email — Delayed Send (exception au niveau 3)

**Règle** : l'email ne peut **jamais** passer en fire & forget total, même au niveau de confiance 3. Il bénéficie toujours d'une fenêtre d'annulation.

```
Email composé par Pensine
    ↓
Notification : "Email prêt — envoi dans [X] min"
    ↓
Fenêtre d'annulation active (countdown visible)
    ↓
[Annuler avant expiration] → brouillon sauvegardé
[Rien / Valider] → envoi automatique après délai
```

Délai configurable dans les settings : 5 / 10 / 30 minutes.

### Comportement Offline

- Les actions restent en queue locale (BDD)
- Exécutées automatiquement à la reconnexion
- Actions intrinsèquement online (email, Telegram) : attendent la connectivité
- Actions locales (Apple Calendar, Apple Reminders) : peuvent s'exécuter hors connexion

### Intégration dans l'UI existante

- **Écran Todos** → section "Actions en attente de validation" intégrée
- Permet le traitement groupé en fin de course/trajet
- Le widget affiche le compteur d'actions en attente

---

## 6. Intégrations Tierces

### Stratégie

**Principe** : Gmail est le seul fournisseur email intégré. ~55-65% des utilisateurs iOS ont un compte Gmail actif ; sur le profil knowledge worker ciblé par Pensine, la proportion est encore plus élevée. Apple Mail sera ajouté si la demande utilisateur le justifie.

### Intégrations validées — Priorité 1

| Service | Plateforme | Protocole | Notes |
|---------|-----------|-----------|-------|
| **Google Calendar** | iOS + Android | OAuth2 (scope partagé Gmail) | Même OAuth que Gmail — coût marginal |
| **Gmail** | iOS + Android | OAuth2 | Intégration email principale |
| **Apple Calendar** | iOS natif | EventKit (pas d'OAuth) | Couvre les utilisateurs non-Google sur iOS |
| **Apple Reminders** | iOS natif | EventKit (pas d'OAuth) | Même framework qu'Apple Calendar |

**Note** : Google Calendar + Gmail partagent le même OAuth2 — les implémenter ensemble coûte quasi le même effort que d'en implémenter un seul.

### Intégrations validées — Priorité 2

| Service | Plateforme | Notes |
|---------|-----------|-------|
| **Telegram** (bots uniquement) | iOS + Android | API gratuite, pas de rollback nécessaire, usage interaction bot |

### Intégrations écartées ou suspendues

| Service | Statut | Motif |
|---------|--------|-------|
| WhatsApp | ⏳ Post-monétisation | API Meta payante |
| Notion | ⏸️ Suspendu | Risque de cannibalisation de valeur Pensine |
| Google Tasks | ❌ Écarté | Trop léger, remplacé par todos internes |
| Todoist / Linear / Jira | ⏸️ Sur demande | Todos restent internes — réexamen si demande massive |
| Apple Notes | ⏸️ Sur demande | Pas de priorité native |
| Signal | ❌ Écarté | Pas d'API officielle, trop complexe |
| SMS | 🔗 Via infrastructure homelab | Brique externe réutilisable (voir section 8) |

---

## 7. Modèle de Settings — Vue d'ensemble

```yaml
Pensine — Paramètres de déclenchement

Mode de déclenchement :
  ☑ Option 1 — Assistant système (Siri / Google)
  ☑ Option 2 — Déclencheur physique
      iOS    : [Back Tap 2x ▾] [Control Center ▾] [Action Button ▾]
      Android: [Quick Tap ▾] [Quick Settings ▾]
  ☑ Option 3 — Widget
      Activer widget home screen : ON/OFF
      Activer widget lock screen : ON/OFF
      Contenu : [Actions en attente ▾] [Idées ▾] [Dernière capture ▾]
  ☑ Option 4 — Wake Word "Hey Pensine"
      Activer : ON/OFF
      ⚠️ Impact batterie estimé : ~2%/h (Android) / ~5%/h (iOS)
      Sensibilité : [Basse ←————→ Haute]
      Actif quand : [Toujours ▾] [Écran éteint seulement ▾] [App background ▾]
      Confirmation : [Bip sonore ▾] [Vibration ▾] [Les deux ▾] [Aucune ▾]

Validation des actions :
  Niveau de confiance : [1 — Validation obligatoire ▾]
  Délai d'annulation (si niveau 2) : [30 secondes ▾]

Email — Delayed Send :
  Délai avant envoi automatique : [10 minutes ▾]
```

---

## 8. Note Infrastructure — SMS Gateway (Projet Satellite)

La délégation SMS dans Pensine s'appuiera sur une **brique d'infrastructure homelab externe**, développée séparément.

**Architecture** :
```
Pensine backend → API SMS Gateway homelab → Device Android (SIM physique) → Réseau GSM → Destinataire
```

**Caractéristiques** :
- Device Android physique avec SIM(s) dédiée(s)
- SMS envoyés comme de vrais SMS humains (non filtrés spam)
- Zéro coût variable par message (hors forfait opérateur)
- Mutualisé entre projets (Pensine + autres)
- Évolution possible vers eSIM dédiée par projet si le besoin le justifie

**Contrat d'interface côté Pensine** : appel API vers le gateway — Pensine est agnostique de ce qu'il y a derrière.

---

## 9. Contraintes Techniques Identifiées

| Contrainte | Impact | Mitigation |
|-----------|--------|-----------|
| Wake word — implémentation custom | Effort R&D significatif | Fork Porcupine comme base d'étude |
| Expo Development Build obligatoire (Option 4) | Pas de test via Expo Go | Déjà en place si native modules utilisés |
| iOS — pas d'accès DSP tiers | Option 4 coûteuse en batterie sur iOS | Opt-in explicite + avertissement clair |
| Fragmentation Android (hors Pixel) | Option 2 à développer par phases | Phase 1 = Pixel 9/10 Pro uniquement |
| App Store — background audio | Risque de rejet si mal justifié | Communication claire dans description app |
| OAuth Google (Calendar + Gmail) | Gestion tokens, refresh, révocation | À spécifier dans stories dédiées |
| Delayed Send email | Gestion état côté BDD (en attente d'envoi) | File d'attente avec timestamp d'expiration |
| SMS Gateway homelab | Dépendance à l'infra personnelle (disponibilité) | Hors scope Pensine — géré côté gateway |

---

## 10. Roadmap Suggérée par Phases

### Phase A — Fondations (prérequis)
- [ ] Pipeline "action exécutable" dans la digestion IA
- [ ] Modèle BDD pour les actions (états : en attente / ambiguë / validée / rejetée / exécutée)
- [ ] Section "Actions en attente" dans l'écran Todos
- [ ] Settings — architecture du panneau de déclenchement

### Phase B — Intégrations calendrier & email
- [ ] Google Calendar + Gmail (OAuth2 mutualisé)
- [ ] Apple Calendar + Apple Reminders (EventKit natif iOS)
- [ ] Delayed Send pour email
- [ ] Gestion offline (queue locale)

### Phase C — Options de déclenchement faible friction
- [ ] Option 3 — Widget (iOS + Android)
- [ ] Option 2 — Back Tap iOS + Quick Tap Pixel 9/10 Pro
- [ ] Option 1 — App Intents (Siri) + App Actions (Google/Gemini)

### Phase D — Messaging
- [ ] Telegram bot integration

### Phase E — Option avancée
- [ ] Option 4 — Wake Word custom (Android DSP d'abord, iOS CPU opt-in)

### Phase F — Post-monétisation
- [ ] WhatsApp Business API
- [ ] Intégrations supplémentaires selon demande

---

*Document généré lors d'une session d'exploration — Mary (Business Analyst)*
*Date : 2026-02-25*
*Dernière mise à jour : 2026-02-25 — Décisions intégrations, rollback, wake word custom, SMS gateway*

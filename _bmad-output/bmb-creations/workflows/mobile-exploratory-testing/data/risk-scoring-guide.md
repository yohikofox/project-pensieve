# Risk Scoring Guide - Murat Framework

## Formule de Base

```
Risk Score = Impact × Probabilité
```

**Range**: 1-25 (Impact: 1-5, Probabilité: 1-5)

**Seuils de Décision**:
- **Risk Score > 12**: BLOCKER - Must fix avant ship
- **Risk Score 6-11**: Fix avant ship recommandé
- **Risk Score < 6**: Acceptable pour backlog

---

## Impact Scale (1-5)

### 5 - Critical Impact
**Affecte la core value proposition**

- Empêche l'utilisateur d'accomplir la tâche principale
- Dégrade significativement la raison d'être de la feature
- Impact sur business metrics critiques

**Exemples**:
- App de messaging: Impossible d'envoyer des messages
- App de photos: Photos ne se sauvegardent pas
- App bancaire: Solde affiché incorrect

### 4 - High Impact
**Affecte la satisfaction utilisateur majeure**

- Dégrade une fonctionnalité importante mais non critique
- Frustration utilisateur élevée
- Impact sur NPS probable

**Exemples**:
- App de messaging: Notifications en retard de 5 minutes
- App de photos: Filtre préféré ne marche pas
- App bancaire: Graphiques de dépenses incorrects

### 3 - Medium Impact
**Affecte le confort d'usage**

- Inconvénient perceptible mais pas bloquant
- L'utilisateur peut accomplir sa tâche avec effort supplémentaire
- Légère frustration

**Exemples**:
- App de messaging: Emoji picker lag
- App de photos: Preview basse résolution
- App bancaire: Historique prend 3s à charger

### 2 - Low Impact
**Affecte le polish**

- Problème cosmétique avec impact fonctionnel mineur
- La plupart des utilisateurs ne remarqueront pas
- Pas de frustration significative

**Exemples**:
- App de messaging: Bulle de chat mal alignée
- App de photos: Bouton de partage légèrement décalé
- App bancaire: Couleur de graphique peu contrastée

### 1 - Minimal Impact
**Cosmétique pur**

- Aucun impact fonctionnel
- Visible seulement si on cherche activement
- Zero impact sur satisfaction

**Exemples**:
- App de messaging: Typo dans tooltip
- App de photos: Icon 1px décalée
- App bancaire: Ombre de carte mal calibrée

---

## Probabilité Scale (1-5)

### 5 - Certain
**Tous les utilisateurs le verront**

- Reproduit 100% du temps
- Sur le happy path principal
- Aucune condition spéciale requise

**Exemples**:
- Bug sur écran d'accueil
- Problème au premier lancement
- Issue sur action la plus commune

### 4 - Likely
**>50% des utilisateurs**

- La majorité des users rencontreront ce bug
- Scénario fréquent mais pas universel
- Reproduit facilement mais nécessite action spécifique

**Exemples**:
- Bug lors du 3ème post consécutif
- Issue quand on utilise feature secondaire populaire
- Problème sur devices iOS (si 60% des users iOS)

### 3 - Possible
**10-50% des utilisateurs**

- Une portion significative mais pas majoritaire
- Requiert combinaison de conditions courantes
- Reproduction modérément difficile

**Exemples**:
- Bug avec images >2MB (20% des photos)
- Issue sur devices avec dark mode activé (30% users)
- Problème après 3 jours d'utilisation

### 2 - Unlikely
**<10% des utilisateurs**

- Scénario rare
- Requiert conditions spécifiques
- Difficile à reproduire

**Exemples**:
- Bug avec emoji spécifiques (🇫🇷 + 🎯 + texte)
- Issue sur iPhone 12 Mini uniquement (2% de base)
- Problème après 50+ items dans liste

### 1 - Rare
**Edge cases extrêmes**

- Scénario pratiquement impossible naturellement
- Nécessite séquence très spécifique d'actions
- Testeur a dû chercher activement ce bug

**Exemples**:
- Bug après 500 swipes rapides consécutifs
- Issue avec nom d'utilisateur de 200 caractères
- Problème si on rotate device 10 fois pendant chargement

---

## Exemples de Calculs

### Exemple 1: Bug de Synchronisation

**Description**: Les données ne se synchronisent pas si l'utilisateur passe de WiFi à 4G pendant le sync.

**Impact**: 4 (High)
- Perte de données temporaire
- Utilisateur doit re-sync manuellement
- Frustration élevée

**Probabilité**: 3 (Possible)
- 20-30% des users passent WiFi↔4G régulièrement
- Timing doit être exact pendant sync (rare)

**Risk Score**: 4 × 3 = **12**
**Catégorie**: BLOCKER (score = 12)

**Décision**: Fix avant ship (juste à la limite du blocker)

---

### Exemple 2: Alignement Bouton

**Description**: Le bouton "Save" est 3px trop bas par rapport au design.

**Impact**: 1 (Minimal)
- Pur cosmétique
- Zero impact fonctionnel

**Probabilité**: 5 (Certain)
- Tous les utilisateurs le voient
- Écran principal

**Risk Score**: 1 × 5 = **5**
**Catégorie**: Backlog

**Décision**: Ship avec story, fix dans backlog

---

### Exemple 3: Crash sur Action Rare

**Description**: L'app crashe si on tape 3 fois sur le logo pendant le chargement.

**Impact**: 5 (Critical)
- Crash complet
- Perte de session

**Probabilité**: 1 (Rare)
- Personne ne fait ça naturellement
- Découvert par testeur en exploration

**Risk Score**: 5 × 1 = **5**
**Catégorie**: Backlog

**Décision**: Documenter, mais pas bloquer pour cette edge case

---

### Exemple 4: Performance Lag

**Description**: Scroll lag perceptible (300ms) dans liste de 50+ items.

**Impact**: 3 (Medium)
- Confort d'usage affecté
- Feature utilisable mais pénible

**Probabilité**: 4 (Likely)
- 60% des users auront 50+ items après 2 semaines
- Reproduit facilement

**Risk Score**: 3 × 4 = **12**
**Catégorie**: BLOCKER (score = 12)

**Décision**: Fix avant ship (performance critique)

---

## Decision Matrix Visuelle

```
PROBABILITÉ
    ↓
5 | 5  10  15  20  25  ← BLOCKER
4 | 4   8  12  16  20  ← FIX BEFORE SHIP
3 | 3   6   9  12  15  ← BACKLOG
2 | 2   4   6   8  10
1 | 1   2   3   4   5
    ___________________
    1   2   3   4   5  → IMPACT

Légende:
■ Rouge (>12): BLOCKER
■ Orange (6-11): FIX BEFORE SHIP
■ Vert (<6): BACKLOG
```

---

## Guide de Discussion avec Testeur

### Questions pour Impact

1. "Si ce bug n'était pas corrigé, quel serait l'impact sur l'utilisateur ?"
2. "Est-ce que ça empêche d'accomplir la tâche, ou juste ça la rend pénible ?"
3. "Est-ce que ça affecte la core value proposition de la feature ?"
4. "Quel serait l'impact sur les metrics (NPS, retention, usage) ?"

### Questions pour Probabilité

1. "Combien d'utilisateurs vont rencontrer ce bug selon toi ?"
2. "Est-ce sur le happy path principal ou un scénario secondaire ?"
3. "Quelles conditions sont nécessaires pour reproduire ?"
4. "Est-ce que tu l'as trouvé naturellement ou en cherchant activement ?"

---

**Source**: Framework Murat (Party Mode Brainstorming)
**Usage**: Step 04 - Risk Scoring pour Bugs Level 1 & 2

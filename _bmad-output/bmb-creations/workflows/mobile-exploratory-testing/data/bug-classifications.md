# Bug Classifications - GLaDOS Framework

## Classification Levels

### Level 0 - Critical Failures (Bloquants Immédiats)

**Définition**: Bugs qui cassent la fonctionnalité core ou créent une expérience utilisateur catastrophique.

**Critères**:

🔴 **Crashes**
- L'application se termine inopinément
- Fermeture forcée sans message d'erreur
- L'app devient complètement inutilisable

🔴 **Data Loss (Perte de Données)**
- Données utilisateur perdues définitivement
- Impossibilité de sauvegarder ou récupérer les données
- Corruption de données

🔴 **Core Flow Broken (Flux Principal Cassé)**
- Le happy path principal est impossible à compléter
- L'utilisateur ne peut pas accomplir la tâche primaire
- Blocage complet de la fonctionnalité principale

🔴 **ANR (Application Not Responding)**
- Plus de 5 secondes de freeze complet
- L'app ne répond plus aux interactions
- Nécessite force quit

**Exemples**:
- Story "Ajout d'entrée": L'app crashe quand on tape sur "Sauvegarder"
- Story "Login": Impossible de se connecter même avec credentials valides
- Story "Search": L'app freeze pendant 10 secondes à chaque recherche

**Action**: BLOCKER - Story ne peut pas être DONE

---

### Level 1 - Major Issues (Story Pas Done)

**Définition**: Bugs qui dégradent significativement l'expérience mais n'empêchent pas l'utilisation.

**Critères**:

🟠 **Degraded UX**
- Feature utilisable mais pénible
- L'utilisateur doit travailler plus dur que nécessaire
- Expérience frustrante mais pas impossible

🟠 **Performance Issues**
- Lag perceptible (>200ms de latence)
- Temps de chargement excessifs
- Animations saccadées

🟠 **Inconsistent State**
- L'UI et les données se désynchronisent
- Affichage incorrect nécessitant refresh
- État de l'app devient incohérent

🟠 **Missing Error Handling**
- L'app plante sur edge cases prévisibles
- Pas de gestion des erreurs réseau
- Messages d'erreur absents ou cryptiques

**Exemples**:
- Story "Sync": La synchronisation prend 15 secondes au lieu de 2-3s
- Story "Edit": Après édition, l'UI n'affiche pas la nouvelle valeur sans refresh
- Story "Upload": Pas de feedback si l'upload échoue, l'utilisateur ne sait pas

**Action**: Évaluer avec Risk Scoring (Impact × Probabilité)
- Si Risk Score > 12: BLOCKER
- Si Risk Score 6-11: Fix avant ship recommandé
- Si Risk Score < 6: Acceptable pour backlog

---

### Level 2 - Minor Issues (Ship Mais Backlog)

**Définition**: Bugs cosmétiques ou edge cases qui n'impactent pas significativement l'utilisabilité.

**Critères**:

🟡 **UI Polish**
- Problèmes d'alignement ou spacing
- Couleurs légèrement off
- Micro-animations manquantes

🟡 **Helpful Errors**
- Messages d'erreur peu clairs mais non bloquants
- L'utilisateur peut comprendre avec effort
- Manque de guidance mais pas de blocage

🟡 **Edge Case Handling**
- Scénarios rares non gérés élégamment
- Cas limites qui fonctionnent mais de manière sous-optimale
- Situations exceptionnelles mal gérées

**Exemples**:
- Story "Profile": La photo de profil est 2px décalée vers la droite
- Story "Notifications": Le badge count affiche "10+" au lieu du nombre exact
- Story "Search": Recherche avec 200 caractères affiche un message tronqué

**Action**: Ship avec la story, créer tickets backlog pour corrections futures

---

## Decision Tree

```
Bug découvert
    ↓
Est-ce que l'app crashe/perd des données/casse le core flow/freeze >5s ?
    ↓ YES → Level 0 - Critical (BLOCKER automatique)
    ↓ NO
        ↓
Est-ce que l'UX est dégradée/performance impact/état incohérent/error handling manquant ?
    ↓ YES → Level 1 - Major (Calculer Risk Score)
    ↓ NO
        ↓
Est-ce un problème de polish/message/edge case ?
    ↓ YES → Level 2 - Minor (Ship avec backlog)
    ↓ NO
        ↓
Peut-être pas un bug? Vérifier avec testeur
```

---

## Exemples Concrets Par Story Type

### Story "Ajout d'Entrée"

- **Level 0**: Crash au save, données perdues après save réussi
- **Level 1**: Bouton save lag 500ms, champs ne se vident pas après save
- **Level 2**: Placeholder text mal aligné, message de succès peu clair

### Story "Authentification"

- **Level 0**: Impossible de login, crash au tap sur "Login", session perdue à chaque restart
- **Level 1**: Login prend 5 secondes, pas de message d'erreur si password faux
- **Level 2**: Bouton "Forgot Password" mal aligné, email validation trop stricte

### Story "Recherche"

- **Level 0**: App freeze pendant recherche, résultats perdus après navigation
- **Level 1**: Recherche lag >300ms par caractère, résultats désynchronisés
- **Level 2**: Placeholder couleur trop claire, message "0 résultats" peu explicite

---

**Source**: Framework GLaDOS (Party Mode Brainstorming)
**Usage**: Step 04 - Bug Classification

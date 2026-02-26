# Rapport des Warnings — Build iOS (2026-02-25)

> Analyse des warnings identifiés lors du build `expo run:ios` (APP_VARIANT=dev).
> Priorité = Effort × (1 / Gain) — plus le score est bas, plus la correction est rentable.

---

## Résumé Exécutif

| # | Warning | Occurrences | Effort | Gain | Priorité |
|---|---------|-------------|--------|------|----------|
| 1 | ld: linker — version mismatch (react-native-audio-api) | ~130+ | Élevé | Moyen | Basse |
| 2 | whisper.rn — module non listé dans "exports" | 1 | Faible | Faible | Moyenne |
| 3 | RecordingNotificationManager — non implémenté sur iOS | 1 | Moyen | Élevé | Haute |
| 4 | SafeAreaView déprécié | N | Moyen | Élevé | Haute |
| 5 | expo-background-fetch déprécié | 1 | Moyen | Élevé | Haute |
| 6 | Hermes run script — s'exécute à chaque build | 1 | Faible | Moyen | Haute |

---

## Détail des Warnings

---

### 1. ld: Linker — Version Mismatch (react-native-audio-api)

**Exemples de messages :**
```
ld: warning: ignoring file libopus.a: found architecture 'x86_64', file was built for archive which is not the architecture being linked
ld: warning: object file (libopus.a) was built for newer iOS Simulator version (18.5) than being linked (15.1)
ld: warning: object file (libssl.a) was built for newer iOS Simulator version (26.0) than being linked (15.1)
```

**Bibliothèques concernées :**
- `libopus.a`, `libopusfile.a` — compilées pour iOS-sim 18.5, cible de build : 15.1
- `libvorbis.a`, `libvorbisenc.a`, `libvorbisfile.a` — compilées pour iOS-sim 18.5, cible : 15.1
- `libssl.a` — compilée pour iOS-sim **26.0** (beta!), cible : 15.1

**Explication technique :**
`react-native-audio-api` distribue des bibliothèques statiques précompilées (`.a`). Ces binaires ont été compilés avec un SDK iOS Simulator plus récent que la cible minimale du projet (`IPHONEOS_DEPLOYMENT_TARGET = 15.1`). Le linker peut les utiliser mais émet des avertissements car il ne peut pas garantir la compatibilité descendante. Le cas de `libssl.a` à 26.0 est particulièrement préoccupant — il s'agit d'un SDK beta qui n'est pas encore sorti officiellement.

**Risques :**
- Crash potentiel sur appareils iOS 15.x si `libssl.a` (26.0) utilise des APIs absentes
- `libssl.a@26.0` suggère l'utilisation de Xcode beta / SDK pré-release par le mainteneur de la lib

**Effort :** Élevé
Nécessite soit de recompiler les `.a` depuis les sources avec la cible 15.1, soit d'attendre une mise à jour de `react-native-audio-api`, soit de relever le `IPHONEOS_DEPLOYMENT_TARGET` (mais cela exclut les utilisateurs iOS 15).

**Gain :** Moyen
L'app fonctionne, mais le risque de crash sur iOS 15 est réel, surtout avec `libssl@26.0`.

**Recommandation :** Ouvrir un issue sur `react-native-audio-api` avec les détails de version. Surveiller les releases. En attendant, envisager d'élever le `DEPLOYMENT_TARGET` à 16.0 si la base utilisateur le permet.

---

### 2. whisper.rn — Module Non Listé dans "exports"

**Message :**
```
WARN Attempted to import the module "whisper.rn" which is not listed in the "exports" of "whisper.rn".
Consider updating your imports to use one of the "exports" defined in the package or update "whisper.rn" to list your import in "exports".
```

**Explication technique :**
Node.js et les bundlers modernes (Metro inclus) respectent le champ `"exports"` de `package.json` (spec ESM Package Exports). Si un import ne correspond à aucune entrée dans ce champ, le bundler émet un warning. Cela indique que `whisper.rn` n'a pas mis à jour son `package.json` pour inclure les exports nécessaires selon la spec moderne.

**Risques :**
Faibles en pratique aujourd'hui — Metro continue de résoudre l'import. Cependant, dans les futures versions de Metro/Expo, ce comportement pourrait devenir une erreur bloquante.

**Effort :** Faible
C'est un problème du mainteneur de `whisper.rn`, pas du projet. Aucune action directe possible sans forker la bibliothèque.

**Gain :** Faible
Le warning est cosmétique pour l'instant.

**Recommandation :** Ouvrir un issue sur le repo `whisper.rn`. Surveiller les nouvelles versions. Le warning disparaîtra à la prochaine release corrigeant le `package.json`.

---

### 3. RecordingNotificationManager — Non Implémenté sur iOS

**Message :**
```
WARN RecordingNotificationManager is not implemented on iOS.
```

**Explication technique :**
`RecordingNotificationManager` est probablement un module natif de `react-native-audio-api` (ou similaire) qui gère les notifications de session d'enregistrement en cours. L'implémentation Android existe mais la version iOS est absente ou stub. Cela signifie que sur iOS, les notifications d'enregistrement actif (icône micro en barre de statut, contrôles depuis l'écran verrouillé) ne fonctionneront pas.

**Risques :**
- Mauvaise UX : l'utilisateur ne verra pas de notification pendant l'enregistrement audio
- Possible violation des guidelines Apple (enregistrement en arrière-plan sans indicateur visible)
- Rejet potentiel de l'App Store si l'audio background est utilisé sans notification appropriée

**Effort :** Moyen
Nécessite d'implémenter la couche iOS native en Swift/Obj-C si le projet maintient un fork, ou d'ouvrir une PR/issue upstream.

**Gain :** Élevé
Impact direct sur l'expérience utilisateur et la conformité App Store.

**Recommandation :** Priorité haute. Vérifier si cette feature est requise par les stories en cours (Epic 6/Capture). Si oui, implémenter ou trouver une alternative (AVAudioSession + UNUserNotificationCenter).

---

### 4. SafeAreaView Déprécié

**Message :**
```
WARN SafeAreaView has been deprecated. Please use SafeAreaView from react-native-safe-area-context instead.
```

**Explication technique :**
React Native a déprécié son composant `SafeAreaView` intégré au profit de celui fourni par `react-native-safe-area-context`. La version RN est moins précise (notamment sur les Dynamic Island, les échancrures variables selon les devices) et ne reçoit plus de mises à jour. `react-native-safe-area-context` est la solution recommandée par Expo et la communauté.

**Risques :**
- Mauvais rendu sur iPhone 14+ (Dynamic Island) ou futurs appareils avec nouvelles formes d'échancrure
- Sera une erreur bloquante dans une future version de React Native

**Effort :** Moyen
Nécessite de trouver toutes les occurrences de `import { SafeAreaView } from 'react-native'` dans la codebase et de les remplacer par `import { SafeAreaView } from 'react-native-safe-area-context'`. La bibliothèque est déjà installée (elle est une dépendance d'expo-router).

**Gain :** Élevé
Amélioration visuelle et compatibilité future garantie. Migration mécanique et sans risque.

**Recommandation :** Créer une story dédiée ou un task de refactoring. Recherche rapide :
```bash
grep -r "from 'react-native'" --include="*.tsx" | grep SafeAreaView
```

---

### 5. expo-background-fetch Déprécié

**Message :**
```
WARN expo-background-fetch: This library is deprecated. Please use expo-background-task instead.
```

**Explication technique :**
Expo a remplacé `expo-background-fetch` par `expo-background-task` (API plus propre, meilleure intégration avec BGTaskScheduler sur iOS et WorkManager sur Android). La version dépréciée continuera à fonctionner à court terme mais ne recevra plus de mises à jour de sécurité ou de bugs.

**Risques :**
- Incompatibilité future avec iOS (BGFetch API Apple potentiellement restreinte)
- Pas de nouvelles fonctionnalités ni corrections de bugs
- Sera possiblement supprimée dans Expo SDK 55 ou 56

**Effort :** Moyen
Migration API nécessaire. `expo-background-task` a une API différente. Nécessite de lire la documentation et d'adapter le code existant. L'API de base est similaire mais les hooks d'enregistrement ont changé.

**Gain :** Élevé
Pérennité du code, conformité avec les recommandations Expo SDK 54+.

**Recommandation :** Migrer avant Expo SDK 55. Consulter : https://docs.expo.dev/versions/latest/sdk/background-task/

---

### 6. Hermes Run Script — S'exécute à Chaque Build

**Message (dans les logs Xcode) :**
```
Run script build phase '[CP-User] [Hermes] Replace Hermes for the right configuration, if needed' will be run during every build because it does not specify any outputs. To address this issue, either add output dependencies to the script phase, or configure it to run in every build by unchecking "Based on dependency analysis" in the script phase.
```

**Explication technique :**
Xcode optimise les builds en sautant les phases de script dont les outputs n'ont pas changé (analyse de dépendances). Si un script ne déclare pas ses `Output Files`, Xcode ne peut pas déterminer s'il doit l'exécuter → il l'exécute à chaque build. C'est une configuration manquante dans le `Podfile`/`Pods` de hermes-engine.

**Risques :**
Pas de risque fonctionnel. Impact uniquement sur la durée des builds (quelques secondes supplémentaires à chaque build incrémental).

**Effort :** Faible
C'est un problème dans la configuration CocoaPods de hermes-engine, géré par React Native / Expo. Pas d'action directe dans le projet sans modifier les Pods.

**Gain :** Moyen
Amélioration mineure des temps de build (builds incrémentaux plus rapides).

**Recommandation :** Ignorer pour l'instant. Ce sera corrigé lors d'une mise à jour de React Native / Expo SDK. Surveiller les release notes.

---

## Priorisation des Actions

### Traiter maintenant (haute priorité)

| Action | Effort estimé | Bénéfice |
|--------|---------------|---------|
| **4. Migrer SafeAreaView** → react-native-safe-area-context | 2-4h | UX + compat future |
| **5. Migrer expo-background-fetch** → expo-background-task | 2-4h | Pérennité SDK |
| **3. Investiguer RecordingNotificationManager** (iOS) | 1j | Conformité App Store |

### Surveiller (moyen terme)

| Action | Quand |
|--------|-------|
| **2. whisper.rn exports** | Lors de la prochaine mise à jour whisper.rn |
| **6. Hermes run script** | Lors de la prochaine mise à jour Expo SDK |

### Accepter le risque (bas terme)

| Action | Condition |
|--------|-----------|
| **1. react-native-audio-api linker mismatch** | Ouvrir issue upstream. Migrer si iOS 15 devient critique |

---

*Rapport généré le 2026-02-25 — Build expo run:ios (APP_VARIANT=dev)*

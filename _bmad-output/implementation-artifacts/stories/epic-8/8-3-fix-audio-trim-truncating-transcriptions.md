# Story 8.3: Fix Audio Trim Truncating Transcriptions

Status: ready-for-dev

<!-- Validation optionnelle : run validate-create-story avant dev-story -->

## Story

As a **user**,
I want **my audio recordings to preserve all spoken content without aggressive automatic silence trimming**,
So that **my transcriptions are complete and include everything I said, especially at the end**.

**GitHub Issue:** [#16 - bug: La fin des transcriptions est tronquée à cause du trim audio post-enregistrement](https://github.com/yohikofox/pensieve/issues/16)

## Context

### État Actuel — Bug Core DÉJÀ Résolu

**IMPORTANT pour le Dev Agent :** La troncature des transcriptions signalée en issue #16 a été partiellement adressée par deux stories précédentes :

1. **Story 13.4 (commit `dddee24`, 2026-02-19)** : `trimSilence()` et `calculateRMS()` ont été **entièrement supprimées** de `AudioConversionService.ts` car elles coupaient les derniers mots. Les constantes `SILENCE_THRESHOLD`, `SILENCE_MARGIN_MS`, `SILENCE_MIN_DURATION_MS` ont également été retirées.

2. **Story 8.20 (commit `9f9a33aec`)** : `NativeTranscriptionEngine` écrasait `finalText` au lieu d'accumuler les segments — bug distinct causant une troncature pour audio >15s. Corrigé.

**Résultat :** Le trim silences n'est plus appliqué. Le pipeline audio actuel dans `AudioConversionService.ts` fait : Decode → Mono → ~~[TrimSilence-SUPPRIMÉ]~~ → Padding (optionnel) → CheckLong → BuildWAV → Write.

### Scope de cette Story 8.3

La story devient donc : **implémenter le trim audio comme feature OPTIONNELLE** (désactivée par défaut) avec une nouvelle implémentation conservative. Les tests `AudioConversionService.preprocessing.test.ts` existent déjà en état RED (méthodes absentes) — ils constituent le spec TDD de l'implémentation.

## Acceptance Criteria

### AC1: Vérifier l'état "Trim désactivé par défaut"

**Given** l'application en état courant post-Story-13.4
**When** j'enregistre un audio et lance la transcription
**Then** aucun trim de silences n'est appliqué automatiquement
**And** le contenu vocal complet (y compris la fin) est transcrit
**And** `AudioConversionService.ts` n'a pas de `trimSilence()` active dans le pipeline principal
**Note Dev :** ✅ **DÉJÀ CONFORME** — trim retiré en Story 13.4. Vérification uniquement, pas de modification requise.

### AC2: Ajouter le toggle "Supprimer les silences" dans les Settings

**Given** je suis dans les réglages de transcription
**When** je navigue vers la section audio de WhisperSettingsScreen (ou SettingsScreen si plus approprié)
**Then** je vois un toggle "Supprimer les silences automatiquement"
**And** le toggle est **OFF par défaut**
**And** le toggle est clairement labellisé avec une description : "Réduit la taille des fichiers audio en supprimant les silences. Peut affecter la complétude des transcriptions."

### AC3: Implémenter trimSilence conservative dans AudioConversionService

**Given** un développeur active le toggle de trim
**When** `convertToWhisperFormat()` est appelé avec `options.trimSilence = true`
**Then** la méthode `trimSilence()` est restaurée dans `AudioConversionService.ts` avec une implémentation conservative
**And** les constantes utilisées sont plus permissives que l'ancienne implémentation :
  - `SILENCE_THRESHOLD = 0.005` (au lieu de l'ancien 0.01 — seuil plus bas, moins agressif)
  - `SILENCE_MARGIN_MS = 200` (marge de sécurité avant/après silence détecté)
  - `SILENCE_MIN_DURATION_MS = 500` (silences courts ignorés)
**And** `calculateRMS()` est restaurée comme méthode privée auxiliaire
**And** la méthode est **uniquement appelée** si `options?.trimSilence === true` dans le pipeline
**And** si `trimSilence` non fourni ou false : `const trimmedBuffer = monoBuffer;` (comportement actuel maintenu)

```typescript
// Pipeline conditionnel (dans convertToWhisperFormat / convertToWhisperFormatWithPadding)
const trimmedBuffer = options?.trimSilence
  ? await this.trimSilence(monoBuffer)
  : monoBuffer;
```

### AC4: Persister le préférence dans settingsStore

**Given** je change le toggle "Supprimer les silences"
**When** je quitte et relance l'application
**Then** ma préférence est conservée
**And** `settingsStore.ts` expose :
  - `audioTrimEnabled: boolean` (default: `false`)
  - `setAudioTrimEnabled: (enabled: boolean) => void`
**And** la valeur est persistée via le mécanisme existant du settingsStore (zustand-persist ou AsyncStorage)
**And** la valeur est lue dans `TranscriptionWorker.ts` lors de l'appel à `convertToWhisperFormat()`

### AC5: Connecter le setting au pipeline de transcription

**Given** `audioTrimEnabled` est exposé dans settingsStore
**When** `TranscriptionWorker.ts` appelle `AudioConversionService`
**Then** le worker lit `audioTrimEnabled` depuis settingsStore et passe `{ trimSilence: audioTrimEnabled }` en options
**And** le comportement runtime est : trim OFF → audio complet (défaut) / trim ON → silences réduits
**And** aucune breaking change dans le contrat de l'API publique de `AudioConversionService` (paramètre `options` optionnel)

### AC6: Les tests existants passent (TDD Green)

**Given** les tests dans `AudioConversionService.preprocessing.test.ts` sont en état RED (méthodes absentes)
**When** j'implémente `trimSilence()` et `calculateRMS()`
**Then** tous les tests du fichier de preprocessing passent (état GREEN)
**And** `npm run test:unit` passe sans régression
**And** `npm run test:acceptance` passe sans régression
**And** `npm run test:architecture` passe (conformité ADR)

### AC7: Pas de régression transcription quand trim désactivé

**Given** le toggle "Supprimer les silences" est OFF (état par défaut)
**When** j'enregistre un audio avec du contenu vocal jusqu'à la fin
**Then** la transcription inclut les derniers mots prononcés
**And** le comportement est identique au comportement actuel (post-Story-13.4)
**And** aucun contenu vocal n'est perdu en comparaison à l'état "disabled"

## Tasks / Subtasks

### Task 1: Audit de conformité (AC1) — lecture seule
- [ ] Subtask 1.1 : Vérifier `AudioConversionService.ts` — confirmer que `trimSilence()` n'est pas dans le pipeline actif
- [ ] Subtask 1.2 : Confirmer que le pipeline est : Decode → Mono → `const trimmedBuffer = monoBuffer;` → CheckLong → BuildWAV → Write
- [ ] Subtask 1.3 : Documenter l'état actuel dans la section "Completion Notes" de cette story
- [ ] Subtask 1.4 : Si tout est conforme, passer à Task 2

### Task 2: Ajouter audioTrimEnabled dans settingsStore (AC4)
- [ ] Subtask 2.1 : Ouvrir `mobile/src/stores/settingsStore.ts`
- [ ] Subtask 2.2 : Ajouter dans l'interface/type du store :
  ```typescript
  audioTrimEnabled: boolean;
  setAudioTrimEnabled: (enabled: boolean) => void;
  ```
- [ ] Subtask 2.3 : Ajouter la valeur par défaut `audioTrimEnabled: false` dans l'initialisation du store
- [ ] Subtask 2.4 : Implémenter le setter `setAudioTrimEnabled`
- [ ] Subtask 2.5 : Vérifier que la persistance fonctionne (même mécanisme que les autres settings boolean du store)

### Task 3: Restaurer trimSilence dans AudioConversionService (AC3)
- [ ] Subtask 3.1 : Ouvrir `mobile/src/contexts/Normalization/services/AudioConversionService.ts`
- [ ] Subtask 3.2 : Ajouter les constantes (après les constantes existantes) :
  ```typescript
  private readonly SILENCE_THRESHOLD = 0.005;    // Seuil RMS (conservative < 0.01 ancien)
  private readonly SILENCE_MARGIN_MS = 200;       // Marge avant/après silence
  private readonly SILENCE_MIN_DURATION_MS = 500; // Durée min de silence à trimmer
  ```
- [ ] Subtask 3.3 : Restaurer `calculateRMS(samples: Float32Array, start: number, end: number): number` — méthode privée
- [ ] Subtask 3.4 : Restaurer `trimSilence(buffer: AudioBuffer): AudioBuffer` — méthode privée
  - Utiliser les tests `AudioConversionService.preprocessing.test.ts` comme spec de comportement
  - Implémenter un algorithme conservatif : ne trimmer QUE les silences longs aux extrémités, jamais le contenu vocal
- [ ] Subtask 3.5 : Modifier la signature de `convertToWhisperFormat()` pour accepter options :
  ```typescript
  async convertToWhisperFormat(
    filePath: string,
    options?: { trimSilence?: boolean }
  ): Promise<Result<string>>
  ```
- [ ] Subtask 3.6 : Modifier `convertToWhisperFormatWithPadding()` de même
- [ ] Subtask 3.7 : Dans le pipeline principal (là où `const trimmedBuffer = monoBuffer;` existe) :
  ```typescript
  const trimmedBuffer = options?.trimSilence
    ? this.trimSilence(monoBuffer)
    : monoBuffer;
  ```
- [ ] Subtask 3.8 : Vérifier que `trimSilence()` est SYNCHRONE (non-async) pour ne pas casser la chaîne Promise existante

### Task 4: Connecter le setting dans TranscriptionWorker (AC5)
- [ ] Subtask 4.1 : Ouvrir `mobile/src/contexts/Normalization/workers/TranscriptionWorker.ts`
- [ ] Subtask 4.2 : Localiser les appels à `audioConversionService.convertToWhisperFormat()` et `.convertToWhisperFormatWithPadding()`
- [ ] Subtask 4.3 : Lire `audioTrimEnabled` depuis settingsStore (ou le recevoir en paramètre si l'architecture préfère éviter le couplage direct)
- [ ] Subtask 4.4 : Passer `{ trimSilence: audioTrimEnabled }` en options lors des appels
- [ ] Subtask 4.5 : S'assurer que le comportement par défaut (false) ne change rien au flow actuel

### Task 5: Ajouter le toggle UI dans les Settings (AC2)
- [ ] Subtask 5.1 : Identifier le meilleur emplacement : `WhisperSettingsScreen.tsx` (section audio/enregistrement) ou `SettingsScreen.tsx`
- [ ] Subtask 5.2 : Ajouter le composant Toggle avec :
  - Label : "Supprimer les silences automatiquement"
  - Description : "Réduit la taille des fichiers audio. Peut affecter la complétude des transcriptions."
  - Valeur : `settingsStore.audioTrimEnabled`
  - Handler : `settingsStore.setAudioTrimEnabled`
- [ ] Subtask 5.3 : Appliquer le pattern NativeWind existant pour le style (cohérent avec les autres toggles du même écran)
- [ ] Subtask 5.4 : S'assurer que le toggle est visible sans nécessiter de scroll excessif

### Task 6: Validation TDD et tests de non-régression (AC6)
- [ ] Subtask 6.1 : Exécuter `npx jest AudioConversionService.preprocessing.test.ts` — vérifier que les tests `trimSilence()` et `calculateRMS()` passent maintenant (GREEN)
- [ ] Subtask 6.2 : Exécuter `npm run test:unit` — aucune régression
- [ ] Subtask 6.3 : Exécuter `npm run test:acceptance` — aucune régression
- [ ] Subtask 6.4 : Exécuter `npm run test:architecture` — conformité ADR maintenue
- [ ] Subtask 6.5 : Test manuel : enregistrer un audio de 30s avec contenu vocal jusqu'à la fin → vérifier transcription complète (toggle OFF)
- [ ] Subtask 6.6 : Fermer l'issue GitHub #16 après merge (commenter : "Fixed in story 8.3 — trim désactivé par défaut depuis Story 13.4, toggle optionnel ajouté")

## Dev Notes

### 🔍 Historique Critique — Pourquoi trimSilence a été supprimée

**Commit `dddee24` (Story 13.4, 2026-02-19) :**
```
M3+M4: trimSilence/calculateRMS supprimées comme dead code
Raison: User reported last word being cut off → full audio is kept without trimming
```

La suppression était une correction d'urgence. La Story 8.3 implémente la solution propre : trim OPTIONNEL et CONSERVATIF.

**L'ancienne implémentation était trop agressive car :**
- `SILENCE_THRESHOLD = 0.01` était probablement trop élevé pour de la parole naturelle
- Il manquait probablement une marge de sécurité suffisante en fin d'audio
- L'algorithme ne distinguait pas bien silence de respiration/pause naturelle

### 🏗️ Architecture AudioConversionService

**Deux points d'entrée publics dans `AudioConversionService.ts` :**

```typescript
// Version standard (Whisper)
convertToWhisperFormat(filePath: string, options?: { trimSilence?: boolean }): Promise<Result<string>>

// Version avec padding en fin (Native Recognition — iOS)
convertToWhisperFormatWithPadding(filePath: string, options?: { trimSilence?: boolean }): Promise<Result<string>>
```

**Pipeline pipeline (6 étapes) :**
```
1. decodeAudioData() → AudioBuffer (16kHz, stéréo→mono via resample)
2. mixToMono() → AudioBuffer mono
3. [NEW] if options?.trimSilence → trimSilence() → AudioBuffer (silences extrémités retirés)
4. [OPTIONNEL] addEndPadding() → 800ms silence fin (uniquement convertToWhisperFormatWithPadding)
5. checkLongAudio() → warn si >10min
6. buildWavFile() → WAV PCM 16-bit
7. writeToCache() → string path
```

**Note :** `addEndPadding()` (800ms de silence en FIN) existe pour aider la reconnaissance native à finaliser le dernier mot — c'est l'INVERSE du trim. Les deux peuvent coexister : trim les silences longs aux EXTRÉMITÉS, puis ajouter du padding à la FIN.

### 📐 Spec Technique — trimSilence Conservative

Les tests dans `AudioConversionService.preprocessing.test.ts` sont la spec de comportement. Implémenter en respectant ces tests :

```typescript
// Algorithme suggéré (conservative):
private trimSilence(buffer: AudioBuffer): AudioBuffer {
  const channelData = buffer.getChannelData(0);
  const sampleRate = buffer.sampleRate;

  // 1. Trouver le premier sample non-silencieux (avec marge)
  const startSample = this.findFirstActiveSample(channelData, sampleRate);
  // 2. Trouver le dernier sample non-silencieux (avec marge IMPORTANTE en fin)
  const endSample = this.findLastActiveSample(channelData, sampleRate);

  // 3. Si pas de contenu détecté → retourner buffer original (sécurité)
  if (startSample >= endSample) return buffer;

  // 4. Extraire le segment actif
  // ... createBuffer + copyFromChannel
}

private calculateRMS(samples: Float32Array, start: number, end: number): number {
  // Implémentation classique RMS sur la fenêtre [start, end]
}
```

**Constantes recommandées (plus conservative que l'ancienne) :**
```typescript
private readonly SILENCE_THRESHOLD = 0.005;    // < 0.01 ancien — plus permissif
private readonly SILENCE_MARGIN_MS = 200;       // 200ms marge aux extrémités
private readonly SILENCE_MIN_DURATION_MS = 500; // Ignorer silences < 500ms
```

### 🔧 TranscriptionWorker — Comment Passer la Préférence

**Option recommandée :** Lire settingsStore directement dans le worker (cohérent avec `isAutoPostProcessEnabled` déjà résolu de cette façon).

```typescript
// Dans TranscriptionWorker.ts, avant l'appel à convertToWhisperFormat :
import { useSettingsStore } from '../../stores/settingsStore';
// OU (si le pattern DI préféré) : injecter ISettings via token

const audioTrimEnabled = settingsStore.getState().audioTrimEnabled;  // ou getState() selon le pattern

const wavPath = await this.audioConversionService.convertToWhisperFormat(
  audioFilePath,
  { trimSilence: audioTrimEnabled }
);
```

**Vérifier le pattern exact** utilisé dans `TranscriptionWorker.ts` pour accéder aux settings (certains workers utilisent `settingsStore.getState()`, d'autres reçoivent les settings via paramètre). Adapter en conséquence.

### 📋 settingsStore — Pattern d'Ajout

Suivre le pattern exact des autres settings boolean dans `settingsStore.ts`. Exemple de référence :
```typescript
// Pattern existant (ligne ~75-77 dans settingsStore.ts)
autoTranscriptionEnabled: boolean,
setAutoTranscription: (enabled: boolean) => void,

// À ajouter de façon similaire :
audioTrimEnabled: boolean,
setAudioTrimEnabled: (enabled: boolean) => void,
```

### 🧪 Tests Existants (État RED → Doit Devenir GREEN)

**Fichier :** `mobile/src/contexts/Normalization/services/__tests__/AudioConversionService.preprocessing.test.ts`

Tests actuellement en état RED (méthodes absentes) :
- `trimSilence()` : "should detect and remove silence using RMS threshold"
- `calculateRMS()` : "should calculate RMS for sine wave samples" + "should return 0 for empty ranges"
- Constants : `SILENCE_THRESHOLD = 0.01`, `MAX_AUDIO_DURATION_MS = 600000`

**Note :** `SILENCE_THRESHOLD = 0.01` dans les tests vs `0.005` recommandé ci-dessus — le test documente l'ancienne valeur. Si on implémente 0.005, le test de la constante échouera. Deux options :
1. Mettre à jour le test pour 0.005 (justifié : ancienne valeur était trop agressive)
2. Garder 0.01 si on pense que c'est correct avec la nouvelle implémentation (marges suffisantes)

Décision à prendre par le Dev Agent après test sur device réel.

### ⚠️ Garde-Fous Critiques

1. **Comportement par défaut INCHANGÉ** : trim désactivé (false) = comportement post-Story-13.4, aucune régression possible
2. **Si trimSilence est async** : ne pas introduire d'async là où le pipeline est synchrone — adapter en conséquence
3. **Tester OBLIGATOIREMENT sur device réel** avant de marquer done : enregistrement vocal 30s, vérifier les derniers mots sont transcrits
4. **NE PAS remplacer** l'AddEndPadding existant — les deux peuvent coexister (trim silences AVANT le padding final)
5. **ADR-023 (Result Pattern)** : Les nouvelles méthodes internes de l'AudioConversionService sont des helpers synchrones privés, pas des Result<T> — ne pas forcer Result sur des méthodes internes triviales

### 🏗️ Architecture Compliance

- **ADR-024 (Clean Code)** : Constantes nommées, pas de magic numbers
- **ADR-021 (DI Lifecycle)** : AudioConversionService est déjà dans le DI container — ne pas modifier son lifecycle
- **ADR-022 (AsyncStorage)** : settingsStore persiste via son mécanisme existant — vérifier qu'il n'utilise pas AsyncStorage pour des données critiques (settings simples : OK)
- **ADR-023 (Result Pattern)** : Méthodes publiques de AudioConversionService retournent déjà `Result<string>` — maintenir ce pattern pour les méthodes modifiées

### Project Structure Notes

**Fichiers à modifier :**
```
mobile/src/stores/settingsStore.ts                              — Ajouter audioTrimEnabled
mobile/src/contexts/Normalization/services/AudioConversionService.ts  — Restaurer trimSilence + options
mobile/src/contexts/Normalization/workers/TranscriptionWorker.ts      — Passer option trim
mobile/src/screens/settings/WhisperSettingsScreen.tsx                 — Toggle UI
```

**Fichiers tests :**
```
mobile/src/contexts/Normalization/services/__tests__/AudioConversionService.preprocessing.test.ts — Tests existants RED → GREEN
```

**Patterns de référence :**
- `mobile/src/_patterns/` — snippets pour patterns récurrents du projet
- Toggle UI existant dans `SettingsScreen.tsx` (lignes ~579-618) — répliquer le même pattern NativeWind

### References

- [Source: GitHub Issue #16](https://github.com/yohikofox/pensieve/issues/16) — Bug report original
- [Source: git commit dddee24] — trimSilence supprimée en Story 13.4, raison documentée
- [Source: git commit 9f9a33aec] — Story 8.20 : NativeTranscriptionEngine accumulation fix
- [Source: mobile/src/contexts/Normalization/services/AudioConversionService.ts] — Pipeline audio (6 étapes)
- [Source: mobile/src/contexts/Normalization/services/__tests__/AudioConversionService.preprocessing.test.ts] — Tests TDD existants (RED, spec de comportement)
- [Source: mobile/src/stores/settingsStore.ts#75-77] — Pattern de référence pour ajout de setting boolean
- [Source: _bmad-output/planning-artifacts/architecture.md] — DDD Bounded Context Normalization

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

### Completion Notes List

### File List

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-02-28 | Story créée depuis issue GitHub #16 — analyse historique confirmant bug résolu en Story 13.4, scope re-défini comme feature optionnelle conservative | yohikofox |

# Story 8.20: Fix Transcription Native — Résultat Tronqué pour Audio Long

Status: review

## Story

En tant qu'utilisateur enregistrant une note vocale de plus de ~15 secondes,
je veux que la transcription native retourne le texte **complet** de mon enregistrement,
afin de ne pas perdre le début de mes idées.

## Acceptance Criteria

1. **AC1 — Accumulation des segments `isFinal` (transcribeFile)**
   La méthode `NativeTranscriptionEngine.transcribeFile` accumule tous les résultats `isFinal` au lieu d'écraser — le texte final est la concaténation de tous les segments.

2. **AC2 — Accumulation en temps réel (startRealTime)**
   La méthode `NativeTranscriptionEngine.startRealTime` accumule les résultats `isFinal` au lieu d'écraser `accumulatedText`.

3. **AC3 — Audio court non régressé**
   Un enregistrement ≤ 10 secondes (un seul segment `isFinal`) retourne un résultat identique à l'implémentation actuelle.

4. **AC4 — Séparateur entre segments**
   Les segments sont joints avec un espace simple pour produire une phrase lisible et sans doublon de ponctuation.

5. **AC5 — Tests unitaires**
   Des tests unitaires couvrent le comportement multi-`isFinal` pour `transcribeFile` et `startRealTime`.

## Tasks / Subtasks

- [x] Task 1 — Corriger `NativeTranscriptionEngine.transcribeFile` (AC1, AC3, AC4)
  - [x] 1.1 : Écrire test unitaire RED — simuler 3 événements `isFinal` successifs → vérifier résultat concaténé
  - [x] 1.2 : Changer `finalText = transcript` en `finalText = finalText ? finalText + " " + transcript : transcript` (ligne ~161)
  - [x] 1.3 : Faire passer le test (GREEN)
  - [x] 1.4 : Vérifier le test AC3 (single `isFinal`) — aucune régression

- [x] Task 2 — Corriger `NativeTranscriptionEngine.startRealTime` (AC2, AC4)
  - [x] 2.1 : Écrire test unitaire RED — simuler 2 événements `isFinal` → vérifier `accumulatedText` cumulé
  - [x] 2.2 : Changer `this.accumulatedText = text` (ligne ~304) en accumulation avec espace
  - [x] 2.3 : Faire passer le test (GREEN)

- [x] Task 3 — Vérification et nettoyage (AC5)
  - [x] 3.1 : Lancer `npm run test:unit` dans `pensieve/mobile/` — zéro régression
  - [x] 3.2 : Mettre à jour le File List

## Dev Notes

### Root Cause

`NativeTranscriptionEngine` utilise `expo-speech-recognition`. Lorsqu'un enregistrement contient plusieurs pauses naturelles (ou dépasse la fenêtre de contexte du moteur natif iOS/Android), le moteur envoie **plusieurs événements `isFinal`** — un par segment de parole.

**Bug 1 — `transcribeFile` (ligne ~161) :**
```typescript
// AVANT (bug) :
finalText = event.results[selectedIndex]?.transcript || "";

// APRÈS (fix) :
const newSegment = event.results[selectedIndex]?.transcript || "";
finalText = finalText ? finalText + " " + newSegment : newSegment;
```

**Bug 2 — `startRealTime` (ligne ~304) :**
```typescript
// AVANT (bug) :
this.accumulatedText = text;

// APRÈS (fix) :
this.accumulatedText = this.accumulatedText ? this.accumulatedText + " " + text : text;
```

Le même pattern d'écrasement existe aux deux endroits. L'utilisateur ne voit que le **dernier segment** transcrit, perdant tout le début de la note.

**Correction cohérence (branche partielle) :**
La branche `else` (isFinal=false) ne modifie plus `accumulatedText` — les résultats partiels sont transitoires et ne doivent pas écraser l'accumulation des segments finaux confirmés.

### Fichiers à modifier

| Fichier | Changement |
|---------|-----------|
| `pensieve/mobile/src/contexts/Normalization/services/NativeTranscriptionEngine.ts` | Bug 1 (l.~161) + Bug 2 (l.~304) + cohérence branche partielle |
| `pensieve/mobile/src/contexts/Normalization/services/__tests__/NativeTranscriptionEngine.test.ts` | Tests unitaires nouveaux (créés) |

### Contraintes

- Ne pas modifier la logique de sélection de la meilleure alternative (index basé sur confidence) — conserver le `selectedIndex` logic pour chaque segment
- Le `confidence` retourné dans le résultat final doit être celui du **dernier** segment (comportement actuel acceptable pour le debug)
- `nativeResults` est construit à partir du dernier événement `isFinal` — conserver ce comportement (metadata, pas texte)

### Contexte Tests

- Tests unitaires : `babel-jest` — fichiers `*.test.ts` dans `src/`
- Commande : `npm run test:unit` dans `pensieve/mobile/`
- Pattern mock `ExpoSpeechRecognitionModule` : jest.mock avec factory (variables capturées via `jest.requireMock` — le hoisting de jest.mock interdit l'usage de `const` externe dans la factory)

### Project Structure Notes

- Bounded Context : `Normalization` (`pensieve/mobile/src/contexts/Normalization/`)
- Engine concret : `services/NativeTranscriptionEngine.ts`
- Interface : `services/ITranscriptionEngine.ts` — ne pas modifier la signature publique
- DI Token : via `TranscriptionEngineService`, enregistré dans `container.ts`

### References

- Bug identifié en session CH avec Amelia (2026-02-20) — analyse manuelle du code
- `NativeTranscriptionEngine.ts` : méthode `transcribeFile` (l.130-261), méthode `startRealTime` (l.266-362)
- `expo-speech-recognition` : comportement multi-`isFinal` documenté pour enregistrements longs
- Story liée : 8-3 (audio trim — problème distinct : suppression des silences)

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

- Jest hoisting: `jest.mock` factory ne peut pas capturer `const` déclarées avant l'appel. Solution: définir le mock object dans la factory et le récupérer via `jest.requireMock`.

### Completion Notes List

- ✅ AC1 : `transcribeFile` — `finalText` accumule avec opérateur ternaire + espace (`l.161-162`)
- ✅ AC2 : `startRealTime` — `accumulatedText` accumule les `isFinal` avec espace (`l.305-307`)
- ✅ AC3 : Test "un seul isFinal" vert avant et après fix (non régressé)
- ✅ AC4 : Séparateur espace simple, pas de doublon — vérifié par `not.toContain("  ")`
- ✅ AC5 : 8 tests unitaires créés dans `NativeTranscriptionEngine.test.ts` — tous verts
- ✅ Cohérence : branche partielle (`isFinal=false`) ne modifie plus `accumulatedText` — évite corruption de l'accumulation entre segments finaux
- ✅ Régression : 184 échecs après fix vs 190 avant (6 réparations additionnelles, 0 régression)

### File List

- `pensieve/mobile/src/contexts/Normalization/services/NativeTranscriptionEngine.ts` (modifié)
- `pensieve/mobile/src/contexts/Normalization/services/__tests__/NativeTranscriptionEngine.test.ts` (créé)

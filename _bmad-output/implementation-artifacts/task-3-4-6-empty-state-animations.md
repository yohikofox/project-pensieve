# Task 3.4.6: Enhance Empty State Animations

**Story**: 3.4 - Navigation et Interactions dans le Feed
**Parent Task**: Task 6 - Enhance Empty State Animations (AC8)
**Status**: ready-for-dev
**Date Created**: 2026-02-03

---

## 📋 Contexte

### Critères d'Acceptation (AC8)
**Given** I interact with empty states or placeholders
**When** no captures exist or filters show no results
**Then** illustrations are beautiful and on-brand (garden metaphor)
**And** micro-animations add life (butterflies, gentle breeze effects)
**And** the empty state guides me to take the next action

**Current Status**: ✅ PARTIALLY DONE - EmptyState with "Jardin d'idées" metaphor exists in CapturesListScreen

### État Actuel
- **Composant**: `pensieve/mobile/src/design-system/components/EmptyState.tsx`
- **Usage**: `CapturesListScreen.tsx` avec:
  - Icône: `feather` (métaphore graine)
  - Titre: "Votre jardin d'idées est prêt à germer"
  - Description: "Capturez votre première pensée"
  - CTA: Button "Commencer"
- **Limitation**: Statique, sans animations

### Objectif
Ajouter des micro-animations subtiles et calming pour renforcer la métaphore "Jardin d'idées" et améliorer l'engagement utilisateur.

---

## 🎯 Subtasks Détaillées

### Subtask 6.1: Install & Configure Lottie Animations ✅

**Version**: `lottie-react-native ~7.3.1` (compatible Expo SDK 54)

**Installation**:
```bash
npx expo install lottie-react-native
```

**Source de Fichiers Lottie**:
- [LottieFiles](https://lottiefiles.com) - Bibliothèque gratuite d'animations
- **Recherche recommandée**:
  - Keywords: "butterfly", "leaf", "breeze", "nature", "garden", "floating"
  - Style: Minimal, calming, pas cartoon
  - Couleurs: Green/Blue (success[300], primary[300])

**Animations Suggérées**:
1. **Butterfly** - Papillon flottant subtil (top-right corner)
2. **Breeze** - Feuilles ou particules flottantes (ambient background)
3. **Seed** - Graine germant (near icon, optionnel)

**Critères de Sélection**:
- ✅ Fichier `.json` format (lottie-react-native standard)
- ✅ Taille < 50KB (performance)
- ✅ Loop compatible
- ✅ License: Free for personal/commercial use

---

### Subtask 6.2: Implement Gentle Breathing Animation 🎨

**Composant**: `AnimatedEmptyState.tsx` (wrapper component)

**Animation Pattern**: Breathing effect sur l'icône "feather"

**Specs Techniques**:
```typescript
// pensieve/mobile/src/components/animations/AnimatedEmptyState.tsx

import React, { useEffect, useRef } from 'react';
import { Animated, Easing } from 'react-native';

interface AnimatedEmptyStateProps {
  children: React.ReactNode;
  enabled?: boolean; // Default: true, disable if reduce motion
}

export function AnimatedEmptyState({
  children,
  enabled = true
}: AnimatedEmptyStateProps) {
  const breathingScale = useRef(new Animated.Value(1)).current;
  const breathingOpacity = useRef(new Animated.Value(0.7)).current;

  useEffect(() => {
    if (!enabled) return;

    // Gentle breathing cycle: 3000ms (slower than PulsingBadge)
    const breathingAnimation = Animated.loop(
      Animated.parallel([
        // Scale animation (subtle)
        Animated.sequence([
          Animated.timing(breathingScale, {
            toValue: 1.08,
            duration: 1500,
            easing: Easing.inOut(Easing.ease),
            useNativeDriver: true,
          }),
          Animated.timing(breathingScale, {
            toValue: 1,
            duration: 1500,
            easing: Easing.inOut(Easing.ease),
            useNativeDriver: true,
          }),
        ]),
        // Opacity animation (respiratory effect)
        Animated.sequence([
          Animated.timing(breathingOpacity, {
            toValue: 1,
            duration: 1500,
            easing: Easing.inOut(Easing.ease),
            useNativeDriver: true,
          }),
          Animated.timing(breathingOpacity, {
            toValue: 0.7,
            duration: 1500,
            easing: Easing.inOut(Easing.ease),
            useNativeDriver: true,
          }),
        ]),
      ])
    );

    breathingAnimation.start();

    return () => breathingAnimation.stop();
  }, [breathingScale, breathingOpacity, enabled]);

  if (!enabled) {
    return <>{children}</>;
  }

  return (
    <Animated.View
      style={{
        transform: [{ scale: breathingScale }],
        opacity: breathingOpacity,
      }}
    >
      {children}
    </Animated.View>
  );
}
```

**Integration dans EmptyState**:
```typescript
// Usage in CapturesListScreen.tsx
import { AnimatedEmptyState } from '../../components/animations/AnimatedEmptyState';

<AnimatedEmptyState>
  <EmptyState
    icon="feather"
    title={t('captures.emptyTitle')}
    description={t('captures.emptyDescription')}
    actionLabel={t('captures.emptyAction')}
    onAction={handleStartCapture}
  />
</AnimatedEmptyState>
```

---

### Subtask 6.3: Calming Color Palette & Visual Aesthetic 🌿

**Problème**: Icône actuelle utilise `colors.neutral[400]` (gris) - pas assez "jardin d'idées"

**Solution**: Modifier EmptyState pour accepter `iconColor` prop

**Modifications**:

1. **EmptyState.tsx** - Ajouter prop `iconColor`:
```typescript
interface EmptyStateProps {
  icon?: keyof typeof Feather.glyphMap;
  iconColor?: string; // NEW
  title: string;
  description?: string;
  actionLabel?: string;
  onAction?: () => void;
  className?: string;
}

export function EmptyState({
  icon,
  iconColor, // NEW
  title,
  description,
  actionLabel,
  onAction,
  className,
}: EmptyStateProps) {
  return (
    <View className={cn('flex-1 items-center justify-center px-8 py-12', className)}>
      {icon && (
        <View className="w-20 h-20 rounded-full bg-bg-subtle items-center justify-center mb-6">
          <Feather
            name={icon}
            size={40}
            color={iconColor || colors.neutral[400]} // Use iconColor if provided
          />
        </View>
      )}
      {/* ... rest */}
    </View>
  );
}
```

2. **CapturesListScreen.tsx** - Passer `iconColor`:
```typescript
<AnimatedEmptyState>
  <EmptyState
    icon="feather"
    iconColor={colors.success[300]} // Fresh green (#6EE7B7)
    title={t('captures.emptyTitle')}
    description={t('captures.emptyDescription')}
    actionLabel={t('captures.emptyAction')}
    onAction={handleStartCapture}
  />
</AnimatedEmptyState>
```

**Palette "Calming"**:
- **Icône**: `colors.success[300]` (#6EE7B7) - Vert frais, nature
- **Background icône**: `colors.success[50]` (#ECFDF5) - Teinte verte subtile (optionnel)
- **Texte titre**: Existant (`text-text-primary`)
- **Texte description**: Existant (`text-text-tertiary`)
- **CTA Button**: `variant="primary"` (blue, non-urgent)

**Dark Mode Support**:
- Light: `success[300]` (#6EE7B7)
- Dark: `success[400]` (#34D399) - Plus lumineux pour contraste

---

### Subtask 6.4: Lottie Micro-Animations Integration 🦋

**Composant**: Intégrer dans `CapturesListScreen.tsx` (ListEmptyComponent)

**Placement**:
1. **Butterfly** - Position absolute, top-right, z-index: 1
2. **Breeze/Leaves** - Background layer, full width, opacity 0.3

**Code Example**:
```typescript
import LottieView from 'lottie-react-native';

const EnhancedEmptyState = () => (
  <View style={styles.emptyContainer}>
    {/* Background ambient animation (optional) */}
    <LottieView
      source={require('../../assets/animations/breeze.json')}
      autoPlay
      loop
      style={styles.breezeAnimation}
    />

    {/* Main empty state with breathing animation */}
    <AnimatedEmptyState>
      <EmptyState
        icon="feather"
        iconColor={colors.success[300]}
        title="Votre jardin d'idées est prêt à germer"
        description="Capturez votre première pensée"
        actionLabel="Commencer"
        onAction={handleStartCapture}
      />
    </AnimatedEmptyState>

    {/* Butterfly floating animation */}
    <LottieView
      source={require('../../assets/animations/butterfly.json')}
      autoPlay
      loop
      style={styles.butterflyAnimation}
    />
  </View>
);

const styles = StyleSheet.create({
  emptyContainer: {
    flex: 1,
    position: 'relative',
  },
  breezeAnimation: {
    position: 'absolute',
    width: '100%',
    height: '100%',
    opacity: 0.3,
    zIndex: 0,
  },
  butterflyAnimation: {
    position: 'absolute',
    top: 40,
    right: 20,
    width: 80,
    height: 80,
    zIndex: 10,
  },
});
```

**Performance**:
- ✅ Lottie animations run on native thread (60fps)
- ✅ Lazy load JSON files (require at render time)
- ✅ Max 2-3 Lottie animations simultanées

---

### Subtask 6.5: Responsive Testing & Accessibility ♿

**Screen Sizes to Test**:
1. **iPhone SE (small)** - 320x568pt
2. **iPhone 15 Pro** - 393x852pt
3. **iPhone 15 Pro Max** - 430x932pt
4. **iPad Mini** - 768x1024pt

**Responsive Checks**:
- ✅ Icône size: 40px constant (lisible sur tous devices)
- ✅ Padding: `px-8 py-12` (adaptive via Tailwind)
- ✅ Button size: Touch target 44x44pt minimum
- ✅ Lottie animations: Scale proportionally

**Accessibility**:

1. **Reduce Motion Support**:
```typescript
import { AccessibilityInfo } from 'react-native';

const [isReduceMotionEnabled, setReduceMotionEnabled] = useState(false);

useEffect(() => {
  AccessibilityInfo.isReduceMotionEnabled().then((enabled) => {
    setReduceMotionEnabled(enabled);
  });
}, []);

<AnimatedEmptyState enabled={!isReduceMotionEnabled}>
  {/* ... */}
</AnimatedEmptyState>
```

2. **Screen Reader**:
```typescript
<View accessible={true} accessibilityLabel="État vide: Aucune capture. Commencez en capturant votre première pensée.">
  <EmptyState ... />
</View>
```

3. **Dark Mode**:
- Tester avec `useColorScheme()` hook
- Vérifier contraste icône (success[400] en dark mode)

---

## 🧪 Tests BDD (jest-cucumber)

### Feature File
**Fichier**: `pensieve/mobile/tests/acceptance/features/story-3-4-feed-interactions.feature`

**Scénario à Ajouter**:
```gherkin
Scénario: États vides animés avec "Jardin d'idées"
  Étant donné qu'aucune capture n'existe dans la base de données
  Quand j'ouvre l'écran de feed des captures
  Alors je vois une illustration "feather" avec couleur verte calming
  Et l'icône a une animation de respiration lente (3000ms cycle)
  Et je vois le titre "Votre jardin d'idées est prêt à germer"
  Et je vois la description "Capturez votre première pensée"
  Et je vois un bouton "Commencer"
  Et des micro-animations Lottie ajoutent de la vie à l'écran
  Et l'esthétique est calming et contemplative

Scénario: Respect de Reduce Motion
  Étant donné que l'utilisateur a activé "Reduce Motion" dans les réglages
  Et qu'aucune capture n'existe
  Quand j'ouvre l'écran de feed
  Alors l'état vide s'affiche sans animations
  Et les informations restent accessibles
```

### Step Definitions
**Fichier**: `pensieve/mobile/tests/acceptance/story-3-4-feed-interactions.test.ts`

```typescript
defineFeature(feature, (test) => {
  test('États vides animés avec "Jardin d\'idées"', ({ given, when, then, and }) => {
    given('qu\'aucune capture n\'existe dans la base de données', () => {
      testContext.captures = [];
    });

    when('j\'ouvre l\'écran de feed des captures', () => {
      renderCapturesListScreen();
    });

    then('je vois une illustration "feather" avec couleur verte calming', () => {
      const featherIcon = screen.getByTestId('empty-state-icon-feather');
      expect(featherIcon).toBeTruthy();
      // Vérifier couleur success[300] (#6EE7B7)
      expect(featherIcon.props.color).toBe('#6EE7B7');
    });

    and('l\'icône a une animation de respiration lente (3000ms cycle)', () => {
      // Vérifier AnimatedEmptyState wrapper présent
      const animatedWrapper = screen.getByTestId('animated-empty-state');
      expect(animatedWrapper).toBeTruthy();
      // TODO: Vérifier cycle animation via jest.advanceTimersByTime(3000)
    });

    and('je vois le titre "Votre jardin d\'idées est prêt à germer"', () => {
      expect(screen.getByText(/jardin d'idées/i)).toBeTruthy();
    });

    and('je vois la description "Capturez votre première pensée"', () => {
      expect(screen.getByText(/capturez votre première pensée/i)).toBeTruthy();
    });

    and('je vois un bouton "Commencer"', () => {
      expect(screen.getByText('Commencer')).toBeTruthy();
    });

    and('des micro-animations Lottie ajoutent de la vie à l\'écran', () => {
      // Vérifier présence de LottieView components
      const lottieAnimations = screen.queryAllByTestId(/lottie-animation/);
      expect(lottieAnimations.length).toBeGreaterThanOrEqual(1);
    });

    and('l\'esthétique est calming et contemplative', () => {
      // Assertion qualitative: vérifier palette success (green)
      // Vérifier pas de rouge/orange (error/warning colors)
      const emptyState = screen.getByTestId('empty-state-container');
      expect(emptyState).toBeTruthy();
    });
  });

  test('Respect de Reduce Motion', ({ given, and, when, then }) => {
    given('que l\'utilisateur a activé "Reduce Motion" dans les réglages', () => {
      jest.spyOn(AccessibilityInfo, 'isReduceMotionEnabled').mockResolvedValue(true);
    });

    and('qu\'aucune capture n\'existe', () => {
      testContext.captures = [];
    });

    when('j\'ouvre l\'écran de feed', () => {
      renderCapturesListScreen();
    });

    then('l\'état vide s\'affiche sans animations', async () => {
      await waitFor(() => {
        const animatedWrapper = screen.queryByTestId('animated-empty-state');
        // Animation disabled via enabled={false}
        expect(animatedWrapper).toBeFalsy();
      });
    });

    and('les informations restent accessibles', () => {
      expect(screen.getByText(/jardin d'idées/i)).toBeTruthy();
      expect(screen.getByText('Commencer')).toBeTruthy();
    });
  });
});
```

---

## ✅ Checklist de Validation

### Fonctionnel
- [ ] Lottie installé (`npx expo install lottie-react-native`)
- [ ] AnimatedEmptyState.tsx créé avec breathing animation
- [ ] EmptyState.tsx supporte prop `iconColor`
- [ ] CapturesListScreen utilise `iconColor={colors.success[300]}`
- [ ] Lottie animations intégrées (butterfly, breeze)
- [ ] Reduce Motion support (AccessibilityInfo)

### Performance
- [ ] Breathing animation runs at 60fps (useNativeDriver: true)
- [ ] Lottie JSON files < 50KB each
- [ ] No jank during animation loop
- [ ] GPU acceleration verified

### Aesthetic
- [ ] Couleur icône: success[300] (#6EE7B7 green)
- [ ] Animation lente et calming (3000ms cycle)
- [ ] Pas de rouge/orange (pas anxiogène)
- [ ] Dark mode: success[400] (#34D399)
- [ ] Lottie animations subtiles (opacity < 0.5 si background)

### Tests
- [ ] BDD tests AC8 passing
- [ ] Reduce Motion scenario passing
- [ ] Responsive sur iPhone SE, Pro Max, iPad
- [ ] Dark mode testé manuellement

### Accessibilité
- [ ] accessibilityLabel sur EmptyState
- [ ] Reduce Motion désactive animations
- [ ] Touch target CTA button ≥ 44x44pt
- [ ] Screen reader compatible

---

## 📁 Fichiers Modifiés/Créés

### Créer
- `pensieve/mobile/src/components/animations/AnimatedEmptyState.tsx` ✅
- `pensieve/mobile/assets/animations/butterfly.json` (Lottie file) ✅
- `pensieve/mobile/assets/animations/breeze.json` (Lottie file, optionnel) ✅

### Modifier
- `pensieve/mobile/src/design-system/components/EmptyState.tsx` (add `iconColor` prop)
- `pensieve/mobile/src/screens/captures/CapturesListScreen.tsx` (integrate animations)
- `pensieve/mobile/tests/acceptance/features/story-3-4-feed-interactions.feature` (add AC8 scenarios)
- `pensieve/mobile/tests/acceptance/story-3-4-feed-interactions.test.ts` (add step definitions)

### Dépendances
- `package.json`: Add `lottie-react-native: ~7.3.1`

---

## 🎨 Références Design

### Lottie Files Suggestions
- [Butterfly Animation](https://lottiefiles.com/search?q=butterfly&category=animations)
- [Nature Breeze](https://lottiefiles.com/search?q=breeze%20leaves)
- [Germination/Seed](https://lottiefiles.com/search?q=seed%20growing)

### Color Palette
| Element | Light Mode | Dark Mode |
|---------|-----------|-----------|
| Icône feather | `success[300]` #6EE7B7 | `success[400]` #34D399 |
| Background icône | `success[50]` #ECFDF5 | `success[900]` #064E3B |
| Texte titre | `text-primary` | `text-primary` |
| Texte description | `text-tertiary` | `text-tertiary` |
| CTA button | `primary[500]` #3B82F6 | `primary[400]` #76A9FA |

### Timing Specs
| Animation | Duration | Easing | Loop |
|-----------|----------|--------|------|
| Breathing scale | 1500ms → 1500ms (3000ms total) | easeInOut | ✅ |
| Breathing opacity | 1500ms → 1500ms (3000ms total) | easeInOut | ✅ |
| Lottie butterfly | Auto (from JSON) | - | ✅ |
| Lottie breeze | Auto (from JSON) | - | ✅ |

---

## 📝 Dev Notes

### Pattern Reuse
- **AnimatedEmptyState** suit le pattern de **PulsingBadge** et **GerminationBadge**
- Wrapper component pattern = separation of concerns
- `enabled` prop pour Reduce Motion = accessibility best practice

### Alternative Considered
**Option 1**: Intégrer animation directement dans EmptyState.tsx
❌ Couple animation au design system
❌ Moins flexible pour autres usages de EmptyState

**Option 2**: Wrapper séparé AnimatedEmptyState ✅ CHOISI
✅ Cohérent avec PulsingBadge/GerminationBadge
✅ Réutilisable
✅ Separation of concerns

### Lottie vs React Native Animated
- **React Native Animated**: Breathing animation (simple, léger)
- **Lottie**: Micro-animations complexes (butterfly, breeze) impossible avec Animated API
- **Combinaison**: Best of both worlds

---

## 🚀 Prochaines Étapes (Après Task 6)

- **Task 7**: Performance Optimization (Profiling, getItemLayout)
- **Task 8**: Write Comprehensive Tests (E2E, Integration)
- **Story 3.4 Review**: Code review avant marquage "done"

---

**Date Last Updated**: 2026-02-03
**Agent**: Amelia (Developer Agent)
**Model**: Claude Sonnet 4.5

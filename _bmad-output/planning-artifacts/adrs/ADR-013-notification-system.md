---
adr: ADR-013
title: "Notification System"
date: 2026-01-19
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Architecture Design"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
---

# ADR-013: Notification System

**Status:** ✅ ACCEPTÉ

**Context:** Notifier l'utilisateur des événements importants (digestion complétée, concordance détectée, todo reminder) sans spammer.

**Decision:** Local notifications MVP, push notifications Post-MVP, opt-in/opt-out granulaire

---

## Architecture Notifications

**Types de notifications:**

```typescript
// Types de notifications
enum NotificationType {
  DIGESTION_COMPLETED = 'digestion_completed',
  TRANSCRIPTION_FAILED = 'transcription_failed',
  CONCORDANCE_DETECTED = 'concordance_detected',
  TODO_REMINDER = 'todo_reminder',
  PROJECT_SUGGESTED = 'project_suggested',
}

// Préférences user (opt-in/opt-out)
interface NotificationPreferences {
  digestionCompleted: boolean;    // Default: true
  concordanceDetected: boolean;   // Default: true
  todoReminders: boolean;         // Default: true
  projectSuggested: boolean;      // Default: true

  // Timing
  quietHours: {
    enabled: boolean;
    start: string;  // "22:00"
    end: string;    // "08:00"
  };
}
```

---

## MVP : Local Notifications Uniquement

```typescript
// Mobile (Expo Notifications)
import * as Notifications from 'expo-notifications';

class NotificationService {
  async scheduleLocal(
    type: NotificationType,
    title: string,
    body: string,
    data?: any,
    triggerSeconds: number = 0
  ) {
    // Vérifier préférences user
    const prefs = await this.getPreferences();
    if (!this.isEnabled(type, prefs)) return;

    // Vérifier quiet hours
    if (this.isQuietHours(prefs.quietHours)) {
      // Reporter après quiet hours
      triggerSeconds = this.calculateDelayAfterQuietHours(prefs.quietHours);
    }

    await Notifications.scheduleNotificationAsync({
      content: {
        title,
        body,
        data,
        sound: true,
        badge: 1,
      },
      trigger: triggerSeconds === 0
        ? null  // Immédiat
        : { seconds: triggerSeconds }
    });
  }

  // Exemples d'usage
  async notifyDigestionCompleted(captureId: string) {
    await this.scheduleLocal(
      NotificationType.DIGESTION_COMPLETED,
      '✨ Capture digérée',
      'Votre capture a été analysée et des idées ont été extraites',
      { captureId },
      0  // Immédiat
    );
  }

  async notifyTodoReminder(todo: Todo) {
    const delay = this.calculateDelayUntilDeadline(todo.deadline);

    await this.scheduleLocal(
      NotificationType.TODO_REMINDER,
      '⏰ Rappel',
      todo.description,
      { todoId: todo.id },
      delay
    );
  }

  async notifyProjectSuggested(project: Project) {
    await this.scheduleLocal(
      NotificationType.PROJECT_SUGGESTED,
      '🌱 Nouveau pattern détecté',
      `"${project.name}" - ${project.ideaIds.length} idées connexes`,
      { projectId: project.id },
      0
    );
  }
}
```

---

## Post-MVP : Push Notifications (Firebase)

```typescript
// Backend envoie push via FCM
class PushNotificationService {
  async sendPush(
    userId: string,
    type: NotificationType,
    title: string,
    body: string,
    data?: any
  ) {
    const user = await this.userService.findById(userId);

    // Récupérer FCM token
    const fcmToken = user.fcmToken;
    if (!fcmToken) return;  // User pas enregistré pour push

    // Vérifier préférences
    const prefs = await this.preferencesService.get(userId);
    if (!this.isEnabled(type, prefs)) return;

    // Envoyer via FCM
    await this.fcm.send({
      token: fcmToken,
      notification: {
        title,
        body,
      },
      data,
      android: {
        priority: 'high',
        notification: {
          sound: 'default',
          channelId: 'pensine_default',
        },
      },
      apns: {
        payload: {
          aps: {
            sound: 'default',
            badge: 1,
          },
        },
      },
    });
  }
}
```

---

## Rationale

- **MVP** : local notifications suffisent (mono-user, app ouverte fréquemment)
- **Post-MVP** : push pour engagement (concordance détectée pendant app fermée)
- **Opt-in/opt-out** : respect préférences user
- **Quiet hours** : pas de spam nocturne
- **Progressive** : commencer simple, ajouter complexité si nécessaire

---

## Conséquences

**Bénéfices:**
- UX non-intrusive : notifications pertinentes uniquement
- Respect préférences : opt-out granulaire + quiet hours
- MVP rapide : local notifications simples (Expo SDK)
- Post-MVP : engagement amélioré avec push

**Trade-offs acceptés:**
- MVP : notifications uniquement si app installée (pas de push)
- Post-MVP : setup Firebase (~2h) + coût FCM (gratuit < 100k/mois)
- Complexité : gestion préférences + timing

**Impact:**
- **Epic 4** : Notifications digestion complétée
- **Epic 5** : Notifications concordance détectée
- **Post-MVP** : Todo reminders avec push notifications

---

## Implementation Status

- ⏳ **Epic 4** : Local notifications digestion
- ⏳ **Epic 5** : Local notifications concordance
- ⏳ **Post-MVP** : Push notifications (Firebase Cloud Messaging)
- ⏳ **Post-MVP** : Todo reminders scheduling

---

## References

- Expo Notifications: https://docs.expo.dev/versions/latest/sdk/notifications/
- Firebase Cloud Messaging: https://firebase.google.com/docs/cloud-messaging
- iOS Notification Center: https://developer.apple.com/documentation/usernotifications
- Android Notification Channels: https://developer.android.com/develop/ui/views/notifications

---

## Validation Criteria

ADR considéré succès SI :
- ⏳ Notifications locales fonctionnent (Epic 4, 5)
- ⏳ Opt-out respecté (user peut désactiver par type)
- ⏳ Quiet hours fonctionnent (pas de notifications nocturnes)
- ⏳ Post-MVP : Push notifications FCM < 500ms delivery

**Review Date :** 2026-03 (après Epic 5)

---

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

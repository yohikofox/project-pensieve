# ADR-037 — Observabilité : Crash Reporting + Télémétrie Custom

**Date:** 2026-02-26
**Statut:** Accepté
**Auteurs:** yohikofox, Winston (Architect)

---

## Contexte

Le projet entre en phase d'utilisation réelle et nécessite de collecter des données techniques
sur les utilisateurs : crashes, performance, analytics comportementaux, infos device.

**Contrainte principale : homelab limité à 8GB de RAM total** pour l'ensemble des services
(PostgreSQL, RabbitMQ, MinIO, NestJS, Next.js apps, Portainer, Cloudflare Tunnel).

## Options Explorées

### Option A — Sentry self-hosted + PostHog self-hosted
- Couverture complète (crashes + analytics)
- **Rejetée** : Sentry self-hosted ~4GB RAM + PostHog self-hosted ~2GB RAM = incompatible avec
  la contrainte homelab

### Option B — GlitchTip self-hosted + module telemetry custom
- GlitchTip : alternative Sentry-compatible ultra-légère (~256MB RAM), utilise les mêmes SDKs
- Module `telemetry` dans le backend NestJS existant pour analytics/performance
- **Considérée** mais soulevait la question : pourquoi pas 100% custom ?

### Option C — 100% custom (PLCrashReporter + handler maison)
**Exploration approfondie de la faisabilité :**

- **Android JVM crashes** : `Thread.setDefaultUncaughtExceptionHandler` — faisable, ~1-2j
- **Crashes C++/natifs (iOS)** : signal handlers POSIX (`SIGSEGV`, `SIGABRT`...) dans un état
  de processus indéfini. Seules les fonctions async-signal-safe sont utilisables (`write()` direct,
  pas de `malloc`, `printf`, `dispatch_async`). PLCrashReporter (Microsoft) résout exactement ce
  problème — c'est la brique sous-jacente de Sentry, Firebase Crashlytics, HockeyApp.
- **Expo native module wrapping PLCrashReporter** : techniquement faisable (précédent :
  `expo-waveform-extractor` déjà présent), estimation ~3j iOS + 1j Android + 1j backend = **~5j**
- **Obfuscation** : sans dSYMs/source maps, les stack traces natives sont des adresses mémoire
  brutes (`0x0000000100a4b2c8`). Gérable manuellement mais limité pour les modules natifs
  (`whisper.rn`, `op-sqlite`, `expo-waveform-extractor`). Les crashes JS restent lisibles.
- **Rejetée à ce stade** : ROI insuffisant sur un MVP — 5j d'ingénierie pour éviter une
  dépendance externe à 0€ et 0 RAM homelab. Voie gardée en réserve si les besoins évoluent.

### Option D (Retenue) — Sentry cloud free tier + module telemetry custom

**Crashes natifs → Sentry cloud free tier**
- 5 000 erreurs/mois, gratuit, zéro RAM homelab
- `@sentry/react-native` + `@sentry/nestjs`
- Couvre les crashes natifs iOS/Android (PLCrashReporter sous le capot)
- Symbolication via upload source maps/dSYMs dans le pipeline de build

**Analytics + Performance + Device info → module `telemetry` custom dans le backend NestJS existant**
- Endpoint `POST /telemetry` stocké dans PostgreSQL existant (zéro nouveau service)
- Collecte : events comportementaux, durées d'opérations (transcription, sync), device info,
  OS version, app version
- Crashes JS rattrapables via `ErrorUtils.setGlobalHandler` côté mobile
- Consultable dans l'interface `/admin` existante

## Décision

**Sentry cloud free tier pour les crashes uniquement + module `telemetry` custom dans le
backend existant pour tout le reste.**

## Architecture Cible

```
Mobile
├── @sentry/react-native     → crashes natifs/JS → Sentry cloud
└── TelemetryService         → events + perfs + device info → POST /telemetry (backend)

Backend (NestJS)
├── @sentry/nestjs           → erreurs backend → Sentry cloud
└── modules/telemetry/       → stockage PostgreSQL, consultable via /admin
    ├── domain/entities/telemetry-event.entity.ts
    ├── application/controllers/telemetry.controller.ts
    └── infrastructure/
```

## Budget RAM Additionnel

| Service | RAM ajoutée |
|---------|-------------|
| Sentry cloud | 0 MB (cloud) |
| Module telemetry NestJS | ~0 MB (process existant) |
| PostgreSQL (shared) | déjà là |
| **Total** | **0 MB** |

## Contraintes RGPD

- Opt-in obligatoire avant activation du tracking (onboarding)
- Suppression des données analytics sur demande (API Sentry + DELETE sur table telemetry)
- Documenter dans la politique de confidentialité
- Le module `rgpd` existant devra couvrir la suppression des events telemetry

## Voie de Migration Future

Si le projet grossit et que la dépendance Sentry cloud devient problématique :
1. Migrer vers GlitchTip self-hosted (SDK identique, zéro changement de code)
2. Ou implémenter le module PLCrashReporter custom (estimation ~5j, cf. Option C ci-dessus)

## Conséquences

- ✅ Zéro RAM supplémentaire sur le homelab
- ✅ Crashes natifs couverts sans réinventer la roue
- ✅ Analytics/performance 100% sous contrôle, données chez nous
- ✅ Voie de sortie claire si Sentry cloud devient contraignant
- ⚠️  Dépendance externe pour les crashes (Sentry/Microsoft) — acceptable à ce stade MVP
- ⚠️  Symbolication native nécessite l'upload dSYMs/source maps dans le pipeline CI/CD

---
adr: ADR-023
title: "Stratégie Unifiée de Gestion des Erreurs - Result Pattern"
date: 2026-02-15
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Clarification ADR-009 §9.5"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
supersedes: "ADR-009 §9.5 (partiel - clarification et extension)"
---

# ADR-023: Stratégie Unifiée de Gestion des Erreurs - Result Pattern

**Date:** 2026-02-15
**Status:** ✅ Accepted
**Context:** Clarification et extension de ADR-009 §9.5 pour couvrir TOUTE la gestion d'erreurs du projet
**Decision Makers:** yohikofox (Product Owner), Winston (Architect)

---

## Context & Problem

### Problème à résoudre

Le projet Pensieve utilise actuellement un **Result Pattern** documenté partiellement dans ADR-009 §9.5, mais uniquement dans le contexte de la synchronisation (`SyncResult`). Cette documentation limitée a créé plusieurs ambiguïtés :

**Ambiguïtés identifiées :**

1. **Scope limité** : ADR-009 §9.5 parle uniquement de `SyncResult` pour la synchronisation
2. **Phrase trompeuse** : "Result Pattern (pas try/catch)" suggère de ne jamais utiliser try/catch, alors que le code l'utilise pour convertir exceptions → Result
3. **Pattern non documenté** : Le code utilise `RepositoryResult<T>` qui n'est pas mentionné dans les ADRs
4. **Règles par couche manquantes** : Aucune directive sur comment gérer les erreurs dans Domain, Repository, Service, Controller, UI
5. **Safety net absent** : Pas de stratégie pour attraper les exceptions non gérées

### Contraintes identifiées

1. **Code existant** : `RepositoryResult<T>` est déjà utilisé dans plusieurs repositories
2. **Multi-plateformes** : Doit fonctionner sur mobile (React Native), backend (NestJS), web (Next.js)
3. **DDD Architecture** : Doit respecter les couches domain/application/infrastructure
4. **TypeScript strict mode** : Doit tirer parti du typage fort et exhaustivité switch
5. **Stabilité critique** : NFR "0 capture perdue" → aucun crash acceptable

### Motivation

**Sans stratégie claire unifiée :**
- ❌ Développeurs hésitent entre throw vs Result
- ❌ Try/catch utilisés incohéremment
- ❌ Exceptions non catchées crashent l'app
- ❌ Code review difficile (pas de standard)
- ❌ Nouveaux développeurs perdus

**Avec ADR-023 :**
- ✅ Règle claire : Result Pattern PARTOUT
- ✅ Try/catch uniquement DB/API externes + root handler
- ✅ Wrappers pour outils système
- ✅ Code predictable et maintenable
- ✅ Onboarding simplifié

---

## Decision

### Décision architecturale unifiée

**Pensieve adopte le Result Pattern comme stratégie UNIQUE de gestion des erreurs à travers TOUTES les couches (Domain, Repository, Service, Controller, UI) sur TOUTES les plateformes (mobile, backend, web).**

### Règle stricte sur try/catch

**Try/catch est autorisé UNIQUEMENT dans 3 cas :**

1. **Appels DB externes** (OP-SQLite, TypeORM, PostgreSQL)
2. **Appels API externes** (fetch, axios, Supabase, OpenAI, MinIO)
3. **Root handler technique** (global error handler pour éviter crash app)

**PARTOUT AILLEURS** : Les fonctions retournent `Result<T>` et **ne throw JAMAIS**.

Si un outil système (EventBus, Logger, Analytics) peut throw, créer un **wrapper** qui convertit exceptions → Result.

---

Cette décision se décompose en **6 règles architecturales** :

---

### 1. Result Pattern - Architecture Principale

**Type générique `Result<T>` :**

```typescript
// mobile/src/contexts/shared/domain/Result.ts
// backend/src/shared/domain/Result.ts

export enum ResultType {
  SUCCESS = "success",
  NOT_FOUND = "not_found",
  DATABASE_ERROR = "database_error",
  VALIDATION_ERROR = "validation_error",
  NETWORK_ERROR = "network_error",
  AUTH_ERROR = "auth_error",
  BUSINESS_ERROR = "business_error",
  UNKNOWN_ERROR = "unknown_error",
}

export type Result<T> = {
  type: ResultType;
  data?: T;
  error?: string;
  retryable?: boolean; // Pour retry logic (sync, queue, upload)
};

// Helper functions
export function success<T>(data: T): Result<T> {
  return { type: ResultType.SUCCESS, data };
}

export function notFound<T>(error?: string): Result<T> {
  return {
    type: ResultType.NOT_FOUND,
    error: error ?? "Resource not found",
    retryable: false
  };
}

export function databaseError<T>(error: string): Result<T> {
  return {
    type: ResultType.DATABASE_ERROR,
    error,
    retryable: true // Database errors peuvent être retried
  };
}

export function validationError<T>(error: string): Result<T> {
  return {
    type: ResultType.VALIDATION_ERROR,
    error,
    retryable: false
  };
}

export function networkError<T>(error: string): Result<T> {
  return {
    type: ResultType.NETWORK_ERROR,
    error,
    retryable: true
  };
}

export function authError<T>(error: string): Result<T> {
  return {
    type: ResultType.AUTH_ERROR,
    error,
    retryable: false // Auth errors nécessitent re-login
  };
}

export function businessError<T>(error: string): Result<T> {
  return {
    type: ResultType.BUSINESS_ERROR,
    error,
    retryable: false
  };
}

export function unknownError<T>(error: string): Result<T> {
  return {
    type: ResultType.UNKNOWN_ERROR,
    error,
    retryable: false
  };
}
```

**Rationale :**
- Enum explicite → TypeScript vérifie exhaustivité des switch
- `retryable` flag → Retry logic centralisée (ADR-009)
- Helpers typés → Pas de construction manuelle de Result
- Generic `<T>` → Type-safe à travers toutes les couches

---

### 2. Interdiction de `throw` (Aucune Exception)

**Règle absolue :**

```typescript
// ❌ INTERDIT - JAMAIS de throw dans le code applicatif
async function createCapture(data: CaptureData): Promise<Result<Capture>> {
  if (!data.rawContent) {
    throw new Error("rawContent is required"); // ❌ FORBIDDEN
  }
  // ...
}

// ✅ CORRECT - Retourner Result
async function createCapture(data: CaptureData): Promise<Result<Capture>> {
  if (!data.rawContent) {
    return validationError("rawContent is required"); // ✅ OK
  }
  // ...
}
```

**Aucune exception à cette règle** :
- Pas d'exception pour validation frameworks
- Pas d'exception pour opérations "auxiliaires"
- Si un outil throw → créer un wrapper (voir règle 4)

**Rationale :**
- Exceptions non catchées crashent l'app → NFR "0 capture perdue" violé
- Result force le code appelant à gérer l'erreur
- TypeScript vérifie exhaustivité des cas d'erreur
- Code plus prévisible et testable

---

### 3. Try/Catch UNIQUEMENT pour DB et API Externes

**Règle stricte : Try/catch UNIQUEMENT pour outils externes hors de notre contrôle**

```typescript
// ✅ CORRECT - Try/catch pour DB externe
async create(data: CreateCaptureData): Promise<Result<Capture>> {
  try {
    // Opération externe (OP-SQLite) qui peut throw
    database.execute(
      "INSERT INTO captures (id, type, state) VALUES (?, ?, ?)",
      [id, data.type, data.state]
    );

    const row = database.execute("SELECT * FROM captures WHERE id = ?", [id]);
    const capture = mapRowToCapture(row.rows[0]);

    return success(capture);
  } catch (error) {
    // Convertir exception → Result (ne PAS re-throw)
    const errorMessage = error instanceof Error ? error.message : "Unknown error";
    return databaseError(`Failed to create capture: ${errorMessage}`);
  }
}
```

**Outils externes autorisés (liste exhaustive) :**

| Outil | Plateforme | Exemple | Raison |
|-------|-----------|---------|--------|
| **OP-SQLite** | Mobile | `database.execute()` | DB externe, peut throw |
| **TypeORM** | Backend | `repository.save()` | ORM externe, peut throw |
| **fetch / axios** | Mobile + Backend | `fetch(url)` | API HTTP, peut throw |
| **Supabase Client** | Mobile + Backend | `supabase.auth.signIn()` | API externe, peut throw |
| **OpenAI SDK** | Backend | `openai.chat.completions.create()` | API externe, peut throw |
| **File System** | Mobile | `FileSystem.readAsStringAsync()` | Expo API, peut throw |
| **Native Modules** | Mobile | `whisper.transcribe()` | Module natif, peut throw |
| **RabbitMQ** | Backend | `channel.sendToQueue()` | Message broker, peut throw |
| **MinIO** | Backend | `minioClient.putObject()` | S3 storage, peut throw |
| **Redis** | Backend | `redis.set()` | Cache externe, peut throw |

**Rationale :**
- Outils externes hors de notre contrôle → peuvent throw
- Try/catch isole l'exception et la convertit en Result
- Pas de propagation d'exception → pas de crash

---

### 4. Pattern Wrapper pour Librairies Externes Système

**Règle : Wrapper nécessaire UNIQUEMENT pour librairies externes système (non DB/API) qui peuvent throw**

**Quand créer un wrapper :**
1. ✅ Librairie externe système (RxJS, Lodash, Moment.js, etc.)
2. ✅ **ET** pas DB/API (déjà couvert par règle 3)
3. ✅ **ET** peut throw des exceptions

**Quand NE PAS créer de wrapper :**
- ❌ Nos propres classes custom (Logger, Analytics, SyncQueue, etc.) → retournent **déjà** `Result<T>`
- ❌ DB/API externes → try/catch directement autorisé (règle 3)

---

#### Exemple : EventBus Wrapper (RxJS)

**Cas d'usage :** Notre projet utilise **RxJS** (librairie externe) pour l'EventBus. RxJS peut throw des exceptions → wrapper nécessaire.

```typescript
// infrastructure/event-bus/EventBusWrapper.ts

export class EventBusWrapper implements IEventBus {
  constructor(private rxjsEventBus: Subject<DomainEvent>) {}

  /**
   * Wrapper qui convertit exceptions RxJS → Result
   */
  publish<T>(event: DomainEvent<T>): Result<void> {
    try {
      // RxJS peut throw si subject errored/completed
      this.rxjsEventBus.next(event);
      return success(undefined);
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : "Unknown error";
      return unknownError(`EventBus publish failed: ${errorMessage}`);
    }
  }

  subscribe<T>(eventType: string, handler: EventHandler<T>): Result<Subscription> {
    try {
      const subscription = this.rxjsEventBus
        .pipe(filter(e => e.type === eventType))
        .subscribe(handler);
      return success(subscription);
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : "Unknown error";
      return unknownError(`EventBus subscribe failed: ${errorMessage}`);
    }
  }
}

// DI registration - Wrapper est injecté partout
container.registerSingleton<IEventBus>(TOKENS.IEventBus, EventBusWrapper);
```

**Utilisation dans repository - PAS de try/catch :**

```typescript
// ✅ CORRECT - EventBus retourne Result, pas de try/catch nécessaire
async create(data: CreateCaptureData): Promise<Result<Capture>> {
  try {
    // Try/catch UNIQUEMENT pour DB externe
    database.execute("INSERT INTO captures...");
    const capture = mapRowToCapture(row);

    // EventBus wrapper retourne Result - PAS de try/catch
    const publishResult = this.eventBus.publish({
      type: "CaptureRecorded",
      payload: { captureId: capture.id }
    });

    // Gérer l'erreur explicitement
    if (publishResult.type !== ResultType.SUCCESS) {
      console.error("[Repository] Event publish failed:", publishResult.error);
      // Continue - capture est créée, event peut attendre
    }

    return success(capture);
  } catch (error) {
    // Catch UNIQUEMENT pour DB
    return databaseError(`Failed to create capture: ${error.message}`);
  }
}
```

---

#### Contre-exemples : PAS de wrapper nécessaire

**SyncQueue, Logger, Analytics, SyncTrigger** sont **nos classes custom** → elles retournent **déjà** `Result<T>` !

```typescript
// ✅ CORRECT - SyncQueueService (classe custom) retourne déjà Result
export class SyncQueueService implements ISyncQueueService {
  async enqueue(entity: string, id: string, operation: string): Promise<Result<void>> {
    try {
      // SQLite = DB externe → try/catch autorisé (règle 3)
      database.execute(
        "INSERT INTO sync_queue (entity, entity_id, operation) VALUES (?, ?, ?)",
        [entity, id, operation]
      );
      return success(undefined);
    } catch (error) {
      return databaseError(`Sync queue enqueue failed: ${error.message}`);
    }
  }
}

// Pas besoin de SyncQueueWrapper - SyncQueueService retourne déjà Result !
```

```typescript
// ✅ CORRECT - Logger (classe custom) retourne déjà Result
export class Logger implements ILogger {
  info(message: string, context?: Record<string, any>): Result<void> {
    try {
      console.log(`[INFO] ${message}`, context); // Console native peut throw (rare)
      return success(undefined);
    } catch (error) {
      return success(undefined); // Silent fail pour logger
    }
  }
}

// Pas besoin de LoggerWrapper - Logger retourne déjà Result !
```

```typescript
// ✅ CORRECT - AnalyticsService (classe custom) retourne déjà Result
export class AnalyticsService implements IAnalyticsService {
  async track(event: string, properties?: Record<string, any>): Promise<Result<void>> {
    try {
      // fetch = API externe → try/catch autorisé (règle 3)
      await fetch('/analytics', {
        method: 'POST',
        body: JSON.stringify({ event, properties })
      });
      return success(undefined);
    } catch (error) {
      return networkError(`Analytics track failed: ${error.message}`);
    }
  }
}

// Pas besoin de AnalyticsWrapper - AnalyticsService retourne déjà Result !
```

---

**Librairies externes système pouvant nécessiter wrappers :**

| Librairie | Wrapper nécessaire ? | Raison |
|-----------|---------------------|--------|
| **RxJS** | ✅ OUI | Système externe, peut throw (subject errored/completed) |
| **Lodash** | ⚠️ Rare | Fonctions pures normalement, wrapper si throw détecté |
| **Moment.js / date-fns** | ⚠️ Rare | Peuvent throw sur dates invalides, wrapper si nécessaire |
| **Crypto (Node.js)** | ⚠️ Rare | Peut throw, wrapper si utilisé directement |

**Nos classes custom - PAS de wrapper :**

| Classe | Wrapper ? | Pourquoi PAS de wrapper |
|--------|-----------|------------------------|
| **Logger** | ❌ NON | Notre code → retourne déjà Result |
| **Analytics** | ❌ NON | Notre code → retourne déjà Result |
| **SyncQueue** | ❌ NON | Notre code → retourne déjà Result (utilise DB avec try/catch autorisé) |
| **SyncTrigger** | ❌ NON | Notre code → retourne déjà Result |

**Rationale :**
- Wrappers UNIQUEMENT pour isolation librairies externes système
- Nos classes custom contrôlées → retournent déjà Result
- DB/API couvertes par règle 3 → pas besoin de wrapper supplémentaire

---

### 5. Root Handler Technique (Safety Net)

**Règle : Global error handler au root de l'application uniquement**

**Mobile (React Native) :**

```typescript
// mobile/src/infrastructure/error-handlers/global-error-handler.ts

import { ErrorUtils } from 'react-native';

export function setupGlobalErrorHandler() {
  // Uncaught JS exceptions
  ErrorUtils.setGlobalHandler((error, isFatal) => {
    console.error('[GlobalErrorHandler] Uncaught exception:', error);

    if (isFatal) {
      // Log to crash reporting service (Sentry, Crashlytics)
      logToErrorTracking(error, { isFatal: true });

      // Graceful degradation
      Alert.alert(
        'Erreur inattendue',
        'Une erreur est survenue. L\'application va redémarrer.',
        [{ text: 'OK', onPress: () => RNRestart.Restart() }]
      );
    } else {
      // Non-fatal - log seulement
      logToErrorTracking(error, { isFatal: false });
    }
  });

  // Unhandled promise rejections
  const promiseRejectionTracker = require('promise/setimmediate/rejection-tracking');
  promiseRejectionTracker.enable({
    allRejections: true,
    onUnhandled: (id, error) => {
      console.error('[GlobalErrorHandler] Unhandled promise rejection:', error);
      logToErrorTracking(error, { type: 'unhandled_promise' });
    },
    onHandled: (id) => {
      console.log('[GlobalErrorHandler] Promise rejection handled:', id);
    }
  });
}

// React Error Boundary
export class GlobalErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    console.error('[ErrorBoundary] Component error:', error);
    logToErrorTracking(error, { componentStack: errorInfo.componentStack });
  }

  render() {
    if (this.state.hasError) {
      return (
        <View>
          <Text>Une erreur est survenue.</Text>
          <Button title="Recharger" onPress={() => this.setState({ hasError: false })} />
        </View>
      );
    }

    return this.props.children;
  }
}
```

**Backend (NestJS) :**

```typescript
// backend/src/infrastructure/filters/global-exception.filter.ts

import { ExceptionFilter, Catch, ArgumentsHost, HttpException, HttpStatus } from '@nestjs/common';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Internal server error';

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      message = exception.message;
    } else if (exception instanceof Error) {
      message = exception.message;
    }

    // Log to error tracking
    console.error('[GlobalExceptionFilter] Uncaught exception:', exception);
    logToErrorTracking(exception, {
      url: request.url,
      method: request.method,
      body: request.body,
    });

    // Return user-friendly error
    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message,
    });
  }
}

// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Register global exception filter
  app.useGlobalFilters(new GlobalExceptionFilter());

  await app.listen(3000);
}
```

**Rationale :**
- Dernière ligne de défense → évite crashes complets
- Log pour diagnostic → Sentry, Crashlytics, CloudWatch
- Graceful degradation → UI affiche erreur user-friendly
- Pas de remplacement du Result Pattern → seulement safety net

---

### 6. Pattern par Couche (DDD)

**Règle : Chaque couche a un rôle spécifique dans la gestion d'erreurs**

#### Domain Layer (Pure Logic)

```typescript
// domain/Capture.model.ts

// ✅ CORRECT - Pure domain logic, JAMAIS de try/catch, JAMAIS d'I/O
export class Capture {
  static validateRawContent(content: string): Result<string> {
    if (!content || content.trim().length === 0) {
      return validationError("rawContent cannot be empty");
    }

    if (content.length > 100000) {
      return validationError("rawContent exceeds maximum length");
    }

    return success(content.trim());
  }

  static calculateDuration(startTime: number, endTime: number): Result<number> {
    const duration = endTime - startTime;

    if (duration < 0) {
      return businessError("endTime must be after startTime");
    }

    if (duration > 24 * 60 * 60 * 1000) {
      return businessError("duration cannot exceed 24 hours");
    }

    return success(duration);
  }
}
```

**Règles domain :**
- ❌ JAMAIS de try/catch (pas d'opérations externes)
- ❌ JAMAIS d'I/O (DB, API, File System)
- ✅ Pure functions avec Result<T>
- ✅ Business validation rules

---

#### Repository Layer (Data Access)

```typescript
// data/CaptureRepository.ts

export class CaptureRepository implements ICaptureRepository {
  constructor(
    @inject(TOKENS.IEventBus) private eventBus: IEventBus,
    @inject(TOKENS.ISyncQueueService) private syncQueue: ISyncQueueService
  ) {}

  async create(data: CreateCaptureData): Promise<Result<Capture>> {
    // Try/catch UNIQUEMENT pour DB externe
    try {
      database.execute("INSERT INTO captures...", [...]);
      const row = database.execute("SELECT * FROM captures WHERE id = ?", [id]);

      if (!row.rows || row.rows.length === 0) {
        return databaseError("Failed to retrieve created capture");
      }

      const capture = mapRowToCapture(row.rows[0]);

      // EventBus wrapper retourne Result - PAS de try/catch
      const publishResult = this.eventBus.publish({
        type: "CaptureRecorded",
        payload: { captureId: capture.id }
      });

      if (publishResult.type !== ResultType.SUCCESS) {
        console.error("[Repository] Event failed:", publishResult.error);
      }

      // SyncQueue wrapper retourne Result - PAS de try/catch
      const enqueueResult = await this.syncQueue.enqueue("capture", capture.id, "create");

      if (enqueueResult.type !== ResultType.SUCCESS) {
        console.error("[Repository] Queue failed:", enqueueResult.error);
      }

      return success(capture);
    } catch (error) {
      // Catch UNIQUEMENT pour DB externe
      const errorMessage = error instanceof Error ? error.message : "Unknown error";
      return databaseError(`Failed to create capture: ${errorMessage}`);
    }
  }

  async findById(id: string): Promise<Result<Capture>> {
    try {
      const result = database.execute("SELECT * FROM captures WHERE id = ?", [id]);

      if (!result.rows || result.rows.length === 0) {
        return notFound(`Capture ${id} not found`);
      }

      const capture = mapRowToCapture(result.rows[0]);
      return success(capture);
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : "Unknown error";
      return databaseError(`Failed to find capture: ${errorMessage}`);
    }
  }
}
```

**Règles repository :**
- ✅ Try/catch UNIQUEMENT pour DB externe
- ✅ Wrappers (EventBus, SyncQueue) retournent Result → pas de try/catch
- ✅ Retourne toujours `Result<T>` (jamais throw)
- ❌ Pas de business logic (domain layer only)

---

#### Service Layer (Application Logic)

```typescript
// services/CaptureService.ts

export class CaptureService {
  constructor(
    private captureRepo: ICaptureRepository,
    private normalizationService: INormalizationService
  ) {}

  async createCapture(data: CreateCaptureData): Promise<Result<Capture>> {
    // Validation domain
    const validationResult = Capture.validateRawContent(data.rawContent);
    if (validationResult.type !== ResultType.SUCCESS) {
      return validationResult as Result<Capture>;
    }

    // Appel repository
    const createResult = await this.captureRepo.create(data);
    if (createResult.type !== ResultType.SUCCESS) {
      return createResult; // Propage l'erreur repository
    }

    const capture = createResult.data!;

    // Normalization (wrapper retourne Result)
    const normalizeResult = await this.normalizationService.normalize(capture.rawContent);

    if (normalizeResult.type !== ResultType.SUCCESS) {
      console.warn('[CaptureService] Normalization failed:', normalizeResult.error);
      // Continue - capture est déjà créée
    } else {
      // Update capture avec normalized text
      await this.captureRepo.update(capture.id, {
        normalizedText: normalizeResult.data
      });
    }

    return success(capture);
  }
}
```

**Règles service :**
- ✅ Compose plusieurs Result<T> (domain + repository)
- ✅ Propage les erreurs ou les transforme si nécessaire
- ❌ JAMAIS de try/catch (dépendances retournent Result)
- ❌ Try/catch UNIQUEMENT si appel API externe direct (rare)

---

#### Controller/API Layer (HTTP)

```typescript
// controllers/CaptureController.ts (Backend NestJS)

@Controller('captures')
export class CaptureController {
  constructor(private captureService: CaptureService) {}

  @Post()
  async createCapture(@Body() dto: CreateCaptureDto): Promise<CaptureResponseDto> {
    const result = await this.captureService.createCapture(dto);

    // Map Result → HTTP response
    switch (result.type) {
      case ResultType.SUCCESS:
        return {
          success: true,
          data: result.data,
        };

      case ResultType.VALIDATION_ERROR:
        throw new BadRequestException(result.error);

      case ResultType.NOT_FOUND:
        throw new NotFoundException(result.error);

      case ResultType.DATABASE_ERROR:
        throw new InternalServerErrorException(result.error);

      case ResultType.AUTH_ERROR:
        throw new UnauthorizedException(result.error);

      default:
        // TypeScript exhaustivité check
        const _exhaustive: never = result.type;
        throw new InternalServerErrorException('Unknown error type');
    }
  }

  @Get(':id')
  async getCapture(@Param('id') id: string): Promise<CaptureResponseDto> {
    const result = await this.captureService.findById(id);

    switch (result.type) {
      case ResultType.SUCCESS:
        return {
          success: true,
          data: result.data,
        };

      case ResultType.NOT_FOUND:
        throw new NotFoundException(result.error);

      default:
        throw new InternalServerErrorException(result.error);
    }
  }
}
```

**Règles controller :**
- ✅ Map Result<T> → HTTP status codes
- ✅ Switch exhaustif sur ResultType
- ✅ Throw HttpException pour NestJS (seule exception au "no throw")
- ❌ Pas de business logic (service layer only)

---

#### UI Layer (React/React Native)

```typescript
// screens/CaptureScreen.tsx

export function CaptureScreen() {
  const [error, setError] = useState<string | null>(null);
  const captureService = useCaptureService();

  const handleCreateCapture = async (data: CreateCaptureData) => {
    const result = await captureService.createCapture(data);

    switch (result.type) {
      case ResultType.SUCCESS:
        Toast.show({ type: 'success', text1: 'Capture créée' });
        navigation.goBack();
        break;

      case ResultType.VALIDATION_ERROR:
        setError(result.error);
        Toast.show({ type: 'error', text1: result.error });
        break;

      case ResultType.DATABASE_ERROR:
        setError('Erreur de base de données');
        Toast.show({ type: 'error', text1: 'Erreur technique, réessayez' });
        break;

      case ResultType.NETWORK_ERROR:
        setError('Erreur réseau');
        Toast.show({ type: 'error', text1: 'Pas de connexion, réessayez' });
        break;

      default:
        setError('Erreur inconnue');
        Toast.show({ type: 'error', text1: 'Une erreur est survenue' });
    }
  };

  return (
    <View>
      <Button title="Créer capture" onPress={handleCreateCapture} />
      {error && <Text style={styles.error}>{error}</Text>}
    </View>
  );
}
```

**Règles UI :**
- ✅ Switch exhaustif sur ResultType
- ✅ Affichage user-friendly des erreurs
- ✅ Toast/Alert pour feedback immédiat
- ❌ JAMAIS de try/catch (services retournent Result)

---

## Consequences

### ✅ Bénéfices

1. **Stabilité maximale**
   - Pas d'exceptions non catchées → 0 crash
   - NFR "0 capture perdue" garanti
   - Global error handler = safety net ultime

2. **Code prévisible et simple**
   - Result force gestion explicite des erreurs
   - TypeScript vérifie exhaustivité
   - Pas de try/catch dispersés dans le code métier

3. **Maintenabilité excellente**
   - Pattern unifié → onboarding simplifié
   - Code review facile (règle stricte claire)
   - Wrappers isolent la complexité

4. **Retry logic centralisée**
   - Flag `retryable` → stratégie retry unifiée (ADR-009)
   - Pas de logique retry dispersée

5. **Debugging facilité**
   - Erreurs explicites avec contexte
   - Logs structurés
   - Stack traces au bon endroit (wrappers)

### ⚠️ Trade-offs acceptés

1. **Verbosité accrue**
   - Switch sur Result vs simple try/catch
   - **Mitigation** : Helpers + wrappers réduisent boilerplate

2. **Wrappers nécessaires**
   - EventBus, SyncQueue, Logger nécessitent wrappers
   - **Mitigation** : Wrappers créés une fois, réutilisés partout

3. **Courbe d'apprentissage**
   - Développeurs habitués à try/catch doivent s'adapter
   - **Mitigation** : Documentation + exemples + code review strict

4. **Migration effort**
   - 7-10 jours pour migrer code existant
   - **Mitigation** : Migration progressive par couche

### 🔄 Impact sur architecture existante

1. **ADR-009 §9.5 clarifié**
   - "Result Pattern (pas try/catch)" → "Result Pattern (try/catch UNIQUEMENT DB/API + root)"

2. **Code existant compatible**
   - `RepositoryResult<T>` déjà utilisé → migration incrémentale facile

3. **DDD architecture renforcée**
   - Séparation couches plus claire
   - Domain layer pure (0 try/catch)
   - Wrappers dans infrastructure layer

---

## Implementation

### Étapes de mise en œuvre

**Implémentation immédiate (avant migration) :**

1. ✅ **Créer ADR-023** (ce document)
2. ⏳ **Créer wrapper EventBus** (RxJS uniquement)
3. ⏳ **Vérifier classes custom** retournent Result :
   - Logger → doit retourner `Result<void>`
   - Analytics → doit retourner `Result<void>`
   - SyncQueue → doit retourner `Result<T>`
   - SyncTrigger → doit retourner `Result<void>`
4. ⏳ **Setup global error handlers** (mobile + backend)
5. ⏳ **Update `project-context.md`** avec règles strictes

**Migration incrémentale (7-10 jours) :**

6. ⏳ **Audit code existant** (1 jour)
   - Identifier tous les try/catch non autorisés
   - Vérifier classes custom retournent Result
7. ⏳ **Migrer repositories** (2-3 jours)
   - Remplacer try/catch imbriqués par appels Result
   - Vérifier try/catch uniquement pour DB/API
8. ⏳ **Migrer services** (2-3 jours)
   - Supprimer try/catch
   - Composer Result<T>
9. ⏳ **Migrer controllers** (1 jour)
   - Map Result → HTTP status
10. ⏳ **Migrer UI** (1 jour)
   - Switch sur ResultType

### Files Modified

**Création nouveaux fichiers :**
```
pensieve/mobile/src/contexts/shared/domain/Result.ts (exists - verify)
pensieve/backend/src/shared/domain/Result.ts (create)
pensieve/mobile/src/infrastructure/event-bus/EventBusWrapper.ts (create - RxJS wrapper uniquement)
pensieve/mobile/src/infrastructure/error-handlers/global-error-handler.ts (create)
pensieve/backend/src/infrastructure/filters/global-exception.filter.ts (create)
_bmad-output/project-context.md (update - add Error Handling section)
```

**Vérification classes custom (doivent déjà retourner Result) :**
```
pensieve/mobile/src/infrastructure/logger/Logger.ts (verify returns Result<void>)
pensieve/mobile/src/infrastructure/analytics/AnalyticsService.ts (verify returns Result<void>)
pensieve/mobile/src/infrastructure/sync/SyncQueueService.ts (verify returns Result<T>)
pensieve/mobile/src/infrastructure/sync/SyncTrigger.ts (verify returns Result<void>)
```

**Migration fichiers existants :**
```
pensieve/mobile/src/contexts/*/data/*Repository.ts (15+ files)
pensieve/mobile/src/contexts/*/services/*.ts (10+ files)
pensieve/backend/src/modules/*/application/services/*.ts (8+ files)
pensieve/backend/src/modules/*/application/controllers/*.ts (6+ files)
```

### Effort réel

- **ADR creation** : 3 heures (✅ done - revised twice)
- **EventBus wrapper creation** : 1 heure
- **Vérification classes custom** : 2 heures
- **Global error handlers** : 2 heures
- **project-context.md update** : 30 minutes
- **Migration code** : 7-10 jours (à planifier)

**Total effort estimé :** 7-10 jours

---

## Validation Criteria

ADR considéré succès SI :

- ✅ **EventBus wrapper créé** : RxJS wrapper opérationnel, retourne Result
- ✅ **Classes custom vérifées** : Logger, Analytics, SyncQueue, SyncTrigger retournent Result
- ✅ **Pas de try/catch illégaux** : 0 try/catch hors DB/API/root
- ✅ **Result Pattern partout** : 100% repositories/services retournent Result<T>
- ✅ **Global handlers actifs** : ErrorBoundary mobile + GlobalExceptionFilter backend opérationnels
- ✅ **Pas de crash** : 0 crash lié à exceptions non catchées (30 jours monitoring)
- ✅ **Code review pass** : 100% PRs respectent la stratégie stricte

**Review Date :** 2026-03 (après migration complète)

---

## References

- **ADR-009 §9.5** : Result Pattern pour sync (clarified et étendu par ADR-023)
- **TypeScript Result Pattern** : https://www.matthewgerstman.com/tech/typescript-result-pattern/
- **Railway Oriented Programming** : https://fsharpforfunandprofit.com/rop/
- **NestJS Exception Filters** : https://docs.nestjs.com/exception-filters
- **React Error Boundaries** : https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary

---

## Anti-Patterns - Ce qu'il NE FAUT PAS faire

### ❌ Anti-Pattern 1 : Throw dans le code applicatif

```typescript
// ❌ WRONG
async createCapture(data: CreateCaptureData): Promise<Capture> {
  if (!data.rawContent) {
    throw new Error("rawContent is required"); // FORBIDDEN
  }
  return await this.repo.save(data);
}

// ✅ CORRECT
async createCapture(data: CreateCaptureData): Promise<Result<Capture>> {
  if (!data.rawContent) {
    return validationError("rawContent is required");
  }
  return await this.repo.create(data);
}
```

---

### ❌ Anti-Pattern 2 : Try/catch sans conversion Result

```typescript
// ❌ WRONG - Try/catch qui re-throw
async create(data): Promise<Capture> {
  try {
    return database.execute(...);
  } catch (error) {
    console.error(error);
    throw error; // ❌ Re-throw = crash potentiel
  }
}

// ✅ CORRECT - Try/catch qui convertit en Result
async create(data): Promise<Result<Capture>> {
  try {
    const result = database.execute(...);
    return success(result);
  } catch (error) {
    return databaseError(error.message); // ✅ Pas de re-throw
  }
}
```

---

### ❌ Anti-Pattern 3 : Try/catch pour outil interne

```typescript
// ❌ WRONG - Try/catch pour EventBus dans repository
async create(data): Promise<Result<Capture>> {
  try {
    const capture = database.execute(...);

    try {
      this.eventBus.publish(event); // ❌ Try/catch interdit ici
    } catch (eventError) {
      console.error(eventError);
    }

    return success(capture);
  } catch (error) {
    return databaseError(error.message);
  }
}

// ✅ CORRECT - EventBus wrapper retourne Result
async create(data): Promise<Result<Capture>> {
  try {
    const capture = database.execute(...);

    // EventBus wrapper retourne Result
    const publishResult = this.eventBus.publish(event);
    if (publishResult.type !== ResultType.SUCCESS) {
      console.error("Event failed:", publishResult.error);
    }

    return success(capture);
  } catch (error) {
    return databaseError(error.message);
  }
}
```

---

### ❌ Anti-Pattern 4 : Switch non exhaustif

```typescript
// ❌ WRONG - Manque cas d'erreur
const result = await service.createCapture(data);

switch (result.type) {
  case ResultType.SUCCESS:
    showToast('Success');
    break;
  // ❌ Manque VALIDATION_ERROR, DATABASE_ERROR, etc.
}

// ✅ CORRECT - Switch exhaustif avec default
const result = await service.createCapture(data);

switch (result.type) {
  case ResultType.SUCCESS:
    showToast('Success');
    break;
  case ResultType.VALIDATION_ERROR:
    showError(result.error);
    break;
  case ResultType.DATABASE_ERROR:
    showError('Technical error');
    break;
  default:
    const _exhaustive: never = result.type;
    showError('Unknown error');
}
```

---

### ❌ Anti-Pattern 5 : Pas de wrapper pour outil système

```typescript
// ❌ WRONG - Utiliser RxJS EventBus directement
class CaptureRepository {
  constructor(private eventBus: RxJSEventBus) {} // ❌ Pas de wrapper

  async create(data): Promise<Result<Capture>> {
    try {
      const capture = database.execute(...);
      this.eventBus.publish(event); // ❌ Peut throw, pas géré
      return success(capture);
    } catch (error) {
      return databaseError(error.message);
    }
  }
}

// ✅ CORRECT - Wrapper qui retourne Result
class CaptureRepository {
  constructor(
    @inject(TOKENS.IEventBus) private eventBus: IEventBus // ✅ Wrapper interface
  ) {}

  async create(data): Promise<Result<Capture>> {
    try {
      const capture = database.execute(...);

      const publishResult = this.eventBus.publish(event); // ✅ Retourne Result
      if (publishResult.type !== ResultType.SUCCESS) {
        console.error("Event failed:", publishResult.error);
      }

      return success(capture);
    } catch (error) {
      return databaseError(error.message);
    }
  }
}
```

---

## Decision Log

**2026-02-15** - Discussion yohikofox + Winston

**Context :**
- Story 6-2 (sync) en cours de review
- Détection d'ambiguïté ADR-009 §9.5 "Result Pattern (pas try/catch)"
- Code utilise try/catch mais documentation dit "pas try/catch"
- Clarification demandée par yohikofox

**Clarification règle stricte :**
- Try/catch UNIQUEMENT pour DB et API externes (services tiers)
- Try/catch technique au root (global error handler)
- TOUT le reste retourne Result
- Si outil throw → créer wrapper qui convertit → Result
- Pas d'exception à cette règle

**Trade-offs discutés :**
- Option A : Try/catch autorisés pour "opérations auxiliaires" → ❌ Rejeté (incohérent)
- Option B : Try/catch UNIQUEMENT DB/API + wrappers pour le reste → ✅ Choisi

**Décision finale :**
- Result Pattern obligatoire partout
- Try/catch strictement limité : DB, API, root handler
- Wrappers pour outils système (EventBus, SyncQueue, Logger)
- Throw interdit sans exception

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

---

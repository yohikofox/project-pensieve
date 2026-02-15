---
adr: ADR-024
title: "Standards Clean Code Appliqués au Projet Pensieve"
date: 2026-02-15
status: "✅ Accepted"
context: "Phase 3 - Solutioning - Standardisation qualité code"
participants:
  - yohikofox (Product Owner)
  - Winston (Architect)
supersedes: "N/A - Nouvelle décision architecturale"
---

# ADR-024: Standards Clean Code Appliqués au Projet Pensieve

**Date:** 2026-02-15
**Status:** ✅ Accepted
**Context:** Définition des règles Clean Code (Uncle Bob) applicables au projet Pensieve
**Decision Makers:** yohikofox (Product Owner), Winston (Architect)

---

## Context & Problem

### Problème à résoudre

Le projet Pensieve est développé par plusieurs contributeurs (développeurs, agents AI) avec des standards de qualité code variables. Sans règles explicites et non-négociables, le code risque de devenir :

**Risques identifiés :**

1. **Inconsistance** : Styles de nommage divergents, patterns contradictoires
2. **Dette technique accumulée** : Magic numbers, code dupliqué, fonctions trop longues
3. **Maintenabilité dégradée** : Difficultés à refactoriser, bug fixes lents
4. **Onboarding difficile** : Nouveaux développeurs perdus dans un code imprévisible
5. **Code reviews interminables** : Pas de standards clairs pour valider les PRs

### Contraintes identifiées

1. **Dogmatisme dangereux** : Certaines règles Clean Code d'Uncle Bob sont trop extrêmes (fonctions 5-10 lignes max)
2. **Pragmatisme nécessaire** : Startup/MVP → vitesse vs. perfection
3. **Multi-plateformes** : Mobile (React Native), Backend (NestJS), Web (Next.js) → règles doivent s'adapter
4. **TypeScript strict mode** : Déjà en place → tirer parti du typage fort
5. **Équipe existante** : Développeurs habitués à certaines conventions

### Motivation

**Sans standards Clean Code clairs :**
- ❌ Code reviews subjectives et conflictuelles
- ❌ Refactoring risqué (pas de patterns clairs)
- ❌ Bugs récurrents (magic numbers, side effects cachés)
- ❌ Onboarding lent (6-8 semaines)

**Avec ADR-024 :**
- ✅ Règles objectives pour code reviews
- ✅ Refactoring sécurisé (standards documentés)
- ✅ Bugs prévenus (validations strictes)
- ✅ Onboarding rapide (2-3 semaines)

---

## Decision

### Décision architecturale

**Pensieve adopte un sous-ensemble pragmatique des règles Clean Code d'Uncle Bob, organisé en 3 niveaux de criticité : NON-NÉGOCIABLES, FORTEMENT RECOMMANDÉS, et CONTEXTUELS.**

Cette décision se décompose en **8 catégories de règles** :

---

### 1. NOMMAGE - Standards Obligatoires

#### 1.1 Noms Révélateurs d'Intention
**Règle :** Les noms doivent expliquer clairement ce que fait le code, sans commentaire.

```typescript
// ❌ WRONG
function get(id: string): Promise<Result<User>> { }
const d = new Date();

// ✅ CORRECT
function getUserById(id: string): Promise<Result<User>> { }
const creationDate = new Date();
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

#### 1.2 Noms Prononçables
**Règle :** Les noms doivent pouvoir être prononcés facilement pour faciliter la communication.

```typescript
// ❌ WRONG
const genYmdhms = Date.now();
const usrActSub = getUserActiveSubscription();

// ✅ CORRECT
const generationTimestamp = Date.now();
const userActiveSubscription = getUserActiveSubscription();
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

#### 1.3 Noms Cherchables & Constantes Nommées
**Règle :** Pas de magic numbers - toutes les valeurs littérales doivent être des constantes nommées.

```typescript
// ❌ WRONG
if (user.status === 3) { }
setTimeout(callback, 86400000);

// ✅ CORRECT
const UserStatus = {
  ACTIVE: 1,
  INACTIVE: 2,
  SUSPENDED: 3,
} as const;

const MILLISECONDS_PER_DAY = 24 * 60 * 60 * 1000;

if (user.status === UserStatus.SUSPENDED) { }
setTimeout(callback, MILLISECONDS_PER_DAY);
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

#### 1.4 Convention Interfaces - Suffixe "Contract"
**Règle :** Les interfaces doivent porter le suffixe `Contract` plutôt que le préfixe `I`.

```typescript
// ❌ WRONG (convention C# legacy)
interface IUser { }
interface IUserRepository { }

// ✅ CORRECT (Pensieve convention)
interface UserContract {
  id: string;
  email: string;
}

// ⚠️ EXCEPTION: Interfaces de services/repositories gardent le préfixe I (convention établie)
interface IUserRepository {
  findById(id: string): Promise<Result<User>>;
}
```

**Rationale :** Suffixe `Contract` rend explicite le rôle d'interface de contrat. Préfixe `I` réservé aux services/repositories (convention DDD existante).

**Niveau :** 🟡 FORTEMENT RECOMMANDÉ pour nouveaux contrats, TOLÉRANCE pour code existant

---

#### 1.5 Pas d'Encodage Hongroise
**Règle :** Pas de préfixes de type (strName, arrUsers, etc.) - TypeScript gère le typage.

```typescript
// ❌ WRONG
const strUserName: string = "John";
const arrUsers: User[] = [];
const bIsActive: boolean = true;

// ✅ CORRECT
const userName: string = "John";
const users: User[] = [];
const isActive: boolean = true;
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

### 2. FONCTIONS - Règles de Structure

#### 2.1 Fonctions Courtes
**Règle :** Viser < 20 lignes idéalement, < 30 lignes acceptable, > 50 lignes signal de refactoring.

```typescript
// ❌ WRONG - 80 lignes, multiple responsabilités
async function processCapture(captureId: string) {
  // Validation (10 lignes)
  // Normalization (20 lignes)
  // Transcription (25 lignes)
  // Sync queue (15 lignes)
  // Event publish (10 lignes)
}

// ✅ CORRECT - Split par responsabilité
async function processCapture(captureId: string): Promise<Result<void>> {
  const validateResult = await validateCapture(captureId);
  if (validateResult.type !== ResultType.SUCCESS) return validateResult;

  const normalizeResult = await normalizeCapture(captureId);
  if (normalizeResult.type !== ResultType.SUCCESS) return normalizeResult;

  const transcribeResult = await transcribeCapture(captureId);
  if (transcribeResult.type !== ResultType.SUCCESS) return transcribeResult;

  await enqueueSyncJob(captureId);
  await publishCaptureProcessedEvent(captureId);

  return success(undefined);
}
```

**Niveau :** 🟡 FORTEMENT RECOMMANDÉ (guideline, pas règle stricte)

---

#### 2.2 Single Responsibility Principle (SRP)
**Règle :** Une fonction = une seule responsabilité clairement définie.

```typescript
// ❌ WRONG - Multiple responsabilités
function saveUserAndSendEmail(user: User) {
  database.save(user);
  emailService.send(user.email, 'Welcome');
  analytics.track('user_created', { userId: user.id });
}

// ✅ CORRECT - Séparation claire
function saveUser(user: User): Promise<Result<User>> {
  return userRepository.create(user);
}

function sendWelcomeEmail(user: User): Promise<Result<void>> {
  return emailService.send(user.email, 'Welcome');
}

function trackUserCreation(user: User): Result<void> {
  return analytics.track('user_created', { userId: user.id });
}

// Composition dans orchestrateur
async function createUser(data: CreateUserData): Promise<Result<User>> {
  const saveResult = await saveUser(data);
  if (saveResult.type !== ResultType.SUCCESS) return saveResult;

  await sendWelcomeEmail(saveResult.data);
  trackUserCreation(saveResult.data);

  return saveResult;
}
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

#### 2.3 Un Seul Niveau d'Abstraction
**Règle :** Ne pas mélanger haut niveau et bas niveau dans la même fonction.

```typescript
// ❌ WRONG - Niveaux d'abstraction mélangés
async function processOrder(orderId: string) {
  const order = await orderRepository.findById(orderId);

  // ⬇️ BAS NIVEAU - Détails SQL/DB
  const query = `UPDATE orders SET status = ? WHERE id = ?`;
  database.execute(query, ['processed', orderId]);

  // ⬆️ HAUT NIVEAU - Business logic
  await paymentService.charge(order.amount);
  await emailService.sendConfirmation(order.customerEmail);
}

// ✅ CORRECT - Niveau d'abstraction uniforme
async function processOrder(orderId: string): Promise<Result<void>> {
  const order = await orderRepository.findById(orderId);
  if (order.type !== ResultType.SUCCESS) return order;

  await orderRepository.markAsProcessed(orderId);
  await paymentService.charge(order.data.amount);
  await emailService.sendConfirmation(order.data.customerEmail);

  return success(undefined);
}
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

#### 2.4 Pas d'Effets de Bord Cachés
**Règle :** Les fonctions doivent être pures quand possible, sinon les side effects doivent être explicites.

```typescript
// ❌ WRONG - Side effect caché
function getUserName(user: User): string {
  analytics.track('user_name_accessed'); // ❌ Side effect non documenté
  return user.name;
}

// ✅ CORRECT - Pure function
function getUserName(user: User): string {
  return user.name;
}

// ✅ CORRECT - Side effect explicite (nom de fonction)
function getUserNameAndTrack(user: User): string {
  analytics.track('user_name_accessed');
  return user.name;
}
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

#### 2.5 Arguments Limités - Options Object Préféré
**Règle :** Max 3 paramètres primitifs, sinon utiliser un options object.

```typescript
// ❌ WRONG - Trop de paramètres
function createUser(
  name: string,
  email: string,
  phone: string,
  country: string,
  tier: string,
  role: string
) { }

// ✅ CORRECT - Options object
interface CreateUserOptions {
  name: string;
  email: string;
  phone: string;
  country: string;
  tier: string;
  role: string;
}

function createUser(options: CreateUserOptions): Promise<Result<User>> {
  // Destructuring pour clarté
  const { name, email, phone, country, tier, role } = options;
  // ...
}
```

**Niveau :** 🔴 NON-NÉGOCIABLE (> 3 paramètres)

---

#### 2.6 Pas de Flag Arguments - Feature Flags Explicites OK
**Règle :** Éviter les booleans bruts, préférer enums/constantes ou feature flags explicites.

```typescript
// ❌ WRONG - Boolean opaque
render(true);
processOrder(orderId, false);

// ✅ CORRECT - Enum explicite
enum RenderMode {
  PRODUCTION = 'production',
  DEBUG = 'debug',
}

render(RenderMode.PRODUCTION);

// ✅ CORRECT - Feature flag explicite
render({ enableExperimentalFeature: featureFlags.newRenderer });

// ✅ ACCEPTABLE - Named object parameter
processOrder(orderId, { skipValidation: false });
```

**Niveau :** 🟡 FORTEMENT RECOMMANDÉ

---

#### 2.7 Command/Query Separation
**Règle :** Séparer fonctions qui modifient l'état (commands) de celles qui lisent (queries) **au niveau domaine/fonctionnel**. Side effects techniques (logs, metrics) sont tolérés.

```typescript
// ❌ WRONG - Command + Query mélangés (domaine)
function saveUser(user: User): User {
  database.save(user);
  return database.findById(user.id); // ❌ Re-query après save
}

// ✅ CORRECT - Command pur
function saveUser(user: User): Promise<Result<void>> {
  logger.info('Saving user', { userId: user.id }); // ✅ OK - technique
  return userRepository.create(user);
}

// ✅ CORRECT - Query pur
function getUserById(id: string): Promise<Result<User>> {
  logger.info('Fetching user', { userId: id }); // ✅ OK - technique
  return userRepository.findById(id);
}
```

**Niveau :** 🔴 NON-NÉGOCIABLE (séparation fonctionnelle)

---

### 3. COMMENTAIRES - Règles de Documentation

#### 3.1 Code Auto-Documenté Prioritaire
**Règle :** Le code doit être lisible sans commentaires. Commentaires = échec du code.

```typescript
// ❌ WRONG - Commentaire pour expliquer code obscur
// Check if user is active and has premium tier
if (u.s === 1 && u.t === 'p') { }

// ✅ CORRECT - Code explicite sans commentaire
const isUserActive = user.status === UserStatus.ACTIVE;
const hasPremiumTier = user.tier === TierType.PREMIUM;

if (isUserActive && hasPremiumTier) { }
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

#### 3.2 TODO Interdit Sans Ticket
**Règle :** Aucun `// TODO` accepté sans ticket associé. Préférer créer une issue immédiatement.

```typescript
// ❌ WRONG
// TODO: Optimize this query later

// ❌ WRONG
// FIXME: This is slow

// ✅ ACCEPTABLE - Avec ticket et deadline
// FIXME(JIRA-1234): Optimize query performance - deadline 2026-02-20

// ✅ MEILLEUR - Créer issue/ticket immédiatement au lieu de TODO
```

**Git Hook Recommandé :**
```bash
# .git/hooks/pre-commit
if git diff --cached | grep -E "//\s*TODO(?!\()"; then
  echo "❌ Commit rejected: TODO without ticket found"
  echo "Use format: // TODO(TICKET-ID): description"
  exit 1
fi
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

#### 3.3 Commentaires Légaux et Warnings Autorisés
**Règle :** Commentaires pour copyright, licences, et warnings techniques sont nécessaires.

```typescript
// ✅ CORRECT - Legal
/**
 * Copyright (c) 2026 Pensieve
 * Licensed under MIT
 */

// ✅ CORRECT - Warning
/**
 * ⚠️ WARNING: This function is NOT thread-safe
 * Must be called within a transaction lock
 */
function updateInventory(item: Item) { }

// ✅ CORRECT - Complex algorithm justification
/**
 * Uses Levenshtein distance for fuzzy matching
 * Time complexity: O(n*m) - acceptable for our use case (n,m < 100)
 */
function fuzzyMatch(a: string, b: string): number { }
```

**Niveau :** 🔴 NON-NÉGOCIABLE (legal), 🟡 FORTEMENT RECOMMANDÉ (warnings)

---

#### 3.4 Pas de Code Commenté
**Règle :** Supprimer immédiatement le code commenté - on a Git pour l'historique.

```typescript
// ❌ WRONG
function processCapture(id: string) {
  // const oldLogic = doSomethingOld();
  // if (oldLogic) {
  //   return handleOldWay();
  // }

  const newLogic = doSomethingNew();
  return handleNewWay(newLogic);
}

// ✅ CORRECT
function processCapture(id: string): Promise<Result<void>> {
  const newLogic = doSomethingNew();
  return handleNewWay(newLogic);
}
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

### 4. FORMATAGE - Standards Automatisés

**Règle :** Utiliser Prettier + ESLint avec configuration stricte - ZERO tolérance.

```json
// .prettierrc
{
  "singleQuote": true,
  "trailingComma": "all",
  "tabWidth": 2,
  "semi": true,
  "printWidth": 100
}
```

**Règles strictes :**
- ✅ Vertical formatting : Lignes vides entre concepts distincts
- ✅ Densité verticale : Grouper code lié sans lignes vides excessives
- ✅ Variables déclarées proche usage : `const`/`let` au point d'utilisation
- ✅ Indentation stricte : 2 espaces (TypeScript convention)
- ✅ Fichiers < 300 lignes : Confortable, > 500 lignes → évaluer split

**Niveau :** 🔴 NON-NÉGOCIABLE (automatisé via pre-commit hooks)

---

### 5. PRINCIPES SOLID - Architecture

#### 5.1 Single Responsibility Principle (SRP)
**Règle :** Une classe/fonction = une seule raison de changer.

```typescript
// ❌ WRONG - Multiple responsabilités
class UserService {
  createUser(data: CreateUserData) { }
  sendWelcomeEmail(user: User) { }
  trackAnalytics(event: string) { }
  validateCreditCard(card: string) { }
}

// ✅ CORRECT - Séparation claire
class UserService {
  createUser(data: CreateUserData): Promise<Result<User>> { }
}

class EmailService {
  sendWelcomeEmail(user: User): Promise<Result<void>> { }
}

class AnalyticsService {
  track(event: string, data: Record<string, any>): Result<void> { }
}

class PaymentService {
  validateCreditCard(card: string): Result<boolean> { }
}
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

#### 5.2 Open/Closed Principle (OCP) - Rule of Three
**Règle :** Créer abstractions SEULEMENT quand le 3ème cas d'usage apparaît.

```typescript
// 1ère fois - Implémentation concrète
class StripePayment {
  process(amount: number): Promise<Result<void>> { }
}

// 2ème fois - Dupliquer temporairement (oui, vraiment!)
class PaypalPayment {
  process(amount: number): Promise<Result<void>> { }
}

// 3ème fois - MAINTENANT abstraire (pattern clair)
interface IPaymentProcessor {
  process(amount: number): Promise<Result<void>>;
}

class StripePayment implements IPaymentProcessor { }
class PaypalPayment implements IPaymentProcessor { }
class CryptoPayment implements IPaymentProcessor { }
```

**Rationale :** Éviter abstractions prématurées - YAGNI (You Ain't Gonna Need It).

**Niveau :** 🟡 FORTEMENT RECOMMANDÉ (Rule of Three), ⚠️ MODÉRÉ (abstraction immédiate)

---

#### 5.3 Liskov Substitution Principle (LSP)
**Règle :** Sous-types doivent être substituables à leur type parent.

```typescript
// ❌ WRONG - Violation LSP
class Rectangle {
  width: number;
  height: number;

  setWidth(w: number) { this.width = w; }
  setHeight(h: number) { this.height = h; }
}

class Square extends Rectangle {
  setWidth(w: number) {
    this.width = w;
    this.height = w; // ❌ Comportement différent du parent
  }
}

// ✅ CORRECT - Composition au lieu d'héritage
interface Shape {
  area(): number;
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}
  area(): number { return this.width * this.height; }
}

class Square implements Shape {
  constructor(private side: number) {}
  area(): number { return this.side * this.side; }
}
```

**Niveau :** 🟡 FORTEMENT RECOMMANDÉ (si héritage utilisé)

---

#### 5.4 Interface Segregation Principle (ISP)
**Règle :** Interfaces ciblées - pas d'interfaces "grasses" avec méthodes inutiles.

```typescript
// ❌ WRONG - Interface trop large
interface IUserRepository {
  findById(id: string): Promise<Result<User>>;
  findByEmail(email: string): Promise<Result<User>>;
  findAll(): Promise<Result<User[]>>;
  create(user: User): Promise<Result<User>>;
  update(id: string, data: Partial<User>): Promise<Result<User>>;
  delete(id: string): Promise<Result<void>>;
  exportToCSV(): Promise<Result<string>>;
  generateReport(): Promise<Result<Report>>;
}

// ✅ CORRECT - Interfaces ségrégées
interface IUserReader {
  findById(id: string): Promise<Result<User>>;
  findByEmail(email: string): Promise<Result<User>>;
}

interface IUserWriter {
  create(user: User): Promise<Result<User>>;
  update(id: string, data: Partial<User>): Promise<Result<User>>;
  delete(id: string): Promise<Result<void>>;
}

interface IUserExporter {
  exportToCSV(): Promise<Result<string>>;
  generateReport(): Promise<Result<Report>>;
}
```

**Niveau :** 🟡 FORTEMENT RECOMMANDÉ

---

#### 5.5 Dependency Inversion Principle (DIP)
**Règle :** Dépendre d'abstractions (interfaces), pas d'implémentations concrètes.

```typescript
// ❌ WRONG - Dépendance sur implémentation
class CaptureService {
  private repo = new CaptureRepository(); // ❌ Couplage fort

  async create(data: CreateCaptureData) {
    return this.repo.create(data);
  }
}

// ✅ CORRECT - Injection d'abstraction
class CaptureService {
  constructor(
    @inject(TOKENS.ICaptureRepository)
    private repo: ICaptureRepository // ✅ Dépend d'abstraction
  ) {}

  async create(data: CreateCaptureData): Promise<Result<Capture>> {
    return this.repo.create(data);
  }
}
```

**Niveau :** 🔴 NON-NÉGOCIABLE (testabilité critique)

---

### 6. GESTION D'ERREURS - Délégation à ADR-023

**Règle :** Toute gestion d'erreurs doit suivre ADR-023 (Result Pattern).

**Résumé strict :**
- ✅ Result Pattern PARTOUT
- ✅ Try/catch UNIQUEMENT pour DB/API externes + root handler
- ❌ JAMAIS de `throw` dans code applicatif
- ✅ Wrappers pour librairies externes système (RxJS, etc.)

**Référence complète :** ADR-023 - Stratégie Unifiée de Gestion des Erreurs

**Niveau :** 🔴 NON-NÉGOCIABLE (voir ADR-023)

---

### 7. TESTS - TDD Obligatoire

#### 7.1 TDD Red-Green-Refactor
**Règle :** Écrire le test AVANT l'implémentation.

```typescript
// 🔴 RED - Test échoue
describe('CaptureService', () => {
  it('should create capture with normalized text', async () => {
    const service = new CaptureService(mockRepo, mockNormalizer);
    const result = await service.createCapture({ rawContent: 'Hello world' });

    expect(result.type).toBe(ResultType.SUCCESS);
    expect(result.data.normalizedText).toBe('hello world');
  });
});

// 🟢 GREEN - Implémenter minimum pour passer
async createCapture(data: CreateCaptureData): Promise<Result<Capture>> {
  const capture = await this.repo.create(data);
  capture.data.normalizedText = data.rawContent.toLowerCase();
  return capture;
}

// 🔵 REFACTOR - Améliorer sans casser tests
async createCapture(data: CreateCaptureData): Promise<Result<Capture>> {
  const createResult = await this.repo.create(data);
  if (createResult.type !== ResultType.SUCCESS) return createResult;

  const normalizeResult = await this.normalizer.normalize(data.rawContent);
  if (normalizeResult.type === ResultType.SUCCESS) {
    createResult.data.normalizedText = normalizeResult.data;
  }

  return createResult;
}
```

**Niveau :** 🔴 NON-NÉGOCIABLE (stories doivent avoir tests BDD)

---

#### 7.2 FIRST Principles
**Règle :** Tests doivent être **F**ast, **I**ndependent, **R**epeatable, **S**elf-validating, **T**imely.

```typescript
// ✅ Fast - < 100ms par test unitaire
// ✅ Independent - Chaque test peut tourner seul
// ✅ Repeatable - Même résultat à chaque exécution
// ✅ Self-validating - expect() assertions claires
// ✅ Timely - Écrits au moment du développement (TDD)
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

#### 7.3 Un Concept Par Test
**Règle :** Chaque test vérifie UN SEUL comportement.

```typescript
// ❌ WRONG - Multiple assertions non liées
it('should handle user operations', async () => {
  const user = await createUser({ name: 'John' });
  expect(user.name).toBe('John');

  const updated = await updateUser(user.id, { email: 'john@test.com' });
  expect(updated.email).toBe('john@test.com');

  await deleteUser(user.id);
  const deleted = await findUser(user.id);
  expect(deleted).toBeNull();
});

// ✅ CORRECT - Tests séparés
it('should create user with name', async () => {
  const user = await createUser({ name: 'John' });
  expect(user.name).toBe('John');
});

it('should update user email', async () => {
  const user = await createUser({ name: 'John' });
  const updated = await updateUser(user.id, { email: 'john@test.com' });
  expect(updated.email).toBe('john@test.com');
});

it('should delete user', async () => {
  const user = await createUser({ name: 'John' });
  await deleteUser(user.id);
  const deleted = await findUser(user.id);
  expect(deleted).toBeNull();
});
```

**Niveau :** 🟡 FORTEMENT RECOMMANDÉ

---

### 8. CODE SMELLS - Detection Stricte

#### 8.1 Code Dupliqué - Rule of Three
**Règle :** Dupliquer 2 fois OK, abstraire à la 3ème occurrence.

```typescript
// 1ère fois - OK
function getUserEmail() {
  return user.email.toLowerCase().trim();
}

// 2ème fois - Dupliquer (oui!)
function getAdminEmail() {
  return admin.email.toLowerCase().trim();
}

// 3ème fois - MAINTENANT abstraire
function normalizeEmail(email: string): string {
  return email.toLowerCase().trim();
}

function getUserEmail() {
  return normalizeEmail(user.email);
}

function getAdminEmail() {
  return normalizeEmail(admin.email);
}
```

**Niveau :** 🟡 FORTEMENT RECOMMANDÉ (Rule of Three)

---

#### 8.2 Fonctions Trop Longues
**Règle :** > 50 lignes = signal de refactoring immédiat.

**Niveau :** 🟡 FORTEMENT RECOMMANDÉ

---

#### 8.3 Classes Trop Grandes (God Objects)
**Règle :** Classes > 300 lignes ou > 10 méthodes publiques = violation SRP probable.

**Niveau :** 🔴 NON-NÉGOCIABLE (violation SRP)

---

#### 8.4 Dead Code
**Règle :** Code jamais appelé = supprimer immédiatement.

```bash
# Utiliser ts-prune pour détecter exports inutilisés
npx ts-prune
```

**Niveau :** 🔴 NON-NÉGOCIABLE

---

## Consequences

### ✅ Bénéfices

1. **Code reviews objectives**
   - Règles claires → pas de débats subjectifs
   - PR reviews 50% plus rapides
   - Moins de friction équipe

2. **Maintenabilité excellente**
   - Refactoring sécurisé (patterns documentés)
   - Bugs prévenus (validations strictes)
   - Dette technique contrôlée

3. **Onboarding accéléré**
   - 6-8 semaines → 2-3 semaines
   - Standards explicites
   - Exemples concrets dans ADR

4. **Qualité code mesurable**
   - ESLint/Prettier automatisés
   - Tests obligatoires (TDD)
   - Coverage minimum enforced

### ⚠️ Trade-offs Acceptés

1. **Verbosité accrue**
   - Noms longs mais explicites
   - Options objects pour fonctions
   - **Mitigation :** Lisibilité > brièveté

2. **Courbe d'apprentissage**
   - Développeurs habitués à style impératif
   - **Mitigation :** Documentation + code reviews pédagogiques

3. **Refactoring initial**
   - Code existant à migrer progressivement
   - **Mitigation :** Migration par story, pas big-bang

---

## Implementation

### Étapes de Mise en Œuvre

**Phase 1 : Outillage (1 jour)**
1. ✅ Créer ADR-024 (ce document)
2. ⏳ Update `project-context.md` avec résumé Clean Code
3. ⏳ Configurer ESLint strict rules
4. ⏳ Configurer Prettier
5. ⏳ Setup pre-commit hooks (TODO check, formatting)

**Phase 2 : Documentation (1 jour)**
6. ⏳ Créer `docs/clean-code-examples.md` avec exemples projet
7. ⏳ Update `CONTRIBUTING.md` avec références ADR-024

**Phase 3 : Enforcement (1 jour)**
8. ⏳ CI/CD : ESLint fails build
9. ⏳ CI/CD : Prettier check
10. ⏳ PR template : Checklist Clean Code

**Phase 4 : Migration Progressive (2-4 semaines)**
11. ⏳ Audit code existant
12. ⏳ Migrer nouveaux fichiers immédiatement
13. ⏳ Refactoring opportuniste (touch file → apply rules)
14. ⏳ Pas de big-bang rewrite

### Files Modified

**Création nouveaux fichiers :**
```
_bmad-output/planning-artifacts/adrs/ADR-024-clean-code-standards.md (✅ this file)
docs/clean-code-examples.md (create)
.eslintrc-strict.js (create or update)
.git/hooks/pre-commit (create)
```

**Update fichiers existants :**
```
_bmad-output/project-context.md (add Clean Code section)
CONTRIBUTING.md (add ADR-024 reference)
.github/pull_request_template.md (add checklist)
```

### Effort Estimé

- **ADR creation** : 4 heures (✅ done)
- **Tooling setup** : 1 jour
- **Documentation** : 1 jour
- **CI/CD enforcement** : 1 jour
- **Migration progressive** : 2-4 semaines (opportuniste)

**Total effort estimé :** 3 jours setup + 2-4 semaines migration

---

## Validation Criteria

ADR considéré succès SI :

- ✅ **ESLint strict** : 100% fichiers passent linting
- ✅ **Prettier formatted** : 0 diff après format
- ✅ **Pre-commit hooks** : Bloque TODO sans ticket
- ✅ **Code reviews** : 90% PRs validées en < 24h (règles claires)
- ✅ **Tests coverage** : > 80% (TDD enforcement)
- ✅ **Onboarding** : < 3 semaines pour nouveau dev
- ✅ **Tech debt** : < 10% fichiers violant standards (30 jours)

**Review Date :** 2026-03 (après migration complète)

---

## References

- **Clean Code (Uncle Bob)** : https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882
- **SOLID Principles** : https://en.wikipedia.org/wiki/SOLID
- **TypeScript ESLint** : https://typescript-eslint.io/
- **Prettier** : https://prettier.io/
- **TDD Best Practices** : https://martinfowler.com/bliki/TestDrivenDevelopment.html
- **ADR-023** : Stratégie Unifiée de Gestion des Erreurs (Result Pattern)

---

## Anti-Patterns - Ce qu'il NE FAUT PAS faire

### ❌ Anti-Pattern 1 : Nommage Obscur

```typescript
// ❌ WRONG
const u = getUserById(id);
const d = new Date();
function get(id: string) { }

// ✅ CORRECT
const user = getUserById(id);
const creationDate = new Date();
function getUserById(id: string): Promise<Result<User>> { }
```

---

### ❌ Anti-Pattern 2 : Magic Numbers

```typescript
// ❌ WRONG
if (user.status === 3) {
  setTimeout(callback, 86400000);
}

// ✅ CORRECT
const UserStatus = {
  ACTIVE: 1,
  INACTIVE: 2,
  SUSPENDED: 3,
} as const;

const MILLISECONDS_PER_DAY = 24 * 60 * 60 * 1000;

if (user.status === UserStatus.SUSPENDED) {
  setTimeout(callback, MILLISECONDS_PER_DAY);
}
```

---

### ❌ Anti-Pattern 3 : God Objects

```typescript
// ❌ WRONG - 500 lignes, 15 méthodes
class UserService {
  create() { }
  update() { }
  delete() { }
  sendEmail() { }
  validateCard() { }
  trackAnalytics() { }
  generateReport() { }
  exportCSV() { }
  // ... 7 autres méthodes
}

// ✅ CORRECT - Séparation SRP
class UserService {
  create() { }
  update() { }
  delete() { }
}

class EmailService {
  send() { }
}

class PaymentService {
  validateCard() { }
}
```

---

### ❌ Anti-Pattern 4 : TODO Sans Ticket

```typescript
// ❌ WRONG
// TODO: Fix this later
// FIXME: This is slow

// ✅ CORRECT
// FIXME(JIRA-1234): Optimize query - deadline 2026-02-20

// ✅ MEILLEUR - Créer issue immédiatement
```

---

### ❌ Anti-Pattern 5 : Code Commenté

```typescript
// ❌ WRONG
function process() {
  // const old = doOld();
  // if (old) return handleOld();

  const new = doNew();
  return handleNew();
}

// ✅ CORRECT - Supprimer, on a Git
function process() {
  const result = doNew();
  return handleNew(result);
}
```

---

## Decision Log

**2026-02-15** - Discussion yohikofox + Winston

**Context :**
- Besoin de standardiser qualité code
- Règles Clean Code d'Uncle Bob trop dogmatiques si appliquées strictement
- Nécessité d'adapter au contexte Pensieve

**Options évaluées :**
1. **Appliquer 100% Clean Code Uncle Bob** → ❌ Rejeté (trop strict, contre-productif)
2. **Pas de standards** → ❌ Rejeté (dette technique incontrôlée)
3. **Sous-ensemble pragmatique avec 3 niveaux** → ✅ Choisi

**Décisions clés :**
- Fonctions < 20 lignes idéal, < 30 acceptable (pas 5-10 strict)
- Open/Closed avec Rule of Three (pas abstraction immédiate)
- TODO interdit sans ticket (git hook enforcement)
- undefined > null (sauf interop API/DB)
- Interfaces suffixe "Contract" (sauf services/repos legacy I prefix)

**Participants :**
- yohikofox (Product Owner)
- Winston (Architect)

---

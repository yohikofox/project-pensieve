# Story 1.2: User Registration

**Story ID:** 1.2
**Epic:** Epic 1 - Foundation & Authentification
**Status:** ready-for-dev
**Story Key:** `1-2-user-registration`

---

## 📋 User Story

**As a** new user
**I want** to create an account with email and password
**So that** my captures are encrypted with my credentials and securely stored

---

## ✅ Acceptance Criteria

### Given I am a new user who has not registered yet

**When** I provide a valid email address and a strong password (min 8 chars, 1 uppercase, 1 number)
**Then** a new user account is created in the Identity Context
**And** my password is hashed using bcrypt before storage
**And** a unique user ID is generated
**And** the User entity is persisted in PostgreSQL
**And** an authentication token (JWT) is returned
**And** I am automatically logged in after registration

### Given I provide an email that already exists

**When** I attempt to register
**Then** I receive an error message "Email already registered"
**And** no duplicate account is created

### Given I provide an invalid email format or weak password

**When** I attempt to register
**Then** I receive appropriate validation error messages
**And** no account is created

### Given registration succeeds

**When** the account is created
**Then** encryption keys are derived from my credentials for data protection (NFR12)

---

## 🎯 Developer Context & Implementation Guide

### 🏗️ Architecture Context

**Bounded Context:** Identity Context (Generic Subdomain)
**Responsibility:** Authentification, gestion utilisateurs
**Aggregate:** `User`

**This is the FIRST user-facing feature** - sets the pattern for all subsequent API development.

---

## 📦 Backend Implementation (NestJS)

### Identity Context Structure

```
src/contexts/identity/
├── domain/
│   ├── entities/
│   │   └── user.entity.ts          # User aggregate root
│   ├── value-objects/
│   │   ├── email.vo.ts             # Email validation
│   │   └── password.vo.ts          # Password validation
│   ├── events/
│   │   └── user-registered.event.ts
│   └── repositories/
│       └── user.repository.interface.ts
├── application/
│   ├── commands/
│   │   ├── register-user.command.ts
│   │   └── register-user.handler.ts
│   └── services/
│       └── auth.service.ts
├── infrastructure/
│   ├── persistence/
│   │   ├── user.schema.ts          # TypeORM entity
│   │   └── user.repository.ts      # TypeORM implementation
│   └── messaging/
│       └── user-events.publisher.ts
└── presentation/
    ├── controllers/
    │   └── auth.controller.ts
    └── dto/
        ├── register.dto.ts
        └── register-response.dto.ts
```

---

## 🔐 Security Requirements

### Password Hashing (bcrypt)

**Library:** `bcrypt` (NOT bcryptjs - use native for performance)
**Version:** ^5.1.0

**Installation:**
```bash
npm install bcrypt
npm install -D @types/bcrypt
```

**Hashing Configuration:**
```typescript
// Salt rounds: 10-12 (12 recommended for production)
const SALT_ROUNDS = 12;

// Example usage
import * as bcrypt from 'bcrypt';

const hashedPassword = await bcrypt.hash(plainPassword, SALT_ROUNDS);
```

**⚠️ CRITICAL:**
- NEVER store plain text passwords
- NEVER log passwords (plain or hashed)
- NEVER return password hash in API responses

---

### JWT Token Generation

**Library:** `@nestjs/jwt` + `jsonwebtoken`
**Version:** @nestjs/jwt ^10.x, jsonwebtoken ^9.x

**Installation:**
```bash
npm install @nestjs/jwt jsonwebtoken
npm install -D @types/jsonwebtoken
```

**JWT Configuration:**
```typescript
// JWT Module configuration
JwtModule.register({
  secret: process.env.JWT_SECRET, // From .env
  signOptions: {
    expiresIn: '7d', // 7 days expiration
    issuer: 'pensine-api',
    audience: 'pensine-mobile'
  }
})
```

**JWT Payload Structure:**
```typescript
interface JwtPayload {
  sub: string;        // User ID (UUID)
  email: string;      // User email
  iat: number;        // Issued at
  exp: number;        // Expiration
}
```

**Token Generation Example:**
```typescript
const payload: JwtPayload = {
  sub: user.id,
  email: user.email,
  iat: Math.floor(Date.now() / 1000),
  exp: Math.floor(Date.now() / 1000) + (7 * 24 * 60 * 60) // 7 days
};

const token = await this.jwtService.signAsync(payload);
```

---

## 🗃️ Database Schema (PostgreSQL + TypeORM)

### User Entity (TypeORM)

```typescript
// src/contexts/identity/infrastructure/persistence/user.schema.ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, UpdateDateColumn, Index } from 'typeorm';

@Entity('users')
export class UserSchema {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  @Index()
  email: string;

  @Column({ select: false }) // Never auto-select password hash
  passwordHash: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @Column({ nullable: true })
  lastLoginAt: Date;

  @Column({ default: true })
  isActive: boolean;

  // Encryption key derivation metadata (NFR12)
  @Column({ nullable: true })
  encryptionSalt: string;
}
```

**Migration:**
```typescript
// migrations/XXXXXX-create-users-table.ts
import { MigrationInterface, QueryRunner, Table, TableIndex } from 'typeorm';

export class CreateUsersTable1234567890 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'users',
        columns: [
          {
            name: 'id',
            type: 'uuid',
            isPrimary: true,
            generationStrategy: 'uuid',
            default: 'uuid_generate_v4()'
          },
          {
            name: 'email',
            type: 'varchar',
            isUnique: true,
            isNullable: false
          },
          {
            name: 'passwordHash',
            type: 'varchar',
            isNullable: false
          },
          {
            name: 'createdAt',
            type: 'timestamp',
            default: 'CURRENT_TIMESTAMP'
          },
          {
            name: 'updatedAt',
            type: 'timestamp',
            default: 'CURRENT_TIMESTAMP'
          },
          {
            name: 'lastLoginAt',
            type: 'timestamp',
            isNullable: true
          },
          {
            name: 'isActive',
            type: 'boolean',
            default: true
          },
          {
            name: 'encryptionSalt',
            type: 'varchar',
            isNullable: true
          }
        ]
      }),
      true
    );

    // Index on email for fast lookups
    await queryRunner.createIndex(
      'users',
      new TableIndex({
        name: 'IDX_USERS_EMAIL',
        columnNames: ['email']
      })
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('users');
  }
}
```

**⚠️ IMPORTANT:**
- Enable UUID extension: `CREATE EXTENSION IF NOT EXISTS "uuid-ossp";`
- Email is indexed for performance
- passwordHash has `select: false` - must explicitly include when needed

---

## 🎨 API Design

### POST /api/auth/register

**Request Body (DTO):**
```typescript
// src/contexts/identity/presentation/dto/register.dto.ts
import { IsEmail, IsString, MinLength, Matches } from 'class-validator';

export class RegisterDto {
  @IsEmail({}, { message: 'Invalid email format' })
  email: string;

  @IsString()
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  @Matches(/^(?=.*[A-Z])(?=.*\d)/, {
    message: 'Password must contain at least 1 uppercase letter and 1 number'
  })
  password: string;
}
```

**Response (Success - 201 Created):**
```typescript
// src/contexts/identity/presentation/dto/register-response.dto.ts
export class RegisterResponseDto {
  id: string;
  email: string;
  accessToken: string;
  expiresIn: number;
  createdAt: Date;
}
```

**Example Response:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 604800,
  "createdAt": "2026-01-19T12:00:00.000Z"
}
```

**Error Responses:**

**400 Bad Request (Validation Errors):**
```json
{
  "statusCode": 400,
  "message": [
    "Invalid email format",
    "Password must be at least 8 characters",
    "Password must contain at least 1 uppercase letter and 1 number"
  ],
  "error": "Bad Request"
}
```

**409 Conflict (Email Already Exists):**
```json
{
  "statusCode": 409,
  "message": "Email already registered",
  "error": "Conflict"
}
```

**500 Internal Server Error:**
```json
{
  "statusCode": 500,
  "message": "Internal server error",
  "error": "Internal Server Error"
}
```

---

## 📱 Mobile Implementation (React Native + Expo)

### Identity Context Structure

```
src/contexts/identity/
├── domain/
│   ├── models/
│   │   └── user.model.ts           # WatermelonDB model
│   └── services/
│       └── auth.service.ts
├── application/
│   ├── hooks/
│   │   ├── useRegister.ts
│   │   └── useAuth.ts
│   └── state/
│       └── auth.context.tsx        # React Context for auth state
└── presentation/
    ├── screens/
    │   └── RegisterScreen.tsx
    ├── components/
    │   ├── RegisterForm.tsx
    │   └── PasswordInput.tsx
    └── validation/
        └── register.validation.ts
```

---

### WatermelonDB User Model

```typescript
// src/contexts/identity/domain/models/user.model.ts
import { Model } from '@nozbe/watermelondb';
import { field, date, readonly } from '@nozbe/watermelondb/decorators';

export class User extends Model {
  static table = 'users';

  @field('email') email!: string;
  @field('access_token') accessToken!: string;
  @readonly @date('created_at') createdAt!: Date;
  @readonly @date('updated_at') updatedAt!: Date;
  @date('last_login_at') lastLoginAt!: Date | null;
}
```

**Schema Definition:**
```typescript
// src/database/schema.ts
import { appSchema, tableSchema } from '@nozbe/watermelondb';

export const schema = appSchema({
  version: 1,
  tables: [
    tableSchema({
      name: 'users',
      columns: [
        { name: 'email', type: 'string', isIndexed: true },
        { name: 'access_token', type: 'string' },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' },
        { name: 'last_login_at', type: 'number', isOptional: true }
      ]
    })
  ]
});
```

---

### API Service (Axios)

```typescript
// src/contexts/identity/domain/services/auth.service.ts
import axios from 'axios';

const API_BASE_URL = process.env.API_BASE_URL || 'http://localhost:3000';

export interface RegisterRequest {
  email: string;
  password: string;
}

export interface RegisterResponse {
  id: string;
  email: string;
  accessToken: string;
  expiresIn: number;
  createdAt: string;
}

export const authService = {
  async register(data: RegisterRequest): Promise<RegisterResponse> {
    const response = await axios.post<RegisterResponse>(
      `${API_BASE_URL}/api/auth/register`,
      data
    );
    return response.data;
  }
};
```

---

### React Hook (useRegister)

```typescript
// src/contexts/identity/application/hooks/useRegister.ts
import { useState } from 'react';
import { authService } from '../../domain/services/auth.service';
import { useDatabase } from '@nozbe/watermelondb/hooks';
import { User } from '../../domain/models/user.model';

export const useRegister = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const database = useDatabase();

  const register = async (email: string, password: string) => {
    setLoading(true);
    setError(null);

    try {
      // Call backend API
      const response = await authService.register({ email, password });

      // Store user + token in local WatermelonDB
      await database.write(async () => {
        await database.get<User>('users').create((user) => {
          user.email = response.email;
          user.accessToken = response.accessToken;
          user.lastLoginAt = new Date();
        });
      });

      return response;
    } catch (err: any) {
      const message = err.response?.data?.message || 'Registration failed';
      setError(message);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  return { register, loading, error };
};
```

---

## ✅ Validation Requirements

### Email Validation

**Pattern:** RFC 5322 compliant
**Library:** `class-validator` (backend), custom regex (mobile)

**Backend:**
```typescript
@IsEmail({}, { message: 'Invalid email format' })
email: string;
```

**Mobile:**
```typescript
const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

export const validateEmail = (email: string): boolean => {
  return EMAIL_REGEX.test(email);
};
```

---

### Password Validation

**Requirements:**
- Minimum 8 characters
- At least 1 uppercase letter
- At least 1 number

**Backend:**
```typescript
@MinLength(8, { message: 'Password must be at least 8 characters' })
@Matches(/^(?=.*[A-Z])(?=.*\d)/, {
  message: 'Password must contain at least 1 uppercase letter and 1 number'
})
password: string;
```

**Mobile:**
```typescript
const PASSWORD_REGEX = /^(?=.*[A-Z])(?=.*\d).{8,}$/;

export const validatePassword = (password: string): {
  valid: boolean;
  errors: string[];
} => {
  const errors: string[] = [];

  if (password.length < 8) {
    errors.push('Password must be at least 8 characters');
  }
  if (!/[A-Z]/.test(password)) {
    errors.push('Password must contain at least 1 uppercase letter');
  }
  if (!/\d/.test(password)) {
    errors.push('Password must contain at least 1 number');
  }

  return {
    valid: errors.length === 0,
    errors
  };
};
```

---

## 🧪 Testing Requirements

### Backend Tests

**Unit Tests (Domain):**
```typescript
// user.entity.spec.ts
describe('User Entity', () => {
  it('should hash password on creation', async () => {
    const user = await User.create('test@example.com', 'Password123');
    expect(user.passwordHash).not.toBe('Password123');
    expect(user.passwordHash).toMatch(/^\$2[aby]\$\d+\$/);
  });

  it('should validate password correctly', async () => {
    const user = await User.create('test@example.com', 'Password123');
    const isValid = await user.validatePassword('Password123');
    expect(isValid).toBe(true);
  });
});
```

**Integration Tests (Repository):**
```typescript
// user.repository.spec.ts
describe('UserRepository', () => {
  it('should save user to database', async () => {
    const user = await User.create('test@example.com', 'Password123');
    const saved = await userRepository.save(user);
    expect(saved.id).toBeDefined();
    expect(saved.email).toBe('test@example.com');
  });

  it('should throw error on duplicate email', async () => {
    await User.create('test@example.com', 'Password123');
    await expect(
      User.create('test@example.com', 'Password456')
    ).rejects.toThrow('Email already registered');
  });
});
```

**E2E Tests (API):**
```typescript
// auth.e2e-spec.ts
describe('/api/auth/register (POST)', () => {
  it('should register new user', () => {
    return request(app.getHttpServer())
      .post('/api/auth/register')
      .send({
        email: 'newuser@example.com',
        password: 'Password123'
      })
      .expect(201)
      .expect((res) => {
        expect(res.body.accessToken).toBeDefined();
        expect(res.body.email).toBe('newuser@example.com');
      });
  });

  it('should return 409 on duplicate email', () => {
    // First registration
    await request(app.getHttpServer())
      .post('/api/auth/register')
      .send({ email: 'test@example.com', password: 'Password123' });

    // Duplicate attempt
    return request(app.getHttpServer())
      .post('/api/auth/register')
      .send({ email: 'test@example.com', password: 'Password456' })
      .expect(409)
      .expect((res) => {
        expect(res.body.message).toBe('Email already registered');
      });
  });

  it('should return 400 on invalid email', () => {
    return request(app.getHttpServer())
      .post('/api/auth/register')
      .send({
        email: 'invalid-email',
        password: 'Password123'
      })
      .expect(400);
  });

  it('should return 400 on weak password', () => {
    return request(app.getHttpServer())
      .post('/api/auth/register')
      .send({
        email: 'test@example.com',
        password: 'weak'
      })
      .expect(400);
  });
});
```

---

### Mobile Tests

**Component Tests:**
```typescript
// RegisterForm.test.tsx
import { render, fireEvent } from '@testing-library/react-native';
import { RegisterForm } from './RegisterForm';

describe('RegisterForm', () => {
  it('should display validation errors', () => {
    const { getByText, getByPlaceholderText } = render(<RegisterForm />);

    const emailInput = getByPlaceholderText('Email');
    const passwordInput = getByPlaceholderText('Password');

    fireEvent.changeText(emailInput, 'invalid-email');
    fireEvent.changeText(passwordInput, 'weak');

    expect(getByText('Invalid email format')).toBeTruthy();
    expect(getByText('Password must be at least 8 characters')).toBeTruthy();
  });

  it('should call onSubmit with valid data', () => {
    const onSubmit = jest.fn();
    const { getByText, getByPlaceholderText } = render(
      <RegisterForm onSubmit={onSubmit} />
    );

    fireEvent.changeText(getByPlaceholderText('Email'), 'test@example.com');
    fireEvent.changeText(getByPlaceholderText('Password'), 'Password123');
    fireEvent.press(getByText('Register'));

    expect(onSubmit).toHaveBeenCalledWith({
      email: 'test@example.com',
      password: 'Password123'
    });
  });
});
```

---

## 🔒 Security Considerations (NFR12)

### Encryption Key Derivation

**Requirement (from NFR12):** "Encryption keys are derived from my credentials for data protection"

**Implementation Strategy:**

1. **User-Specific Encryption Salt** (stored in DB)
   - Generate random salt on registration
   - Store in `encryptionSalt` column
   - Used for deriving encryption keys for user data

2. **Key Derivation Function (PBKDF2)**
   ```typescript
   import { pbkdf2Sync } from 'crypto';

   const deriveEncryptionKey = (password: string, salt: string): Buffer => {
     return pbkdf2Sync(password, salt, 100000, 32, 'sha256');
   };
   ```

3. **Storage:**
   - Encryption key NEVER stored in database
   - Only salt is stored
   - Key is derived on-demand from user password

**⚠️ MVP Scope:**
- Basic implementation: Store salt
- Full encryption: Post-MVP (TDE or Column-level)
- Current: Device-level encryption + infrastructure-level

---

## ⚠️ Common Pitfalls to Avoid

### ❌ DO NOT:
1. **Store plain text passwords** (use bcrypt hash)
2. **Return password hash in API responses** (security violation)
3. **Log passwords** (even hashed)
4. **Use weak password validation** (enforce uppercase + number)
5. **Skip email uniqueness check** (race conditions)
6. **Forget to set JWT expiration** (7 days standard)
7. **Use bcryptjs** (use native `bcrypt` for performance)
8. **Skip database migration** (users table MUST exist)

### ✅ DO:
1. **Hash password with bcrypt (12 salt rounds)**
2. **Generate UUID for user ID** (PostgreSQL uuid-ossp)
3. **Index email column** (fast lookups)
4. **Return JWT token immediately** (auto-login after registration)
5. **Handle duplicate email gracefully** (409 Conflict)
6. **Validate input on both client and server**
7. **Store user + token in WatermelonDB** (offline access)
8. **Test all error scenarios** (duplicate email, weak password, invalid email)

---

## 🔗 References

### Official Documentation
- [bcrypt npm](https://www.npmjs.com/package/bcrypt)
- [NestJS JWT](https://docs.nestjs.com/security/authentication#jwt-functionality)
- [TypeORM Entities](https://typeorm.io/entities)
- [class-validator](https://github.com/typestack/class-validator)
- [WatermelonDB Models](https://nozbe.github.io/WatermelonDB/Model.html)

### Architecture Documents
- **Main Architecture:** `architecture.md` (Identity Context, User aggregate)
- **PRD:** `prd.md` (FR25: Create account)
- **NFR12:** Data encryption requirements

### Related Stories
- **Previous:** 1.1 Project Foundation & Infrastructure Setup
- **Next:** 1.3 User Login
- **Epic:** Epic 1 - Foundation & Authentification

---

## ✅ Definition of Done

**This story is DONE when:**

1. ✅ **Backend Implementation:**
   - User entity (TypeORM) created with all fields
   - Database migration executed (users table created)
   - Email uniqueness constraint enforced
   - Password hashing with bcrypt (12 rounds)
   - JWT token generation working
   - POST /api/auth/register endpoint functional
   - Input validation (email format, password strength)
   - Error handling (409 duplicate, 400 validation)

2. ✅ **Mobile Implementation:**
   - User model (WatermelonDB) created
   - RegisterScreen UI created
   - RegisterForm component with validation
   - useRegister hook implemented
   - API integration working (axios)
   - Local storage (WatermelonDB) after successful registration
   - Error display (duplicate email, validation errors)

3. ✅ **Testing:**
   - Backend unit tests passing (User entity, bcrypt)
   - Backend integration tests passing (UserRepository)
   - Backend E2E tests passing (register endpoint)
   - Mobile component tests passing (RegisterForm)
   - Mobile hook tests passing (useRegister)

4. ✅ **Security:**
   - Password never stored in plain text
   - Password hash never returned in API
   - JWT token properly signed and expires in 7 days
   - Email uniqueness enforced at DB level
   - Encryption salt generated and stored (NFR12)

5. ✅ **Verification:**
   - Can register new user via Postman/curl
   - Can register new user via mobile app
   - Duplicate email returns 409 error
   - Weak password returns 400 error
   - Invalid email returns 400 error
   - JWT token is valid and can be decoded
   - User persisted in PostgreSQL
   - User + token persisted in WatermelonDB

---

## 📝 Developer Notes

_This section will be filled by the developer during implementation._

**Implementation Log:**
- [ ] Backend User entity created
- [ ] Database migration created and executed
- [ ] Password hashing implemented
- [ ] JWT token generation implemented
- [ ] Register endpoint created
- [ ] Mobile User model created
- [ ] RegisterScreen created
- [ ] API integration completed
- [ ] All tests passing

**Challenges Encountered:**
_TBD_

**Learnings for Next Stories:**
_TBD_

---

**Story Created:** 2026-01-19
**Created by:** Scrum Master (BMAD Methodology)
**Context Engine Analysis:** ✅ Complete
**Ready for Development:** ✅ Yes

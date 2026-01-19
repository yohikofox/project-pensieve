# Story 1.3: User Login

**Story ID:** 1.3
**Epic:** Epic 1 - Foundation & Authentification
**Status:** ready-for-dev
**Story Key:** `1-3-user-login`

---

## 📋 User Story

**As a** registered user
**I want** to log in with my email and password
**So that** I can access my encrypted captures securely

---

## ✅ Acceptance Criteria

### Given I am a registered user with valid credentials

**When** I provide my correct email and password
**Then** my credentials are validated against the hashed password in the database
**And** a JWT access token is generated and returned
**And** my user session is established
**And** I am redirected to the main app screen

### Given I provide an incorrect password

**When** I attempt to login
**Then** I receive an error message "Invalid email or password"
**And** no session is created
**And** the failed attempt is logged for security monitoring

### Given I provide an email that doesn't exist

**When** I attempt to login
**Then** I receive an error message "Invalid email or password" (same message to prevent email enumeration)
**And** no session is created

### Given I successfully log in

**When** the JWT token is issued
**Then** the token has a reasonable expiration time (e.g., 7 days)
**And** the token includes my user ID for data isolation (NFR13)

---

## 🎯 Developer Context & Implementation Guide

### 🏗️ Architecture Context

**Bounded Context:** Identity Context (Generic Subdomain)
**Responsibility:** Authentification, gestion utilisateurs
**Aggregate:** `User`

**This story complements Story 1.2 (User Registration)** - reuses User entity, JWT infrastructure, and authentication patterns.

---

## 📦 Backend Implementation (NestJS)

### Identity Context Structure (Extends Story 1.2)

```
src/contexts/identity/
├── domain/
│   ├── entities/
│   │   └── user.entity.ts          # Reuse from 1.2
│   └── value-objects/
│       └── password.vo.ts          # Add validatePassword method
├── application/
│   ├── commands/
│   │   ├── login-user.command.ts   # NEW
│   │   └── login-user.handler.ts   # NEW
│   └── services/
│       └── auth.service.ts         # Extend from 1.2
└── presentation/
    ├── controllers/
    │   └── auth.controller.ts      # Add login endpoint
    └── dto/
        ├── login.dto.ts            # NEW
        └── login-response.dto.ts   # NEW (reuse register-response structure)
```

---

## 🔐 Security Requirements

### Password Validation (bcrypt.compare)

**Library:** `bcrypt` (already installed in Story 1.2)

**Password Comparison:**
```typescript
import * as bcrypt from 'bcrypt';

// In User entity or AuthService
async validatePassword(plainPassword: string): Promise<boolean> {
  return await bcrypt.compare(plainPassword, this.passwordHash);
}
```

**⚠️ CRITICAL SECURITY:**
1. **NEVER return different error messages** for "user not found" vs "wrong password"
   - **Reason:** Prevents email enumeration attacks
   - **Always return:** "Invalid email or password"

2. **Log failed login attempts** for security monitoring
   - Store: timestamp, email attempted, IP address (optional)
   - Use for: brute force detection, account security alerts

3. **Use constant-time comparison** (bcrypt.compare does this automatically)
   - Prevents timing attacks

---

### JWT Token Generation (Reuse from Story 1.2)

**Configuration:** Same as Story 1.2
- 7 days expiration
- Include user ID (sub) for data isolation (NFR13)

**JWT Payload Structure:**
```typescript
interface JwtPayload {
  sub: string;        // User ID (UUID) - NFR13: Data isolation
  email: string;      // User email
  iat: number;        // Issued at
  exp: number;        // Expiration
}
```

---

## 🎨 API Design

### POST /api/auth/login

**Request Body (DTO):**
```typescript
// src/contexts/identity/presentation/dto/login.dto.ts
import { IsEmail, IsString, MinLength } from 'class-validator';

export class LoginDto {
  @IsEmail({}, { message: 'Invalid email format' })
  email: string;

  @IsString()
  @MinLength(1, { message: 'Password is required' })
  password: string;
}
```

**Response (Success - 200 OK):**
```typescript
// src/contexts/identity/presentation/dto/login-response.dto.ts
export class LoginResponseDto {
  id: string;
  email: string;
  accessToken: string;
  expiresIn: number;
  lastLoginAt: Date;
}
```

**Example Response:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 604800,
  "lastLoginAt": "2026-01-19T12:00:00.000Z"
}
```

**Error Responses:**

**401 Unauthorized (Invalid Credentials):**
```json
{
  "statusCode": 401,
  "message": "Invalid email or password",
  "error": "Unauthorized"
}
```

**⚠️ CRITICAL:** Return same message for both:
- Email doesn't exist
- Password is incorrect

**400 Bad Request (Validation Errors):**
```json
{
  "statusCode": 400,
  "message": [
    "Invalid email format",
    "Password is required"
  ],
  "error": "Bad Request"
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

## 🛠️ Backend Implementation Details

### Auth Controller (Extend from Story 1.2)

```typescript
// src/contexts/identity/presentation/controllers/auth.controller.ts
import { Controller, Post, Body, HttpCode, HttpStatus } from '@nestjs/common';
import { LoginDto } from '../dto/login.dto';
import { LoginResponseDto } from '../dto/login-response.dto';
import { AuthService } from '../../application/services/auth.service';

@Controller('api/auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  // From Story 1.2
  @Post('register')
  async register(@Body() dto: RegisterDto): Promise<RegisterResponseDto> {
    // Implementation from 1.2
  }

  // NEW for Story 1.3
  @Post('login')
  @HttpCode(HttpStatus.OK) // 200 OK for successful login
  async login(@Body() dto: LoginDto): Promise<LoginResponseDto> {
    return await this.authService.login(dto.email, dto.password);
  }
}
```

---

### Auth Service (Extend from Story 1.2)

```typescript
// src/contexts/identity/application/services/auth.service.ts
import { Injectable, UnauthorizedException, Logger } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UserRepository } from '../../infrastructure/persistence/user.repository';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  private readonly logger = new Logger(AuthService.name);

  constructor(
    private readonly userRepository: UserRepository,
    private readonly jwtService: JwtService
  ) {}

  // From Story 1.2
  async register(email: string, password: string) {
    // Implementation from 1.2
  }

  // NEW for Story 1.3
  async login(email: string, password: string): Promise<LoginResponseDto> {
    // Find user by email (with password hash included)
    const user = await this.userRepository.findByEmail(email, true);

    // Check if user exists AND password is correct
    if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
      // Log failed attempt for security monitoring
      this.logger.warn(`Failed login attempt for email: ${email}`);

      // CRITICAL: Same message for user not found vs wrong password
      throw new UnauthorizedException('Invalid email or password');
    }

    // Update last login timestamp
    user.lastLoginAt = new Date();
    await this.userRepository.save(user);

    // Generate JWT token
    const payload: JwtPayload = {
      sub: user.id,
      email: user.email,
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + (7 * 24 * 60 * 60) // 7 days
    };

    const accessToken = await this.jwtService.signAsync(payload);

    return {
      id: user.id,
      email: user.email,
      accessToken,
      expiresIn: 7 * 24 * 60 * 60, // 7 days in seconds
      lastLoginAt: user.lastLoginAt
    };
  }
}
```

**Key Implementation Notes:**
1. **findByEmail with password:** Must explicitly include passwordHash (select: false by default)
2. **Single conditional check:** `!user || !validPassword` prevents timing attacks
3. **Logging:** Log failed attempts (email only, NEVER password)
4. **Update lastLoginAt:** Track when user last accessed the app

---

### User Repository (Extend from Story 1.2)

```typescript
// src/contexts/identity/infrastructure/persistence/user.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { UserSchema } from './user.schema';

@Injectable()
export class UserRepository {
  constructor(
    @InjectRepository(UserSchema)
    private readonly repo: Repository<UserSchema>
  ) {}

  // From Story 1.2
  async save(user: UserSchema): Promise<UserSchema> {
    return await this.repo.save(user);
  }

  // NEW for Story 1.3
  async findByEmail(email: string, includePassword = false): Promise<UserSchema | null> {
    const query = this.repo
      .createQueryBuilder('user')
      .where('user.email = :email', { email });

    // Explicitly select password hash if needed (for login)
    if (includePassword) {
      query.addSelect('user.passwordHash');
    }

    return await query.getOne();
  }
}
```

**⚠️ IMPORTANT:**
- By default, passwordHash is NOT selected (select: false in schema)
- Must explicitly add it for login validation
- NEVER return passwordHash in API responses

---

## 📱 Mobile Implementation (React Native + Expo)

### Identity Context Structure (Extends Story 1.2)

```
src/contexts/identity/
├── domain/
│   ├── models/
│   │   └── user.model.ts           # Reuse from 1.2
│   └── services/
│       └── auth.service.ts         # Extend with login method
├── application/
│   ├── hooks/
│   │   ├── useLogin.ts             # NEW
│   │   ├── useRegister.ts          # From 1.2
│   │   └── useAuth.ts              # Extend with login state
│   └── state/
│       └── auth.context.tsx        # Manage auth state
└── presentation/
    ├── screens/
    │   ├── LoginScreen.tsx         # NEW
    │   └── RegisterScreen.tsx      # From 1.2
    └── components/
        └── LoginForm.tsx           # NEW
```

---

### API Service (Extend from Story 1.2)

```typescript
// src/contexts/identity/domain/services/auth.service.ts
import axios from 'axios';

const API_BASE_URL = process.env.API_BASE_URL || 'http://localhost:3000';

export interface LoginRequest {
  email: string;
  password: string;
}

export interface LoginResponse {
  id: string;
  email: string;
  accessToken: string;
  expiresIn: number;
  lastLoginAt: string;
}

export const authService = {
  // From Story 1.2
  async register(data: RegisterRequest): Promise<RegisterResponse> {
    // Implementation from 1.2
  },

  // NEW for Story 1.3
  async login(data: LoginRequest): Promise<LoginResponse> {
    const response = await axios.post<LoginResponse>(
      `${API_BASE_URL}/api/auth/login`,
      data
    );
    return response.data;
  }
};
```

---

### React Hook (useLogin)

```typescript
// src/contexts/identity/application/hooks/useLogin.ts
import { useState } from 'react';
import { authService } from '../../domain/services/auth.service';
import { useDatabase } from '@nozbe/watermelondb/hooks';
import { User } from '../../domain/models/user.model';
import { useNavigation } from '@react-navigation/native';

export const useLogin = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const database = useDatabase();
  const navigation = useNavigation();

  const login = async (email: string, password: string) => {
    setLoading(true);
    setError(null);

    try {
      // Call backend API
      const response = await authService.login({ email, password });

      // Update or create user in WatermelonDB
      await database.write(async () => {
        const usersCollection = database.get<User>('users');

        // Check if user already exists locally
        const existingUsers = await usersCollection
          .query(Q.where('email', email))
          .fetch();

        if (existingUsers.length > 0) {
          // Update existing user
          const user = existingUsers[0];
          await user.update((u) => {
            u.accessToken = response.accessToken;
            u.lastLoginAt = new Date(response.lastLoginAt);
          });
        } else {
          // Create new user record
          await usersCollection.create((user) => {
            user.email = response.email;
            user.accessToken = response.accessToken;
            user.lastLoginAt = new Date(response.lastLoginAt);
          });
        }
      });

      // Navigate to main app screen
      navigation.reset({
        index: 0,
        routes: [{ name: 'MainApp' }], // Replace with your main screen name
      });

      return response;
    } catch (err: any) {
      const message = err.response?.data?.message || 'Login failed';
      setError(message);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  return { login, loading, error };
};
```

**Key Implementation Notes:**
1. **Update or create:** Check if user exists locally, update if yes, create if no
2. **Navigation reset:** Replace entire navigation stack to prevent back to login
3. **Error handling:** Display backend error message (e.g., "Invalid email or password")

---

### LoginScreen UI

```typescript
// src/contexts/identity/presentation/screens/LoginScreen.tsx
import React, { useState } from 'react';
import { View, TextInput, Button, Text, ActivityIndicator } from 'react-native';
import { useLogin } from '../../application/hooks/useLogin';

export const LoginScreen: React.FC = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const { login, loading, error } = useLogin();

  const handleLogin = async () => {
    try {
      await login(email, password);
      // Navigation handled by useLogin hook
    } catch (err) {
      // Error displayed via error state
    }
  };

  return (
    <View style={{ padding: 20 }}>
      <Text style={{ fontSize: 24, marginBottom: 20 }}>Login</Text>

      {error && (
        <Text style={{ color: 'red', marginBottom: 10 }}>{error}</Text>
      )}

      <TextInput
        placeholder="Email"
        value={email}
        onChangeText={setEmail}
        keyboardType="email-address"
        autoCapitalize="none"
        autoCorrect={false}
        style={{ borderWidth: 1, padding: 10, marginBottom: 10 }}
      />

      <TextInput
        placeholder="Password"
        value={password}
        onChangeText={setPassword}
        secureTextEntry
        style={{ borderWidth: 1, padding: 10, marginBottom: 20 }}
      />

      {loading ? (
        <ActivityIndicator size="large" />
      ) : (
        <Button title="Login" onPress={handleLogin} />
      )}
    </View>
  );
};
```

---

## 🔒 Security Considerations

### Prevent Email Enumeration

**Attack Vector:** Attacker tries different emails to discover which accounts exist.

**Mitigation:**
- ✅ Return same error message for "user not found" vs "wrong password"
- ✅ Use constant-time password comparison (bcrypt.compare handles this)
- ✅ Same response time for both scenarios (avoid timing attacks)

**Example:**
```typescript
// ❌ BAD: Different messages
if (!user) {
  throw new UnauthorizedException('User not found');
}
if (!validPassword) {
  throw new UnauthorizedException('Wrong password');
}

// ✅ GOOD: Same message
if (!user || !validPassword) {
  throw new UnauthorizedException('Invalid email or password');
}
```

---

### Failed Login Logging

**Purpose:** Security monitoring, brute force detection

**What to Log:**
- ✅ Timestamp
- ✅ Email attempted
- ✅ IP address (optional, if available)
- ❌ NEVER log password (plain or hashed)

**Implementation:**
```typescript
this.logger.warn(`Failed login attempt for email: ${email}`);

// Optional: Log to separate security audit table
await this.auditLogRepository.save({
  event: 'FAILED_LOGIN',
  email,
  ipAddress: request.ip, // If available from request
  timestamp: new Date()
});
```

**Future Enhancement (Post-MVP):**
- Rate limiting (e.g., max 5 attempts per 15 minutes)
- Account lockout after N failed attempts
- CAPTCHA after repeated failures
- Email alert to user on multiple failed attempts

---

### NFR13: Data Isolation

**Requirement:** "The token includes my user ID for data isolation"

**Implementation:**
```typescript
const payload: JwtPayload = {
  sub: user.id, // ← User ID for data isolation (NFR13)
  email: user.email,
  iat: Math.floor(Date.now() / 1000),
  exp: Math.floor(Date.now() / 1000) + (7 * 24 * 60 * 60)
};
```

**Usage in Future Stories:**
- All API requests include JWT token in Authorization header
- Backend extracts user ID from token (`sub` claim)
- All database queries filter by user ID: `WHERE userId = {sub}`
- Ensures users can NEVER access other users' data

---

## 🧪 Testing Requirements

### Backend Tests

**Unit Tests (AuthService):**
```typescript
// auth.service.spec.ts
describe('AuthService - login', () => {
  it('should return JWT token for valid credentials', async () => {
    const result = await authService.login('test@example.com', 'Password123');
    expect(result.accessToken).toBeDefined();
    expect(result.email).toBe('test@example.com');
  });

  it('should throw UnauthorizedException for wrong password', async () => {
    await expect(
      authService.login('test@example.com', 'WrongPassword')
    ).rejects.toThrow(UnauthorizedException);
  });

  it('should throw UnauthorizedException for non-existent email', async () => {
    await expect(
      authService.login('nonexistent@example.com', 'Password123')
    ).rejects.toThrow(UnauthorizedException);
  });

  it('should return same error message for both scenarios', async () => {
    let errorMessage1: string;
    let errorMessage2: string;

    try {
      await authService.login('test@example.com', 'WrongPassword');
    } catch (err: any) {
      errorMessage1 = err.message;
    }

    try {
      await authService.login('nonexistent@example.com', 'Password123');
    } catch (err: any) {
      errorMessage2 = err.message;
    }

    expect(errorMessage1).toBe(errorMessage2);
    expect(errorMessage1).toBe('Invalid email or password');
  });

  it('should update lastLoginAt on successful login', async () => {
    const before = new Date();
    await authService.login('test@example.com', 'Password123');
    const user = await userRepository.findByEmail('test@example.com');
    expect(user.lastLoginAt.getTime()).toBeGreaterThanOrEqual(before.getTime());
  });

  it('should log failed login attempts', async () => {
    const loggerSpy = jest.spyOn(logger, 'warn');

    try {
      await authService.login('test@example.com', 'WrongPassword');
    } catch (err) {
      // Expected
    }

    expect(loggerSpy).toHaveBeenCalledWith(
      expect.stringContaining('Failed login attempt')
    );
  });
});
```

---

**E2E Tests (API):**
```typescript
// auth.e2e-spec.ts
describe('/api/auth/login (POST)', () => {
  beforeEach(async () => {
    // Create test user
    await request(app.getHttpServer())
      .post('/api/auth/register')
      .send({ email: 'test@example.com', password: 'Password123' });
  });

  it('should login with valid credentials', () => {
    return request(app.getHttpServer())
      .post('/api/auth/login')
      .send({
        email: 'test@example.com',
        password: 'Password123'
      })
      .expect(200)
      .expect((res) => {
        expect(res.body.accessToken).toBeDefined();
        expect(res.body.email).toBe('test@example.com');
        expect(res.body.lastLoginAt).toBeDefined();
      });
  });

  it('should return 401 for wrong password', () => {
    return request(app.getHttpServer())
      .post('/api/auth/login')
      .send({
        email: 'test@example.com',
        password: 'WrongPassword'
      })
      .expect(401)
      .expect((res) => {
        expect(res.body.message).toBe('Invalid email or password');
      });
  });

  it('should return 401 for non-existent email', () => {
    return request(app.getHttpServer())
      .post('/api/auth/login')
      .send({
        email: 'nonexistent@example.com',
        password: 'Password123'
      })
      .expect(401)
      .expect((res) => {
        expect(res.body.message).toBe('Invalid email or password');
      });
  });

  it('should return 400 for invalid email format', () => {
    return request(app.getHttpServer())
      .post('/api/auth/login')
      .send({
        email: 'invalid-email',
        password: 'Password123'
      })
      .expect(400);
  });

  it('should return same error for both wrong password and non-existent user', async () => {
    const res1 = await request(app.getHttpServer())
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'Wrong' })
      .expect(401);

    const res2 = await request(app.getHttpServer())
      .post('/api/auth/login')
      .send({ email: 'fake@example.com', password: 'Password123' })
      .expect(401);

    expect(res1.body.message).toBe(res2.body.message);
  });
});
```

---

### Mobile Tests

**Hook Tests:**
```typescript
// useLogin.test.ts
import { renderHook, act } from '@testing-library/react-hooks';
import { useLogin } from './useLogin';

describe('useLogin', () => {
  it('should login successfully with valid credentials', async () => {
    const { result } = renderHook(() => useLogin());

    await act(async () => {
      await result.current.login('test@example.com', 'Password123');
    });

    expect(result.current.error).toBeNull();
    expect(result.current.loading).toBe(false);
  });

  it('should set error on invalid credentials', async () => {
    const { result } = renderHook(() => useLogin());

    await act(async () => {
      try {
        await result.current.login('test@example.com', 'WrongPassword');
      } catch (err) {
        // Expected
      }
    });

    expect(result.current.error).toBe('Invalid email or password');
  });

  it('should update loading state', async () => {
    const { result } = renderHook(() => useLogin());

    expect(result.current.loading).toBe(false);

    act(() => {
      result.current.login('test@example.com', 'Password123');
    });

    expect(result.current.loading).toBe(true);
  });
});
```

---

## ⚠️ Common Pitfalls to Avoid

### ❌ DO NOT:
1. **Return different error messages** for "user not found" vs "wrong password" (email enumeration risk)
2. **Log passwords** (plain or hashed)
3. **Skip lastLoginAt update** (useful for security monitoring)
4. **Forget to include passwordHash** when querying user for login (select: false by default)
5. **Use !== for password comparison** (use bcrypt.compare)
6. **Allow unlimited login attempts** (implement rate limiting in future)
7. **Store multiple sessions per user** in WatermelonDB without cleanup

### ✅ DO:
1. **Use same error message** for all authentication failures
2. **Log failed attempts** (email only, for security monitoring)
3. **Update lastLoginAt** on successful login
4. **Use bcrypt.compare** for password validation (constant-time)
5. **Reset navigation stack** after successful login (prevent back to login screen)
6. **Include user ID in JWT** (NFR13: data isolation)
7. **Test email enumeration prevention** (same error message, same response time)
8. **Handle network errors gracefully** on mobile (offline, timeout)

---

## 🔗 References

### Official Documentation
- [bcrypt npm](https://www.npmjs.com/package/bcrypt)
- [NestJS JWT](https://docs.nestjs.com/security/authentication#jwt-functionality)
- [TypeORM Query Builder](https://typeorm.io/select-query-builder)
- [React Navigation Reset](https://reactnavigation.org/docs/navigation-actions/#reset)

### Security Best Practices
- [OWASP: Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Preventing Username Enumeration](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/03-Identity_Management_Testing/04-Testing_for_Account_Enumeration_and_Guessable_User_Account)

### Architecture Documents
- **Main Architecture:** `architecture.md` (Identity Context, NFR13)
- **PRD:** `prd.md` (FR26: Login)
- **NFR13:** Data isolation via user ID in token

### Related Stories
- **Previous:** 1.2 User Registration (JWT infrastructure, User entity)
- **Next:** 1.4 User Logout (token invalidation)
- **Epic:** Epic 1 - Foundation & Authentification

---

## ✅ Definition of Done

**This story is DONE when:**

1. ✅ **Backend Implementation:**
   - Login endpoint POST /api/auth/login created
   - Password validation with bcrypt.compare
   - JWT token generation (7 days expiration)
   - lastLoginAt updated on successful login
   - Same error message for user not found vs wrong password
   - Failed login attempts logged
   - Input validation (email format)

2. ✅ **Mobile Implementation:**
   - LoginScreen UI created
   - LoginForm component with validation
   - useLogin hook implemented
   - API integration working (axios)
   - WatermelonDB update/create user after login
   - Navigation reset to MainApp after successful login
   - Error display (invalid credentials)

3. ✅ **Security:**
   - Email enumeration prevention verified (same error message)
   - Password never logged
   - Constant-time password comparison (bcrypt)
   - User ID included in JWT token (NFR13)
   - Failed attempts logged for monitoring

4. ✅ **Testing:**
   - Backend unit tests passing (AuthService)
   - Backend E2E tests passing (login endpoint)
   - Mobile hook tests passing (useLogin)
   - Mobile component tests passing (LoginScreen)
   - Email enumeration test passing (same error message)

5. ✅ **Verification:**
   - Can login with valid credentials via Postman/curl
   - Can login with valid credentials via mobile app
   - Wrong password returns 401 with "Invalid email or password"
   - Non-existent email returns 401 with same message
   - JWT token is valid and includes user ID
   - lastLoginAt is updated in database
   - User data persisted in WatermelonDB
   - Navigation to main app screen works

---

## 📝 Developer Notes

_This section will be filled by the developer during implementation._

**Implementation Log:**
- [ ] Backend login endpoint created
- [ ] Password validation with bcrypt implemented
- [ ] JWT token generation working
- [ ] lastLoginAt update implemented
- [ ] Failed login logging implemented
- [ ] Mobile LoginScreen created
- [ ] useLogin hook implemented
- [ ] Navigation reset working
- [ ] All tests passing

**Challenges Encountered:**
_TBD_

**Learnings for Next Stories:**
- Pattern established for authentication flows
- JWT infrastructure reusable for logout (Story 1.4)
- Error handling pattern for security-sensitive operations

---

**Story Created:** 2026-01-19
**Created by:** Scrum Master (BMAD Methodology)
**Context Engine Analysis:** ✅ Complete
**Ready for Development:** ✅ Yes

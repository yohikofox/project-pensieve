# Story 1.4: User Logout

**Story ID:** 1.4
**Epic:** Epic 1 - Foundation & Authentification
**Status:** ready-for-dev
**Story Key:** `1-4-user-logout`

---

## 📋 User Story

**As a** logged-in user
**I want** to log out of my account
**So that** my session is terminated and my data is secure on shared devices

---

## ✅ Acceptance Criteria

### Given I am currently logged in with an active session

**When** I trigger the logout action
**Then** my JWT token is invalidated (added to Redis blacklist)
**And** my local session data is cleared from the mobile app
**And** I am redirected to the login screen
**And** subsequent API requests with the old token are rejected

### Given I log out

**When** I close and reopen the app
**Then** I remain logged out and must authenticate again

### Given I log out

**When** any pending local data exists (offline captures)
**Then** the data remains securely stored locally for future sync
**And** I am warned if unsynchronized data exists (optional notification)

---

## 🎯 Developer Context & Implementation Guide

### 🏗️ Architecture Context

**Bounded Context:** Identity Context (Generic Subdomain)
**Responsibility:** Authentification, gestion utilisateurs, session management
**Aggregate:** `User`

**This story completes the authentication cycle:** Register → Login → **Logout**

**Key Challenge:** JWT tokens are stateless by design. Logout requires server-side token invalidation using Redis blacklist.

---

## 📦 Backend Implementation (NestJS)

### Identity Context Structure (Extends Stories 1.2, 1.3)

```
src/contexts/identity/
├── domain/
│   └── entities/
│       └── user.entity.ts          # Reuse from 1.2
├── application/
│   ├── commands/
│   │   ├── logout-user.command.ts   # NEW
│   │   └── logout-user.handler.ts   # NEW
│   └── services/
│       ├── auth.service.ts          # Extend with logout
│       └── token-blacklist.service.ts # NEW - Redis integration
├── infrastructure/
│   └── cache/
│       └── redis.service.ts         # Redis client (from Story 1.1)
└── presentation/
    ├── controllers/
    │   └── auth.controller.ts       # Add logout endpoint
    ├── guards/
    │   └── jwt-auth.guard.ts        # NEW - Check blacklist
    └── dto/
        └── logout-response.dto.ts   # NEW
```

---

## 🔐 JWT Token Invalidation Strategy

### Why Token Invalidation is Needed

**Problem:** JWT tokens are stateless and self-contained. Once issued, they remain valid until expiration (7 days).

**Without logout:** If a user logs out, their old token could still be used to access the API until expiration.

**Solution:** Token blacklist using Redis

---

### Redis Token Blacklist

**Library:** `ioredis` (already installed in Story 1.1)

**Service Implementation:**
```typescript
// src/contexts/identity/application/services/token-blacklist.service.ts
import { Injectable } from '@nestjs/common';
import { RedisService } from '../../../infrastructure/cache/redis.service';

@Injectable()
export class TokenBlacklistService {
  constructor(private readonly redisService: RedisService) {}

  /**
   * Add token to blacklist
   * TTL = remaining token validity time
   */
  async blacklistToken(token: string, expirationSeconds: number): Promise<void> {
    const key = `blacklist:${token}`;
    await this.redisService.set(key, '1', expirationSeconds);
  }

  /**
   * Check if token is blacklisted
   */
  async isBlacklisted(token: string): Promise<boolean> {
    const key = `blacklist:${token}`;
    const result = await this.redisService.get(key);
    return result !== null;
  }
}
```

**Key Design Decisions:**
1. **Redis TTL = Token TTL:** Blacklist entry expires when token would have expired naturally
2. **Key Pattern:** `blacklist:{token}` for easy identification
3. **Simple value:** Store `"1"` (we only need existence check)

---

### JWT Auth Guard (Token Validation + Blacklist Check)

```typescript
// src/contexts/identity/presentation/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { TokenBlacklistService } from '../../application/services/token-blacklist.service';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private readonly tokenBlacklistService: TokenBlacklistService) {
    super();
  }

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // First, validate JWT signature and expiration (passport-jwt)
    const isValid = await super.canActivate(context);
    if (!isValid) {
      return false;
    }

    // Extract token from Authorization header
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);

    if (!token) {
      throw new UnauthorizedException('Token not found');
    }

    // Check if token is blacklisted
    const isBlacklisted = await this.tokenBlacklistService.isBlacklisted(token);
    if (isBlacklisted) {
      throw new UnauthorizedException('Token has been invalidated');
    }

    return true;
  }

  private extractTokenFromHeader(request: any): string | null {
    const authHeader = request.headers.authorization;
    if (!authHeader) {
      return null;
    }

    const [type, token] = authHeader.split(' ');
    return type === 'Bearer' ? token : null;
  }
}
```

**Usage in Controllers:**
```typescript
@Controller('api/captures')
export class CapturesController {
  @UseGuards(JwtAuthGuard) // ← Will check blacklist
  @Get()
  async getCaptures() {
    // Protected route
  }
}
```

---

## 🎨 API Design

### POST /api/auth/logout

**Request Headers:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**No Request Body Required**

**Response (Success - 200 OK):**
```typescript
// src/contexts/identity/presentation/dto/logout-response.dto.ts
export class LogoutResponseDto {
  message: string;
  loggedOutAt: Date;
}
```

**Example Response:**
```json
{
  "message": "Successfully logged out",
  "loggedOutAt": "2026-01-19T12:00:00.000Z"
}
```

**Error Responses:**

**401 Unauthorized (No Token or Invalid Token):**
```json
{
  "statusCode": 401,
  "message": "Unauthorized",
  "error": "Unauthorized"
}
```

**401 Unauthorized (Token Already Blacklisted):**
```json
{
  "statusCode": 401,
  "message": "Token has been invalidated",
  "error": "Unauthorized"
}
```

---

## 🛠️ Backend Implementation Details

### Auth Controller (Extend from Stories 1.2, 1.3)

```typescript
// src/contexts/identity/presentation/controllers/auth.controller.ts
import { Controller, Post, Body, UseGuards, Request } from '@nestjs/common';
import { JwtAuthGuard } from '../guards/jwt-auth.guard';
import { AuthService } from '../../application/services/auth.service';
import { LogoutResponseDto } from '../dto/logout-response.dto';

@Controller('api/auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  // From Story 1.2
  @Post('register')
  async register(@Body() dto: RegisterDto) { /* ... */ }

  // From Story 1.3
  @Post('login')
  async login(@Body() dto: LoginDto) { /* ... */ }

  // NEW for Story 1.4
  @Post('logout')
  @UseGuards(JwtAuthGuard) // Requires valid, non-blacklisted token
  async logout(@Request() req): Promise<LogoutResponseDto> {
    const token = this.extractTokenFromHeader(req);
    return await this.authService.logout(token);
  }

  private extractTokenFromHeader(request: any): string {
    const authHeader = request.headers.authorization;
    const [, token] = authHeader.split(' ');
    return token;
  }
}
```

---

### Auth Service (Extend from Stories 1.2, 1.3)

```typescript
// src/contexts/identity/application/services/auth.service.ts
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { TokenBlacklistService } from './token-blacklist.service';

@Injectable()
export class AuthService {
  constructor(
    private readonly jwtService: JwtService,
    private readonly tokenBlacklistService: TokenBlacklistService
  ) {}

  // From Story 1.2
  async register(email: string, password: string) { /* ... */ }

  // From Story 1.3
  async login(email: string, password: string) { /* ... */ }

  // NEW for Story 1.4
  async logout(token: string): Promise<LogoutResponseDto> {
    // Decode token to get expiration time
    const decoded = this.jwtService.decode(token) as any;
    const expirationTime = decoded.exp;
    const currentTime = Math.floor(Date.now() / 1000);

    // Calculate remaining TTL (in seconds)
    const remainingTTL = expirationTime - currentTime;

    // Add token to blacklist with remaining TTL
    if (remainingTTL > 0) {
      await this.tokenBlacklistService.blacklistToken(token, remainingTTL);
    }

    return {
      message: 'Successfully logged out',
      loggedOutAt: new Date()
    };
  }
}
```

**Key Implementation Notes:**
1. **Calculate remaining TTL:** Only blacklist for time token would have been valid
2. **Check remainingTTL > 0:** Prevent negative TTL if token near expiration
3. **No database update needed:** Logout is stateless, only Redis entry

---

## 📱 Mobile Implementation (React Native + Expo)

### Identity Context Structure (Extends Stories 1.2, 1.3)

```
src/contexts/identity/
├── domain/
│   └── services/
│       └── auth.service.ts         # Extend with logout method
├── application/
│   ├── hooks/
│   │   ├── useLogout.ts            # NEW
│   │   └── useAuth.ts              # Extend with logout state
│   └── state/
│       └── auth.context.tsx        # Manage logout state
└── presentation/
    ├── screens/
    │   └── ProfileScreen.tsx       # NEW (with logout button)
    └── components/
        └── LogoutButton.tsx        # NEW
```

---

### API Service (Extend from Stories 1.2, 1.3)

```typescript
// src/contexts/identity/domain/services/auth.service.ts
import axios from 'axios';

const API_BASE_URL = process.env.API_BASE_URL || 'http://localhost:3000';

export interface LogoutResponse {
  message: string;
  loggedOutAt: string;
}

export const authService = {
  // From Story 1.2
  async register(data: RegisterRequest): Promise<RegisterResponse> { /* ... */ },

  // From Story 1.3
  async login(data: LoginRequest): Promise<LoginResponse> { /* ... */ },

  // NEW for Story 1.4
  async logout(accessToken: string): Promise<LogoutResponse> {
    const response = await axios.post<LogoutResponse>(
      `${API_BASE_URL}/api/auth/logout`,
      {},
      {
        headers: {
          Authorization: `Bearer ${accessToken}`
        }
      }
    );
    return response.data;
  }
};
```

---

### React Hook (useLogout)

```typescript
// src/contexts/identity/application/hooks/useLogout.ts
import { useState } from 'react';
import { authService } from '../../domain/services/auth.service';
import { useDatabase } from '@nozbe/watermelondb/hooks';
import { User } from '../../domain/models/user.model';
import { useNavigation } from '@react-navigation/native';
import { Q } from '@nozbe/watermelondb';

export const useLogout = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const database = useDatabase();
  const navigation = useNavigation();

  const logout = async () => {
    setLoading(true);
    setError(null);

    try {
      // Get current user and token from WatermelonDB
      const usersCollection = database.get<User>('users');
      const users = await usersCollection.query().fetch();

      if (users.length === 0) {
        throw new Error('No user session found');
      }

      const user = users[0];
      const accessToken = user.accessToken;

      // Call backend API to blacklist token
      await authService.logout(accessToken);

      // Clear user data from WatermelonDB
      await database.write(async () => {
        await user.markAsDeleted(); // Soft delete
        // OR await user.destroyPermanently(); // Hard delete
      });

      // Reset navigation to login screen
      navigation.reset({
        index: 0,
        routes: [{ name: 'Login' }],
      });
    } catch (err: any) {
      const message = err.response?.data?.message || 'Logout failed';
      setError(message);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  return { logout, loading, error };
};
```

**Key Implementation Notes:**
1. **Get token from WatermelonDB:** Need token to send to backend
2. **Backend invalidation first:** Blacklist token before local cleanup
3. **Clear local data:** Delete user record from WatermelonDB
4. **Navigation reset:** Prevent back navigation to authenticated screens
5. **Keep offline captures:** Only delete User record, not Capture data

---

### Logout Button Component

```typescript
// src/contexts/identity/presentation/components/LogoutButton.tsx
import React from 'react';
import { TouchableOpacity, Text, ActivityIndicator, Alert } from 'react-native';
import { useLogout } from '../../application/hooks/useLogout';
import { useDatabase } from '@nozbe/watermelondb/hooks';
import { Capture } from '../../../capture/domain/models/capture.model';
import { Q } from '@nozbe/watermelondb';

export const LogoutButton: React.FC = () => {
  const { logout, loading } = useLogout();
  const database = useDatabase();

  const handleLogout = async () => {
    // Check for unsynced captures
    const capturesCollection = database.get<Capture>('captures');
    const unsyncedCaptures = await capturesCollection
      .query(Q.where('sync_status', 'pending'))
      .fetchCount();

    if (unsyncedCaptures > 0) {
      // Warn user about unsynced data (AC3)
      Alert.alert(
        'Unsynced Data',
        `You have ${unsyncedCaptures} capture(s) that haven't been synced. They will remain on this device.`,
        [
          {
            text: 'Cancel',
            style: 'cancel'
          },
          {
            text: 'Logout Anyway',
            onPress: async () => {
              await logout();
            }
          }
        ]
      );
    } else {
      // No unsynced data, logout directly
      await logout();
    }
  };

  return (
    <TouchableOpacity
      onPress={handleLogout}
      disabled={loading}
      style={{ padding: 10, backgroundColor: 'red' }}
    >
      {loading ? (
        <ActivityIndicator color="white" />
      ) : (
        <Text style={{ color: 'white' }}>Logout</Text>
      )}
    </TouchableOpacity>
  );
};
```

**Key Features:**
1. **Check unsynced captures:** Query WatermelonDB for `sync_status = 'pending'`
2. **Warn user:** Alert dialog if unsynced data exists (AC3)
3. **User choice:** Cancel or proceed with logout
4. **Preserve captures:** Offline data remains on device after logout

---

## 🔒 Security Considerations

### Token Blacklist Security

**Why Blacklist is Necessary:**
- Prevents use of stolen tokens after logout
- Enables immediate session termination
- Critical for shared devices (security requirement)

**Performance Considerations:**
- Redis is fast (sub-millisecond lookups)
- Guard checks blacklist on EVERY protected request
- TTL ensures automatic cleanup (no manual maintenance)

**Alternative Approaches (Not Used):**
1. **Short-lived tokens + Refresh tokens:** More complex, not needed for MVP
2. **Database blacklist:** Too slow for every request
3. **No blacklist:** Insecure, tokens valid until expiration

---

### Data Preservation on Logout

**Requirement (AC3):** "The data remains securely stored locally for future sync"

**Implementation:**
```typescript
// ✅ CORRECT: Only delete User record
await database.write(async () => {
  await user.markAsDeleted();
});

// ❌ INCORRECT: Don't delete all data
await database.write(async () => {
  await database.unsafeResetDatabase(); // DON'T DO THIS
});
```

**What to Keep:**
- ✅ Captures (audio, text)
- ✅ Thoughts (digested content)
- ✅ Todos (actions)
- ✅ Ideas (insights)

**What to Delete:**
- ❌ User record (email, token)
- ❌ Session data

**Rationale:**
- User may want to log in with different account
- Offline captures should sync when user logs back in
- Data loss = 0 tolerance (NFR6)

---

## 🧪 Testing Requirements

### Backend Tests

**Unit Tests (AuthService):**
```typescript
// auth.service.spec.ts
describe('AuthService - logout', () => {
  it('should blacklist token on logout', async () => {
    const token = 'valid.jwt.token';
    await authService.logout(token);

    const isBlacklisted = await tokenBlacklistService.isBlacklisted(token);
    expect(isBlacklisted).toBe(true);
  });

  it('should calculate correct TTL for blacklist', async () => {
    const token = jwtService.sign({ sub: 'user-id', exp: Math.floor(Date.now() / 1000) + 3600 });

    const setSpy = jest.spyOn(redisService, 'set');
    await authService.logout(token);

    expect(setSpy).toHaveBeenCalledWith(
      expect.any(String),
      '1',
      expect.any(Number)
    );

    const ttl = setSpy.mock.calls[0][2];
    expect(ttl).toBeGreaterThan(0);
    expect(ttl).toBeLessThanOrEqual(3600);
  });

  it('should not blacklist already expired token', async () => {
    const expiredToken = jwtService.sign({
      sub: 'user-id',
      exp: Math.floor(Date.now() / 1000) - 3600 // Expired 1 hour ago
    });

    const setSpy = jest.spyOn(redisService, 'set');
    await authService.logout(expiredToken);

    expect(setSpy).not.toHaveBeenCalled();
  });
});
```

---

**Integration Tests (JwtAuthGuard):**
```typescript
// jwt-auth.guard.spec.ts
describe('JwtAuthGuard', () => {
  it('should allow request with valid non-blacklisted token', async () => {
    const token = await generateValidToken();
    const request = createMockRequest(token);

    const canActivate = await jwtAuthGuard.canActivate(createMockContext(request));
    expect(canActivate).toBe(true);
  });

  it('should reject request with blacklisted token', async () => {
    const token = await generateValidToken();
    await tokenBlacklistService.blacklistToken(token, 3600);

    const request = createMockRequest(token);

    await expect(
      jwtAuthGuard.canActivate(createMockContext(request))
    ).rejects.toThrow(UnauthorizedException);
  });

  it('should reject request with expired token', async () => {
    const expiredToken = await generateExpiredToken();
    const request = createMockRequest(expiredToken);

    await expect(
      jwtAuthGuard.canActivate(createMockContext(request))
    ).rejects.toThrow(UnauthorizedException);
  });

  it('should reject request without token', async () => {
    const request = createMockRequest(null);

    await expect(
      jwtAuthGuard.canActivate(createMockContext(request))
    ).rejects.toThrow(UnauthorizedException);
  });
});
```

---

**E2E Tests (API):**
```typescript
// auth.e2e-spec.ts
describe('/api/auth/logout (POST)', () => {
  let accessToken: string;

  beforeEach(async () => {
    // Register and login to get token
    await request(app.getHttpServer())
      .post('/api/auth/register')
      .send({ email: 'test@example.com', password: 'Password123' });

    const loginResponse = await request(app.getHttpServer())
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'Password123' });

    accessToken = loginResponse.body.accessToken;
  });

  it('should logout successfully with valid token', () => {
    return request(app.getHttpServer())
      .post('/api/auth/logout')
      .set('Authorization', `Bearer ${accessToken}`)
      .expect(200)
      .expect((res) => {
        expect(res.body.message).toBe('Successfully logged out');
        expect(res.body.loggedOutAt).toBeDefined();
      });
  });

  it('should reject subsequent requests with logged-out token', async () => {
    // Logout
    await request(app.getHttpServer())
      .post('/api/auth/logout')
      .set('Authorization', `Bearer ${accessToken}`)
      .expect(200);

    // Try to use token again (should fail)
    return request(app.getHttpServer())
      .get('/api/captures') // Protected route
      .set('Authorization', `Bearer ${accessToken}`)
      .expect(401)
      .expect((res) => {
        expect(res.body.message).toContain('invalidated');
      });
  });

  it('should return 401 for logout without token', () => {
    return request(app.getHttpServer())
      .post('/api/auth/logout')
      .expect(401);
  });

  it('should return 401 for logout with already blacklisted token', async () => {
    // Logout once
    await request(app.getHttpServer())
      .post('/api/auth/logout')
      .set('Authorization', `Bearer ${accessToken}`)
      .expect(200);

    // Try to logout again with same token
    return request(app.getHttpServer())
      .post('/api/auth/logout')
      .set('Authorization', `Bearer ${accessToken}`)
      .expect(401);
  });
});
```

---

### Mobile Tests

**Hook Tests:**
```typescript
// useLogout.test.ts
import { renderHook, act } from '@testing-library/react-hooks';
import { useLogout } from './useLogout';

describe('useLogout', () => {
  it('should logout successfully', async () => {
    const { result } = renderHook(() => useLogout());

    await act(async () => {
      await result.current.logout();
    });

    expect(result.current.error).toBeNull();
    expect(result.current.loading).toBe(false);
  });

  it('should clear local user data', async () => {
    const { result } = renderHook(() => useLogout());

    await act(async () => {
      await result.current.logout();
    });

    const users = await database.get('users').query().fetch();
    expect(users.length).toBe(0);
  });

  it('should preserve offline captures', async () => {
    // Create test capture
    await database.write(async () => {
      await database.get('captures').create((capture) => {
        capture.type = 'text';
        capture.content = 'Test thought';
        capture.syncStatus = 'pending';
      });
    });

    const { result } = renderHook(() => useLogout());

    await act(async () => {
      await result.current.logout();
    });

    const captures = await database.get('captures').query().fetch();
    expect(captures.length).toBe(1); // Still exists
  });
});
```

---

## ⚠️ Common Pitfalls to Avoid

### ❌ DO NOT:
1. **Skip token blacklist** (tokens remain valid until expiration = security risk)
2. **Delete offline captures on logout** (NFR6: zero data loss)
3. **Use database for blacklist** (too slow for every request)
4. **Forget to check blacklist in guard** (must check on EVERY protected route)
5. **Allow navigation back to authenticated screens** (use navigation.reset)
6. **Blacklist token with infinite TTL** (use remaining token TTL)
7. **Logout without backend call** (token still valid server-side)

### ✅ DO:
1. **Use Redis for blacklist** (fast, automatic expiration)
2. **Calculate remaining TTL** (efficient memory usage)
3. **Preserve offline data** (captures, thoughts, todos)
4. **Warn about unsynced data** (optional notification)
5. **Reset navigation stack** (prevent back button to authenticated screens)
6. **Check blacklist in JwtAuthGuard** (apply to all protected routes)
7. **Clear user record from WatermelonDB** (logout state persists)
8. **Test token invalidation** (E2E test verifies blacklist works)

---

## 🔗 References

### Official Documentation
- [Redis Commands](https://redis.io/commands/)
- [ioredis npm](https://www.npmjs.com/package/ioredis)
- [NestJS Guards](https://docs.nestjs.com/guards)
- [Passport JWT Strategy](http://www.passportjs.org/packages/passport-jwt/)
- [React Navigation Reset](https://reactnavigation.org/docs/navigation-actions/#reset)

### Security Best Practices
- [JWT Token Revocation](https://auth0.com/blog/blacklist-json-web-token-api-keys/)
- [OWASP: Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)

### Architecture Documents
- **Main Architecture:** `architecture.md` (Identity Context, Redis cache)
- **PRD:** `prd.md` (FR27: Logout)
- **NFR6:** Zero data loss tolerance

### Related Stories
- **Previous:** 1.3 User Login (JWT infrastructure, auth patterns)
- **Next:** 1.5 Password Recovery
- **Epic:** Epic 1 - Foundation & Authentification

---

## ✅ Definition of Done

**This story is DONE when:**

1. ✅ **Backend Implementation:**
   - Logout endpoint POST /api/auth/logout created
   - TokenBlacklistService implemented (Redis integration)
   - JwtAuthGuard extended to check blacklist
   - Token TTL calculated correctly
   - Guard applied to all protected routes

2. ✅ **Mobile Implementation:**
   - useLogout hook implemented
   - Logout button component created
   - Profile/Settings screen with logout option
   - User data cleared from WatermelonDB
   - Navigation reset to Login screen
   - Unsynced data warning implemented

3. ✅ **Security:**
   - Token added to Redis blacklist
   - Blacklisted tokens rejected by guard
   - TTL = remaining token validity
   - Logout persists after app restart

4. ✅ **Data Preservation:**
   - Offline captures NOT deleted
   - User warned about unsynced data
   - Only User record deleted from WatermelonDB

5. ✅ **Testing:**
   - Backend unit tests passing (AuthService, TokenBlacklistService)
   - Backend integration tests passing (JwtAuthGuard)
   - Backend E2E tests passing (logout endpoint, token invalidation)
   - Mobile hook tests passing (useLogout)
   - Mobile component tests passing (LogoutButton)

6. ✅ **Verification:**
   - Can logout via Postman/curl with valid token
   - Can logout via mobile app
   - Old token rejected after logout (401)
   - User redirected to login screen
   - Cannot navigate back to authenticated screens
   - Logout state persists after app restart
   - Offline captures preserved after logout
   - Warning shown if unsynced data exists

---

## 📝 Developer Notes

_This section will be filled by the developer during implementation._

**Implementation Log:**
- [ ] TokenBlacklistService created
- [ ] Redis integration working
- [ ] JwtAuthGuard extended with blacklist check
- [ ] Logout endpoint created
- [ ] useLogout hook implemented
- [ ] LogoutButton component created
- [ ] Navigation reset working
- [ ] Data preservation verified
- [ ] All tests passing

**Challenges Encountered:**
_TBD_

**Learnings for Next Stories:**
- Token blacklist pattern reusable for other token types (refresh tokens in future)
- Guard pattern applies to all protected endpoints
- Data preservation critical for offline-first apps

---

**Story Created:** 2026-01-19
**Created by:** Scrum Master (BMAD Methodology)
**Context Engine Analysis:** ✅ Complete
**Ready for Development:** ✅ Yes

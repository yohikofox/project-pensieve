# Story 1.5: Password Recovery

**Epic:** Epic 1 - Foundation & Authentification
**Story ID:** 1.5
**Created:** 2026-01-19
**Status:** Ready for Development

---

## User Story

As a **registered user who forgot my password**,
I want **to reset my password via email**,
So that **I can regain access to my account and encrypted captures**.

---

## Acceptance Criteria

### AC1: Request Password Reset

**Given** I am a registered user who forgot my password
**When** I request a password reset with my email address
**Then** a unique password reset token is generated with 1-hour expiration
**And** a password reset email is sent to my registered email address
**And** the email contains a secure reset link with the token
**And** the system does not reveal whether the email exists (security best practice)

### AC2: Navigate to Reset Form

**Given** I receive the password reset email
**When** I click the reset link within 1 hour
**Then** I am directed to a password reset form
**And** the token is validated before allowing password change

### AC3: Complete Password Reset

**Given** I am on the password reset form with a valid token
**When** I provide a new strong password (min 8 chars, 1 uppercase, 1 number)
**Then** my password is updated in the database (bcrypt hashed)
**And** the reset token is invalidated immediately
**And** all existing sessions are terminated (security measure)
**And** I receive a confirmation that my password was changed
**And** I can log in with the new password

### AC4: Handle Invalid/Expired Tokens

**Given** I attempt to use an expired or invalid reset token
**When** I try to reset my password
**Then** I receive an error message "Reset link expired or invalid"
**And** I am prompted to request a new reset link

---

## Implementation Details

### Architecture Context

**Bounded Context:** Identity Context (Generic subdomain)
**Tech Stack:** NestJS (Backend), React Native + Expo (Mobile), Nodemailer (Email), Redis (Token blacklist)

**Related ADRs:**
- ADR-010.1: JWT Authentication Strategy
- ADR-010.4: HTTPS/TLS Enforcement (secure reset links)
- ADR-010.5: RBAC and Data Isolation

**Related Stories:**
- Story 1.2: User Registration (password validation patterns)
- Story 1.3: User Login (password comparison patterns)
- Story 1.4: User Logout (token blacklist patterns)

---

### Backend Implementation

#### 1. Database Schema - Password Reset Tokens

Create a new entity to store password reset tokens:

```typescript
// src/modules/identity/domain/entities/password-reset-token.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn, Index, ManyToOne, JoinColumn } from 'typeorm';
import { UserSchema } from './user.entity';

@Entity('password_reset_tokens')
export class PasswordResetTokenSchema {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  @Index()
  userId: string;

  @ManyToOne(() => UserSchema, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'userId' })
  user: UserSchema;

  // Token hashé (SHA-256) pour sécurité
  @Column()
  @Index()
  tokenHash: string;

  @Column({ type: 'timestamp' })
  expiresAt: Date;

  @Column({ default: false })
  used: boolean;

  @CreateDateColumn()
  createdAt: Date;
}
```

**Migration:**
```bash
npm run typeorm migration:generate -- -n CreatePasswordResetTokensTable
npm run typeorm migration:run
```

---

#### 2. Email Service (Nodemailer)

Configure email sending service:

```typescript
// src/modules/shared/infrastructure/email/email.service.ts
import { Injectable } from '@nestjs/common';
import * as nodemailer from 'nodemailer';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class EmailService {
  private transporter: nodemailer.Transporter;

  constructor(private configService: ConfigService) {
    this.transporter = nodemailer.createTransport({
      host: this.configService.get('SMTP_HOST'),
      port: this.configService.get('SMTP_PORT'),
      secure: this.configService.get('SMTP_SECURE') === 'true', // true for 465, false for 587
      auth: {
        user: this.configService.get('SMTP_USER'),
        pass: this.configService.get('SMTP_PASS'),
      },
    });
  }

  async sendPasswordResetEmail(
    email: string,
    resetToken: string,
  ): Promise<void> {
    const resetUrl = `${this.configService.get('MOBILE_APP_SCHEME')}://reset-password?token=${resetToken}`;

    const mailOptions = {
      from: this.configService.get('SMTP_FROM') || 'noreply@pensine.app',
      to: email,
      subject: 'Réinitialisation de votre mot de passe - Pensine',
      html: `
        <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
          <h2>Réinitialisation de mot de passe</h2>

          <p>Vous avez demandé la réinitialisation de votre mot de passe Pensine.</p>

          <p>Cliquez sur le lien ci-dessous pour créer un nouveau mot de passe :</p>

          <p style="margin: 30px 0;">
            <a href="${resetUrl}"
               style="background-color: #007AFF;
                      color: white;
                      padding: 12px 24px;
                      text-decoration: none;
                      border-radius: 6px;
                      display: inline-block;">
              Réinitialiser mon mot de passe
            </a>
          </p>

          <p><strong>Ce lien expire dans 1 heure.</strong></p>

          <p>Si vous n'avez pas demandé cette réinitialisation, ignorez cet email.</p>

          <hr style="margin: 40px 0; border: none; border-top: 1px solid #ddd;">

          <p style="color: #999; font-size: 12px;">
            Vous ne pouvez pas cliquer sur le lien ? Copiez-collez cette URL dans votre navigateur :<br>
            ${resetUrl}
          </p>
        </div>
      `,
      text: `
Réinitialisation de mot de passe - Pensine

Vous avez demandé la réinitialisation de votre mot de passe.

Ouvrez ce lien pour créer un nouveau mot de passe :
${resetUrl}

Ce lien expire dans 1 heure.

Si vous n'avez pas demandé cette réinitialisation, ignorez cet email.
      `,
    };

    await this.transporter.sendMail(mailOptions);
  }

  async sendPasswordChangedConfirmation(email: string): Promise<void> {
    const mailOptions = {
      from: this.configService.get('SMTP_FROM') || 'noreply@pensine.app',
      to: email,
      subject: 'Votre mot de passe a été modifié - Pensine',
      html: `
        <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
          <h2>Mot de passe modifié avec succès</h2>

          <p>Votre mot de passe Pensine vient d'être modifié.</p>

          <p><strong>Si vous n'êtes pas à l'origine de ce changement, veuillez contacter le support immédiatement.</strong></p>

          <p>Toutes vos sessions ont été déconnectées par mesure de sécurité.</p>
        </div>
      `,
      text: `
Mot de passe modifié avec succès - Pensine

Votre mot de passe Pensine vient d'être modifié.

Si vous n'êtes pas à l'origine de ce changement, veuillez contacter le support immédiatement.

Toutes vos sessions ont été déconnectées par mesure de sécurité.
      `,
    };

    await this.transporter.sendMail(mailOptions);
  }
}
```

**Environment Variables (.env):**
```bash
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password
SMTP_FROM="Pensine <noreply@pensine.app>"
MOBILE_APP_SCHEME=pensine
```

**Gmail Configuration (Development):**
1. Enable 2FA on Google Account
2. Generate App Password: https://myaccount.google.com/apppasswords
3. Use App Password as `SMTP_PASS`

---

#### 3. Password Reset Service

Create service to handle reset logic:

```typescript
// src/modules/identity/application/services/password-reset.service.ts
import { Injectable, BadRequestException, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, LessThan } from 'typeorm';
import { PasswordResetTokenSchema } from '../../domain/entities/password-reset-token.entity';
import { UserSchema } from '../../domain/entities/user.entity';
import { EmailService } from '../../../shared/infrastructure/email/email.service';
import { TokenBlacklistService } from './token-blacklist.service';
import * as crypto from 'crypto';
import * as bcrypt from 'bcrypt';

@Injectable()
export class PasswordResetService {
  constructor(
    @InjectRepository(PasswordResetTokenSchema)
    private resetTokenRepository: Repository<PasswordResetTokenSchema>,
    @InjectRepository(UserSchema)
    private userRepository: Repository<UserSchema>,
    private emailService: EmailService,
    private tokenBlacklistService: TokenBlacklistService,
  ) {}

  /**
   * Request password reset - Send email with reset token
   * SECURITY: Always return same message (don't reveal if email exists)
   */
  async requestPasswordReset(email: string): Promise<{ message: string }> {
    const user = await this.userRepository.findOne({ where: { email } });

    // ✅ SECURITY: Always return success even if user doesn't exist
    // This prevents email enumeration attacks
    if (!user) {
      return {
        message: 'If this email exists, you will receive a password reset link shortly.',
      };
    }

    // Generate secure random token (32 bytes = 256 bits)
    const resetToken = crypto.randomBytes(32).toString('hex');

    // Hash token before storing (SHA-256)
    const tokenHash = crypto
      .createHash('sha256')
      .update(resetToken)
      .digest('hex');

    // Set expiration to 1 hour from now
    const expiresAt = new Date();
    expiresAt.setHours(expiresAt.getHours() + 1);

    // Invalidate any existing reset tokens for this user
    await this.resetTokenRepository.update(
      { userId: user.id, used: false },
      { used: true },
    );

    // Create new reset token
    await this.resetTokenRepository.save({
      userId: user.id,
      tokenHash,
      expiresAt,
    });

    // Send email with reset token (plain text token, not hash)
    await this.emailService.sendPasswordResetEmail(user.email, resetToken);

    return {
      message: 'If this email exists, you will receive a password reset link shortly.',
    };
  }

  /**
   * Validate reset token and return user email
   * Used by mobile app to verify token before showing reset form
   */
  async validateResetToken(token: string): Promise<{ email: string }> {
    const tokenHash = crypto
      .createHash('sha256')
      .update(token)
      .digest('hex');

    const resetToken = await this.resetTokenRepository.findOne({
      where: { tokenHash, used: false },
      relations: ['user'],
    });

    if (!resetToken) {
      throw new BadRequestException('Reset link expired or invalid');
    }

    // Check expiration
    if (new Date() > resetToken.expiresAt) {
      throw new BadRequestException('Reset link expired or invalid');
    }

    return { email: resetToken.user.email };
  }

  /**
   * Reset password with token
   */
  async resetPassword(
    token: string,
    newPassword: string,
  ): Promise<{ message: string }> {
    // Validate password strength
    this.validatePasswordStrength(newPassword);

    const tokenHash = crypto
      .createHash('sha256')
      .update(token)
      .digest('hex');

    const resetToken = await this.resetTokenRepository.findOne({
      where: { tokenHash, used: false },
      relations: ['user'],
    });

    if (!resetToken) {
      throw new BadRequestException('Reset link expired or invalid');
    }

    // Check expiration
    if (new Date() > resetToken.expiresAt) {
      throw new BadRequestException('Reset link expired or invalid');
    }

    const user = resetToken.user;

    // Hash new password
    const saltRounds = 12;
    const passwordHash = await bcrypt.hash(newPassword, saltRounds);

    // Update user password
    await this.userRepository.update(user.id, { passwordHash });

    // Mark token as used
    await this.resetTokenRepository.update(resetToken.id, { used: true });

    // ✅ SECURITY: Invalidate ALL existing sessions (blacklist all JWT tokens)
    // This is critical: if someone got access to user's account, they're now logged out
    await this.invalidateAllUserSessions(user.id);

    // Send confirmation email
    await this.emailService.sendPasswordChangedConfirmation(user.email);

    return {
      message: 'Password successfully reset. All sessions have been logged out for security.',
    };
  }

  /**
   * Invalidate all JWT tokens for a user
   * Since we can't enumerate all tokens, we use a different strategy:
   * - Add userId to Redis with current timestamp
   * - In JwtAuthGuard, reject tokens issued before this timestamp
   */
  private async invalidateAllUserSessions(userId: string): Promise<void> {
    const currentTimestamp = Math.floor(Date.now() / 1000);
    const key = `password-reset:${userId}`;

    // Store timestamp for 30 days (refresh token max lifetime)
    await this.tokenBlacklistService.set(key, currentTimestamp.toString(), 30 * 24 * 60 * 60);
  }

  /**
   * Validate password strength
   * Same rules as registration
   */
  private validatePasswordStrength(password: string): void {
    if (password.length < 8) {
      throw new BadRequestException('Password must be at least 8 characters');
    }

    if (!/[A-Z]/.test(password)) {
      throw new BadRequestException('Password must contain at least one uppercase letter');
    }

    if (!/[0-9]/.test(password)) {
      throw new BadRequestException('Password must contain at least one number');
    }
  }

  /**
   * Cleanup expired tokens (run as cron job)
   */
  async cleanupExpiredTokens(): Promise<void> {
    await this.resetTokenRepository.delete({
      expiresAt: LessThan(new Date()),
    });
  }
}
```

---

#### 4. Updated Token Blacklist Service

Extend the service from Story 1.4 to support password reset invalidation:

```typescript
// src/modules/identity/application/services/token-blacklist.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Redis } from 'ioredis';
import { InjectRedis } from '@nestjs-modules/ioredis';

@Injectable()
export class TokenBlacklistService {
  constructor(
    @InjectRedis() private readonly redisClient: Redis,
  ) {}

  // Existing methods from Story 1.4
  async blacklistToken(token: string, expirationSeconds: number): Promise<void> {
    const key = `blacklist:${token}`;
    await this.redisClient.set(key, '1', 'EX', expirationSeconds);
  }

  async isBlacklisted(token: string): Promise<boolean> {
    const key = `blacklist:${token}`;
    const result = await this.redisClient.get(key);
    return result !== null;
  }

  // ✅ NEW: Generic set method for password reset tracking
  async set(key: string, value: string, expirationSeconds: number): Promise<void> {
    await this.redisClient.set(key, value, 'EX', expirationSeconds);
  }

  async get(key: string): Promise<string | null> {
    return await this.redisClient.get(key);
  }

  /**
   * Check if token was issued before password reset
   * Called from JwtAuthGuard
   */
  async isTokenIssuedBeforePasswordReset(userId: string, tokenIssuedAt: number): Promise<boolean> {
    const key = `password-reset:${userId}`;
    const resetTimestamp = await this.get(key);

    if (!resetTimestamp) {
      return false; // No password reset occurred
    }

    // Token issued before password reset = invalid
    return tokenIssuedAt < parseInt(resetTimestamp, 10);
  }
}
```

---

#### 5. Updated JwtAuthGuard

Extend guard to check password reset invalidation:

```typescript
// src/modules/identity/infrastructure/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { TokenBlacklistService } from '../../application/services/token-blacklist.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(
    private tokenBlacklistService: TokenBlacklistService,
    private jwtService: JwtService,
  ) {
    super();
  }

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);

    if (!token) {
      throw new UnauthorizedException('No token provided');
    }

    // ✅ Check 1: Token in blacklist (logout)
    const isBlacklisted = await this.tokenBlacklistService.isBlacklisted(token);
    if (isBlacklisted) {
      throw new UnauthorizedException('Token has been revoked');
    }

    // Decode token to get userId and iat
    const decoded = this.jwtService.decode(token) as any;
    if (!decoded || !decoded.sub || !decoded.iat) {
      throw new UnauthorizedException('Invalid token');
    }

    // ✅ Check 2: Token issued before password reset
    const issuedBeforeReset = await this.tokenBlacklistService.isTokenIssuedBeforePasswordReset(
      decoded.sub,
      decoded.iat,
    );

    if (issuedBeforeReset) {
      throw new UnauthorizedException('Session expired due to password change');
    }

    // Continue with normal JWT validation
    return super.canActivate(context) as Promise<boolean>;
  }

  private extractTokenFromHeader(request: any): string | null {
    const authHeader = request.headers.authorization;
    if (!authHeader) return null;

    const [type, token] = authHeader.split(' ');
    return type === 'Bearer' ? token : null;
  }
}
```

---

#### 6. Password Reset Controller

Create endpoints for password reset flow:

```typescript
// src/modules/identity/infrastructure/http/controllers/password-reset.controller.ts
import { Controller, Post, Body, Get, Query } from '@nestjs/common';
import { PasswordResetService } from '../../../application/services/password-reset.service';

class ForgotPasswordDto {
  email: string;
}

class ResetPasswordDto {
  token: string;
  newPassword: string;
}

@Controller('api/auth')
export class PasswordResetController {
  constructor(private passwordResetService: PasswordResetService) {}

  @Post('forgot-password')
  async forgotPassword(@Body() dto: ForgotPasswordDto) {
    return this.passwordResetService.requestPasswordReset(dto.email);
  }

  @Get('validate-reset-token')
  async validateResetToken(@Query('token') token: string) {
    return this.passwordResetService.validateResetToken(token);
  }

  @Post('reset-password')
  async resetPassword(@Body() dto: ResetPasswordDto) {
    return this.passwordResetService.resetPassword(dto.token, dto.newPassword);
  }
}
```

---

#### 7. Cron Job for Token Cleanup

Add scheduled task to cleanup expired tokens:

```typescript
// src/modules/identity/infrastructure/tasks/password-reset-cleanup.task.ts
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { PasswordResetService } from '../../application/services/password-reset.service';

@Injectable()
export class PasswordResetCleanupTask {
  constructor(private passwordResetService: PasswordResetService) {}

  @Cron(CronExpression.EVERY_DAY_AT_3AM)
  async handleCleanup() {
    await this.passwordResetService.cleanupExpiredTokens();
  }
}
```

**Install dependency:**
```bash
npm install @nestjs/schedule
```

**Register in module:**
```typescript
// src/modules/identity/identity.module.ts
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot(),
    // ...
  ],
  providers: [PasswordResetCleanupTask, ...],
})
export class IdentityModule {}
```

---

### Mobile Implementation

#### 1. Deep Linking Configuration

Configure app to handle reset password links:

```typescript
// app.json
{
  "expo": {
    "scheme": "pensine",
    "ios": {
      "bundleIdentifier": "com.pensine.app",
      "associatedDomains": ["applinks:pensine.app"]
    },
    "android": {
      "package": "com.pensine.app",
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [
            {
              "scheme": "https",
              "host": "pensine.app",
              "pathPrefix": "/reset-password"
            },
            {
              "scheme": "pensine"
            }
          ],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

**Install dependency:**
```bash
npx expo install expo-linking
```

---

#### 2. Forgot Password Screen

Create screen for requesting password reset:

```typescript
// src/screens/auth/ForgotPasswordScreen.tsx
import React, { useState } from 'react';
import { View, Text, TextInput, TouchableOpacity, Alert, StyleSheet } from 'react-native';
import { useForgotPassword } from '../../hooks/auth/useForgotPassword';

export const ForgotPasswordScreen = ({ navigation }) => {
  const [email, setEmail] = useState('');
  const { requestReset, loading } = useForgotPassword();

  const handleSubmit = async () => {
    if (!email.trim()) {
      Alert.alert('Error', 'Please enter your email address');
      return;
    }

    try {
      await requestReset(email);
      Alert.alert(
        'Check your email',
        'If this email exists, you will receive a password reset link shortly.',
        [{ text: 'OK', onPress: () => navigation.goBack() }]
      );
    } catch (error) {
      Alert.alert('Error', error.message || 'Failed to send reset email');
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Forgot Password</Text>
      <Text style={styles.subtitle}>
        Enter your email address and we'll send you a link to reset your password.
      </Text>

      <TextInput
        style={styles.input}
        placeholder="Email"
        value={email}
        onChangeText={setEmail}
        keyboardType="email-address"
        autoCapitalize="none"
        autoComplete="email"
        editable={!loading}
      />

      <TouchableOpacity
        style={[styles.button, loading && styles.buttonDisabled]}
        onPress={handleSubmit}
        disabled={loading}
      >
        <Text style={styles.buttonText}>
          {loading ? 'Sending...' : 'Send Reset Link'}
        </Text>
      </TouchableOpacity>

      <TouchableOpacity
        style={styles.backButton}
        onPress={() => navigation.goBack()}
      >
        <Text style={styles.backButtonText}>Back to Login</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 24,
    backgroundColor: '#fff',
    justifyContent: 'center',
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    marginBottom: 12,
  },
  subtitle: {
    fontSize: 16,
    color: '#666',
    marginBottom: 32,
  },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 16,
    fontSize: 16,
    marginBottom: 16,
  },
  button: {
    backgroundColor: '#007AFF',
    padding: 16,
    borderRadius: 8,
    alignItems: 'center',
  },
  buttonDisabled: {
    opacity: 0.6,
  },
  buttonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  backButton: {
    marginTop: 16,
    alignItems: 'center',
  },
  backButtonText: {
    color: '#007AFF',
    fontSize: 16,
  },
});
```

---

#### 3. Reset Password Screen

Create screen for resetting password with token:

```typescript
// src/screens/auth/ResetPasswordScreen.tsx
import React, { useState, useEffect } from 'react';
import { View, Text, TextInput, TouchableOpacity, Alert, StyleSheet, ActivityIndicator } from 'react-native';
import { useResetPassword } from '../../hooks/auth/useResetPassword';
import { useRoute } from '@react-navigation/native';

export const ResetPasswordScreen = ({ navigation }) => {
  const route = useRoute();
  const { token } = route.params || {};

  const [newPassword, setNewPassword] = useState('');
  const [confirmPassword, setConfirmPassword] = useState('');
  const [email, setEmail] = useState<string | null>(null);

  const { validateToken, resetPassword, loading, validating } = useResetPassword();

  useEffect(() => {
    if (token) {
      validateTokenOnMount();
    } else {
      Alert.alert('Error', 'Invalid reset link', [
        { text: 'OK', onPress: () => navigation.replace('Login') }
      ]);
    }
  }, [token]);

  const validateTokenOnMount = async () => {
    try {
      const result = await validateToken(token);
      setEmail(result.email);
    } catch (error) {
      Alert.alert('Error', 'Reset link expired or invalid', [
        { text: 'OK', onPress: () => navigation.replace('Login') }
      ]);
    }
  };

  const handleSubmit = async () => {
    if (!newPassword.trim()) {
      Alert.alert('Error', 'Please enter a new password');
      return;
    }

    if (newPassword !== confirmPassword) {
      Alert.alert('Error', 'Passwords do not match');
      return;
    }

    if (newPassword.length < 8) {
      Alert.alert('Error', 'Password must be at least 8 characters');
      return;
    }

    if (!/[A-Z]/.test(newPassword)) {
      Alert.alert('Error', 'Password must contain at least one uppercase letter');
      return;
    }

    if (!/[0-9]/.test(newPassword)) {
      Alert.alert('Error', 'Password must contain at least one number');
      return;
    }

    try {
      await resetPassword(token, newPassword);
      Alert.alert(
        'Success',
        'Your password has been reset successfully. All sessions have been logged out for security.',
        [{ text: 'OK', onPress: () => navigation.replace('Login') }]
      );
    } catch (error) {
      Alert.alert('Error', error.message || 'Failed to reset password');
    }
  };

  if (validating) {
    return (
      <View style={[styles.container, styles.centerContent]}>
        <ActivityIndicator size="large" color="#007AFF" />
        <Text style={styles.loadingText}>Validating reset link...</Text>
      </View>
    );
  }

  if (!email) {
    return null; // Will redirect via Alert
  }

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Reset Password</Text>
      <Text style={styles.subtitle}>
        Creating new password for: {email}
      </Text>

      <TextInput
        style={styles.input}
        placeholder="New Password"
        value={newPassword}
        onChangeText={setNewPassword}
        secureTextEntry
        autoCapitalize="none"
        editable={!loading}
      />

      <TextInput
        style={styles.input}
        placeholder="Confirm Password"
        value={confirmPassword}
        onChangeText={setConfirmPassword}
        secureTextEntry
        autoCapitalize="none"
        editable={!loading}
      />

      <Text style={styles.requirements}>
        Password must contain:
        {'\n'}• At least 8 characters
        {'\n'}• At least 1 uppercase letter
        {'\n'}• At least 1 number
      </Text>

      <TouchableOpacity
        style={[styles.button, loading && styles.buttonDisabled]}
        onPress={handleSubmit}
        disabled={loading}
      >
        <Text style={styles.buttonText}>
          {loading ? 'Resetting...' : 'Reset Password'}
        </Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 24,
    backgroundColor: '#fff',
    justifyContent: 'center',
  },
  centerContent: {
    justifyContent: 'center',
    alignItems: 'center',
  },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    marginBottom: 12,
  },
  subtitle: {
    fontSize: 16,
    color: '#666',
    marginBottom: 32,
  },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 16,
    fontSize: 16,
    marginBottom: 16,
  },
  requirements: {
    fontSize: 14,
    color: '#666',
    marginBottom: 24,
    lineHeight: 20,
  },
  button: {
    backgroundColor: '#007AFF',
    padding: 16,
    borderRadius: 8,
    alignItems: 'center',
  },
  buttonDisabled: {
    opacity: 0.6,
  },
  buttonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  loadingText: {
    marginTop: 16,
    fontSize: 16,
    color: '#666',
  },
});
```

---

#### 4. Update Login Screen

Add "Forgot Password?" link:

```typescript
// src/screens/auth/LoginScreen.tsx
// ... existing code ...

<TouchableOpacity
  style={styles.forgotPasswordButton}
  onPress={() => navigation.navigate('ForgotPassword')}
>
  <Text style={styles.forgotPasswordText}>Forgot Password?</Text>
</TouchableOpacity>

// ... existing code ...

const styles = StyleSheet.create({
  // ... existing styles ...
  forgotPasswordButton: {
    alignItems: 'center',
    marginTop: 16,
  },
  forgotPasswordText: {
    color: '#007AFF',
    fontSize: 16,
  },
});
```

---

#### 5. Custom Hooks

Create hooks for password reset logic:

```typescript
// src/hooks/auth/useForgotPassword.ts
import { useState } from 'react';
import axios from 'axios';
import { API_URL } from '../../config';

export const useForgotPassword = () => {
  const [loading, setLoading] = useState(false);

  const requestReset = async (email: string) => {
    setLoading(true);
    try {
      const response = await axios.post(`${API_URL}/api/auth/forgot-password`, {
        email: email.toLowerCase().trim(),
      });
      return response.data;
    } catch (error) {
      throw new Error(error.response?.data?.message || 'Failed to send reset email');
    } finally {
      setLoading(false);
    }
  };

  return { requestReset, loading };
};
```

```typescript
// src/hooks/auth/useResetPassword.ts
import { useState } from 'react';
import axios from 'axios';
import { API_URL } from '../../config';

export const useResetPassword = () => {
  const [loading, setLoading] = useState(false);
  const [validating, setValidating] = useState(false);

  const validateToken = async (token: string): Promise<{ email: string }> => {
    setValidating(true);
    try {
      const response = await axios.get(`${API_URL}/api/auth/validate-reset-token`, {
        params: { token },
      });
      return response.data;
    } catch (error) {
      throw new Error(error.response?.data?.message || 'Invalid reset token');
    } finally {
      setValidating(false);
    }
  };

  const resetPassword = async (token: string, newPassword: string) => {
    setLoading(true);
    try {
      const response = await axios.post(`${API_URL}/api/auth/reset-password`, {
        token,
        newPassword,
      });
      return response.data;
    } catch (error) {
      throw new Error(error.response?.data?.message || 'Failed to reset password');
    } finally {
      setLoading(false);
    }
  };

  return { validateToken, resetPassword, loading, validating };
};
```

---

#### 6. Deep Link Handling

Handle incoming reset password links:

```typescript
// App.tsx or navigation setup
import * as Linking from 'expo-linking';
import { useEffect } from 'react';

const linking = {
  prefixes: ['pensine://', 'https://pensine.app'],
  config: {
    screens: {
      ResetPassword: 'reset-password',
      // ... other screens
    },
  },
};

export default function App() {
  useEffect(() => {
    // Handle deep links when app is already open
    const subscription = Linking.addEventListener('url', handleDeepLink);

    return () => subscription.remove();
  }, []);

  const handleDeepLink = ({ url }: { url: string }) => {
    // Linking automatically navigates to ResetPassword screen with token param
  };

  return (
    <NavigationContainer linking={linking}>
      {/* ... navigation */}
    </NavigationContainer>
  );
}
```

---

#### 7. Navigation Configuration

Update navigation to include new screens:

```typescript
// src/navigation/AuthNavigator.tsx
import { ForgotPasswordScreen } from '../screens/auth/ForgotPasswordScreen';
import { ResetPasswordScreen } from '../screens/auth/ResetPasswordScreen';

const AuthStack = createStackNavigator();

export const AuthNavigator = () => {
  return (
    <AuthStack.Navigator>
      <AuthStack.Screen name="Login" component={LoginScreen} />
      <AuthStack.Screen
        name="ForgotPassword"
        component={ForgotPasswordScreen}
        options={{ title: 'Forgot Password' }}
      />
      <AuthStack.Screen
        name="ResetPassword"
        component={ResetPasswordScreen}
        options={{ title: 'Reset Password' }}
      />
      <AuthStack.Screen name="Register" component={RegisterScreen} />
    </AuthStack.Navigator>
  );
};
```

---

### Security Considerations

#### 1. Email Enumeration Prevention

**Vulnerability:** Attacker can discover registered emails by analyzing different responses.

**Mitigation:**
```typescript
// ✅ CORRECT: Same message for existing and non-existing emails
return {
  message: 'If this email exists, you will receive a password reset link shortly.',
};

// ❌ WRONG: Different messages reveal email existence
if (!user) {
  throw new NotFoundException('Email not found');
}
```

---

#### 2. Token Security

**Best Practices:**
- ✅ Generate cryptographically secure tokens: `crypto.randomBytes(32)`
- ✅ Hash tokens before storing (SHA-256): prevents database leak exploitation
- ✅ Short expiration (1 hour): limits attack window
- ✅ One-time use: invalidate immediately after password reset
- ✅ Invalidate on successful reset: `used: true`

**Token Flow:**
```
1. Generate: crypto.randomBytes(32).toString('hex')
   → "a1b2c3d4e5f6..."

2. Hash: SHA-256(token)
   → "9f86d081884c7d659a2feaa0c55ad015..."

3. Store: tokenHash in DB

4. Send: Plain token via email

5. Verify: Hash received token and compare with DB
```

---

#### 3. Session Invalidation

**Critical Requirement:** All existing sessions MUST be terminated after password reset.

**Implementation:**
```typescript
// Store password reset timestamp in Redis
const key = `password-reset:${userId}`;
await redisClient.set(key, currentTimestamp, 'EX', 30 * 24 * 60 * 60);

// In JwtAuthGuard, check if token issued before reset
const resetTimestamp = await redisClient.get(`password-reset:${userId}`);
if (resetTimestamp && tokenIssuedAt < parseInt(resetTimestamp)) {
  throw new UnauthorizedException('Session expired due to password change');
}
```

**Why this works:**
- JWT tokens contain `iat` (issued at) claim
- After password reset, we store reset timestamp
- Any token issued BEFORE reset timestamp is rejected
- No need to blacklist individual tokens

---

#### 4. Rate Limiting

**Prevent abuse:** Limit password reset requests per email/IP.

**Implementation (Post-MVP):**
```typescript
import { ThrottlerGuard, Throttle } from '@nestjs/throttler';

@Controller('api/auth')
@UseGuards(ThrottlerGuard)
export class PasswordResetController {
  @Post('forgot-password')
  @Throttle({ default: { limit: 3, ttl: 60000 } }) // 3 requests per minute
  async forgotPassword(@Body() dto: ForgotPasswordDto) {
    // ...
  }
}
```

**Install:**
```bash
npm install @nestjs/throttler
```

---

#### 5. HTTPS Enforcement

**Critical:** Reset links MUST be sent over HTTPS only.

**Backend:**
```typescript
// src/main.ts
app.use((req, res, next) => {
  if (req.header('x-forwarded-proto') !== 'https' && process.env.NODE_ENV === 'production') {
    return res.redirect(`https://${req.header('host')}${req.url}`);
  }
  next();
});
```

**Email Links:**
```typescript
// Force HTTPS in production
const protocol = process.env.NODE_ENV === 'production' ? 'https' : 'pensine';
const resetUrl = `${protocol}://pensine.app/reset-password?token=${resetToken}`;
```

---

### Testing Requirements

#### 1. Backend Unit Tests

```typescript
// src/modules/identity/application/services/__tests__/password-reset.service.spec.ts
describe('PasswordResetService', () => {
  describe('requestPasswordReset', () => {
    it('should send reset email for existing user', async () => {
      const result = await service.requestPasswordReset('user@example.com');
      expect(result.message).toContain('If this email exists');
      expect(emailService.sendPasswordResetEmail).toHaveBeenCalledWith(
        'user@example.com',
        expect.any(String),
      );
    });

    it('should return same message for non-existing email (prevent enumeration)', async () => {
      const result = await service.requestPasswordReset('nonexistent@example.com');
      expect(result.message).toContain('If this email exists');
      expect(emailService.sendPasswordResetEmail).not.toHaveBeenCalled();
    });

    it('should invalidate previous reset tokens', async () => {
      await service.requestPasswordReset('user@example.com');
      await service.requestPasswordReset('user@example.com');

      const tokens = await resetTokenRepository.find({
        where: { userId: user.id, used: false },
      });
      expect(tokens).toHaveLength(1); // Only newest token active
    });

    it('should hash token before storing', async () => {
      await service.requestPasswordReset('user@example.com');

      const token = await resetTokenRepository.findOne({
        where: { userId: user.id },
      });
      expect(token.tokenHash).toHaveLength(64); // SHA-256 hex length
    });
  });

  describe('validateResetToken', () => {
    it('should validate valid unexpired token', async () => {
      const token = await createResetToken(user);
      const result = await service.validateResetToken(token);
      expect(result.email).toBe(user.email);
    });

    it('should reject expired token', async () => {
      const expiredToken = await createExpiredResetToken(user);
      await expect(service.validateResetToken(expiredToken)).rejects.toThrow(
        'Reset link expired or invalid',
      );
    });

    it('should reject invalid token', async () => {
      await expect(service.validateResetToken('invalid-token')).rejects.toThrow(
        'Reset link expired or invalid',
      );
    });

    it('should reject used token', async () => {
      const token = await createResetToken(user);
      await service.resetPassword(token, 'NewPass123');

      await expect(service.validateResetToken(token)).rejects.toThrow(
        'Reset link expired or invalid',
      );
    });
  });

  describe('resetPassword', () => {
    it('should update password with bcrypt hash', async () => {
      const token = await createResetToken(user);
      await service.resetPassword(token, 'NewPass123');

      const updatedUser = await userRepository.findOne({ where: { id: user.id } });
      expect(updatedUser.passwordHash).not.toBe(user.passwordHash);
      expect(await bcrypt.compare('NewPass123', updatedUser.passwordHash)).toBe(true);
    });

    it('should invalidate token after use', async () => {
      const token = await createResetToken(user);
      await service.resetPassword(token, 'NewPass123');

      const resetToken = await resetTokenRepository.findOne({
        where: { userId: user.id },
      });
      expect(resetToken.used).toBe(true);
    });

    it('should invalidate all user sessions', async () => {
      const token = await createResetToken(user);
      await service.resetPassword(token, 'NewPass123');

      const resetTimestamp = await redisClient.get(`password-reset:${user.id}`);
      expect(resetTimestamp).toBeTruthy();
    });

    it('should send confirmation email', async () => {
      const token = await createResetToken(user);
      await service.resetPassword(token, 'NewPass123');

      expect(emailService.sendPasswordChangedConfirmation).toHaveBeenCalledWith(
        user.email,
      );
    });

    it('should enforce password strength requirements', async () => {
      const token = await createResetToken(user);

      await expect(service.resetPassword(token, 'weak')).rejects.toThrow(
        'Password must be at least 8 characters',
      );

      await expect(service.resetPassword(token, 'nouppercase123')).rejects.toThrow(
        'Password must contain at least one uppercase letter',
      );

      await expect(service.resetPassword(token, 'NoNumbers')).rejects.toThrow(
        'Password must contain at least one number',
      );
    });
  });

  describe('cleanupExpiredTokens', () => {
    it('should delete expired tokens', async () => {
      await createExpiredResetToken(user);
      await service.cleanupExpiredTokens();

      const tokens = await resetTokenRepository.find({
        where: { userId: user.id },
      });
      expect(tokens).toHaveLength(0);
    });

    it('should preserve unexpired tokens', async () => {
      await createResetToken(user);
      await service.cleanupExpiredTokens();

      const tokens = await resetTokenRepository.find({
        where: { userId: user.id },
      });
      expect(tokens).toHaveLength(1);
    });
  });
});
```

---

#### 2. Integration Tests

```typescript
// test/password-reset.e2e-spec.ts
describe('Password Reset (e2e)', () => {
  it('should complete full password reset flow', async () => {
    // 1. Request reset
    const response1 = await request(app.getHttpServer())
      .post('/api/auth/forgot-password')
      .send({ email: 'user@example.com' })
      .expect(201);

    expect(response1.body.message).toContain('If this email exists');

    // Extract token from email (mock)
    const token = emailService.lastSentToken;

    // 2. Validate token
    const response2 = await request(app.getHttpServer())
      .get('/api/auth/validate-reset-token')
      .query({ token })
      .expect(200);

    expect(response2.body.email).toBe('user@example.com');

    // 3. Reset password
    const response3 = await request(app.getHttpServer())
      .post('/api/auth/reset-password')
      .send({ token, newPassword: 'NewPass123' })
      .expect(201);

    expect(response3.body.message).toContain('successfully reset');

    // 4. Login with new password
    const response4 = await request(app.getHttpServer())
      .post('/api/auth/login')
      .send({ email: 'user@example.com', password: 'NewPass123' })
      .expect(200);

    expect(response4.body.accessToken).toBeDefined();
  });

  it('should invalidate old sessions after password reset', async () => {
    // 1. Login and get token
    const loginResponse = await request(app.getHttpServer())
      .post('/api/auth/login')
      .send({ email: 'user@example.com', password: 'OldPass123' })
      .expect(200);

    const oldToken = loginResponse.body.accessToken;

    // 2. Reset password
    const resetToken = await createResetToken(user);
    await request(app.getHttpServer())
      .post('/api/auth/reset-password')
      .send({ token: resetToken, newPassword: 'NewPass123' })
      .expect(201);

    // 3. Try to use old token (should fail)
    await request(app.getHttpServer())
      .get('/api/profile')
      .set('Authorization', `Bearer ${oldToken}`)
      .expect(401);
  });
});
```

---

#### 3. Mobile Unit Tests

```typescript
// src/hooks/auth/__tests__/useForgotPassword.test.ts
describe('useForgotPassword', () => {
  it('should send reset request', async () => {
    const { result } = renderHook(() => useForgotPassword());

    await act(async () => {
      await result.current.requestReset('user@example.com');
    });

    expect(axios.post).toHaveBeenCalledWith(
      expect.stringContaining('/api/auth/forgot-password'),
      { email: 'user@example.com' },
    );
  });

  it('should handle errors', async () => {
    axios.post.mockRejectedValue({
      response: { data: { message: 'Network error' } },
    });

    const { result } = renderHook(() => useForgotPassword());

    await expect(
      result.current.requestReset('user@example.com')
    ).rejects.toThrow('Network error');
  });
});
```

---

#### 4. Mobile E2E Tests (Detox)

```typescript
// e2e/password-reset.e2e.ts
describe('Password Reset Flow', () => {
  it('should complete password reset from login screen', async () => {
    // 1. Navigate to Forgot Password
    await element(by.id('loginScreen')).tap();
    await element(by.text('Forgot Password?')).tap();

    // 2. Enter email
    await element(by.id('emailInput')).typeText('user@example.com');
    await element(by.text('Send Reset Link')).tap();

    // 3. Verify success message
    await expect(element(by.text('Check your email'))).toBeVisible();

    // 4. Simulate deep link (token from email)
    await device.openURL({ url: 'pensine://reset-password?token=abc123' });

    // 5. Enter new password
    await element(by.id('newPasswordInput')).typeText('NewPass123');
    await element(by.id('confirmPasswordInput')).typeText('NewPass123');
    await element(by.text('Reset Password')).tap();

    // 6. Verify success and redirect to login
    await expect(element(by.text('Success'))).toBeVisible();
    await element(by.text('OK')).tap();
    await expect(element(by.id('loginScreen'))).toBeVisible();
  });

  it('should show error for expired token', async () => {
    await device.openURL({ url: 'pensine://reset-password?token=expired-token' });

    await expect(element(by.text('Reset link expired or invalid'))).toBeVisible();
  });
});
```

---

### Definition of Done

- [ ] **Backend:**
  - [ ] `password_reset_tokens` table created with migration
  - [ ] `EmailService` configured with Nodemailer + SMTP
  - [ ] `PasswordResetService` with all methods implemented
  - [ ] `TokenBlacklistService` extended for password reset invalidation
  - [ ] `JwtAuthGuard` checks token issued before password reset
  - [ ] POST `/api/auth/forgot-password` endpoint working
  - [ ] GET `/api/auth/validate-reset-token` endpoint working
  - [ ] POST `/api/auth/reset-password` endpoint working
  - [ ] Cron job for expired token cleanup scheduled
  - [ ] All backend unit tests passing (100% coverage)
  - [ ] Integration tests passing

- [ ] **Mobile:**
  - [ ] Deep linking configured (app.json)
  - [ ] `ForgotPasswordScreen` implemented
  - [ ] `ResetPasswordScreen` implemented
  - [ ] `useForgotPassword` hook implemented
  - [ ] `useResetPassword` hook implemented
  - [ ] "Forgot Password?" link added to LoginScreen
  - [ ] Navigation updated with new screens
  - [ ] Deep link handling working
  - [ ] All mobile unit tests passing
  - [ ] E2E tests passing

- [ ] **Security:**
  - [ ] Email enumeration prevented (same message always)
  - [ ] Tokens hashed before storage (SHA-256)
  - [ ] Token expiration enforced (1 hour)
  - [ ] One-time use enforced (used flag)
  - [ ] All sessions invalidated after reset
  - [ ] Password strength validation working
  - [ ] HTTPS enforced for reset links (production)

- [ ] **Documentation:**
  - [ ] Environment variables documented (.env.example)
  - [ ] SMTP configuration guide (Gmail setup)
  - [ ] Deep linking setup instructions
  - [ ] Security considerations documented

---

## Common Pitfalls

### ❌ DO NOT:

1. **Reveal email existence:**
```typescript
// ❌ WRONG: Different messages
if (!user) {
  throw new NotFoundException('Email not found');
}
return { message: 'Reset email sent' };
```

2. **Store tokens in plain text:**
```typescript
// ❌ WRONG: Plain token in DB
await resetTokenRepository.save({
  userId: user.id,
  token: resetToken, // Vulnerable to DB leak
});
```

3. **Forget to invalidate old sessions:**
```typescript
// ❌ WRONG: Password changed but old tokens still work
await userRepository.update(user.id, { passwordHash });
// Missing: invalidateAllUserSessions(user.id)
```

4. **Use long expiration times:**
```typescript
// ❌ WRONG: 24 hours is too long
expiresAt.setHours(expiresAt.getHours() + 24);

// ✅ CORRECT: 1 hour max
expiresAt.setHours(expiresAt.getHours() + 1);
```

5. **Allow token reuse:**
```typescript
// ❌ WRONG: Token can be used multiple times
await userRepository.update(user.id, { passwordHash });
// Missing: mark token as used
```

6. **Use weak token generation:**
```typescript
// ❌ WRONG: Predictable tokens
const token = Math.random().toString(36);

// ✅ CORRECT: Cryptographically secure
const token = crypto.randomBytes(32).toString('hex');
```

---

### ✅ DO:

1. **Always return same message:**
```typescript
return {
  message: 'If this email exists, you will receive a password reset link shortly.',
};
```

2. **Hash tokens before storage:**
```typescript
const tokenHash = crypto.createHash('sha256').update(resetToken).digest('hex');
await resetTokenRepository.save({ userId, tokenHash, expiresAt });
```

3. **Invalidate all sessions:**
```typescript
await this.invalidateAllUserSessions(user.id);
await this.emailService.sendPasswordChangedConfirmation(user.email);
```

4. **Use short expiration:**
```typescript
const expiresAt = new Date();
expiresAt.setHours(expiresAt.getHours() + 1); // 1 hour only
```

5. **Enforce one-time use:**
```typescript
await resetTokenRepository.update(resetToken.id, { used: true });
```

6. **Use crypto.randomBytes:**
```typescript
const resetToken = crypto.randomBytes(32).toString('hex'); // 256 bits
```

---

## Non-Functional Requirements

**Security (NFR10-NFR14):**
- ✅ Tokens hashed with SHA-256 before storage
- ✅ HTTPS enforcement for reset links (production)
- ✅ Email enumeration prevention
- ✅ Session invalidation on password change
- ✅ Rate limiting on forgot-password endpoint (Post-MVP)

**Reliability (NFR6-NFR9):**
- ✅ Email delivery confirmation (Nodemailer)
- ✅ Graceful error handling (invalid tokens, expired tokens)
- ✅ Atomic operations (token creation, password update)

**Performance (NFR1-NFR5):**
- ✅ Token validation < 100ms (Redis lookup)
- ✅ Password hash generation optimized (bcrypt saltRounds=12)
- ✅ Email sending async (non-blocking)

---

## Additional Resources

**Email Templates:**
- Customizable HTML templates in EmailService
- Support for both HTML and plain text

**SMTP Providers:**
- Gmail (development): Free, requires App Password
- SendGrid (production): Reliable, scalable
- AWS SES (production): Cost-effective, high deliverability

**Deep Linking:**
- Expo Linking: https://docs.expo.dev/guides/linking/
- Universal Links (iOS): https://developer.apple.com/ios/universal-links/
- App Links (Android): https://developer.android.com/training/app-links

**Security References:**
- OWASP Password Reset Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- NIST Password Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html

---

**Story Ready for Development** ✅

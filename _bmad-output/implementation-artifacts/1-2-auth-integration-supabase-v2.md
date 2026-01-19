# Story 1.2: Authentication Integration (Supabase Cloud)

**Story ID:** 1.2
**Epic:** Epic 1 - Foundation & Authentification
**Status:** ready-for-dev
**Story Key:** `1-2-auth-integration-supabase`
**Version:** 2.0 (Updated per ADR-016: Hybrid Architecture)

---

## 📋 User Story

**As a** user
**I want** to authenticate using email/password, Google, or Apple
**So that** I can securely access my personal Pensine account

---

## ✅ Acceptance Criteria

### AC1: Email/Password Authentication

**Given** I am a new user
**When** I register with email and password
**Then:**
- ✅ Account created in Supabase
- ✅ Confirmation email sent automatically
- ✅ JWT token received and stored locally
- ✅ Redirected to main app screen

**And When** I login with email and password
**Then:**
- ✅ JWT token received
- ✅ Session persisted (AsyncStorage)
- ✅ User info accessible in app

### AC2: Google Sign-In

**Given** I want to use Google authentication
**When** I tap "Continue with Google"
**Then:**
- ✅ Google OAuth consent screen opens
- ✅ After consent, redirected back to app (`pensine://auth/callback`)
- ✅ JWT token received and stored
- ✅ User account created/linked automatically
- ✅ Email and name populated from Google profile

### AC3: Apple Sign-In

**Given** I want to use Apple authentication
**When** I tap "Sign in with Apple"
**Then:**
- ✅ Apple authentication sheet opens
- ✅ After Face ID/Touch ID, redirected back to app
- ✅ JWT token received and stored
- ✅ User account created/linked automatically
- ✅ Email handled correctly (including "Hide My Email")

### AC4: Logout

**Given** I am logged in
**When** I tap "Logout"
**Then:**
- ✅ JWT token cleared from storage
- ✅ WatermelonDB User record deleted (local only)
- ✅ Unsynced data warning shown if applicable
- ✅ Redirected to login screen
- ✅ Session terminated (Supabase)

### AC5: Password Recovery

**Given** I forgot my password
**When** I request password reset
**Then:**
- ✅ Password reset email sent (Supabase managed)
- ✅ Email contains magic link
- ✅ Magic link opens app via deep link
- ✅ Password update screen shown
- ✅ New password validated (min 8 chars, 1 uppercase, 1 number)
- ✅ Password updated successfully
- ✅ Automatically logged in with new password

### AC6: Session Persistence

**Given** I close and reopen the app
**When** I have a valid session
**Then:**
- ✅ Automatically logged in (no re-auth required)
- ✅ JWT token refreshed automatically if expired
- ✅ User info loaded from Supabase

---

## 🎯 Implementation Details

### Part 1: Mobile Screens (React Native)

#### Screen 1.1: Login Screen

```typescript
// src/contexts/identity/screens/LoginScreen.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Alert,
  Platform,
} from 'react-native';
import { supabase } from '../../../lib/supabase';
import * as AppleAuthentication from 'expo-apple-authentication';
import * as WebBrowser from 'expo-web-browser';
import { makeRedirectUri } from 'expo-auth-session';

WebBrowser.maybeCompleteAuthSession();

export const LoginScreen = ({ navigation }) => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);

  // Email/Password Login
  const handleEmailLogin = async () => {
    if (!email || !password) {
      Alert.alert('Error', 'Please fill in all fields');
      return;
    }

    setLoading(true);
    try {
      const { data, error } = await supabase.auth.signInWithPassword({
        email: email.toLowerCase().trim(),
        password,
      });

      if (error) throw error;

      // Navigation handled by auth state listener
    } catch (error) {
      Alert.alert('Login Failed', error.message);
    } finally {
      setLoading(false);
    }
  };

  // Google Sign-In
  const handleGoogleSignIn = async () => {
    setLoading(true);
    try {
      const redirectUrl = makeRedirectUri({ path: 'auth/callback' });

      const { data, error } = await supabase.auth.signInWithOAuth({
        provider: 'google',
        options: {
          redirectTo: redirectUrl,
          skipBrowserRedirect: true,
        },
      });

      if (error) throw error;

      // Open browser for OAuth
      const result = await WebBrowser.openAuthSessionAsync(
        data.url,
        redirectUrl,
      );

      if (result.type === 'success') {
        const { url } = result;
        // Extract tokens from URL
        const params = new URLSearchParams(url.split('#')[1]);
        const accessToken = params.get('access_token');
        const refreshToken = params.get('refresh_token');

        if (accessToken && refreshToken) {
          // Set session
          await supabase.auth.setSession({
            access_token: accessToken,
            refresh_token: refreshToken,
          });
        }
      }
    } catch (error) {
      Alert.alert('Google Sign-In Failed', error.message);
    } finally {
      setLoading(false);
    }
  };

  // Apple Sign-In
  const handleAppleSignIn = async () => {
    if (Platform.OS !== 'ios') {
      Alert.alert('Error', 'Apple Sign-In only available on iOS');
      return;
    }

    setLoading(true);
    try {
      const credential = await AppleAuthentication.signInAsync({
        requestedScopes: [
          AppleAuthentication.AppleAuthenticationScope.FULL_NAME,
          AppleAuthentication.AppleAuthenticationScope.EMAIL,
        ],
      });

      const { identityToken } = credential;

      if (!identityToken) {
        throw new Error('No identity token received');
      }

      // Exchange Apple token for Supabase session
      const { data, error } = await supabase.auth.signInWithIdToken({
        provider: 'apple',
        token: identityToken,
      });

      if (error) throw error;

      // Navigation handled by auth state listener
    } catch (error) {
      if (error.code === 'ERR_CANCELED') {
        // User cancelled, do nothing
      } else {
        Alert.alert('Apple Sign-In Failed', error.message);
      }
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Welcome to Pensine</Text>
      <Text style={styles.subtitle}>
        Capture your thoughts, incubate your ideas
      </Text>

      {/* Email Input */}
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

      {/* Password Input */}
      <TextInput
        style={styles.input}
        placeholder="Password"
        value={password}
        onChangeText={setPassword}
        secureTextEntry
        autoCapitalize="none"
        editable={!loading}
      />

      {/* Login Button */}
      <TouchableOpacity
        style={[styles.button, styles.primaryButton, loading && styles.buttonDisabled]}
        onPress={handleEmailLogin}
        disabled={loading}
      >
        <Text style={styles.buttonText}>
          {loading ? 'Signing in...' : 'Sign In'}
        </Text>
      </TouchableOpacity>

      {/* Forgot Password */}
      <TouchableOpacity
        style={styles.forgotPasswordButton}
        onPress={() => navigation.navigate('ForgotPassword')}
      >
        <Text style={styles.forgotPasswordText}>Forgot Password?</Text>
      </TouchableOpacity>

      {/* Divider */}
      <View style={styles.divider}>
        <View style={styles.dividerLine} />
        <Text style={styles.dividerText}>OR</Text>
        <View style={styles.dividerLine} />
      </View>

      {/* Google Sign-In */}
      <TouchableOpacity
        style={[styles.button, styles.googleButton]}
        onPress={handleGoogleSignIn}
        disabled={loading}
      >
        <Text style={styles.googleButtonText}>Continue with Google</Text>
      </TouchableOpacity>

      {/* Apple Sign-In (iOS only) */}
      {Platform.OS === 'ios' && (
        <TouchableOpacity
          style={[styles.button, styles.appleButton]}
          onPress={handleAppleSignIn}
          disabled={loading}
        >
          <Text style={styles.appleButtonText}>Sign in with Apple</Text>
        </TouchableOpacity>
      )}

      {/* Register Link */}
      <View style={styles.registerContainer}>
        <Text style={styles.registerText}>Don't have an account? </Text>
        <TouchableOpacity onPress={() => navigation.navigate('Register')}>
          <Text style={styles.registerLink}>Sign Up</Text>
        </TouchableOpacity>
      </View>
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
    fontSize: 32,
    fontWeight: 'bold',
    marginBottom: 8,
    textAlign: 'center',
  },
  subtitle: {
    fontSize: 16,
    color: '#666',
    marginBottom: 32,
    textAlign: 'center',
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
    padding: 16,
    borderRadius: 8,
    alignItems: 'center',
    marginBottom: 12,
  },
  primaryButton: {
    backgroundColor: '#007AFF',
  },
  googleButton: {
    backgroundColor: '#fff',
    borderWidth: 1,
    borderColor: '#ddd',
  },
  appleButton: {
    backgroundColor: '#000',
  },
  buttonDisabled: {
    opacity: 0.6,
  },
  buttonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  googleButtonText: {
    color: '#000',
    fontSize: 16,
    fontWeight: '600',
  },
  appleButtonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  forgotPasswordButton: {
    alignItems: 'center',
    marginBottom: 24,
  },
  forgotPasswordText: {
    color: '#007AFF',
    fontSize: 14,
  },
  divider: {
    flexDirection: 'row',
    alignItems: 'center',
    marginVertical: 24,
  },
  dividerLine: {
    flex: 1,
    height: 1,
    backgroundColor: '#ddd',
  },
  dividerText: {
    marginHorizontal: 16,
    color: '#999',
    fontSize: 14,
  },
  registerContainer: {
    flexDirection: 'row',
    justifyContent: 'center',
    marginTop: 24,
  },
  registerText: {
    color: '#666',
    fontSize: 14,
  },
  registerLink: {
    color: '#007AFF',
    fontSize: 14,
    fontWeight: '600',
  },
});
```

#### Screen 1.2: Register Screen

```typescript
// src/contexts/identity/screens/RegisterScreen.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Alert,
} from 'react-native';
import { supabase } from '../../../lib/supabase';

export const RegisterScreen = ({ navigation }) => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [confirmPassword, setConfirmPassword] = useState('');
  const [loading, setLoading] = useState(false);

  const validatePassword = (pwd: string): string | null => {
    if (pwd.length < 8) {
      return 'Password must be at least 8 characters';
    }
    if (!/[A-Z]/.test(pwd)) {
      return 'Password must contain at least one uppercase letter';
    }
    if (!/[0-9]/.test(pwd)) {
      return 'Password must contain at least one number';
    }
    return null;
  };

  const handleRegister = async () => {
    // Validation
    if (!email || !password || !confirmPassword) {
      Alert.alert('Error', 'Please fill in all fields');
      return;
    }

    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(email)) {
      Alert.alert('Error', 'Please enter a valid email address');
      return;
    }

    const passwordError = validatePassword(password);
    if (passwordError) {
      Alert.alert('Error', passwordError);
      return;
    }

    if (password !== confirmPassword) {
      Alert.alert('Error', 'Passwords do not match');
      return;
    }

    setLoading(true);
    try {
      const { data, error } = await supabase.auth.signUp({
        email: email.toLowerCase().trim(),
        password,
      });

      if (error) throw error;

      Alert.alert(
        'Success',
        'Account created! Please check your email to confirm your account.',
        [{ text: 'OK', onPress: () => navigation.navigate('Login') }],
      );
    } catch (error) {
      Alert.alert('Registration Failed', error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Create Account</Text>

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

      <TextInput
        style={styles.input}
        placeholder="Password"
        value={password}
        onChangeText={setPassword}
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
        onPress={handleRegister}
        disabled={loading}
      >
        <Text style={styles.buttonText}>
          {loading ? 'Creating Account...' : 'Sign Up'}
        </Text>
      </TouchableOpacity>

      <View style={styles.loginContainer}>
        <Text style={styles.loginText}>Already have an account? </Text>
        <TouchableOpacity onPress={() => navigation.navigate('Login')}>
          <Text style={styles.loginLink}>Sign In</Text>
        </TouchableOpacity>
      </View>
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
    fontSize: 32,
    fontWeight: 'bold',
    marginBottom: 32,
    textAlign: 'center',
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
  loginContainer: {
    flexDirection: 'row',
    justifyContent: 'center',
    marginTop: 24,
  },
  loginText: {
    color: '#666',
    fontSize: 14,
  },
  loginLink: {
    color: '#007AFF',
    fontSize: 14,
    fontWeight: '600',
  },
});
```

#### Screen 1.3: Forgot Password Screen

```typescript
// src/contexts/identity/screens/ForgotPasswordScreen.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Alert,
} from 'react-native';
import { supabase } from '../../../lib/supabase';

export const ForgotPasswordScreen = ({ navigation }) => {
  const [email, setEmail] = useState('');
  const [loading, setLoading] = useState(false);

  const handleResetPassword = async () => {
    if (!email.trim()) {
      Alert.alert('Error', 'Please enter your email address');
      return;
    }

    setLoading(true);
    try {
      const { error } = await supabase.auth.resetPasswordForEmail(
        email.toLowerCase().trim(),
        {
          redirectTo: 'pensine://reset-password',
        },
      );

      if (error) throw error;

      Alert.alert(
        'Check Your Email',
        'Password reset instructions have been sent to your email address.',
        [{ text: 'OK', onPress: () => navigation.goBack() }],
      );
    } catch (error) {
      Alert.alert('Error', error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Forgot Password</Text>
      <Text style={styles.subtitle}>
        Enter your email address and we'll send you a link to reset your
        password.
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
        onPress={handleResetPassword}
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

#### Screen 1.4: Reset Password Screen

```typescript
// src/contexts/identity/screens/ResetPasswordScreen.tsx
import React, { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  Alert,
} from 'react-native';
import { supabase } from '../../../lib/supabase';
import { useRoute } from '@react-navigation/native';

export const ResetPasswordScreen = ({ navigation }) => {
  const route = useRoute();
  const { accessToken } = route.params || {};

  const [newPassword, setNewPassword] = useState('');
  const [confirmPassword, setConfirmPassword] = useState('');
  const [loading, setLoading] = useState(false);

  const validatePassword = (pwd: string): string | null => {
    if (pwd.length < 8) return 'Password must be at least 8 characters';
    if (!/[A-Z]/.test(pwd)) return 'Password must contain at least one uppercase letter';
    if (!/[0-9]/.test(pwd)) return 'Password must contain at least one number';
    return null;
  };

  const handleResetPassword = async () => {
    if (!newPassword || !confirmPassword) {
      Alert.alert('Error', 'Please fill in all fields');
      return;
    }

    const passwordError = validatePassword(newPassword);
    if (passwordError) {
      Alert.alert('Error', passwordError);
      return;
    }

    if (newPassword !== confirmPassword) {
      Alert.alert('Error', 'Passwords do not match');
      return;
    }

    setLoading(true);
    try {
      const { error } = await supabase.auth.updateUser({
        password: newPassword,
      });

      if (error) throw error;

      Alert.alert(
        'Success',
        'Your password has been reset successfully.',
        [{ text: 'OK', onPress: () => navigation.replace('Login') }],
      );
    } catch (error) {
      Alert.alert('Error', error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Reset Password</Text>

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
        onPress={handleResetPassword}
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
  title: {
    fontSize: 28,
    fontWeight: 'bold',
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
});
```

#### Navigation Configuration

```typescript
// src/navigation/AuthNavigator.tsx
import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { LoginScreen } from '../contexts/identity/screens/LoginScreen';
import { RegisterScreen } from '../contexts/identity/screens/RegisterScreen';
import { ForgotPasswordScreen } from '../contexts/identity/screens/ForgotPasswordScreen';
import { ResetPasswordScreen } from '../contexts/identity/screens/ResetPasswordScreen';

const Stack = createNativeStackNavigator();

export const AuthNavigator = () => {
  return (
    <Stack.Navigator
      screenOptions={{
        headerShown: false,
      }}
    >
      <Stack.Screen name="Login" component={LoginScreen} />
      <Stack.Screen name="Register" component={RegisterScreen} />
      <Stack.Screen name="ForgotPassword" component={ForgotPasswordScreen} />
      <Stack.Screen name="ResetPassword" component={ResetPasswordScreen} />
    </Stack.Navigator>
  );
};
```

#### Auth State Listener

```typescript
// src/contexts/identity/hooks/useAuthListener.ts
import { useEffect, useState } from 'react';
import { supabase } from '../../../lib/supabase';
import type { Session, User } from '@supabase/supabase-js';

export const useAuthListener = () => {
  const [session, setSession] = useState<Session | null>(null);
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Get initial session
    supabase.auth.getSession().then(({ data: { session } }) => {
      setSession(session);
      setUser(session?.user ?? null);
      setLoading(false);
    });

    // Listen for auth changes
    const {
      data: { subscription },
    } = supabase.auth.onAuthStateChange((_event, session) => {
      setSession(session);
      setUser(session?.user ?? null);
      setLoading(false);
    });

    return () => subscription.unsubscribe();
  }, []);

  return { session, user, loading };
};
```

#### Root Navigator with Auth Check

```typescript
// App.tsx
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { useAuthListener } from './src/contexts/identity/hooks/useAuthListener';
import { AuthNavigator } from './src/navigation/AuthNavigator';
import { MainNavigator } from './src/navigation/MainNavigator';
import { ActivityIndicator, View } from 'react-native';

export default function App() {
  const { user, loading } = useAuthListener();

  if (loading) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" />
      </View>
    );
  }

  return (
    <NavigationContainer>
      {user ? <MainNavigator /> : <AuthNavigator />}
    </NavigationContainer>
  );
}
```

---

### Part 2: Backend Integration

#### Health Check Endpoint (Testing)

```typescript
// src/modules/identity/infrastructure/controllers/auth.controller.ts
import { Controller, Get, UseGuards, Request } from '@nestjs/common';
import { SupabaseAuthGuard } from '../../../shared/infrastructure/guards/supabase-auth.guard';

@Controller('api/auth')
export class AuthController {
  @Get('me')
  @UseGuards(SupabaseAuthGuard)
  async getCurrentUser(@Request() req) {
    return {
      userId: req.user.id,
      email: req.user.email,
      createdAt: req.user.created_at,
    };
  }

  @Get('health')
  async health() {
    return { status: 'ok', auth: 'supabase' };
  }
}
```

---

## 📊 Definition of Done

### Mobile

- [ ] **Screens Implemented:**
  - [ ] LoginScreen (email/password + social)
  - [ ] RegisterScreen
  - [ ] ForgotPasswordScreen
  - [ ] ResetPasswordScreen

- [ ] **Authentication Flows Working:**
  - [ ] Email/Password registration
  - [ ] Email/Password login
  - [ ] Google Sign-In (iOS + Android)
  - [ ] Apple Sign-In (iOS)
  - [ ] Password reset via email
  - [ ] Logout

- [ ] **Session Management:**
  - [ ] JWT token stored in AsyncStorage
  - [ ] Auto-refresh token when expired
  - [ ] Auth state listener working
  - [ ] Navigation based on auth state

- [ ] **Deep Linking:**
  - [ ] OAuth callbacks handled (`pensine://auth/callback`)
  - [ ] Password reset links handled (`pensine://reset-password`)

### Backend

- [ ] **Auth Guard:**
  - [ ] SupabaseAuthGuard implemented
  - [ ] JWT validation working locally
  - [ ] User info extracted from token
  - [ ] Endpoint `/api/auth/me` returns user info

### Testing

- [ ] **Manual Testing:**
  - [ ] Register new account (email confirmation received)
  - [ ] Login with email/password
  - [ ] Login with Google
  - [ ] Login with Apple (iOS)
  - [ ] Forgot password (email received)
  - [ ] Reset password via magic link
  - [ ] Logout and re-login
  - [ ] Close/reopen app (session persisted)

---

## 🚨 Common Pitfalls

### ❌ DO NOT:

1. **Implement custom password hashing:**
   - Supabase handles all password operations

2. **Store plain passwords:**
   - Never log or store passwords locally

3. **Hardcode Supabase keys:**
   - Use environment variables

4. **Forget email confirmation:**
   - Supabase requires email confirmation by default

5. **Ignore token refresh:**
   - Supabase client handles this automatically

### ✅ DO:

1. **Handle OAuth redirect correctly:**
   - Parse URL parameters from deep link
   - Extract `access_token` and `refresh_token`

2. **Validate passwords client-side:**
   - Min 8 chars, 1 uppercase, 1 number
   - Prevents unnecessary API calls

3. **Show loading states:**
   - Disable buttons during auth operations

4. **Handle errors gracefully:**
   - Display user-friendly error messages

5. **Test on both platforms:**
   - iOS and Android have different OAuth flows

---

## 📚 References

- **ADR-016:** Hybrid Architecture
- **Supabase Auth Docs:** https://supabase.com/docs/guides/auth
- **Expo Auth Session:** https://docs.expo.dev/versions/latest/sdk/auth-session/
- **Apple Authentication:** https://docs.expo.dev/versions/latest/sdk/apple-authentication/

---

## ⏱️ Estimated Time

- **Mobile Screens:** 4-6 hours
- **Navigation Setup:** 1 hour
- **Deep Linking Config:** 1 hour
- **Backend Auth Guard:** 1 hour
- **Testing:** 2 hours

**Total:** ~9-11 hours (1-2 days)

---

**Story Ready for Development** ✅

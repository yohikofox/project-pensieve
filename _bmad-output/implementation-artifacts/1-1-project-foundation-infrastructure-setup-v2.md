# Story 1.1: Project Foundation & Infrastructure Setup

**Story ID:** 1.1
**Epic:** Epic 1 - Foundation & Authentification
**Status:** ready-for-dev
**Story Key:** `1-1-project-foundation-infrastructure-setup`
**Version:** 2.0 (Updated per ADR-016: Hybrid Architecture)

---

## 📋 User Story

**As a** developer
**I want** the foundational project structure with hybrid architecture (Supabase Cloud auth + Homelab storage)
**So that** I can build Pensine MVP with optimal time-to-market and zero infrastructure cost

---

## ✅ Acceptance Criteria

### AC1: Mobile Project Initialized

**Given** I am setting up the Pensine mobile app
**When** I initialize the React Native project
**Then:**
- ✅ Expo custom dev client created with TypeScript strict mode
- ✅ Supabase client configured (`@supabase/supabase-js`)
- ✅ WatermelonDB configured for offline-first storage
- ✅ React Navigation configured (Stack + Bottom Tabs)
- ✅ Project structure follows DDD bounded contexts layout
- ✅ Deep linking configured for OAuth callbacks (`pensine://`)

### AC2: Backend Project Initialized

**Given** I am setting up the Pensine backend
**When** I initialize the NestJS project
**Then:**
- ✅ NestJS project created with TypeScript strict mode
- ✅ Supabase Auth Guard configured (JWT validation)
- ✅ PostgreSQL connection configured (app database)
- ✅ RabbitMQ message broker connected
- ✅ MinIO client configured (S3-compatible storage)
- ✅ DDD folder structure with bounded contexts

### AC3: Homelab Infrastructure Running

**Given** I have a homelab with Docker
**When** I deploy the infrastructure stack
**Then:**
- ✅ PostgreSQL container running (app database only)
- ✅ RabbitMQ container running (message queue)
- ✅ MinIO container running (audio storage)
- ✅ Cloudflare Tunnel configured (api.pensine.app, storage.pensine.app)
- ✅ All services accessible from Internet via tunnel
- ✅ Health checks passing for all services

### AC4: Supabase Cloud Configured

**Given** I need authentication services
**When** I create Supabase Cloud project
**Then:**
- ✅ Supabase project created (Free tier)
- ✅ Email/Password provider enabled
- ✅ Google OAuth provider configured
- ✅ Apple Sign-In provider configured
- ✅ JWT secret obtained and documented
- ✅ SMTP configured for password reset emails

---

## 🎯 Architecture Overview (ADR-016)

### Hybrid Stack Decision

**Decision (per ADR-016):**
- **Auth:** Supabase Cloud (managed) - €0/mois
- **Storage:** MinIO Homelab (self-hosted) - €0/mois
- **Backend/DB/Queue:** Homelab (self-hosted) - €0/mois

**Rationale:**
- Social login (Google/Apple) trivial avec Supabase (20 min vs 10-15 jours custom)
- Storage illimité homelab (vs $750/an Supabase Cloud storage)
- Time-to-market optimal (1 jour setup vs 2 semaines full self-hosted)

### Architecture Diagram

```
┌──────────────────────────────────────────────────────┐
│              MOBILE APP                              │
│          (React Native + Expo)                       │
│                                                      │
│  Auth: @supabase/supabase-js                        │
│  Storage: Direct presigned URLs (MinIO)             │
│  Offline: WatermelonDB                              │
└──────────────────────────────────────────────────────┘
               │              │              │
               │              │              │
         ┌─────┴───┐    ┌─────┴───┐    ┌────┴────┐
         │  Auth   │    │   API   │    │ Storage │
         │ (Cloud) │    │(Homelab)│    │(Homelab)│
         └─────────┘    └─────────┘    └─────────┘
               │              │              │
               ▼              ▼              ▼
    ┌─────────────────┐  ┌──────────────────────────┐
    │ Supabase Cloud  │  │       HOMELAB            │
    │                 │  │                          │
    │ ✅ GoTrue Auth  │  │ ✅ NestJS Backend        │
    │ ✅ Google OAuth │  │ ✅ PostgreSQL            │
    │ ✅ Apple OAuth  │  │ ✅ MinIO (Storage)       │
    │ ✅ SMTP Emails  │  │ ✅ RabbitMQ              │
    │                 │  │ ✅ Cloudflare Tunnel     │
    │ FREE tier       │  │ €0/mois                  │
    │ 50K MAU         │  │                          │
    └─────────────────┘  └──────────────────────────┘
```

---

## 🛠️ Implementation Details

### Part 1: Supabase Cloud Setup (30 minutes)

#### Step 1.1: Create Supabase Project

```bash
# 1. Go to https://supabase.com/dashboard
# 2. Click "New Project"
# 3. Fill:
#    - Name: pensine
#    - Database Password: [generate strong password]
#    - Region: Europe (Frankfurt) - RGPD compliance
# 4. Wait ~2 minutes for provisioning
```

#### Step 1.2: Configure Authentication Providers

**Enable Email/Password:**
```
Dashboard → Authentication → Providers → Email
- Enable Email provider: ✅
- Confirm email: ✅ (enabled)
- Email templates: Customize later
```

**Configure Google OAuth:**
```
Dashboard → Authentication → Providers → Google

1. Get credentials from Google Cloud Console:
   https://console.cloud.google.com/

   - Create project "Pensine"
   - Enable Google+ API
   - Create OAuth 2.0 credentials:
     - Application type: Web application
     - Authorized redirect URIs:
       https://xxxxx.supabase.co/auth/v1/callback

2. Copy Client ID & Client Secret to Supabase

3. Enable Google provider: ✅
```

**Configure Apple Sign-In:**
```
Dashboard → Authentication → Providers → Apple

1. Get credentials from Apple Developer Portal:
   https://developer.apple.com/

   - Create App ID with "Sign in with Apple" capability
   - Create Service ID
   - Create Private Key (.p8 file)
   - Configure Return URLs:
     https://xxxxx.supabase.co/auth/v1/callback

2. Copy Service ID, Team ID, Key ID, Private Key to Supabase

3. Enable Apple provider: ✅
```

#### Step 1.3: Get Configuration Values

```bash
# Dashboard → Settings → API

# Copy these values (needed for .env):
SUPABASE_URL=https://xxxxx.supabase.co
SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
SUPABASE_SERVICE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9... (keep secret!)

# JWT Secret (for backend validation)
# Dashboard → Settings → API → JWT Settings → JWT Secret
JWT_SECRET=your-super-secret-jwt-secret-32-chars-min
```

---

### Part 2: Homelab Infrastructure Setup (4-6 hours)

#### Step 2.1: Create Docker Compose Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ==========================================
  # DATABASE: PostgreSQL (App DB only)
  # ==========================================
  postgres:
    image: postgres:15-alpine
    container_name: pensine-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: pensine
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: pensine
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - pensine-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pensine"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ==========================================
  # QUEUE: RabbitMQ
  # ==========================================
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: pensine-queue
    restart: unless-stopped
    environment:
      RABBITMQ_DEFAULT_USER: pensine
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    networks:
      - pensine-network
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ==========================================
  # STORAGE: MinIO (S3-compatible)
  # ==========================================
  minio:
    image: minio/minio:latest
    container_name: pensine-storage
    restart: unless-stopped
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    volumes:
      - minio-data:/data
      # OU montage NAS:
      # - /path/to/nas/pensine-audios:/data
    ports:
      - "9000:9000"  # API
      - "9001:9001"  # Web Console
    networks:
      - pensine-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # ==========================================
  # BACKEND: NestJS (placeholder, built later)
  # ==========================================
  # backend:
  #   build: ./backend
  #   container_name: pensine-api
  #   restart: unless-stopped
  #   ports:
  #     - "3000:3000"
  #   environment:
  #     DATABASE_URL: postgres://pensine:${POSTGRES_PASSWORD}@postgres:5432/pensine
  #     RABBITMQ_URL: amqp://pensine:${RABBITMQ_PASSWORD}@rabbitmq:5672
  #     SUPABASE_URL: ${SUPABASE_URL}
  #     SUPABASE_ANON_KEY: ${SUPABASE_ANON_KEY}
  #     JWT_SECRET: ${JWT_SECRET}
  #     MINIO_ENDPOINT: minio
  #     MINIO_PORT: 9000
  #     MINIO_ACCESS_KEY: ${MINIO_ROOT_USER}
  #     MINIO_SECRET_KEY: ${MINIO_ROOT_PASSWORD}
  #   depends_on:
  #     - postgres
  #     - rabbitmq
  #     - minio
  #   networks:
  #     - pensine-network

networks:
  pensine-network:
    driver: bridge

volumes:
  postgres-data:
    driver: local
  rabbitmq-data:
    driver: local
  minio-data:
    driver: local
```

#### Step 2.2: Create Environment File

```bash
# .env
POSTGRES_PASSWORD=your-secure-postgres-password-here
RABBITMQ_PASSWORD=your-secure-rabbitmq-password-here

MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=your-secure-minio-password-here

# Supabase (from Step 1.3)
SUPABASE_URL=https://xxxxx.supabase.co
SUPABASE_ANON_KEY=eyJhbGc...
SUPABASE_SERVICE_KEY=eyJhbGc... # Backend admin operations
JWT_SECRET=your-jwt-secret-from-supabase
```

#### Step 2.3: Start Infrastructure

```bash
# Start services
docker-compose up -d

# Check all healthy
docker-compose ps

# Expected output:
# pensine-db       Up (healthy)
# pensine-queue    Up (healthy)
# pensine-storage  Up (healthy)

# Access services:
# PostgreSQL:   localhost:5432
# RabbitMQ UI:  http://localhost:15672 (user: pensine)
# MinIO Console: http://localhost:9001 (user: minioadmin)
```

#### Step 2.4: Initialize MinIO Bucket

```bash
# Install MinIO Client (mc)
# macOS:
brew install minio/stable/mc

# Linux:
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Configure mc
mc alias set local http://localhost:9000 minioadmin your-minio-password

# Create bucket for audio files
mc mb local/pensine-audios

# Set bucket policy (public read, private write)
mc anonymous set download local/pensine-audios

# Verify
mc ls local
# Output: [DATE] [SIZE] pensine-audios/
```

---

### Part 3: Cloudflare Tunnel Setup (1-2 hours)

#### Step 3.1: Install Cloudflared

```bash
# macOS:
brew install cloudflared

# Linux:
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Verify installation
cloudflared --version
```

#### Step 3.2: Authenticate Cloudflare Account

```bash
# Login to Cloudflare
cloudflared tunnel login

# Browser opens → Select domain (pensine.app)
# Creates cert.pem in ~/.cloudflared/
```

#### Step 3.3: Create Tunnel

```bash
# Create tunnel
cloudflared tunnel create pensine

# Output:
# Tunnel credentials written to: ~/.cloudflared/<TUNNEL-ID>.json
# Created tunnel pensine with id <TUNNEL-ID>

# Copy tunnel ID for next steps
export TUNNEL_ID=<TUNNEL-ID>
```

#### Step 3.4: Configure Tunnel Routes

```bash
# Create config file
mkdir -p ~/.cloudflared
cat > ~/.cloudflared/config.yml <<EOF
tunnel: ${TUNNEL_ID}
credentials-file: /Users/$(whoami)/.cloudflared/${TUNNEL_ID}.json

ingress:
  # API Backend (NestJS)
  - hostname: api.pensine.app
    service: http://localhost:3000

  # Storage (MinIO)
  - hostname: storage.pensine.app
    service: http://localhost:9000

  # Catch-all rule (required)
  - service: http_status:404
EOF

# Configure DNS (creates CNAME records)
cloudflared tunnel route dns pensine api.pensine.app
cloudflared tunnel route dns pensine storage.pensine.app

# Output:
# Created CNAME api.pensine.app → <TUNNEL-ID>.cfargotunnel.com
# Created CNAME storage.pensine.app → <TUNNEL-ID>.cfargotunnel.com
```

#### Step 3.5: Start Tunnel

```bash
# Run tunnel (foreground for testing)
cloudflared tunnel run pensine

# Or run as service (background)
cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared

# Verify tunnel status
cloudflared tunnel info pensine

# Test access
curl https://storage.pensine.app/minio/health/live
# Output: OK (if MinIO running)
```

---

### Part 4: Mobile App Initialization (2 hours)

#### Step 4.1: Create Expo Project

```bash
# Create Expo app with TypeScript
npx create-expo-app@latest pensine-mobile --template blank-typescript

cd pensine-mobile

# Install dependencies
npm install @supabase/supabase-js
npm install @react-native-async-storage/async-storage
npm install @react-navigation/native
npm install @react-navigation/native-stack
npm install @react-navigation/bottom-tabs
npm install @watermelondb/react-native
npm install react-native-screens react-native-safe-area-context

# Expo dependencies
npx expo install expo-linking
npx expo install expo-auth-session
npx expo install expo-web-browser
```

#### Step 4.2: Configure Supabase Client

```typescript
// src/lib/supabase.ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = 'https://xxxxx.supabase.co';
const supabaseAnonKey = 'eyJhbGc...'; // Your anon key

export const supabase = createClient(supabaseUrl, supabaseAnonKey, {
  auth: {
    storage: AsyncStorage,
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: false,
  },
});
```

#### Step 4.3: Configure Deep Linking

```json
// app.json
{
  "expo": {
    "name": "Pensine",
    "slug": "pensine",
    "version": "1.0.0",
    "scheme": "pensine",
    "ios": {
      "bundleIdentifier": "com.pensine.app",
      "supportsTablet": true
    },
    "android": {
      "package": "com.pensine.app",
      "adaptiveIcon": {
        "backgroundColor": "#ffffff"
      }
    },
    "plugins": [
      "expo-router"
    ]
  }
}
```

#### Step 4.4: Initialize WatermelonDB

```bash
# Install WatermelonDB
npm install @nozbe/watermelondb
npm install --save-dev @babel/plugin-proposal-decorators

# Configure Babel
cat > babel.config.js <<EOF
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: [
      ['@babel/plugin-proposal-decorators', { legacy: true }],
    ],
  };
};
EOF
```

```typescript
// src/database/index.ts
import { Database } from '@nozbe/watermelondb';
import SQLiteAdapter from '@nozbe/watermelondb/adapters/sqlite';

import { schema } from './schema';
import { migrations } from './migrations';
// Models will be imported here later

const adapter = new SQLiteAdapter({
  schema,
  migrations,
  dbName: 'pensine',
});

export const database = new Database({
  adapter,
  modelClasses: [
    // Models will be added here
  ],
});
```

#### Step 4.5: Project Structure (DDD)

```bash
mkdir -p src/{lib,database,contexts,navigation,hooks,components}

# Bounded Contexts (mobile-side representations)
mkdir -p src/contexts/{capture,knowledge,opportunity,action,identity}

# Each context structure:
# src/contexts/capture/
#   - models/        (WatermelonDB models)
#   - screens/       (React Native screens)
#   - hooks/         (Custom hooks)
#   - services/      (API clients)
```

---

### Part 5: Backend Initialization (2 hours)

#### Step 5.1: Create NestJS Project

```bash
# Create NestJS app
npx @nestjs/cli new pensine-backend

cd pensine-backend

# Install dependencies
npm install @nestjs/config
npm install @nestjs/typeorm typeorm pg
npm install @nestjs/microservices amqplib
npm install @supabase/supabase-js
npm install minio

# Development dependencies
npm install --save-dev @types/node
```

#### Step 5.2: Configure Supabase Auth Guard

```typescript
// src/modules/shared/infrastructure/guards/supabase-auth.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';
import { createClient } from '@supabase/supabase-js';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class SupabaseAuthGuard implements CanActivate {
  private supabase;

  constructor(private configService: ConfigService) {
    this.supabase = createClient(
      this.configService.get('SUPABASE_URL'),
      this.configService.get('SUPABASE_ANON_KEY'),
    );
  }

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);

    if (!token) {
      throw new UnauthorizedException('No token provided');
    }

    // Validate JWT locally (no network call to Supabase)
    const { data: { user }, error } = await this.supabase.auth.getUser(token);

    if (error || !user) {
      throw new UnauthorizedException('Invalid token');
    }

    request.user = user; // Attach user to request
    return true;
  }

  private extractToken(request: any): string | null {
    const authHeader = request.headers.authorization;
    if (!authHeader) return null;

    const [type, token] = authHeader.split(' ');
    return type === 'Bearer' ? token : null;
  }
}
```

#### Step 5.3: Configure MinIO Service

```typescript
// src/modules/shared/infrastructure/storage/minio.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as Minio from 'minio';

@Injectable()
export class MinioService {
  private client: Minio.Client;
  private bucket: string;

  constructor(private config: ConfigService) {
    this.client = new Minio.Client({
      endPoint: this.config.get('MINIO_ENDPOINT'),
      port: parseInt(this.config.get('MINIO_PORT')),
      useSSL: this.config.get('MINIO_USE_SSL') === 'true',
      accessKey: this.config.get('MINIO_ACCESS_KEY'),
      secretKey: this.config.get('MINIO_SECRET_KEY'),
    });

    this.bucket = 'pensine-audios';
    this.ensureBucket();
  }

  private async ensureBucket() {
    const exists = await this.client.bucketExists(this.bucket);
    if (!exists) {
      await this.client.makeBucket(this.bucket, 'us-east-1');
    }
  }

  async presignedPutObject(
    objectName: string,
    expiry: number = 3600,
  ): Promise<string> {
    return await this.client.presignedPutObject(
      this.bucket,
      objectName,
      expiry,
    );
  }

  async presignedGetObject(
    objectName: string,
    expiry: number = 86400,
  ): Promise<string> {
    return await this.client.presignedGetObject(
      this.bucket,
      objectName,
      expiry,
    );
  }

  async removeObject(objectName: string): Promise<void> {
    await this.client.removeObject(this.bucket, objectName);
  }
}
```

#### Step 5.4: Configure PostgreSQL + TypeORM

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
    }),
    TypeOrmModule.forRootAsync({
      useFactory: () => ({
        type: 'postgres',
        url: process.env.DATABASE_URL,
        autoLoadEntities: true,
        synchronize: process.env.NODE_ENV !== 'production',
      }),
    }),
  ],
})
export class AppModule {}
```

#### Step 5.5: Project Structure (DDD)

```bash
mkdir -p src/modules/{shared,capture,knowledge,opportunity,action,identity}

# Bounded Context structure (example: Capture)
# src/modules/capture/
#   - domain/
#     - entities/       (TypeORM entities)
#     - value-objects/
#     - events/        (Domain events)
#   - application/
#     - services/      (Use cases)
#     - dto/          (Data Transfer Objects)
#   - infrastructure/
#     - controllers/   (HTTP/REST endpoints)
#     - repositories/  (TypeORM repositories)
```

#### Step 5.6: Environment Configuration

```bash
# .env
NODE_ENV=development

# Database
DATABASE_URL=postgres://pensine:your-password@localhost:5432/pensine

# RabbitMQ
RABBITMQ_URL=amqp://pensine:your-password@localhost:5672

# Supabase Auth
SUPABASE_URL=https://xxxxx.supabase.co
SUPABASE_ANON_KEY=eyJhbGc...
SUPABASE_SERVICE_KEY=eyJhbGc... # For admin operations
JWT_SECRET=your-jwt-secret

# MinIO Storage
MINIO_ENDPOINT=localhost
MINIO_PORT=9000
MINIO_USE_SSL=false
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=your-minio-password
```

---

## 📊 Definition of Done

### Infrastructure

- [ ] **Supabase Cloud:**
  - [ ] Project created (Free tier)
  - [ ] Email/Password provider enabled
  - [ ] Google OAuth configured
  - [ ] Apple Sign-In configured
  - [ ] JWT secret documented

- [ ] **Homelab Docker Stack:**
  - [ ] PostgreSQL running and healthy
  - [ ] RabbitMQ running and healthy
  - [ ] MinIO running and healthy
  - [ ] MinIO bucket `pensine-audios` created
  - [ ] All services accessible via `localhost`

- [ ] **Cloudflare Tunnel:**
  - [ ] Tunnel created and running
  - [ ] DNS configured: `api.pensine.app`
  - [ ] DNS configured: `storage.pensine.app`
  - [ ] HTTPS working (automatic SSL)
  - [ ] Health checks passing from Internet

### Mobile App

- [ ] **Expo Project:**
  - [ ] Created with TypeScript strict mode
  - [ ] Supabase client configured
  - [ ] WatermelonDB initialized
  - [ ] React Navigation configured
  - [ ] Deep linking configured (`pensine://`)
  - [ ] DDD folder structure created
  - [ ] `npm start` works without errors

### Backend

- [ ] **NestJS Project:**
  - [ ] Created with TypeScript strict mode
  - [ ] Supabase Auth Guard implemented
  - [ ] MinIO Service implemented
  - [ ] PostgreSQL connection working
  - [ ] RabbitMQ connection working
  - [ ] DDD folder structure created
  - [ ] `npm run start:dev` works without errors

### Documentation

- [ ] **README.md** created with:
  - [ ] Architecture diagram (hybrid cloud/homelab)
  - [ ] Setup instructions (all steps)
  - [ ] Environment variables documented
  - [ ] Troubleshooting section

- [ ] **.env.example** files created:
  - [ ] Backend `.env.example`
  - [ ] Mobile `.env.example` (if applicable)

---

## 🚨 Common Pitfalls

### ❌ DO NOT:

1. **Use boilerplate/starters:**
   - ❌ NestJS DDD boilerplate
   - ❌ Expo full-stack starter
   - Reason: ADR-007 compliance

2. **Implement custom auth:**
   - ❌ bcrypt password hashing
   - ❌ JWT token generation
   - ❌ OAuth2 flows from scratch
   - Reason: Delegated to Supabase (ADR-016)

3. **Install Redis:**
   - ❌ Not needed without JWT blacklist
   - Reason: Simplified per ADR-016

4. **Expose homelab directly:**
   - ❌ Port forwarding without tunnel
   - Reason: Security risk, use Cloudflare Tunnel

5. **Hardcode credentials:**
   - ❌ Supabase keys in code
   - ❌ Database passwords in code
   - Use environment variables

### ✅ DO:

1. **Follow DDD structure strictly:**
   - Bounded Contexts = folders
   - Domain/Application/Infrastructure layers

2. **Use Supabase client correctly:**
   - Mobile: `@supabase/supabase-js`
   - Backend: Auth Guard only (validation)

3. **MinIO presigned URLs:**
   - Backend generates URLs
   - Mobile uploads/downloads direct

4. **Test each component:**
   - Supabase auth working
   - MinIO upload/download working
   - Cloudflare Tunnel accessible

5. **Document everything:**
   - Architecture decisions
   - Setup steps
   - Troubleshooting

---

## 📚 References

- **ADR-016:** Hybrid Architecture (Cloud Auth + Homelab Storage)
- **ADR-007:** From Scratch Approach
- **Supabase Docs:** https://supabase.com/docs
- **MinIO Docs:** https://min.io/docs
- **Cloudflare Tunnel:** https://developers.cloudflare.com/cloudflare-one/connections/connect-apps
- **WatermelonDB:** https://watermelondb.dev/docs
- **NestJS:** https://docs.nestjs.com

---

## ⏱️ Estimated Time

- **Supabase Cloud Setup:** 30 minutes
- **Homelab Infrastructure:** 4-6 hours
- **Cloudflare Tunnel:** 1-2 hours
- **Mobile App Init:** 2 hours
- **Backend Init:** 2 hours

**Total:** ~10-13 hours (spread over 1-2 jours)

---

**Story Ready for Development** ✅

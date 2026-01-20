# Epic 1 - Action Items Tracker

**Source:** Epic 1 Retrospective
**Date Created:** 2026-01-20
**Epic Target:** Epic 2 (Capture & Transcription)

---

## 🔥 Priority 1: CRITICAL (MUST COMPLETE BEFORE STORY 2.1)

### AI-1: Setup Mobile Jest Configuration

**Status:** 🔴 TODO
**Owner:** Dev Agent
**Deadline:** Before Story 2.1 starts
**Estimated Effort:** 1 hour

**Why Critical:**
- Mobile tests exist but cannot run
- Blocks E2E validation on mobile
- Android blind spot (user has no Android device)
- User feedback: "tu ne peux constater les soucis sur android"

**Tasks:**
- [ ] Install Jest dependencies in pensieve/mobile/
  ```bash
  npm install --save-dev jest @testing-library/react-native @testing-library/jest-native
  ```
- [ ] Create `pensieve/mobile/jest.config.js`
  ```javascript
  module.exports = {
    preset: 'jest-expo',
    transformIgnorePatterns: [
      'node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|react-native-svg)'
    ],
    setupFilesAfterEnv: ['<rootDir>/jest-setup.js'],
  };
  ```
- [ ] Add test scripts to `pensieve/mobile/package.json`
  ```json
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
  ```
- [ ] Create `pensieve/mobile/jest-setup.js` with mocks
- [ ] Run existing tests: `npm test`
- [ ] Verify SettingsScreen.test.tsx passes
- [ ] Document test commands in `pensieve/mobile/README.md`

**Success Criteria:**
- ✅ `npm test` runs successfully in pensieve/mobile/
- ✅ All 16 SettingsScreen tests pass
- ✅ No configuration errors

**User Benefit:**
- Automated validation replaces manual QA
- Android coverage without device/emulator

---

### AI-2: Add CI Quality Gate (GitHub Actions)

**Status:** 🔴 TODO
**Owner:** Dev Agent
**Deadline:** Before Story 2.1 starts
**Estimated Effort:** 1-2 hours

**Why Critical:**
- Enforce Definition of Done automatically
- Prevent merges without tests (Story 1.3 pattern)
- Reduce user QA burden
- User feedback: "j'ai passé trop de temps à tester l'app pour te valider"

**Tasks:**
- [ ] Create `.github/workflows/ci.yml`
  ```yaml
  name: CI
  on: [push, pull_request]

  jobs:
    backend-tests:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v3
        - uses: actions/setup-node@v3
          with:
            node-version: '18'
        - name: Install dependencies
          run: cd pensieve/backend && npm ci
        - name: Run tests
          run: cd pensieve/backend && npm test -- --coverage
        - name: Check coverage
          run: |
            cd pensieve/backend
            COVERAGE=$(npx nyc report --reporter=text-summary | grep 'Statements' | awk '{print $3}' | sed 's/%//')
            if (( $(echo "$COVERAGE < 80" | bc -l) )); then
              echo "Coverage $COVERAGE% is below 80%"
              exit 1
            fi

    mobile-tests:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v3
        - uses: actions/setup-node@v3
          with:
            node-version: '18'
        - name: Install dependencies
          run: cd pensieve/mobile && npm ci
        - name: Run tests
          run: cd pensieve/mobile && npm test -- --coverage
        - name: Check coverage
          run: |
            cd pensieve/mobile
            COVERAGE=$(npx nyc report --reporter=text-summary | grep 'Statements' | awk '{print $3}' | sed 's/%//')
            if (( $(echo "$COVERAGE < 80" | bc -l) )); then
              echo "Coverage $COVERAGE% is below 80%"
              exit 1
            fi
  ```
- [ ] Add status badge to README
- [ ] Configure branch protection rules (require CI to pass)
- [ ] Test CI with a dummy PR

**Success Criteria:**
- ✅ CI runs on every push/PR
- ✅ Backend tests run and check coverage >80%
- ✅ Mobile tests run and check coverage >80%
- ✅ PRs cannot merge if CI fails
- ✅ Status badge shows in README

**User Benefit:**
- Automatic enforcement of test requirements
- No more manual review to check if tests exist
- Confidence that code is tested before merge

---

### AI-3: Enforce TDD Workflow

**Status:** 🔴 TODO
**Owner:** Dev Agent (process change)
**Deadline:** Starting Story 2.1
**Estimated Effort:** 0 hours (process change, not code)

**Why Critical:**
- Prevent retroactive test creation (Story 1.3 pattern)
- Reduce user QA burden from the start
- User feedback: "j'ai passé trop de temps à tester l'app"

**Workflow:**

**Red Phase (Write Failing Test FIRST):**
```bash
# 1. Create test file BEFORE implementation
touch src/feature.test.ts

# 2. Write failing test
describe('Feature', () => {
  it('should do X when Y happens', () => {
    // Arrange
    const input = setupTestData();

    // Act
    const result = featureFunction(input);

    // Assert
    expect(result).toBe(expectedOutput);
  });
});

# 3. Run test → RED (fails)
npm test
```

**Green Phase (Minimal Implementation):**
```bash
# 4. Write minimal code to pass test
touch src/feature.ts
# ... implement just enough to pass test

# 5. Run test → GREEN (passes)
npm test
```

**Refactor Phase (Clean Up):**
```bash
# 6. Refactor while keeping tests green
# ... improve code quality, extract functions, etc.

# 7. Verify tests still pass
npm test
```

**Tasks:**
- [ ] Document TDD workflow in `CONTRIBUTING.md`
- [ ] Add TDD reminder to Story 2.1 template
- [ ] Commit to TDD cycle for ALL Epic 2 stories
- [ ] NO code review without tests (enforced by AI-2 CI)

**Success Criteria:**
- ✅ 100% of Epic 2 stories follow TDD cycle
- ✅ Tests written BEFORE or DURING implementation (never after)
- ✅ User does not have to manually test features

**User Benefit:**
- Features work on first try (tests validate)
- User can trust implementation without manual QA
- Faster iteration (no back-and-forth for bug fixes)

---

## ⚠️ Priority 2: HIGH (COMPLETE DURING EPIC 2)

### AI-4: Add Story 1.2 (Auth) Tests

**Status:** 🔴 TODO
**Owner:** Dev Agent
**Deadline:** During Epic 2 (backfill Epic 1 gap)
**Estimated Effort:** 3-4 hours

**Why High Priority:**
- Story 1.2 has 0% test coverage
- Auth is critical functionality
- Gap from Epic 1 needs to be filled

**Tasks:**
- [ ] Create `LoginScreen.test.tsx`
  - Email/password login flow
  - Google Sign-In flow
  - Apple Sign-In flow (iOS)
  - Error handling (invalid credentials)
  - Loading states
- [ ] Create `RegisterScreen.test.tsx`
  - Registration flow
  - Password validation (8 chars, uppercase, number)
  - Email validation
  - Confirmation email flow
  - Error handling
- [ ] Create `ForgotPasswordScreen.test.tsx`
  - Password reset request
  - Email sent confirmation
  - Error handling
- [ ] Create `ResetPasswordScreen.test.tsx`
  - Password update flow
  - Password validation
  - Deep link handling
  - Error handling

**Success Criteria:**
- ✅ Auth module has >80% test coverage
- ✅ All auth flows validated by tests
- ✅ CI passes for auth module

**User Benefit:**
- Auth bugs caught automatically
- User doesn't need to manually test login/register flows

---

### AI-5: Add Infrastructure Health Checks

**Status:** 🔴 TODO
**Owner:** Dev Agent
**Deadline:** Story 2.1 (while working on capture infrastructure)
**Estimated Effort:** 1 hour

**Why High Priority:**
- Validate infrastructure is running correctly
- Reduce manual checks (`docker-compose ps`)
- Early detection of service failures

**Tasks:**
- [ ] Create `src/modules/shared/infrastructure/controllers/health.controller.ts`
  ```typescript
  @Controller('api/health')
  export class HealthController {
    @Get()
    async checkHealth() {
      const checks = {
        postgres: await this.checkPostgreSQL(),
        rabbitmq: await this.checkRabbitMQ(),
        minio: await this.checkMinIO(),
        cloudflare: await this.checkCloudflareTunnel(), // optional
      };

      const allHealthy = Object.values(checks).every(c => c.status === 'ok');

      return {
        status: allHealthy ? 'healthy' : 'degraded',
        timestamp: new Date().toISOString(),
        checks,
      };
    }
  }
  ```
- [ ] Implement health check methods for each service
- [ ] Add `npm run health-check` script
- [ ] Document health check endpoint in README

**Success Criteria:**
- ✅ `GET /api/health` returns status of all services
- ✅ `npm run health-check` runs from command line
- ✅ Health check detects service failures

**User Benefit:**
- Quick validation that infrastructure is running
- Automated monitoring (can be polled by monitoring tools)

---

## 📋 Priority 3: MEDIUM (BACKLOG - AFTER EPIC 2)

### AI-6: Automate Infrastructure Setup

**Status:** 🟡 BACKLOG
**Owner:** Dev Agent
**Deadline:** After Epic 2
**Estimated Effort:** 4-6 hours

**Why Medium Priority:**
- User feedback: "docker compose fait bien le travail" (not urgent now)
- Manual setup works for single developer
- Becomes important when onboarding new developers

**Tasks:**
- [ ] Create `scripts/setup-supabase.sh`
  - Use Supabase CLI for project creation
  - Configure auth providers via CLI
  - Generate .env file automatically
- [ ] Create `scripts/setup-cloudflare.sh`
  - Automate tunnel creation
  - Configure DNS via CLI
  - Generate credentials file
- [ ] Create `scripts/setup-minio.sh`
  - Use mc CLI for bucket creation
  - Set bucket policies
  - Generate access keys
- [ ] Create `scripts/setup-all.sh` (orchestrates all setup)
- [ ] Document in README

**Success Criteria:**
- ✅ New developer can setup in <30 min
- ✅ Setup is reproducible across machines
- ✅ No manual dashboard configuration needed

---

### AI-7: Document Production Security Considerations

**Status:** 🟡 BACKLOG
**Owner:** Dev Agent or SM
**Deadline:** Before production deployment (after Epic 4-5)
**Estimated Effort:** 2-3 hours

**Why Medium Priority:**
- User concern: "j'appréhende l'ouverture de certains protocoles (udp, quick etc ...)"
- Not needed until production deployment
- Important for security hardening

**Tasks:**
- [ ] Document required ports/protocols
  - TCP 443 (HTTPS via Cloudflare Tunnel)
  - UDP 443 (QUIC via Cloudflare, if enabled)
  - Internal: PostgreSQL 5432, RabbitMQ 5672, MinIO 9000
- [ ] Create security hardening guide
  - Firewall rules (homelab DMZ isolation)
  - Rate limiting (Cloudflare WAF)
  - DDoS protection (Cloudflare)
  - SSL/TLS configuration (Let's Encrypt via Cloudflare)
- [ ] Network architecture diagram
  - Internet → Cloudflare Tunnel → Homelab
  - Show DMZ isolation
  - Show internal network segmentation
- [ ] Document Cloudflare security features
  - WAF rules
  - Bot protection
  - Rate limiting policies

**Success Criteria:**
- ✅ Security checklist for production deployment
- ✅ Network architecture documented
- ✅ User concerns addressed (UDP/QUIC protocols)

---

## 📊 Progress Tracking

**Epic 2 Readiness:**
- [ ] AI-1: Setup Mobile Jest ← BLOCKER
- [ ] AI-2: Add CI Quality Gate ← BLOCKER
- [ ] AI-3: Enforce TDD Workflow ← BLOCKER

**Epic 2 Completion:**
- [ ] AI-4: Add Story 1.2 Tests
- [ ] AI-5: Add Health Checks

**Future Epics:**
- [ ] AI-6: Automate Infrastructure Setup
- [ ] AI-7: Document Production Security

---

## 🎯 Next Steps

1. **IMMEDIATE:** Complete AI-1, AI-2, AI-3 before starting Story 2.1
2. **DURING EPIC 2:** Complete AI-4, AI-5
3. **AFTER EPIC 2:** AI-6, AI-7 backlog items

**Goal:** Eliminate user QA burden by automating all testing.

**User Feedback to Address:**
> "j'ai passé trop de temps à tester simplement et en profondeur l'app pour te valider ou t'invalider des tâches"

**Solution:** Priority 1 items (AI-1, AI-2, AI-3) directly address this problem.

---

**Last Updated:** 2026-01-20
**Status:** Ready for Epic 2

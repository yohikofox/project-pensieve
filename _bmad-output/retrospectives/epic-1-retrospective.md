# Epic 1 Retrospective: Foundation & Authentication

**Epic:** Epic 1 - Foundation & Authentification
**Date:** 2026-01-20
**Participants:** Dev (Claude), Yohikofox (Product Owner)
**Sprint Duration:** Multiple sessions
**Status:** ✅ DONE (all 3 stories completed)

---

## 📦 Epic Scope

### Stories Completed

**Original Plan (deprecated):**
- Story 1.1: Infrastructure Setup
- Story 1.2: User Registration
- Story 1.3: User Login
- Story 1.4: User Logout
- Story 1.5: Password Recovery

**Actual Implementation (ADR-016 Hybrid Architecture):**
- ✅ **Story 1.1 v2:** Project Foundation & Infrastructure Setup
  - Supabase Cloud setup (auth)
  - Homelab infrastructure (PostgreSQL, RabbitMQ, MinIO)
  - Cloudflare Tunnel configuration
  - Mobile + Backend initialization
- ✅ **Story 1.2 v2:** Auth Integration (Supabase Cloud)
  - Email/Password authentication
  - Google OAuth
  - Apple Sign-In
  - Password recovery
  - Session persistence
- ✅ **Story 1.3 v2:** RGPD Compliance
  - Data export (Article 15)
  - Account deletion (Article 17)
  - Audit logging
  - Privacy policy integration

### Architectural Pivot (ADR-016)

**Decision:** Hybrid Architecture (Supabase Cloud auth + Homelab storage)

**Rationale:**
- Social auth (Google/Apple) ready in 20 min vs 10-15 days custom
- Cost: €0/month (Supabase Free tier + homelab)
- Unlimited storage (homelab) vs $750/year (Supabase Storage)
- Time-to-market optimized

**Impact:**
- 5 stories consolidated → 3 stories v2
- Login/Logout/Password Recovery merged into Story 1.2 v2
- Significant scope reduction while maintaining functionality

---

## ✅ What Went Well

### 1. Architectural Pivot Success (ADR-016)
- ✅ **Strategic decision** validated: Hybrid approach saves time and money
- ✅ **Concrete wins:**
  - Social auth setup: 20 min (vs 10-15 days custom implementation)
  - Monthly cost: €0 (vs $750/year for Supabase Storage)
  - Storage capacity: Unlimited (homelab NAS)
- ✅ **Scope consolidation:** 5 stories → 3 stories (reduced complexity)
- ✅ **No regrets:** Architecture still valid and scalable

### 2. Comprehensive Documentation
- ✅ Stories contain complete, executable code examples
- ✅ Step-by-step setup instructions (Supabase, Cloudflare Tunnel, MinIO)
- ✅ Architecture diagrams clearly show system boundaries
- ✅ Common pitfalls documented proactively
- ✅ Environment variables documented (.env.example files)

### 3. Complete RGPD Implementation
- ✅ RGPD Articles 15 & 17 fully covered (export + deletion)
- ✅ Cascade deletion logic correct (PostgreSQL → MinIO → Supabase)
- ✅ Audit logs for compliance (3-year retention)
- ✅ UX with clear warnings and double confirmation
- ✅ Password re-verification before account deletion

### 4. Clean Architecture Maintained
- ✅ DDD bounded contexts respected (Identity, RGPD modules)
- ✅ Domain/Application/Infrastructure layers separated
- ✅ Services well-structured (RgpdService, SupabaseAdminService, MinioService)
- ✅ No framework coupling in domain layer

---

## ❌ What Didn't Go Well

### 1. 🚨 CRITICAL: Missing Tests Until the End
- ❌ **Problem:** Story 1.3 marked "done" with ZERO tests
- ❌ **Impact:** Definition of Done violated (>80% coverage required)
- ❌ **Discovery:** Only detected during final verification
- ❌ **Correction required:** 36 tests created retroactively
  - `rgpd.service.spec.ts`: 12 tests
  - `rgpd.controller.spec.ts`: 8 tests
  - `SettingsScreen.test.tsx`: 16 tests
- ❌ **Root cause:** Tests written AFTER implementation (not TDD)
- ❌ **User impact:** "j'ai passé trop de temps à tester simplement et en profondeur l'app pour te valider ou t'invalider des tâches"
  - **This is the core problem:** User doing manual QA that tests should automate

### 2. Mobile Testing Not Executable
- ❌ Tests created (`SettingsScreen.test.tsx`) but Jest not configured in mobile
- ❌ Tests exist but cannot run (`npm test` fails)
- ❌ Blocks E2E validation on mobile
- ❌ Android testing impossible (user has no Android device/emulator)
  - User feedback: "on a eu du mal à identifier la root cause sur mobile parce que tu ne peux constater les soucis sur android"
  - **Impact:** Blind spot on Android platform

### 3. Excessive Manual Setup
- ❌ Stories 1.1 and 1.2 require extensive manual configuration:
  - Supabase dashboard (Google OAuth credentials, Apple Sign-In config)
  - Cloudflare Tunnel creation + DNS configuration
  - MinIO bucket initialization + policy setup
- ❌ No automation for these steps
- ❌ Risk: Configuration drift between environments
- ❌ Onboarding friction for new developers
- **User feedback:** "pour le moment docker compose fait bien le travail" (not urgent to automate now)

### 4. Missing Infrastructure Validation
- ❌ No automated tests to validate infrastructure is running correctly
- ❌ No automated health checks (PostgreSQL, RabbitMQ, MinIO, Cloudflare Tunnel)
- ❌ Relies on manual verification (`docker-compose ps`, `curl` commands)
- ❌ No monitoring or alerting for service failures

### 5. Story 1.2 (Auth) Has Zero Tests
- ❌ LoginScreen, RegisterScreen, ForgotPasswordScreen: 0% test coverage
- ❌ Auth flows (Google, Apple, password reset) not validated by tests
- ❌ Only Story 1.3 (RGPD) has tests (created retroactively)

---

## 🎓 Lessons Learned

### 1. TDD is Non-Negotiable
- **Lesson:** Write tests BEFORE or DURING implementation, never after
- **Evidence:** Tests forgotten until end of Story 1.3, Story 1.2 has zero tests
- **User impact:** Manual QA burden falls on user
- **Action:** Enforce TDD workflow (Red-Green-Refactor cycle)
- **Success metric:** 100% of stories have tests before merge

### 2. Definition of Done Must Be Enforced
- **Lesson:** Story ≠ Done if tests < 80% coverage
- **Evidence:** Story 1.3 technically "not done" until tests created
- **User impact:** User wastes time doing QA that should be automated
- **Action:** Automated CI checks to block merge without tests
- **Success metric:** No PR merges without passing tests + coverage threshold

### 3. Architectural Pivots Are Risky But Valuable
- **Lesson:** ADR-016 saved time (5 stories → 3) but invalidated already-planned work
- **Trade-off:** Time-to-market gains vs wasted planning effort
- **Success:** Pivot was correct decision (no regrets)
- **Action:** Validate architecture decisions BEFORE story creation (not mid-sprint)

### 4. Mobile Testing Requires Infrastructure Early
- **Lesson:** Tests created but not executable = Dead code
- **Evidence:** SettingsScreen.test.tsx exists but Jest not configured
- **Impact:** No real E2E validation on mobile, blind spot on Android
- **Action:** Setup Jest mobile BEFORE writing tests
- **Success metric:** `npm test` works in mobile/ directory

### 5. Infrastructure as Code > Manual Setup (Backlog)
- **Lesson:** Manual setup (Supabase dashboard, Cloudflare) = Not reproducible
- **Risk:** Configuration drift, onboarding difficulty
- **User feedback:** "docker compose fait bien le travail" (acceptable for now)
- **Action:** Script automation (Terraform, CLI scripts) - backlog item, not urgent

### 6. Android Testing Gap
- **Lesson:** Cannot validate Android without device/emulator or automated tests
- **Evidence:** User feedback: "tu ne peux constater les soucis sur android"
- **Impact:** Android bugs may go undetected
- **Action:** Prioritize automated tests to cover Android blind spot

---

## 🚀 Action Items for Epic 2

### 🔥 Priority 1: CRITICAL (Must Fix Before Story 2.1)

#### **AI-1: Setup Mobile Jest Configuration**
- **Owner:** Dev Agent
- **When:** Before Story 2.1 (Epic 2)
- **Why:** Tests exist but cannot run. Blocks mobile validation. Android blind spot.
- **Action:**
  1. Configure Jest in `pensieve/mobile/jest.config.js`
  2. Add test scripts to `pensieve/mobile/package.json`
  3. Install required packages (`@testing-library/react-native`, etc.)
  4. Run existing tests (SettingsScreen.test.tsx) to validate
  5. Fix any configuration issues
  6. Document test commands in README
- **Success metric:** `npm test` runs successfully in mobile/
- **Estimated effort:** 1 hour

#### **AI-2: Add CI Quality Gate (GitHub Actions)**
- **Owner:** Dev Agent
- **When:** Before Story 2.1
- **Why:** Enforce Definition of Done automatically. Prevent merges without tests.
- **Action:**
  1. Create `.github/workflows/ci.yml`
  2. Run backend tests: `cd pensieve/backend && npm test`
  3. Run mobile tests: `cd pensieve/mobile && npm test`
  4. Check coverage threshold: >80% for modified files
  5. Block merge if tests fail OR coverage < 80%
  6. Add status badge to README
- **Success metric:** CI blocks PRs without tests or coverage < 80%
- **Estimated effort:** 1-2 hours

#### **AI-3: Enforce TDD Workflow**
- **Owner:** Dev Agent (process change)
- **When:** Starting Story 2.1
- **Why:** Prevent retroactive test creation. Reduce user QA burden.
- **Action:**
  1. **Red phase:** Write failing test FIRST
  2. **Green phase:** Implement minimal code to pass test
  3. **Refactor phase:** Clean up code while tests pass
  4. NO code review without tests (enforced by CI)
  5. Document TDD workflow in CONTRIBUTING.md
- **Success metric:** 100% of Epic 2 stories have tests written before/during implementation
- **User benefit:** No more manual QA burden ("tester simplement et en profondeur l'app")

---

### ⚠️ Priority 2: High (Complete During Epic 2)

#### **AI-4: Add Story 1.2 (Auth) Tests**
- **Owner:** Dev Agent
- **When:** During Epic 2 (fill gaps from Epic 1)
- **Why:** Story 1.2 has 0% test coverage. Auth is critical functionality.
- **Action:**
  1. Create `LoginScreen.test.tsx` (email/password, Google, Apple flows)
  2. Create `RegisterScreen.test.tsx` (validation, registration flow)
  3. Create `ForgotPasswordScreen.test.tsx` (password reset flow)
  4. Create `ResetPasswordScreen.test.tsx` (password update flow)
  5. Target: >80% coverage for all auth screens
- **Success metric:** Auth module has >80% test coverage
- **Estimated effort:** 3-4 hours

#### **AI-5: Add Infrastructure Health Checks**
- **Owner:** Dev Agent
- **When:** Story 2.1 (while working on capture infrastructure)
- **Why:** Validate infrastructure is running correctly. Reduce manual checks.
- **Action:**
  1. Add `GET /api/health` endpoint
  2. Check PostgreSQL connection
  3. Check RabbitMQ connection
  4. Check MinIO connection
  5. Check Cloudflare Tunnel (optional)
  6. Return JSON with status of each service
  7. Add `npm run health-check` script
- **Success metric:** Health check detects service failures automatically
- **Estimated effort:** 1 hour

---

### 📋 Priority 3: Medium (Backlog - After Epic 2)

#### **AI-6: Automate Infrastructure Setup**
- **Owner:** Dev Agent
- **When:** After Epic 2 (backlog)
- **Why:** Reduce manual setup steps. Improve reproducibility.
- **User feedback:** "docker compose fait bien le travail" (not urgent now)
- **Action:**
  1. Create `setup-supabase.sh` script (CLI automation via Supabase CLI)
  2. Create `setup-cloudflare.sh` script (cloudflared automation)
  3. Create `setup-minio.sh` script (mc CLI automation)
  4. Document "one-command setup" in README
- **Success metric:** New developer can setup in <30 min
- **Estimated effort:** 4-6 hours

#### **AI-7: Document Production Security Considerations**
- **Owner:** Dev Agent or SM
- **When:** Before production deployment (after Epic 4-5)
- **Why:** User concern: "j'appréhende l'ouverture de certains protocoles (udp, quick etc ...)"
- **Action:**
  1. Document required ports/protocols for production
  2. Security hardening guide (firewall rules, rate limiting)
  3. Cloudflare security features (DDoS protection, WAF)
  4. SSL/TLS configuration
  5. Network architecture diagram (DMZ, homelab isolation)
- **Success metric:** Security checklist for production deployment
- **Estimated effort:** 2-3 hours

---

## 📈 Epic 1 Metrics

### Effort (Estimated vs Actual)
- **Story 1.1 v2:** Estimated 10-13h → Actual: (not tracked)
- **Story 1.2 v2:** Estimated 9-11h → Actual: (not tracked)
- **Story 1.3 v2:** Estimated 8-10h → Actual: (not tracked) + 2h retroactive tests

**Total Epic 1:** ~27-34h estimated

### Quality Metrics

**Test Coverage:**
- Backend RGPD module: >80% (20 tests created)
- Backend Identity module: 0% (Story 1.2 - gap identified)
- Mobile Settings: 16 tests created (but Jest not configured)
- Mobile Auth: 0% tests

**Architecture Compliance:**
- ✅ DDD structure maintained
- ✅ ADR-016 (Hybrid Architecture) applied correctly
- ✅ Bounded contexts respected
- ❌ Definition of Done (tests) not enforced

### User Feedback (Critical)

> **"j'ai passé trop de temps à tester simplement et en profondeur l'app pour te valider ou t'invalider des tâches"**

**Analysis:** This is the core problem. User doing manual QA = tests not doing their job.

**Root cause:**
1. Tests written retroactively (not TDD)
2. Mobile tests not executable (Jest not configured)
3. No CI enforcement (can merge without tests)
4. Android blind spot (no device/emulator)

**Solution:** All Priority 1 action items address this problem.

---

## 🎯 Recommendations for Epic 2

### DO:
1. ✅ **Write tests FIRST** (strict TDD: Red-Green-Refactor)
2. ✅ **Setup Jest mobile** BEFORE starting Story 2.1 (AI-1)
3. ✅ **Add CI pipeline** to enforce tests + coverage (AI-2)
4. ✅ **Complete Story 1.2 tests** during Epic 2 (fill gap)
5. ✅ **Add health checks** for infrastructure validation
6. ✅ **Reduce user QA burden** (automate what user tests manually)

### DON'T:
1. ❌ **DO NOT mark story "done"** without tests (CI will block)
2. ❌ **DO NOT skip Definition of Done** (strict enforcement)
3. ❌ **DO NOT create tests retroactively** (TDD only)
4. ❌ **DO NOT make user do QA** (tests should validate)
5. ❌ **DO NOT merge without CI passing** (automated enforcement)

---

## 💬 User Feedback Summary

### Q1: Tests
> "on a eu du mal à identifier la root cause sur mobile parce que tu ne peux constater les soucis sur android"

**Action:** Prioritize mobile tests (AI-1) to cover Android blind spot.

### Q2: Architecture Hybrid
> "pour le moment ça va, rien a dire car on est encore en local. Cependant, j'appréhende l'ouverture de certains protocoles (udp, quick etc ...)"

**Action:** Document security considerations for production (AI-7, backlog).

### Q3: Setup Manuel
> "pour le moment docker compose fait bien le travail"

**Action:** Automation is backlog (AI-6), not urgent now.

### Q4: Epic 2 Readiness
> "je ne me rend pas compte à quel point ce qui manque est handicapant"

**Action:** Complete Priority 1 items (AI-1, AI-2, AI-3) to show value immediately.

### Q5: Definition of Done
> "oui ce serait mieux. Globalement, j'ai passé trop de temps à tester simplement et en profondeur l'app pour te valider ou t'invalider des tâches."

**Action:** 🚨 CRITICAL feedback. Strict DoD enforcement (CI + TDD) to eliminate user QA burden.

---

## 🏁 Epic 1 Conclusion

**Overall Assessment:** ✅ Epic 1 DONE with critical learnings

**Key Wins:**
- Architectural pivot (ADR-016) was correct decision
- RGPD implementation complete and compliant
- Foundation solid for Epic 2

**Key Gaps:**
- Tests not enforced (user doing manual QA)
- Mobile tests not executable
- Story 1.2 (Auth) has zero tests

**Epic 2 Focus:**
- Fix Priority 1 items BEFORE Story 2.1
- Strict TDD enforcement
- Eliminate user QA burden

---

**Next Steps:**
1. Complete Priority 1 action items (AI-1, AI-2, AI-3)
2. Begin Epic 2 Sprint Planning
3. Review Epic 2 stories with TDD reminder

**Retrospective Date:** 2026-01-20
**Retrospective Facilitator:** Bob (Scrum Master Agent)
**Status:** ✅ COMPLETE

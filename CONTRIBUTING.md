# Contributing to Pensine

Thank you for contributing to Pensine! This guide outlines our development workflow and quality standards.

---

## 📋 Table of Contents

1. [Development Philosophy](#development-philosophy)
2. [Test-Driven Development (TDD)](#test-driven-development-tdd)
3. [Definition of Done](#definition-of-done)
4. [Testing Strategy](#testing-strategy)
5. [CI/CD Pipeline](#cicd-pipeline)
6. [Code Review Process](#code-review-process)
7. [Git Workflow](#git-workflow)
8. [Development Environment](#development-environment)

---

## 🎯 Development Philosophy

Pensine follows a **test-first, quality-focused** development approach:

- **TDD is non-negotiable**: All code must be written following Red-Green-Refactor cycle
- **Quality over speed**: Stories are not complete without comprehensive test coverage
- **Backend-first testing**: Focus on backend tests (80% of business logic)
- **CI enforcement**: All quality gates must pass before merge

### Why This Matters

> "Globalement, j'ai passé trop de temps à tester simplement et en profondeur l'app pour te valider ou t'invalider des tâches."
> — User feedback, Epic 1 Retrospective

Our TDD workflow eliminates manual QA burden by ensuring automated test coverage from the start.

---

## 🔴🟢🔵 Test-Driven Development (TDD)

### The Red-Green-Refactor Cycle

All development MUST follow this cycle:

#### 1️⃣ **RED** - Write Failing Test
```bash
# Before writing any implementation code
cd pensieve/backend  # or pensieve/mobile
npm test -- path/to/feature.test.ts
```

**Requirements:**
- Write test that describes expected behavior
- Test must FAIL (proving it tests something real)
- Commit the failing test: `git commit -m "test: Add failing test for [feature]"`

#### 2️⃣ **GREEN** - Make Test Pass
```bash
# Write minimal code to make test pass
npm test -- path/to/feature.test.ts
```

**Requirements:**
- Write simplest code that makes test pass
- Do NOT over-engineer or add extra features
- Run test repeatedly until it passes

#### 3️⃣ **REFACTOR** - Improve Code Quality
```bash
# Clean up implementation while keeping tests green
npm test -- path/to/feature.test.ts
```

**Requirements:**
- Improve code structure, readability, performance
- Keep all tests passing during refactoring
- Run full test suite: `npm test`

### Example TDD Workflow

**User Story**: "As a user, I want to create a thought with title and content"

```bash
# 1. RED - Write failing test
cd pensieve/backend
cat > src/modules/knowledge/thoughts/thoughts.service.spec.ts <<EOF
describe('ThoughtsService', () => {
  it('should create a thought with title and content', async () => {
    const result = await service.createThought({
      title: 'My Idea',
      content: 'This is brilliant!'
    });
    expect(result).toHaveProperty('id');
    expect(result.title).toBe('My Idea');
  });
});
EOF

npm test -- thoughts.service.spec.ts  # Should FAIL
git add .
git commit -m "test: Add failing test for thought creation"

# 2. GREEN - Implement minimal solution
# ... edit thoughts.service.ts ...
npm test -- thoughts.service.spec.ts  # Should PASS

# 3. REFACTOR - Improve code quality
# ... refactor thoughts.service.ts ...
npm test  # All tests should still PASS

git add .
git commit -m "feat: Implement thought creation with validation"
```

---

## ✅ Definition of Done

A story is NOT complete until ALL criteria are met:

### Code Requirements
- [ ] All acceptance criteria implemented
- [ ] Code follows TypeScript strict mode conventions
- [ ] No TypeScript errors (`npx tsc --noEmit`)
- [ ] Linting passes (`npm run lint`)

### Testing Requirements (MANDATORY)
- [ ] **TDD workflow followed** (tests written BEFORE implementation)
- [ ] **Backend: >80% coverage** for branches, functions, lines, statements
- [ ] **Mobile: Best effort** (blocked by Expo SDK 54, see `pensieve/mobile/TESTING.md`)
- [ ] All tests pass locally: `npm test`
- [ ] CI pipeline passes (all quality gates green)

### Documentation Requirements
- [ ] Code comments for complex logic
- [ ] Update relevant documentation files
- [ ] Story file updated with completion notes

### Review Requirements
- [ ] Self-review completed
- [ ] PR created with descriptive title and body
- [ ] CI checks passing
- [ ] Code review approved

---

## 🧪 Testing Strategy

### Backend Testing (Primary Focus)

**Location**: `pensieve/backend/src/**/*.spec.ts`

**Coverage Requirements**: **Minimum 80%** for:
- Branches
- Functions
- Lines
- Statements

**Run Tests**:
```bash
cd pensieve/backend

# Run all tests
npm test

# Run with coverage
npm run test:cov

# Watch mode (TDD)
npm run test:watch

# CI mode
npm run test:ci
```

**Writing Backend Tests**:
```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { MyService } from './my-service.service';

describe('MyService', () => {
  let service: MyService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [MyService],
    }).compile();

    service = module.get<MyService>(MyService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  // RED: Write failing test first
  it('should do something specific', () => {
    const result = service.doSomething();
    expect(result).toBe('expected value');
  });
});
```

### Mobile Testing (Known Limitation)

**Status**: ⚠️ **BLOCKED** by Expo SDK 54 + jest-expo incompatibility

**Location**: `pensieve/mobile/src/**/*.test.ts`

**Current Approach**:
- Backend tests cover 80% of business logic
- Manual QA required for mobile UI
- E2E tests planned for future (Detox/Maestro)

**Documentation**: See [`pensieve/mobile/TESTING.md`](pensieve/mobile/TESTING.md) for details

---

## 🚀 CI/CD Pipeline

### GitHub Actions Workflow

All PRs must pass these quality gates:

#### 1. Backend Tests & Coverage
- Runs full test suite
- Enforces **80% minimum coverage**
- Uploads coverage to Codecov
- **Blocks merge if coverage < 80%**

#### 2. Linting
- Runs ESLint on backend code
- Checks code style consistency

#### 3. TypeScript Type Checking
- Validates backend TypeScript (`npx tsc --noEmit`)
- Validates mobile TypeScript (`npx tsc --noEmit`)
- **Blocks merge if type errors exist**

#### 4. Quality Gate
- Combines all checks
- **Fails if ANY check fails**
- Prevents merging broken code

### Running CI Locally

Before pushing, ensure all checks pass:

```bash
# Backend tests
cd pensieve/backend
npm test -- --coverage --ci
npm run lint
npx tsc --noEmit

# Mobile type check
cd pensieve/mobile
npx tsc --noEmit
```

### CI Status Badges

Check CI status at:
- [![CI](https://github.com/yohikofox/pensine/actions/workflows/ci.yml/badge.svg)](https://github.com/yohikofox/pensine/actions/workflows/ci.yml)
- [![Backend Coverage](https://codecov.io/gh/yohikofox/pensine/branch/main/graph/badge.svg?flag=backend)](https://codecov.io/gh/yohikofox/pensine)

---

## 👀 Code Review Process

### Before Requesting Review

1. **Self-review your changes**:
   - Read through the diff carefully
   - Check for commented-out code, debug statements
   - Verify tests are meaningful and comprehensive

2. **Run quality checks**:
   ```bash
   npm test -- --coverage
   npm run lint
   npx tsc --noEmit
   ```

3. **Update documentation**:
   - Update story file with completion notes
   - Add code comments for complex logic
   - Update relevant markdown docs

### Creating a Pull Request

**Title Format**:
```
feat(module): Brief description of change

Examples:
- feat(auth): Add JWT token refresh mechanism
- fix(capture): Handle audio recording permission denial
- test(knowledge): Add tests for thought creation flow
```

**PR Body Template**:
```markdown
## Summary
Brief description of what this PR does

## Related Story
- Story X.Y: [Story Title]
- Acceptance Criteria: AC1, AC2, AC3

## Test Coverage
- Backend coverage: XX%
- New tests added: X
- Edge cases covered: [list]

## Testing Checklist
- [ ] All tests pass locally
- [ ] CI pipeline passes
- [ ] Manual testing completed (if applicable)

## Screenshots (if UI changes)
[Add screenshots for mobile UI changes]
```

### Review Checklist

Reviewers should verify:
- [ ] TDD workflow followed (tests in separate commit?)
- [ ] Test coverage >80% for backend
- [ ] All acceptance criteria met
- [ ] No obvious bugs or security issues
- [ ] Code is readable and maintainable
- [ ] CI pipeline passing

---

## 🌿 Git Workflow

### Branch Naming

```bash
# Feature branches
git checkout -b story/X.Y-brief-description

# Bug fixes
git checkout -b fix/issue-description

# Technical debt
git checkout -b tech/improvement-description

Examples:
- story/2.1-capture-audio-recording
- fix/auth-token-refresh-bug
- tech/add-thought-service-tests
```

### Commit Messages

Follow conventional commits format:

```bash
# Feature
git commit -m "feat(module): Add feature description"

# Bug fix
git commit -m "fix(module): Fix bug description"

# Tests
git commit -m "test(module): Add tests for feature"

# Documentation
git commit -m "docs: Update CONTRIBUTING.md with TDD workflow"

# Refactoring
git commit -m "refactor(module): Improve code structure"

Examples:
git commit -m "test: Add failing test for thought creation (RED)"
git commit -m "feat(knowledge): Implement thought creation service (GREEN)"
git commit -m "refactor(knowledge): Extract validation logic (REFACTOR)"
```

### TDD Commit Pattern

For TDD workflow, use separate commits for each phase:

```bash
# 1. RED phase
git add tests/
git commit -m "test: Add failing test for [feature]"

# 2. GREEN + REFACTOR phase (can be combined)
git add src/ tests/
git commit -m "feat: Implement [feature] with tests passing"
```

---

## 💻 Development Environment

### Prerequisites

- **Node.js**: v20.x (LTS)
- **Docker**: Latest (for backend services)
- **Docker Compose**: Latest
- **Git**: Latest

### Backend Setup

```bash
cd pensieve/backend

# Install dependencies
npm ci

# Start local infrastructure
docker-compose up -d

# Run migrations
npm run migration:run

# Run tests
npm test

# Start development server
npm run start:dev
```

### Mobile Setup

```bash
cd pensieve/mobile

# Install dependencies
npm ci

# Start Expo dev server
npm start

# Run on Android
npm run android

# Run on iOS
npm run ios

# Type check
npx tsc --noEmit
```

### Environment Variables

**Backend** (`.env`):
```env
DATABASE_URL=postgresql://user:pass@localhost:5432/pensine
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_KEY=your-service-key
```

**Mobile** (`.env`):
```env
EXPO_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
EXPO_PUBLIC_API_URL=http://homelab.local:3000
```

---

## 🆘 Getting Help

### Documentation
- [Product Requirements Document](_bmad-output/planning-artifacts/prd.md)
- [Architecture Document](_bmad-output/planning-artifacts/architecture.md)
- [UX Design Specification](_bmad-output/planning-artifacts/ux-design-specification.md)
- [Epics & Stories](_bmad-output/planning-artifacts/epics.md)

### Common Issues

#### "Tests failing with Expo Winter runtime error"
- See [`pensieve/mobile/TESTING.md`](pensieve/mobile/TESTING.md)
- Known issue with Expo SDK 54 + jest-expo
- Focus on backend tests for now

#### "Coverage below 80%"
- Run `npm run test:cov` to see coverage report
- Add tests for uncovered branches/functions
- Focus on edge cases and error handling

#### "TypeScript errors in CI"
- Run `npx tsc --noEmit` locally
- Fix type errors before pushing
- Enable TypeScript strict mode in IDE

---

## 📚 Additional Resources

- [BMAD Methodology](_bmad-output/bmm-workflow-status.yaml)
- [Epic 1 Retrospective](_bmad-output/retrospectives/epic-1-retrospective.md)
- [Epic 1 Action Items](_bmad-output/retrospectives/epic-1-action-items.md)
- [Sprint Status](_bmad-output/implementation-artifacts/sprint-status.yaml)

---

**Happy coding! 🎉**

Remember: **Write tests first, code second. Quality is not negotiable.**

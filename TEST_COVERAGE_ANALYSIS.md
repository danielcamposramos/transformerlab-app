# Test Coverage Analysis & Recommendations
**Transformer Lab App - v0.25.0**
**Generated:** 2025-11-18

---

## Executive Summary

**Current Test Coverage:** ~0.5% (1 superficial test out of 200+ components)

**Critical Findings:**
- ❌ No utility function tests (8+ pure functions untested)
- ❌ No authentication flow tests (complex token refresh logic)
- ❌ No API client tests (SWR hooks, fetchers, error handling)
- ❌ No business logic tests (log parser, data transformations)
- ❌ No component integration tests
- ❌ Missing test infrastructure (mocks, helpers, utilities)

**Recommended Approach:** 4-phase implementation starting with highest-value, lowest-effort areas (utility functions) and progressing to complex integration tests.

---

## Table of Contents

1. [Current State Analysis](#current-state-analysis)
2. [Priority 1: Utility Functions (CRITICAL)](#priority-1-utility-functions-critical)
3. [Priority 2: Business Logic (HIGH)](#priority-2-business-logic-high)
4. [Priority 3: API Client Layer (HIGH)](#priority-3-api-client-layer-high)
5. [Priority 4: Component Tests (MEDIUM)](#priority-4-component-tests-medium)
6. [Testing Infrastructure Setup](#testing-infrastructure-setup)
7. [Implementation Roadmap](#implementation-roadmap)
8. [Metrics & Goals](#metrics--goals)

---

## Current State Analysis

### Existing Tests
```
src/__tests__/App.test.tsx (10 lines)
├── Single render test
└── No assertions on functionality
```

### Jest Configuration ✓
- **Status:** Properly configured in package.json
- **Environment:** jsdom
- **Transform:** ts-jest
- **Module Mapping:** CSS/assets mocked
- **Coverage:** Not currently tracked

### Missing Infrastructure
- [ ] Test utilities (`renderWithProviders`, `mockAuthContext`)
- [ ] API mocking (MSW - Mock Service Worker)
- [ ] Electron API mocks (`window.storage`, `window.platform`)
- [ ] Custom matchers
- [ ] Test fixtures/factories
- [ ] Coverage thresholds

---

## Priority 1: Utility Functions (CRITICAL)

**File:** `src/renderer/lib/utils.ts` (229 lines)
**Complexity:** Low
**Test Value:** ⭐⭐⭐⭐⭐
**Effort:** ⚡ (Low)

### Why Critical?
- Pure functions = easiest to test
- High usage across codebase
- Edge cases exist (NaN, negative numbers, empty arrays)
- Quick wins to establish testing culture

### Functions to Test

#### 1. `formatBytes(bytes: number, decimals?: number)`
**Importance:** Used throughout UI for file sizes
**Edge Cases:**
```typescript
// Test cases needed:
✓ formatBytes(0) === '0 Bytes'
✓ formatBytes(1024) === '1 KiB'
✓ formatBytes(1536, 2) === '1.50 KiB'
✓ formatBytes(1073741824) === '1 GiB'
✓ formatBytes(-100) === '0 Bytes'  // Negative handling
✓ formatBytes(NaN) === '0 Bytes'   // Invalid input
✓ formatBytes(Infinity) === '0 Bytes'
✓ formatBytes(1048576, 0) === '1 MiB'  // Zero decimals
```

#### 2. `filterByFilters(data, searchText, filters)`
**Importance:** Core search/filter logic
**Edge Cases:**
```typescript
// Test cases needed:
✓ Empty data array
✓ Empty search text (return all)
✓ Case-insensitive search
✓ Multiple filters applied
✓ "All" filter value (should not filter)
✓ No matches found
✓ Partial name matches
```

#### 3. `generateFriendlyName()`
**Importance:** Generates experiment names
**Edge Cases:**
```typescript
// Test cases needed:
✓ Returns capitalized format (e.g., "BeautifulPanda")
✓ Always returns valid adjective + animal combo
✓ Different results on multiple calls (randomness)
✓ Name length is reasonable
```

#### 4. `jobChipColor(status: string)`
**Importance:** UI status indicators
**Edge Cases:**
```typescript
// Test cases needed:
✓ 'COMPLETE' → success color
✓ 'QUEUED' → warning color
✓ 'FAILED' → danger color
✓ 'RUNNING' → specific RGB
✓ 'STOPPED' → warning color
✓ Unknown status → neutral color
✓ Case sensitivity handling
```

#### 5. `clamp(n, min, max)`
**Importance:** Number validation
**Edge Cases:**
```typescript
// Test cases needed:
✓ Value within range (unchanged)
✓ Value below min (clamped to min)
✓ Value above max (clamped to max)
✓ Equal to min/max boundaries
```

#### 6. `mixColorWithBackground(color: string, percent?: string)`
**Importance:** Theme color generation
**Edge Cases:**
```typescript
// Test cases needed:
✓ Default percent (50%)
✓ Custom percent
✓ Returns valid color-mix CSS
```

### Recommended Test File Structure
```
src/__tests__/lib/
├── utils.test.ts          (100+ assertions)
└── utils.fixtures.ts      (test data)
```

### Example Test Pattern
```typescript
describe('formatBytes', () => {
  it('should format zero bytes', () => {
    expect(formatBytes(0)).toBe('0 Bytes')
  })

  it('should format kilobytes with default decimals', () => {
    expect(formatBytes(1536)).toBe('1.50 KiB')
  })

  it('should handle negative numbers gracefully', () => {
    expect(formatBytes(-100)).toBe('0 Bytes')
  })

  it('should handle NaN input', () => {
    expect(formatBytes(NaN)).toBe('0 Bytes')
  })
})
```

**Estimated Time:** 2-3 hours
**Expected Coverage:** 100% of utils.ts
**ROI:** Immediate value, establishes testing patterns

---

## Priority 2: Business Logic (HIGH)

### 2.1 Orchestrator Log Parser ⭐⭐⭐⭐⭐

**File:** `src/renderer/lib/orchestrator-log-parser.ts` (157 lines)
**Complexity:** Medium-High
**Test Value:** ⭐⭐⭐⭐⭐
**Effort:** ⚡⚡ (Medium)

**Why High Priority?**
- Complex state machine with multiple transitions
- Parses streaming logs in production
- ANSI code stripping has edge cases
- Failure impacts job progress tracking

**Critical Test Scenarios:**

#### State Transitions
```typescript
describe('OrchestratorLogParser', () => {
  let parser: OrchestratorLogParser

  beforeEach(() => {
    parser = new OrchestratorLogParser()
  })

  describe('SSE format parsing', () => {
    it('should parse data: prefixed log entries', () => {
      const input = 'data: {"log_line":"Instance is up","status":"running"}'
      const state = parser.parseLogData(input)
      expect(state.machineFound).toBe(true)
    })

    it('should handle multiple data: lines', () => {
      const input = `
data: {"log_line":"Instance is up"}
data: {"log_line":"Cluster launched: 192.168.1.1"}
data: {"status":"completed"}
      `.trim()
      const state = parser.parseLogData(input)
      expect(state.machineFound).toBe(true)
      expect(state.ipAllocated).toBe(true)
      expect(state.isCompleted).toBe(true)
    })

    it('should skip malformed JSON gracefully', () => {
      const input = 'data: {invalid json}'
      expect(() => parser.parseLogData(input)).not.toThrow()
    })
  })

  describe('ANSI code stripping', () => {
    it('should remove ANSI escape codes', () => {
      const input = 'data: {"log_line":"\\u001b[32m✓\\u001b[0m Instance is up"}'
      const state = parser.parseLogData(input)
      expect(state.machineFound).toBe(true)
      expect(parser.getLogLines()[0]).not.toContain('\\u001b')
    })
  })

  describe('progress state machine', () => {
    it('should transition through all states in order', () => {
      const logs = [
        'data: {"log_line":"✓ Instance chosen"}',
        'data: {"log_line":"Cluster launched: 10.0.0.1"}',
        'data: {"log_line":"Syncing files..."}',
        'data: {"log_line":"Job deployed"}',
        'data: {"log_line":"Storage mounted"}',
        'data: {"log_line":"SDK initialized"}',
        'data: {"status":"completed"}',
      ]

      logs.forEach(log => parser.parseLogData(log))
      const state = parser.getProgressState()

      expect(state.machineFound).toBe(true)
      expect(state.ipAllocated).toBe(true)
      expect(state.environmentSetup).toBe(true)
      expect(state.jobDeployed).toBe(true)
      expect(state.diskMounted).toBe(true)
      expect(state.sdkInitialized).toBe(true)
      expect(state.isCompleted).toBe(true)
    })

    it('should handle partial completion states', () => {
      parser.parseLogData('data: {"log_line":"Instance is up"}')
      const state = parser.getProgressState()

      expect(state.machineFound).toBe(true)
      expect(state.isCompleted).toBe(false)
    })
  })

  describe('reset functionality', () => {
    it('should reset all state', () => {
      parser.parseLogData('data: {"log_line":"Instance is up"}')
      parser.reset()
      const state = parser.getProgressState()

      expect(state.machineFound).toBe(false)
      expect(parser.getLogLines()).toEqual([])
    })
  })
})
```

**Test Coverage Goals:**
- ✓ SSE format parsing (valid, invalid, multi-line)
- ✓ ANSI code stripping (various escape sequences)
- ✓ All 8 state transitions
- ✓ State immutability (getProgressState returns copy)
- ✓ Edge cases (empty input, Unicode characters)

**Estimated Time:** 3-4 hours
**Expected Coverage:** 95%+ of orchestrator-log-parser.ts

---

## Priority 3: API Client Layer (HIGH)

### 3.1 Authentication Context ⭐⭐⭐⭐⭐

**File:** `src/renderer/lib/authContext.ts` (309 lines)
**Complexity:** High
**Test Value:** ⭐⭐⭐⭐⭐
**Effort:** ⚡⚡⚡ (High - requires mocking)

**Why Critical?**
- Handles token refresh on 401
- Complex async flows
- localStorage integration
- Affects entire application security

**Critical Test Scenarios:**

```typescript
import { renderHook, act, waitFor } from '@testing-library/react'
import { AuthProvider, useAuth, getAccessToken, updateAccessToken } from './authContext'

// Mock localStorage
const localStorageMock = (() => {
  let store = {}
  return {
    getItem: (key) => store[key] || null,
    setItem: (key, value) => { store[key] = value.toString() },
    removeItem: (key) => { delete store[key] },
    clear: () => { store = {} },
  }
})()

Object.defineProperty(window, 'localStorage', { value: localStorageMock })

describe('AuthContext', () => {
  beforeEach(() => {
    localStorageMock.clear()
    global.fetch = jest.fn()
  })

  describe('token management', () => {
    it('should initialize token from localStorage', () => {
      localStorageMock.setItem('access_token', 'test-token')
      expect(getAccessToken()).toBe('test-token')
    })

    it('should update token and notify listeners', () => {
      const listener = jest.fn()
      const unsub = subscribeAuthChange(listener)

      updateAccessToken('new-token')

      expect(getAccessToken()).toBe('new-token')
      expect(localStorage.getItem('access_token')).toBe('new-token')
      expect(listener).toHaveBeenCalled()

      unsub()
    })

    it('should remove token on logout', async () => {
      updateAccessToken('test-token')

      const { result } = renderHook(() => useAuth(), {
        wrapper: ({ children }) => <AuthProvider>{children}</AuthProvider>
      })

      await act(async () => {
        await result.current.logout()
      })

      expect(getAccessToken()).toBeNull()
      expect(localStorage.getItem('access_token')).toBeNull()
    })
  })

  describe('token refresh on 401', () => {
    it('should refresh token on 401 response', async () => {
      // First call returns 401
      // Second call (refresh) returns new token
      // Third call (retry) succeeds
      global.fetch
        .mockResolvedValueOnce({ status: 401, ok: false })  // Initial request
        .mockResolvedValueOnce({                             // Refresh endpoint
          ok: true,
          json: async () => ({ access_token: 'new-token' })
        })
        .mockResolvedValueOnce({ ok: true, json: async () => ({}) })  // Retry

      updateAccessToken('old-token')

      const response = await fetchWithAuth('/api/test')

      expect(global.fetch).toHaveBeenCalledTimes(3)
      expect(getAccessToken()).toBe('new-token')
      expect(response.ok).toBe(true)
    })

    it('should logout if refresh fails', async () => {
      global.fetch
        .mockResolvedValueOnce({ status: 401, ok: false })
        .mockResolvedValueOnce({ ok: false })  // Refresh fails

      updateAccessToken('old-token')

      await expect(fetchWithAuth('/api/test')).rejects.toThrow()
      expect(getAccessToken()).toBeNull()
    })
  })

  describe('login flow', () => {
    it('should successfully login and store token', async () => {
      global.fetch.mockResolvedValueOnce({
        ok: true,
        json: async () => ({ access_token: 'login-token' })
      })

      const { result } = renderHook(() => useAuth(), {
        wrapper: ({ children }) => <AuthProvider>{children}</AuthProvider>
      })

      await act(async () => {
        await result.current.login()
      })

      expect(getAccessToken()).toBe('login-token')
      expect(result.current.isAuthenticated).toBe(true)
    })

    it('should handle login failure', async () => {
      global.fetch.mockResolvedValueOnce({
        ok: false,
        status: 401,
        json: async () => ({ error: 'Invalid credentials' })
      })

      const { result } = renderHook(() => useAuth(), {
        wrapper: ({ children }) => <AuthProvider>{children}</AuthProvider>
      })

      await act(async () => {
        await result.current.login()
      })

      expect(getAccessToken()).toBeNull()
      expect(result.current.isAuthenticated).toBe(false)
    })
  })
})
```

**Test Coverage Goals:**
- ✓ Token initialization from localStorage
- ✓ Token update and listener notifications
- ✓ 401 auto-refresh flow (success & failure)
- ✓ Login/logout flows
- ✓ User data fetching with SWR
- ✓ Multiple concurrent refresh attempts (race condition)

**Estimated Time:** 6-8 hours
**Expected Coverage:** 80%+ of authContext.ts

---

### 3.2 API Hooks & Functions ⭐⭐⭐⭐

**Files:**
- `src/renderer/lib/api-client/hooks.ts` (221 lines)
- `src/renderer/lib/api-client/functions.ts` (496 lines)

**Complexity:** Medium-High
**Test Value:** ⭐⭐⭐⭐
**Effort:** ⚡⚡⚡ (High)

**Critical Hooks to Test:**

#### `useAPI()` - Core data fetching hook
```typescript
describe('useAPI', () => {
  it('should fetch data with authentication', async () => {
    const mockData = [{ id: 1, name: 'Test Model' }]

    global.fetch.mockResolvedValueOnce({
      ok: true,
      json: async () => mockData
    })

    const { result } = renderHook(() =>
      useAPI('models', ['list']),
      { wrapper: SWRConfig }
    )

    await waitFor(() => expect(result.current.isLoading).toBe(false))

    expect(result.current.data).toEqual(mockData)
    expect(result.current.error).toBeUndefined()
  })

  it('should skip fetch when params contain null/undefined', () => {
    const { result } = renderHook(() =>
      useAPI('experiment', ['get'], { id: null })
    )

    expect(result.current.data).toBeUndefined()
    expect(global.fetch).not.toHaveBeenCalled()
  })

  it('should return unauthorized status on 401', async () => {
    global.fetch.mockResolvedValueOnce({
      status: 401,
      ok: false
    })

    const { result } = renderHook(() =>
      useAPI('models', ['list'])
    )

    await waitFor(() => expect(result.current.isLoading).toBe(false))

    expect(result.current.data.status).toBe('unauthorized')
  })

  it('should handle network errors', async () => {
    global.fetch.mockRejectedValueOnce(new Error('Network error'))

    const { result } = renderHook(() =>
      useAPI('models', ['list'])
    )

    await waitFor(() => expect(result.current.error).toBeDefined())
  })
})
```

#### `useModelStatus()` - Polling hook
```typescript
describe('useModelStatus', () => {
  it('should poll every 2 seconds', async () => {
    jest.useFakeTimers()

    global.fetch.mockResolvedValue({
      ok: true,
      json: async () => ({ status: 'running' })
    })

    renderHook(() => useModelStatus())

    expect(global.fetch).toHaveBeenCalledTimes(1)

    jest.advanceTimersByTime(2000)
    await waitFor(() => expect(global.fetch).toHaveBeenCalledTimes(2))

    jest.advanceTimersByTime(2000)
    await waitFor(() => expect(global.fetch).toHaveBeenCalledTimes(3))

    jest.useRealTimers()
  })
})
```

**Functions to Test:**

#### `login()`, `logout()`, `registerUser()`
```typescript
describe('authentication functions', () => {
  describe('login', () => {
    it('should successfully login with valid credentials', async () => {
      global.fetch.mockResolvedValueOnce({
        ok: true,
        json: async () => ({ access_token: 'test-token' })
      })

      const result = await login('test@example.com', 'password')

      expect(result.status).toBe('success')
      expect(await getAccessToken()).toBe('test-token')
    })

    it('should return unauthorized for invalid credentials', async () => {
      global.fetch.mockResolvedValueOnce({
        ok: true,
        json: async () => ({}) // No token
      })

      const result = await login('test@example.com', 'wrong')

      expect(result.status).toBe('unauthorized')
    })

    it('should handle network errors', async () => {
      global.fetch.mockRejectedValueOnce(new Error('Network failed'))

      const result = await login('test@example.com', 'password')

      expect(result.status).toBe('error')
      expect(result.message).toContain('Login exception')
    })
  })
})
```

**Estimated Time:** 8-10 hours
**Expected Coverage:** 75%+ of hooks.ts and functions.ts

---

## Priority 4: Component Tests (MEDIUM)

### 4.1 Critical Interactive Components

#### ChatBubble.tsx ⭐⭐⭐⭐
**File:** `src/renderer/components/Experiment/Interact/ChatBubble.tsx` (282 lines)

**Test Scenarios:**
```typescript
describe('ChatBubble', () => {
  it('should render user message', () => {
    render(<ChatBubble role="user" content="Hello" />)
    expect(screen.getByText('Hello')).toBeInTheDocument()
  })

  it('should render assistant message with markdown', () => {
    render(<ChatBubble role="assistant" content="**Bold** text" />)
    expect(screen.getByText('Bold')).toHaveStyle({ fontWeight: 'bold' })
  })

  it('should render streaming message with cursor', () => {
    render(<ChatBubble role="assistant" content="Typing..." isStreaming />)
    expect(screen.getByText('Typing...')).toBeInTheDocument()
    expect(screen.getByTestId('streaming-cursor')).toBeInTheDocument()
  })

  it('should render code blocks with syntax highlighting', () => {
    const code = '```python\nprint("hello")\n```'
    render(<ChatBubble role="assistant" content={code} />)
    expect(screen.getByText('print("hello")')).toBeInTheDocument()
  })

  it('should show copy button on hover', async () => {
    render(<ChatBubble role="assistant" content="Test message" />)

    const bubble = screen.getByText('Test message').closest('div')
    fireEvent.mouseEnter(bubble)

    await waitFor(() => {
      expect(screen.getByLabelText('Copy')).toBeVisible()
    })
  })
})
```

#### NotificationSystem.tsx ⭐⭐⭐
**File:** `src/renderer/components/Shared/NotificationSystem.tsx`

**Test Scenarios:**
```typescript
describe('NotificationSystem', () => {
  it('should display notification', () => {
    const { showNotification } = renderWithNotificationContext()

    act(() => {
      showNotification('Test message', 'success')
    })

    expect(screen.getByText('Test message')).toBeInTheDocument()
  })

  it('should auto-dismiss after timeout', async () => {
    jest.useFakeTimers()

    const { showNotification } = renderWithNotificationContext()

    act(() => {
      showNotification('Test', 'info', 3000)
    })

    expect(screen.getByText('Test')).toBeInTheDocument()

    jest.advanceTimersByTime(3000)

    await waitFor(() => {
      expect(screen.queryByText('Test')).not.toBeInTheDocument()
    })

    jest.useRealTimers()
  })

  it('should queue multiple notifications', () => {
    const { showNotification } = renderWithNotificationContext()

    act(() => {
      showNotification('First', 'info')
      showNotification('Second', 'warning')
      showNotification('Third', 'error')
    })

    expect(screen.getAllByRole('alert')).toHaveLength(3)
  })
})
```

**Estimated Time:** 12-15 hours for 10 critical components
**Expected Coverage:** 60%+ of critical component files

---

## Testing Infrastructure Setup

### Prerequisites Checklist

#### 1. Install Testing Dependencies ✓
```bash
npm install --save-dev @testing-library/react@14.0.0
npm install --save-dev @testing-library/jest-dom@6.1.5
npm install --save-dev @testing-library/user-event@14.5.1
npm install --save-dev msw@2.0.0  # API mocking
npm install --save-dev jest-environment-jsdom@29.5.0
```

#### 2. Create Test Utilities
**File:** `src/__tests__/test-utils.tsx`
```typescript
import { render, RenderOptions } from '@testing-library/react'
import { ReactElement } from 'react'
import { AuthProvider } from '../renderer/lib/authContext'
import { SWRConfig } from 'swr'

// Mock window.storage for Electron
const mockStorage = {
  get: jest.fn().mockResolvedValue(null),
  set: jest.fn().mockResolvedValue(undefined),
  delete: jest.fn().mockResolvedValue(undefined),
}

Object.defineProperty(window, 'storage', {
  value: mockStorage,
  writable: true,
})

// Mock window.platform
Object.defineProperty(window, 'platform', {
  value: {
    appmode: 'desktop',
    os: 'linux',
  },
  writable: true,
})

interface CustomRenderOptions extends RenderOptions {
  initialAuth?: { token: string | null }
}

function AllTheProviders({ children, initialAuth }: any) {
  return (
    <SWRConfig value={{ dedupingInterval: 0, provider: () => new Map() }}>
      <AuthProvider>
        {children}
      </AuthProvider>
    </SWRConfig>
  )
}

function customRender(
  ui: ReactElement,
  options?: CustomRenderOptions,
) {
  return render(ui, { wrapper: AllTheProviders, ...options })
}

export * from '@testing-library/react'
export { customRender as render }
export { mockStorage }
```

#### 3. Setup MSW (Mock Service Worker)
**File:** `src/__tests__/mocks/handlers.ts`
```typescript
import { http, HttpResponse } from 'msw'

export const handlers = [
  // Models endpoint
  http.get('http://localhost:8338/api/models/list', () => {
    return HttpResponse.json([
      { id: '1', name: 'Test Model', architecture: 'LlamaForCausalLM' },
    ])
  }),

  // Auth endpoints
  http.post('http://localhost:8338/api/auth/login', async ({ request }) => {
    const formData = await request.formData()
    const username = formData.get('username')

    if (username === 'test@example.com') {
      return HttpResponse.json({ access_token: 'mock-token' })
    }
    return new HttpResponse(null, { status: 401 })
  }),

  // Experiments
  http.get('http://localhost:8338/api/experiment/:id', ({ params }) => {
    return HttpResponse.json({
      id: params.id,
      name: 'Test Experiment',
    })
  }),
]
```

**File:** `src/__tests__/mocks/server.ts`
```typescript
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

export const server = setupServer(...handlers)
```

**File:** `src/__tests__/setup.ts`
```typescript
import '@testing-library/jest-dom'
import { server } from './mocks/server'

// Establish API mocking before all tests
beforeAll(() => server.listen({ onUnhandledRequest: 'warn' }))

// Reset handlers after each test
afterEach(() => server.resetHandlers())

// Clean up after tests finish
afterAll(() => server.close())
```

#### 4. Update package.json
```json
{
  "jest": {
    "setupFilesAfterEnv": ["<rootDir>/src/__tests__/setup.ts"],
    "collectCoverageFrom": [
      "src/**/*.{ts,tsx}",
      "!src/**/*.d.ts",
      "!src/__tests__/**",
      "!src/**/index.{ts,tsx}"
    ],
    "coverageThresholds": {
      "global": {
        "statements": 50,
        "branches": 40,
        "functions": 50,
        "lines": 50
      }
    }
  }
}
```

---

## Implementation Roadmap

### Phase 1: Foundation (Week 1)
**Goal:** Establish testing infrastructure and quick wins

- [ ] Install testing dependencies
- [ ] Create test utilities (`test-utils.tsx`, MSW setup)
- [ ] Configure coverage reporting
- [ ] Write utility function tests (utils.ts) - **100% coverage**
- [ ] Document testing patterns

**Deliverable:** 15-20 tests, infrastructure in place

---

### Phase 2: Business Logic (Week 2-3)
**Goal:** Test critical business logic

- [ ] Test orchestrator log parser - **95% coverage**
- [ ] Test authentication context - **80% coverage**
- [ ] Test API client functions (login, logout, register)
- [ ] Test custom hooks (useAPI, useModelStatus)

**Deliverable:** 40-60 tests, critical paths covered

---

### Phase 3: Component Coverage (Week 4-6)
**Goal:** Test user-facing components

**Week 4: Shared Components**
- [ ] NotificationSystem
- [ ] Modal dialogs
- [ ] Form inputs

**Week 5: Experiment Components**
- [ ] ChatBubble
- [ ] JobProgress
- [ ] DatasetTable

**Week 6: Complex Features**
- [ ] Workflow editor basics
- [ ] RAG query interface
- [ ] Model selection

**Deliverable:** 80-120 tests, major components covered

---

### Phase 4: Integration & E2E (Week 7-8)
**Goal:** Test complete user flows

- [ ] Authentication flow (login → token refresh → logout)
- [ ] Model download → experiment creation → chat
- [ ] Dataset upload → processing → training job
- [ ] Error boundary and fallback behavior

**Deliverable:** 20-30 integration tests

---

## Metrics & Goals

### Coverage Targets (6-month horizon)

| Category | Current | 3 Months | 6 Months |
|----------|---------|----------|----------|
| **Overall** | 0.5% | 50% | 70% |
| **Utilities** | 0% | 100% | 100% |
| **Business Logic** | 0% | 80% | 90% |
| **API Client** | 0% | 70% | 85% |
| **Components** | 0% | 40% | 60% |
| **Integration** | 0% | 20% | 40% |

### Success Metrics

**Quantitative:**
- [ ] 200+ total tests
- [ ] < 5% test flakiness
- [ ] < 30s test suite execution time
- [ ] 70%+ code coverage
- [ ] All critical paths covered

**Qualitative:**
- [ ] Developers write tests for new features
- [ ] CI fails on test failures
- [ ] Regression bugs caught by tests
- [ ] Refactoring confidence increased

---

## Quick Start Commands

```bash
# Run all tests
npm test

# Run with coverage
npm test -- --coverage

# Run specific test file
npm test utils.test.ts

# Run in watch mode
npm test -- --watch

# Update snapshots
npm test -- -u

# Run only changed tests
npm test -- --onlyChanged
```

---

## Appendix: File Reference

### High-Priority Test Files to Create

```
src/__tests__/
├── setup.ts                           # Global test setup
├── test-utils.tsx                     # Custom render with providers
├── mocks/
│   ├── handlers.ts                    # MSW request handlers
│   ├── server.ts                      # MSW server setup
│   └── electron.ts                    # Electron API mocks
├── lib/
│   ├── utils.test.ts                  # ⭐⭐⭐⭐⭐ Priority 1
│   ├── orchestrator-log-parser.test.ts # ⭐⭐⭐⭐⭐ Priority 2
│   ├── authContext.test.ts            # ⭐⭐⭐⭐⭐ Priority 3
│   └── api-client/
│       ├── hooks.test.ts              # ⭐⭐⭐⭐ Priority 3
│       └── functions.test.ts          # ⭐⭐⭐⭐ Priority 3
├── components/
│   ├── Shared/
│   │   └── NotificationSystem.test.tsx # ⭐⭐⭐ Priority 4
│   └── Experiment/
│       └── Interact/
│           └── ChatBubble.test.tsx    # ⭐⭐⭐⭐ Priority 4
└── integration/
    ├── auth-flow.test.tsx             # Future
    └── model-interaction.test.tsx     # Future
```

---

## Conclusion

**Immediate Action Items:**
1. ✅ Review this document with team
2. ⚡ Set up testing infrastructure (2-3 hours)
3. ⚡ Write first utility tests (2-3 hours)
4. 📅 Schedule Phase 1 completion (1 week)

**Long-term Vision:**
- Testing culture established
- New features ship with tests
- Regression bugs prevented
- Refactoring confidence high
- CI/CD pipeline robust

---

**Document Version:** 1.0
**Author:** Test Coverage Analysis
**Next Review:** After Phase 1 completion

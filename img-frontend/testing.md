# Testing Guide

## Overview

This document provides comprehensive testing strategies and best practices for the Deed-O Frontend application, covering unit testing, integration testing, end-to-end testing, and performance testing.

## Testing Philosophy

### Testing Pyramid

```
    /\     E2E Tests (Few)
   /  \    - Critical user journeys
  /____\   - Cross-browser compatibility
 /      \  
/________\ Integration Tests (Some)
          - Component interactions
          - API integrations
          - State management

________________ Unit Tests (Many)
                - Pure functions
                - Component logic
                - Utilities
```

### Testing Principles

1. **Test Behavior, Not Implementation**: Focus on what the component does, not how it does it
2. **Write Tests First**: Use TDD/BDD approach for critical features
3. **Keep Tests Simple**: Each test should verify one specific behavior
4. **Use Real User Interactions**: Test how users actually interact with the application
5. **Mock External Dependencies**: Isolate units under test
6. **Maintain Test Quality**: Treat test code with the same care as production code

## Testing Stack

### Core Testing Libraries

```json
{
  "devDependencies": {
    "@testing-library/react": "^13.4.0",
    "@testing-library/jest-dom": "^5.16.5",
    "@testing-library/user-event": "^14.4.3",
    "vitest": "^0.34.0",
    "@vitest/ui": "^0.34.0",
    "jsdom": "^22.1.0",
    "msw": "^1.3.0",
    "playwright": "^1.37.0",
    "@playwright/test": "^1.37.0",
    "@storybook/react": "^7.4.0",
    "@storybook/testing-library": "^0.2.0"
  }
}
```

### Test Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    css: true,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/test/',
        '**/*.d.ts',
        '**/*.config.*',
        '**/coverage/**',
      ],
      thresholds: {
        global: {
          branches: 80,
          functions: 80,
          lines: 80,
          statements: 80,
        },
      },
    },
  },
  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
    },
  },
});
```

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom';
import { cleanup } from '@testing-library/react';
import { afterEach, beforeAll, afterAll } from 'vitest';
import { server } from './mocks/server';

// Setup MSW
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterAll(() => server.close());
afterEach(() => {
  server.resetHandlers();
  cleanup();
});

// Mock IntersectionObserver
global.IntersectionObserver = class IntersectionObserver {
  constructor() {}
  observe() {
    return null;
  }
  disconnect() {
    return null;
  }
  unobserve() {
    return null;
  }
};

// Mock ResizeObserver
global.ResizeObserver = class ResizeObserver {
  constructor() {}
  observe() {
    return null;
  }
  disconnect() {
    return null;
  }
  unobserve() {
    return null;
  }
};

// Mock matchMedia
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: (query: string) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: () => {},
    removeListener: () => {},
    addEventListener: () => {},
    removeEventListener: () => {},
    dispatchEvent: () => {},
  }),
});
```

## Unit Testing

### 1. Component Testing

```typescript
// src/components/ui/__tests__/Button.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { Button } from '../Button';

describe('Button', () => {
  it('renders with correct text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole('button', { name: /click me/i })).toBeInTheDocument();
  });
  
  it('handles click events', async () => {
    const user = userEvent.setup();
    const handleClick = vi.fn();
    
    render(<Button onClick={handleClick}>Click me</Button>);
    
    await user.click(screen.getByRole('button'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });
  
  it('applies variant styles correctly', () => {
    const { rerender } = render(<Button variant="primary">Primary</Button>);
    expect(screen.getByRole('button')).toHaveClass('bg-primary');
    
    rerender(<Button variant="secondary">Secondary</Button>);
    expect(screen.getByRole('button')).toHaveClass('bg-secondary');
  });
  
  it('disables button when loading', () => {
    render(<Button loading>Loading</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
    expect(screen.getByText(/loading/i)).toBeInTheDocument();
  });
  
  it('supports different sizes', () => {
    const { rerender } = render(<Button size="sm">Small</Button>);
    expect(screen.getByRole('button')).toHaveClass('text-sm');
    
    rerender(<Button size="lg">Large</Button>);
    expect(screen.getByRole('button')).toHaveClass('text-lg');
  });
});
```

```typescript
// src/components/forms/__tests__/ProductForm.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ProductForm } from '../ProductForm';
import { productService } from '../../../services/product.service';

// Mock the product service
vi.mock('../../../services/product.service', () => ({
  productService: {
    createProduct: vi.fn(),
    updateProduct: vi.fn(),
  },
}));

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });
  
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('ProductForm', () => {
  const mockProduct = {
    id: '1',
    name: 'Test Product',
    description: 'Test Description',
    price: 99.99,
    category: 'electronics',
  };
  
  it('renders form fields correctly', () => {
    render(<ProductForm />, { wrapper: createWrapper() });
    
    expect(screen.getByLabelText(/product name/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/description/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/price/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/category/i)).toBeInTheDocument();
  });
  
  it('validates required fields', async () => {
    const user = userEvent.setup();
    render(<ProductForm />, { wrapper: createWrapper() });
    
    await user.click(screen.getByRole('button', { name: /save/i }));
    
    await waitFor(() => {
      expect(screen.getByText(/product name is required/i)).toBeInTheDocument();
      expect(screen.getByText(/price is required/i)).toBeInTheDocument();
    });
  });
  
  it('submits form with valid data', async () => {
    const user = userEvent.setup();
    const mockCreate = vi.mocked(productService.createProduct);
    mockCreate.mockResolvedValue(mockProduct);
    
    render(<ProductForm />, { wrapper: createWrapper() });
    
    await user.type(screen.getByLabelText(/product name/i), 'Test Product');
    await user.type(screen.getByLabelText(/description/i), 'Test Description');
    await user.type(screen.getByLabelText(/price/i), '99.99');
    await user.selectOptions(screen.getByLabelText(/category/i), 'electronics');
    
    await user.click(screen.getByRole('button', { name: /save/i }));
    
    await waitFor(() => {
      expect(mockCreate).toHaveBeenCalledWith({
        name: 'Test Product',
        description: 'Test Description',
        price: 99.99,
        category: 'electronics',
      });
    });
  });
  
  it('populates form when editing existing product', () => {
    render(<ProductForm product={mockProduct} />, { wrapper: createWrapper() });
    
    expect(screen.getByDisplayValue('Test Product')).toBeInTheDocument();
    expect(screen.getByDisplayValue('Test Description')).toBeInTheDocument();
    expect(screen.getByDisplayValue('99.99')).toBeInTheDocument();
  });
});
```

### 2. Hook Testing

```typescript
// src/hooks/__tests__/useAuth.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useAuth } from '../useAuth';
import { authService } from '../../services/auth.service';

// Mock auth service
vi.mock('../../services/auth.service', () => ({
  authService: {
    login: vi.fn(),
    logout: vi.fn(),
    getCurrentUser: vi.fn(),
    refreshToken: vi.fn(),
  },
}));

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });
  
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('useAuth', () => {
  beforeEach(() => {
    vi.clearAllMocks();
    localStorage.clear();
  });
  
  it('returns initial auth state', () => {
    const { result } = renderHook(() => useAuth(), {
      wrapper: createWrapper(),
    });
    
    expect(result.current.user).toBeNull();
    expect(result.current.isAuthenticated).toBe(false);
    expect(result.current.isLoading).toBe(true);
  });
  
  it('handles successful login', async () => {
    const mockUser = { id: '1', email: 'test@example.com', role: 'vendor' };
    const mockLogin = vi.mocked(authService.login);
    mockLogin.mockResolvedValue({ user: mockUser, token: 'mock-token' });
    
    const { result } = renderHook(() => useAuth(), {
      wrapper: createWrapper(),
    });
    
    await result.current.login('test@example.com', 'password');
    
    await waitFor(() => {
      expect(result.current.user).toEqual(mockUser);
      expect(result.current.isAuthenticated).toBe(true);
    });
  });
  
  it('handles login failure', async () => {
    const mockLogin = vi.mocked(authService.login);
    mockLogin.mockRejectedValue(new Error('Invalid credentials'));
    
    const { result } = renderHook(() => useAuth(), {
      wrapper: createWrapper(),
    });
    
    await expect(result.current.login('test@example.com', 'wrong-password'))
      .rejects.toThrow('Invalid credentials');
    
    expect(result.current.user).toBeNull();
    expect(result.current.isAuthenticated).toBe(false);
  });
  
  it('handles logout', async () => {
    const mockLogout = vi.mocked(authService.logout);
    mockLogout.mockResolvedValue(undefined);
    
    // Set initial authenticated state
    localStorage.setItem('token', 'mock-token');
    
    const { result } = renderHook(() => useAuth(), {
      wrapper: createWrapper(),
    });
    
    await result.current.logout();
    
    await waitFor(() => {
      expect(result.current.user).toBeNull();
      expect(result.current.isAuthenticated).toBe(false);
    });
  });
});
```

### 3. Utility Function Testing

```typescript
// src/lib/__tests__/utils.test.ts
import { describe, it, expect } from 'vitest';
import { cn, formatCurrency, debounce, validateEmail } from '../utils';

describe('utils', () => {
  describe('cn (className utility)', () => {
    it('combines class names correctly', () => {
      expect(cn('base', 'additional')).toBe('base additional');
      expect(cn('base', undefined, 'additional')).toBe('base additional');
      expect(cn('base', false && 'conditional')).toBe('base');
      expect(cn('base', true && 'conditional')).toBe('base conditional');
    });
  });
  
  describe('formatCurrency', () => {
    it('formats currency correctly', () => {
      expect(formatCurrency(1234.56)).toBe('$1,234.56');
      expect(formatCurrency(0)).toBe('$0.00');
      expect(formatCurrency(1234.56, 'EUR')).toBe('€1,234.56');
    });
    
    it('handles invalid input', () => {
      expect(formatCurrency(NaN)).toBe('$0.00');
      expect(formatCurrency(Infinity)).toBe('$0.00');
    });
  });
  
  describe('debounce', () => {
    it('debounces function calls', async () => {
      let callCount = 0;
      const fn = () => callCount++;
      const debouncedFn = debounce(fn, 100);
      
      debouncedFn();
      debouncedFn();
      debouncedFn();
      
      expect(callCount).toBe(0);
      
      await new Promise(resolve => setTimeout(resolve, 150));
      expect(callCount).toBe(1);
    });
  });
  
  describe('validateEmail', () => {
    it('validates email addresses correctly', () => {
      expect(validateEmail('test@example.com')).toBe(true);
      expect(validateEmail('user.name+tag@domain.co.uk')).toBe(true);
      expect(validateEmail('invalid-email')).toBe(false);
      expect(validateEmail('test@')).toBe(false);
      expect(validateEmail('@example.com')).toBe(false);
    });
  });
});
```

## Integration Testing

### 1. API Integration Tests

```typescript
// src/test/mocks/handlers.ts
import { rest } from 'msw';

const API_BASE_URL = 'http://localhost:3000/api';

export const handlers = [
  // Auth endpoints
  rest.post(`${API_BASE_URL}/auth/login`, (req, res, ctx) => {
    const { email, password } = req.body as any;
    
    if (email === 'test@example.com' && password === 'password') {
      return res(
        ctx.status(200),
        ctx.json({
          user: {
            id: '1',
            email: 'test@example.com',
            role: 'vendor',
            name: 'Test User',
          },
          token: 'mock-jwt-token',
        })
      );
    }
    
    return res(
      ctx.status(401),
      ctx.json({ message: 'Invalid credentials' })
    );
  }),
  
  rest.post(`${API_BASE_URL}/auth/logout`, (req, res, ctx) => {
    return res(ctx.status(200));
  }),
  
  rest.get(`${API_BASE_URL}/auth/me`, (req, res, ctx) => {
    const authHeader = req.headers.get('Authorization');
    
    if (authHeader === 'Bearer mock-jwt-token') {
      return res(
        ctx.status(200),
        ctx.json({
          id: '1',
          email: 'test@example.com',
          role: 'vendor',
          name: 'Test User',
        })
      );
    }
    
    return res(ctx.status(401));
  }),
  
  // Product endpoints
  rest.get(`${API_BASE_URL}/products`, (req, res, ctx) => {
    const page = req.url.searchParams.get('page') || '1';
    const limit = req.url.searchParams.get('limit') || '10';
    
    return res(
      ctx.status(200),
      ctx.json({
        products: [
          {
            id: '1',
            name: 'Test Product 1',
            description: 'Description 1',
            price: 99.99,
            category: 'electronics',
          },
          {
            id: '2',
            name: 'Test Product 2',
            description: 'Description 2',
            price: 149.99,
            category: 'clothing',
          },
        ],
        pagination: {
          page: parseInt(page),
          limit: parseInt(limit),
          total: 2,
          totalPages: 1,
        },
      })
    );
  }),
  
  rest.post(`${API_BASE_URL}/products`, (req, res, ctx) => {
    const product = req.body as any;
    
    return res(
      ctx.status(201),
      ctx.json({
        id: '3',
        ...product,
        createdAt: new Date().toISOString(),
      })
    );
  }),
  
  // Chat endpoints
  rest.get(`${API_BASE_URL}/groups`, (req, res, ctx) => {
    return res(
      ctx.status(200),
      ctx.json([
        {
          id: '1',
          name: 'General Discussion',
          type: 'public',
          memberCount: 25,
        },
        {
          id: '2',
          name: 'Vendor Support',
          type: 'private',
          memberCount: 5,
        },
      ])
    );
  }),
  
  rest.get(`${API_BASE_URL}/groups/:groupId/messages`, (req, res, ctx) => {
    const { groupId } = req.params;
    
    return res(
      ctx.status(200),
      ctx.json([
        {
          id: '1',
          content: 'Hello everyone!',
          sender: {
            id: '1',
            name: 'Test User',
            avatar: 'https://example.com/avatar.jpg',
          },
          timestamp: new Date().toISOString(),
          groupId,
        },
      ])
    );
  }),
];
```

```typescript
// src/test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

### 2. Component Integration Tests

```typescript
// src/pages/__tests__/VendorDashboard.integration.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, beforeEach } from 'vitest';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';
import { VendorDashboard } from '../vendor/VendorDashboard';
import { AuthProvider } from '../../contexts/AuthContext';

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });
  
  return ({ children }: { children: React.ReactNode }) => (
    <BrowserRouter>
      <QueryClientProvider client={queryClient}>
        <AuthProvider>
          {children}
        </AuthProvider>
      </QueryClientProvider>
    </BrowserRouter>
  );
};

describe('VendorDashboard Integration', () => {
  beforeEach(() => {
    // Set up authenticated user
    localStorage.setItem('token', 'mock-jwt-token');
  });
  
  it('loads and displays vendor dashboard data', async () => {
    render(<VendorDashboard />, { wrapper: createWrapper() });
    
    // Check loading state
    expect(screen.getByText(/loading/i)).toBeInTheDocument();
    
    // Wait for data to load
    await waitFor(() => {
      expect(screen.getByText(/vendor dashboard/i)).toBeInTheDocument();
    });
    
    // Check if products are displayed
    expect(screen.getByText('Test Product 1')).toBeInTheDocument();
    expect(screen.getByText('Test Product 2')).toBeInTheDocument();
  });
  
  it('allows creating a new product', async () => {
    const user = userEvent.setup();
    render(<VendorDashboard />, { wrapper: createWrapper() });
    
    await waitFor(() => {
      expect(screen.getByText(/vendor dashboard/i)).toBeInTheDocument();
    });
    
    // Click add product button
    await user.click(screen.getByRole('button', { name: /add product/i }));
    
    // Fill out product form
    await user.type(screen.getByLabelText(/product name/i), 'New Product');
    await user.type(screen.getByLabelText(/description/i), 'New Description');
    await user.type(screen.getByLabelText(/price/i), '199.99');
    
    // Submit form
    await user.click(screen.getByRole('button', { name: /save/i }));
    
    // Check if product was added
    await waitFor(() => {
      expect(screen.getByText('New Product')).toBeInTheDocument();
    });
  });
  
  it('handles navigation between dashboard sections', async () => {
    const user = userEvent.setup();
    render(<VendorDashboard />, { wrapper: createWrapper() });
    
    await waitFor(() => {
      expect(screen.getByText(/vendor dashboard/i)).toBeInTheDocument();
    });
    
    // Navigate to analytics section
    await user.click(screen.getByRole('link', { name: /analytics/i }));
    
    await waitFor(() => {
      expect(screen.getByText(/sales analytics/i)).toBeInTheDocument();
    });
    
    // Navigate to orders section
    await user.click(screen.getByRole('link', { name: /orders/i }));
    
    await waitFor(() => {
      expect(screen.getByText(/order management/i)).toBeInTheDocument();
    });
  });
});
```

## End-to-End Testing

### 1. Playwright Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['json', { outputFile: 'test-results/results.json' }],
  ],
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] },
    },
  ],
  
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
});
```

### 2. E2E Test Examples

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('should login successfully with valid credentials', async ({ page }) => {
    await page.goto('/');
    
    // Navigate to login
    await page.click('text=Login');
    
    // Fill login form
    await page.fill('[data-testid="email-input"]', 'vendor@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    
    // Submit form
    await page.click('[data-testid="login-button"]');
    
    // Check successful login
    await expect(page).toHaveURL('/vendor/dashboard');
    await expect(page.locator('[data-testid="user-menu"]')).toBeVisible();
  });
  
  test('should show error for invalid credentials', async ({ page }) => {
    await page.goto('/login');
    
    await page.fill('[data-testid="email-input"]', 'invalid@example.com');
    await page.fill('[data-testid="password-input"]', 'wrongpassword');
    
    await page.click('[data-testid="login-button"]');
    
    await expect(page.locator('[data-testid="error-message"]')).toContainText(
      'Invalid credentials'
    );
  });
  
  test('should logout successfully', async ({ page }) => {
    // Login first
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'vendor@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');
    
    await expect(page).toHaveURL('/vendor/dashboard');
    
    // Logout
    await page.click('[data-testid="user-menu"]');
    await page.click('text=Logout');
    
    await expect(page).toHaveURL('/');
    await expect(page.locator('text=Login')).toBeVisible();
  });
});
```

```typescript
// e2e/vendor-workflow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Vendor Workflow', () => {
  test.beforeEach(async ({ page }) => {
    // Login as vendor
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'vendor@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');
    await expect(page).toHaveURL('/vendor/dashboard');
  });
  
  test('should create a new product', async ({ page }) => {
    // Navigate to products
    await page.click('text=Products');
    
    // Click add product
    await page.click('[data-testid="add-product-button"]');
    
    // Fill product form
    await page.fill('[data-testid="product-name"]', 'Test Product E2E');
    await page.fill('[data-testid="product-description"]', 'E2E Test Description');
    await page.fill('[data-testid="product-price"]', '299.99');
    await page.selectOption('[data-testid="product-category"]', 'electronics');
    
    // Upload image
    await page.setInputFiles(
      '[data-testid="product-image"]',
      'e2e/fixtures/test-image.jpg'
    );
    
    // Submit form
    await page.click('[data-testid="save-product-button"]');
    
    // Verify product was created
    await expect(page.locator('text=Test Product E2E')).toBeVisible();
    await expect(page.locator('text=$299.99')).toBeVisible();
  });
  
  test('should edit existing product', async ({ page }) => {
    await page.click('text=Products');
    
    // Click edit on first product
    await page.click('[data-testid="product-item"]:first-child [data-testid="edit-button"]');
    
    // Update product name
    await page.fill('[data-testid="product-name"]', 'Updated Product Name');
    
    // Save changes
    await page.click('[data-testid="save-product-button"]');
    
    // Verify update
    await expect(page.locator('text=Updated Product Name')).toBeVisible();
  });
  
  test('should view analytics dashboard', async ({ page }) => {
    await page.click('text=Analytics');
    
    // Check analytics elements
    await expect(page.locator('[data-testid="sales-chart"]')).toBeVisible();
    await expect(page.locator('[data-testid="revenue-metric"]')).toBeVisible();
    await expect(page.locator('[data-testid="orders-metric"]')).toBeVisible();
  });
});
```

```typescript
// e2e/chat.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Chat Functionality', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'vendor@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');
  });
  
  test('should send and receive messages', async ({ page, context }) => {
    // Open chat
    await page.click('text=Chat');
    
    // Select a group
    await page.click('[data-testid="group-item"]:first-child');
    
    // Send a message
    const messageText = `Test message ${Date.now()}`;
    await page.fill('[data-testid="message-input"]', messageText);
    await page.click('[data-testid="send-button"]');
    
    // Verify message appears
    await expect(page.locator(`text=${messageText}`)).toBeVisible();
  });
  
  test('should upload file in chat', async ({ page }) => {
    await page.click('text=Chat');
    await page.click('[data-testid="group-item"]:first-child');
    
    // Upload file
    await page.click('[data-testid="file-upload-button"]');
    await page.setInputFiles(
      '[data-testid="file-input"]',
      'e2e/fixtures/test-document.pdf'
    );
    
    // Verify file upload
    await expect(page.locator('[data-testid="file-message"]')).toBeVisible();
  });
});
```

## Performance Testing

### 1. Lighthouse Integration

```typescript
// e2e/performance.spec.ts
import { test, expect } from '@playwright/test';
import { playAudit } from 'playwright-lighthouse';

test.describe('Performance Tests', () => {
  test('should meet performance benchmarks on homepage', async ({ page }) => {
    await page.goto('/');
    
    await playAudit({
      page,
      thresholds: {
        performance: 90,
        accessibility: 95,
        'best-practices': 90,
        seo: 85,
      },
      port: 9222,
    });
  });
  
  test('should load vendor dashboard quickly', async ({ page }) => {
    // Login first
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'vendor@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');
    
    // Measure dashboard load time
    const startTime = Date.now();
    await page.goto('/vendor/dashboard');
    await page.waitForSelector('[data-testid="dashboard-content"]');
    const loadTime = Date.now() - startTime;
    
    expect(loadTime).toBeLessThan(3000); // Should load within 3 seconds
  });
});
```

### 2. Load Testing

```typescript
// scripts/loadTest.ts
import { chromium } from 'playwright';

interface LoadTestConfig {
  url: string;
  concurrentUsers: number;
  duration: number; // in seconds
  rampUpTime: number; // in seconds
}

class LoadTester {
  private config: LoadTestConfig;
  private results: Array<{ timestamp: number; responseTime: number; success: boolean }> = [];
  
  constructor(config: LoadTestConfig) {
    this.config = config;
  }
  
  async run(): Promise<void> {
    console.log(`Starting load test with ${this.config.concurrentUsers} users`);
    
    const browser = await chromium.launch();
    const promises: Promise<void>[] = [];
    
    // Ramp up users gradually
    const rampUpInterval = (this.config.rampUpTime * 1000) / this.config.concurrentUsers;
    
    for (let i = 0; i < this.config.concurrentUsers; i++) {
      setTimeout(() => {
        promises.push(this.simulateUser(browser, i));
      }, i * rampUpInterval);
    }
    
    // Wait for test duration
    await new Promise(resolve => setTimeout(resolve, this.config.duration * 1000));
    
    await browser.close();
    
    this.generateReport();
  }
  
  private async simulateUser(browser: any, userId: number): Promise<void> {
    const context = await browser.newContext();
    const page = await context.newPage();
    
    const endTime = Date.now() + (this.config.duration * 1000);
    
    while (Date.now() < endTime) {
      try {
        const startTime = Date.now();
        
        // Simulate user journey
        await page.goto(this.config.url);
        await page.waitForLoadState('networkidle');
        
        // Random user actions
        await this.performRandomActions(page);
        
        const responseTime = Date.now() - startTime;
        
        this.results.push({
          timestamp: Date.now(),
          responseTime,
          success: true,
        });
        
        // Wait between requests
        await page.waitForTimeout(Math.random() * 2000 + 1000);
        
      } catch (error) {
        this.results.push({
          timestamp: Date.now(),
          responseTime: 0,
          success: false,
        });
      }
    }
    
    await context.close();
  }
  
  private async performRandomActions(page: any): Promise<void> {
    const actions = [
      () => page.click('text=Products'),
      () => page.click('text=About'),
      () => page.click('text=Contact'),
      () => page.fill('[data-testid="search-input"]', 'test'),
    ];
    
    const randomAction = actions[Math.floor(Math.random() * actions.length)];
    try {
      await randomAction();
    } catch (error) {
      // Ignore action errors
    }
  }
  
  private generateReport(): void {
    const successfulRequests = this.results.filter(r => r.success);
    const failedRequests = this.results.filter(r => !r.success);
    
    const responseTimes = successfulRequests.map(r => r.responseTime);
    const avgResponseTime = responseTimes.reduce((a, b) => a + b, 0) / responseTimes.length;
    const p95ResponseTime = responseTimes.sort((a, b) => a - b)[Math.floor(responseTimes.length * 0.95)];
    
    console.log('\n=== Load Test Results ===');
    console.log(`Total Requests: ${this.results.length}`);
    console.log(`Successful Requests: ${successfulRequests.length}`);
    console.log(`Failed Requests: ${failedRequests.length}`);
    console.log(`Success Rate: ${(successfulRequests.length / this.results.length * 100).toFixed(2)}%`);
    console.log(`Average Response Time: ${avgResponseTime.toFixed(2)}ms`);
    console.log(`95th Percentile Response Time: ${p95ResponseTime}ms`);
  }
}

// Run load test
const loadTest = new LoadTester({
  url: 'http://localhost:5173',
  concurrentUsers: 10,
  duration: 60,
  rampUpTime: 10,
});

loadTest.run().catch(console.error);
```

## Visual Regression Testing

### 1. Storybook Integration

```typescript
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/react-vite';

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(js|jsx|ts|tsx|mdx)'],
  addons: [
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
    '@storybook/addon-a11y',
    '@storybook/addon-viewport',
  ],
  framework: {
    name: '@storybook/react-vite',
    options: {},
  },
  features: {
    interactionsDebugger: true,
  },
};

export default config;
```

```typescript
// src/components/ui/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'UI/Button',
  component: Button,
  parameters: {
    layout: 'centered',
  },
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: { type: 'select' },
      options: ['primary', 'secondary', 'outline', 'ghost'],
    },
    size: {
      control: { type: 'select' },
      options: ['sm', 'md', 'lg'],
    },
  },
};

export default meta;
type Story = StoryObj<typeof meta>;

export const Primary: Story = {
  args: {
    variant: 'primary',
    children: 'Primary Button',
  },
};

export const Secondary: Story = {
  args: {
    variant: 'secondary',
    children: 'Secondary Button',
  },
};

export const Loading: Story = {
  args: {
    loading: true,
    children: 'Loading Button',
  },
};

export const Disabled: Story = {
  args: {
    disabled: true,
    children: 'Disabled Button',
  },
};
```

### 2. Visual Testing with Playwright

```typescript
// e2e/visual.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Visual Regression Tests', () => {
  test('homepage should match visual baseline', async ({ page }) => {
    await page.goto('/');
    await page.waitForLoadState('networkidle');
    
    await expect(page).toHaveScreenshot('homepage.png');
  });
  
  test('vendor dashboard should match visual baseline', async ({ page }) => {
    // Login first
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'vendor@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');
    
    await page.waitForLoadState('networkidle');
    
    await expect(page).toHaveScreenshot('vendor-dashboard.png');
  });
  
  test('product form should match visual baseline', async ({ page }) => {
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'vendor@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.click('[data-testid="login-button"]');
    
    await page.click('text=Products');
    await page.click('[data-testid="add-product-button"]');
    
    await expect(page.locator('[data-testid="product-form"]')).toHaveScreenshot('product-form.png');
  });
});
```

## Test Scripts

```json
{
  "scripts": {
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest --coverage",
    "test:watch": "vitest --watch",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:debug": "playwright test --debug",
    "test:visual": "playwright test e2e/visual.spec.ts",
    "test:visual:update": "playwright test e2e/visual.spec.ts --update-snapshots",
    "test:load": "ts-node scripts/loadTest.ts",
    "test:all": "npm run test && npm run test:e2e",
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build"
  }
}
```

## CI/CD Integration

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      - run: npm run test:coverage
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
  
  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run build
      - run: npm run test:e2e
      
      - uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
  
  visual-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run build
      - run: npm run test:visual
```

## Testing Best Practices

### 1. Test Organization

- **Co-locate tests**: Keep test files close to the code they test
- **Use descriptive names**: Test names should clearly describe what is being tested
- **Group related tests**: Use `describe` blocks to organize related tests
- **Follow AAA pattern**: Arrange, Act, Assert

### 2. Test Data Management

- **Use factories**: Create test data factories for consistent test objects
- **Avoid hardcoded values**: Use variables and constants for test data
- **Clean up after tests**: Ensure tests don't affect each other
- **Use realistic data**: Test data should resemble production data

### 3. Mocking Strategy

- **Mock external dependencies**: Don't test third-party libraries
- **Use MSW for API mocking**: Intercept network requests at the network level
- **Mock at the right level**: Mock at the boundary of your system
- **Keep mocks simple**: Don't over-engineer mock implementations

### 4. Performance Considerations

- **Run tests in parallel**: Use parallel execution for faster feedback
- **Optimize test setup**: Minimize expensive setup operations
- **Use test databases**: Separate test data from production
- **Profile slow tests**: Identify and optimize slow-running tests

## Next Steps

For more detailed information:
- [Performance Guide](./performance.md) - Performance testing strategies
- [Deployment Guide](./deployment.md) - Testing in CI/CD pipelines
- [Architecture Guide](./architecture.md) - Testable architecture patterns
- [Components Guide](./components.md) - Component testing patterns
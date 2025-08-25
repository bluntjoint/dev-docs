# Authentication & Security

## Overview

This document provides a comprehensive guide to the authentication and security implementation in the Deed-O Frontend application, covering authentication flows, security measures, and best practices.

## Authentication Architecture

### Technology Stack

- **JWT (JSON Web Tokens)**: Stateless authentication
- **SSO Integration**: Third-party authentication provider
- **Role-Based Access Control (RBAC)**: Multi-role authorization
- **Secure Storage**: Token management and persistence
- **HTTPS**: Encrypted data transmission

### Authentication Flow

```
Authentication Flow
├── Login Methods
│   ├── Email/Password Login
│   ├── SSO Authentication
│   └── Social Login (Future)
├── Token Management
│   ├── JWT Token Storage
│   ├── Token Refresh
│   ├── Token Validation
│   └── Automatic Logout
├── Role-Based Access
│   ├── Vendor Access
│   ├── Moderator Access
│   ├── Customer Access
│   └── Admin Access (Future)
└── Security Measures
    ├── CSRF Protection
    ├── XSS Prevention
    ├── Input Validation
    └── Secure Headers
```

## Authentication Implementation

### 1. Authentication Service

```typescript
// src/services/auth.service.ts
import axios from 'axios';
import { useAuthStore } from '../stores/auth.store';

interface LoginCredentials {
  email: string;
  password: string;
}

interface RegisterData {
  firstName: string;
  lastName: string;
  email: string;
  password: string;
  role: 'vendor' | 'customer';
  acceptTerms: boolean;
}

interface AuthResponse {
  user: User;
  token: string;
  refreshToken: string;
  expiresIn: number;
}

class AuthService {
  private baseURL = process.env.VITE_API_URL;
  
  // Email/Password Login
  async login(credentials: LoginCredentials): Promise<AuthResponse> {
    try {
      const response = await axios.post(`${this.baseURL}/auth/login`, credentials);
      
      const { user, token, refreshToken, expiresIn } = response.data;
      
      // Store authentication data
      this.setAuthData({ user, token, refreshToken, expiresIn });
      
      return response.data;
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }
  
  // User Registration
  async register(data: RegisterData): Promise<AuthResponse> {
    try {
      const response = await axios.post(`${this.baseURL}/auth/register`, data);
      
      const { user, token, refreshToken, expiresIn } = response.data;
      
      // Store authentication data
      this.setAuthData({ user, token, refreshToken, expiresIn });
      
      return response.data;
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }
  
  // SSO Authentication
  async ssoLogin(provider: string): Promise<string> {
    try {
      const response = await axios.get(`${this.baseURL}/auth/sso/${provider}`);
      return response.data.redirectUrl;
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }
  
  // Handle SSO Callback
  async handleSSOCallback(code: string, state: string): Promise<AuthResponse> {
    try {
      const response = await axios.post(`${this.baseURL}/auth/sso/callback`, {
        code,
        state,
      });
      
      const { user, token, refreshToken, expiresIn } = response.data;
      
      // Store authentication data
      this.setAuthData({ user, token, refreshToken, expiresIn });
      
      return response.data;
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }
  
  // Token Refresh
  async refreshToken(): Promise<AuthResponse> {
    try {
      const refreshToken = this.getRefreshToken();
      
      if (!refreshToken) {
        throw new Error('No refresh token available');
      }
      
      const response = await axios.post(`${this.baseURL}/auth/refresh`, {
        refreshToken,
      });
      
      const { user, token, refreshToken: newRefreshToken, expiresIn } = response.data;
      
      // Update stored authentication data
      this.setAuthData({ user, token, refreshToken: newRefreshToken, expiresIn });
      
      return response.data;
    } catch (error) {
      // If refresh fails, logout user
      this.logout();
      throw error;
    }
  }
  
  // Logout
  async logout(): Promise<void> {
    try {
      const token = this.getToken();
      
      if (token) {
        await axios.post(`${this.baseURL}/auth/logout`, {}, {
          headers: { Authorization: `Bearer ${token}` }
        });
      }
    } catch (error) {
      console.error('Logout error:', error);
    } finally {
      // Clear local authentication data
      this.clearAuthData();
    }
  }
  
  // Forgot Password
  async forgotPassword(email: string): Promise<void> {
    try {
      await axios.post(`${this.baseURL}/auth/forgot-password`, { email });
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }
  
  // Reset Password
  async resetPassword(token: string, newPassword: string): Promise<void> {
    try {
      await axios.post(`${this.baseURL}/auth/reset-password`, {
        token,
        password: newPassword,
      });
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }
  
  // Verify Email
  async verifyEmail(token: string): Promise<void> {
    try {
      await axios.post(`${this.baseURL}/auth/verify-email`, { token });
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }
  
  // Get Current User
  async getCurrentUser(): Promise<User> {
    try {
      const response = await axios.get(`${this.baseURL}/auth/me`);
      return response.data.user;
    } catch (error) {
      this.handleAuthError(error);
      throw error;
    }
  }
  
  // Private Methods
  private setAuthData({ user, token, refreshToken, expiresIn }: {
    user: User;
    token: string;
    refreshToken: string;
    expiresIn: number;
  }): void {
    // Store in Zustand store
    useAuthStore.getState().login(user, token);
    
    // Store refresh token securely
    localStorage.setItem('refreshToken', refreshToken);
    
    // Set token expiration
    const expirationTime = Date.now() + (expiresIn * 1000);
    localStorage.setItem('tokenExpiration', expirationTime.toString());
    
    // Setup automatic token refresh
    this.setupTokenRefresh(expiresIn);
  }
  
  private clearAuthData(): void {
    // Clear Zustand store
    useAuthStore.getState().logout();
    
    // Clear local storage
    localStorage.removeItem('refreshToken');
    localStorage.removeItem('tokenExpiration');
    
    // Clear any refresh timers
    this.clearTokenRefresh();
  }
  
  private getToken(): string | null {
    return useAuthStore.getState().token;
  }
  
  private getRefreshToken(): string | null {
    return localStorage.getItem('refreshToken');
  }
  
  private setupTokenRefresh(expiresIn: number): void {
    // Refresh token 5 minutes before expiration
    const refreshTime = (expiresIn - 300) * 1000;
    
    if (refreshTime > 0) {
      setTimeout(() => {
        this.refreshToken().catch(console.error);
      }, refreshTime);
    }
  }
  
  private clearTokenRefresh(): void {
    // Clear any existing refresh timers
    // Implementation depends on how timers are stored
  }
  
  private handleAuthError(error: any): void {
    if (error.response?.status === 401) {
      // Unauthorized - clear auth data
      this.clearAuthData();
    }
    
    console.error('Authentication error:', error);
  }
}

export const authService = new AuthService();
```

### 2. Authentication Hooks

```typescript
// src/hooks/useAuth.ts
import { useEffect } from 'react';
import { useNavigate, useLocation } from 'react-router-dom';
import { useAuthStore } from '../stores/auth.store';
import { authService } from '../services/auth.service';

export const useAuth = () => {
  const {
    user,
    token,
    isAuthenticated,
    isLoading,
    login,
    logout,
    updateUser,
    setLoading,
  } = useAuthStore();
  
  const navigate = useNavigate();
  const location = useLocation();
  
  // Initialize authentication state
  useEffect(() => {
    const initializeAuth = async () => {
      setLoading(true);
      
      try {
        // Check if user is already authenticated
        if (token && !user) {
          // Fetch current user data
          const currentUser = await authService.getCurrentUser();
          updateUser(currentUser);
        }
        
        // Check token expiration
        const tokenExpiration = localStorage.getItem('tokenExpiration');
        if (tokenExpiration && Date.now() > parseInt(tokenExpiration)) {
          // Token expired, try to refresh
          await authService.refreshToken();
        }
      } catch (error) {
        console.error('Auth initialization error:', error);
        // Clear invalid auth data
        logout();
      } finally {
        setLoading(false);
      }
    };
    
    initializeAuth();
  }, [token, user, updateUser, logout, setLoading]);
  
  // Login function
  const handleLogin = async (credentials: LoginCredentials) => {
    setLoading(true);
    
    try {
      const response = await authService.login(credentials);
      
      // Redirect to intended page or dashboard
      const from = location.state?.from || getDashboardRoute(response.user.role);
      navigate(from, { replace: true });
      
      return response;
    } catch (error) {
      throw error;
    } finally {
      setLoading(false);
    }
  };
  
  // Register function
  const handleRegister = async (data: RegisterData) => {
    setLoading(true);
    
    try {
      const response = await authService.register(data);
      
      // Redirect to dashboard
      const dashboardRoute = getDashboardRoute(response.user.role);
      navigate(dashboardRoute, { replace: true });
      
      return response;
    } catch (error) {
      throw error;
    } finally {
      setLoading(false);
    }
  };
  
  // SSO login function
  const handleSSOLogin = async (provider: string) => {
    try {
      const redirectUrl = await authService.ssoLogin(provider);
      window.location.href = redirectUrl;
    } catch (error) {
      throw error;
    }
  };
  
  // Logout function
  const handleLogout = async () => {
    setLoading(true);
    
    try {
      await authService.logout();
      navigate('/login', { replace: true });
    } catch (error) {
      console.error('Logout error:', error);
    } finally {
      setLoading(false);
    }
  };
  
  // Helper function to get dashboard route based on role
  const getDashboardRoute = (role: string) => {
    switch (role) {
      case 'vendor':
        return '/vendor';
      case 'moderator':
        return '/moderator';
      case 'admin':
        return '/admin';
      default:
        return '/';
    }
  };
  
  return {
    user,
    token,
    isAuthenticated,
    isLoading,
    login: handleLogin,
    register: handleRegister,
    ssoLogin: handleSSOLogin,
    logout: handleLogout,
    updateUser,
  };
};

// Role-based hooks
export const useRequireAuth = (requiredRole?: string) => {
  const { isAuthenticated, user, isLoading } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();
  
  useEffect(() => {
    if (!isLoading) {
      if (!isAuthenticated) {
        // Redirect to login with return URL
        navigate('/login', {
          state: { from: location.pathname },
          replace: true,
        });
      } else if (requiredRole && user?.role !== requiredRole) {
        // Redirect to unauthorized page or home
        navigate('/', { replace: true });
      }
    }
  }, [isAuthenticated, user, isLoading, requiredRole, navigate, location]);
  
  return { isAuthenticated, user, isLoading };
};

export const useRequireRole = (allowedRoles: string[]) => {
  const { user } = useAuth();
  
  const hasRole = user && allowedRoles.includes(user.role);
  
  return { hasRole, userRole: user?.role };
};
```

### 3. Authentication Components

```typescript
// src/components/auth/LoginForm.tsx
import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useAuth } from '../../hooks/useAuth';
import { Button } from '../ui/button';
import { Input } from '../ui/input';
import { Alert } from '../ui/alert';

const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(6, 'Password must be at least 6 characters'),
  rememberMe: z.boolean().default(false),
});

type LoginFormData = z.infer<typeof loginSchema>;

export const LoginForm = () => {
  const [error, setError] = useState<string | null>(null);
  const { login, ssoLogin, isLoading } = useAuth();
  
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
  });
  
  const onSubmit = async (data: LoginFormData) => {
    setError(null);
    
    try {
      await login({
        email: data.email,
        password: data.password,
      });
    } catch (err: any) {
      setError(
        err.response?.data?.message || 
        'Login failed. Please check your credentials.'
      );
    }
  };
  
  const handleSSOLogin = async (provider: string) => {
    setError(null);
    
    try {
      await ssoLogin(provider);
    } catch (err: any) {
      setError(
        err.response?.data?.message || 
        'SSO login failed. Please try again.'
      );
    }
  };
  
  return (
    <div className="max-w-md mx-auto">
      <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
        <div>
          <h2 className="text-2xl font-bold text-center">Sign In</h2>
          <p className="text-center text-gray-600 mt-2">
            Welcome back to Deed-O
          </p>
        </div>
        
        {error && (
          <Alert variant="destructive">
            {error}
          </Alert>
        )}
        
        <div>
          <label htmlFor="email" className="block text-sm font-medium">
            Email Address
          </label>
          <Input
            {...register('email')}
            type="email"
            id="email"
            placeholder="Enter your email"
            error={errors.email?.message}
          />
        </div>
        
        <div>
          <label htmlFor="password" className="block text-sm font-medium">
            Password
          </label>
          <Input
            {...register('password')}
            type="password"
            id="password"
            placeholder="Enter your password"
            error={errors.password?.message}
          />
        </div>
        
        <div className="flex items-center justify-between">
          <label className="flex items-center">
            <input
              {...register('rememberMe')}
              type="checkbox"
              className="rounded border-gray-300"
            />
            <span className="ml-2 text-sm">Remember me</span>
          </label>
          
          <a 
            href="/forgot-password" 
            className="text-sm text-blue-600 hover:underline"
          >
            Forgot password?
          </a>
        </div>
        
        <Button
          type="submit"
          className="w-full"
          disabled={isSubmitting || isLoading}
        >
          {isSubmitting || isLoading ? 'Signing in...' : 'Sign In'}
        </Button>
        
        <div className="relative">
          <div className="absolute inset-0 flex items-center">
            <div className="w-full border-t border-gray-300" />
          </div>
          <div className="relative flex justify-center text-sm">
            <span className="px-2 bg-white text-gray-500">Or continue with</span>
          </div>
        </div>
        
        <Button
          type="button"
          variant="outline"
          className="w-full"
          onClick={() => handleSSOLogin('google')}
          disabled={isLoading}
        >
          <GoogleIcon className="w-5 h-5 mr-2" />
          Sign in with Google
        </Button>
        
        <p className="text-center text-sm">
          Don't have an account?{' '}
          <a href="/register" className="text-blue-600 hover:underline">
            Sign up
          </a>
        </p>
      </form>
    </div>
  );
};
```

### 4. Protected Route Components

```typescript
// src/components/auth/ProtectedRoute.tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '../../hooks/useAuth';
import { Spinner } from '../ui/spinner';

interface ProtectedRouteProps {
  children: React.ReactNode;
  requiredRole?: string;
  allowedRoles?: string[];
  fallback?: React.ReactNode;
}

export const ProtectedRoute = ({
  children,
  requiredRole,
  allowedRoles,
  fallback,
}: ProtectedRouteProps) => {
  const { isAuthenticated, user, isLoading } = useAuth();
  const location = useLocation();
  
  // Show loading spinner while checking authentication
  if (isLoading) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <Spinner size="lg" />
      </div>
    );
  }
  
  // Redirect to login if not authenticated
  if (!isAuthenticated) {
    return (
      <Navigate 
        to="/login" 
        state={{ from: location.pathname }}
        replace 
      />
    );
  }
  
  // Check role-based access
  if (requiredRole && user?.role !== requiredRole) {
    return fallback || <Navigate to="/" replace />;
  }
  
  if (allowedRoles && !allowedRoles.includes(user?.role || '')) {
    return fallback || <Navigate to="/" replace />;
  }
  
  return <>{children}</>;
};

// Role-specific route components
export const VendorRoute = ({ children }: { children: React.ReactNode }) => (
  <ProtectedRoute requiredRole="vendor">
    {children}
  </ProtectedRoute>
);

export const ModeratorRoute = ({ children }: { children: React.ReactNode }) => (
  <ProtectedRoute requiredRole="moderator">
    {children}
  </ProtectedRoute>
);

export const AdminRoute = ({ children }: { children: React.ReactNode }) => (
  <ProtectedRoute allowedRoles={['admin', 'moderator']}>
    {children}
  </ProtectedRoute>
);
```

### 5. SSO Callback Handler

```typescript
// src/components/auth/SSOCallback.tsx
import { useEffect, useState } from 'react';
import { useNavigate, useSearchParams } from 'react-router-dom';
import { authService } from '../../services/auth.service';
import { Spinner } from '../ui/spinner';
import { Alert } from '../ui/alert';

export const SSOCallback = () => {
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [searchParams] = useSearchParams();
  const navigate = useNavigate();
  
  useEffect(() => {
    const handleCallback = async () => {
      try {
        const code = searchParams.get('code');
        const state = searchParams.get('state');
        const error = searchParams.get('error');
        
        if (error) {
          throw new Error(searchParams.get('error_description') || 'SSO authentication failed');
        }
        
        if (!code || !state) {
          throw new Error('Invalid SSO callback parameters');
        }
        
        // Handle SSO callback
        const response = await authService.handleSSOCallback(code, state);
        
        // Redirect to appropriate dashboard
        const dashboardRoute = getDashboardRoute(response.user.role);
        navigate(dashboardRoute, { replace: true });
        
      } catch (err: any) {
        console.error('SSO callback error:', err);
        setError(err.message || 'Authentication failed');
      } finally {
        setIsLoading(false);
      }
    };
    
    handleCallback();
  }, [searchParams, navigate]);
  
  const getDashboardRoute = (role: string) => {
    switch (role) {
      case 'vendor':
        return '/vendor';
      case 'moderator':
        return '/moderator';
      default:
        return '/';
    }
  };
  
  if (isLoading) {
    return (
      <div className="flex flex-col items-center justify-center min-h-screen">
        <Spinner size="lg" />
        <p className="mt-4 text-gray-600">Completing authentication...</p>
      </div>
    );
  }
  
  if (error) {
    return (
      <div className="flex flex-col items-center justify-center min-h-screen max-w-md mx-auto">
        <Alert variant="destructive" className="mb-4">
          <h3 className="font-semibold">Authentication Failed</h3>
          <p>{error}</p>
        </Alert>
        
        <button
          onClick={() => navigate('/login')}
          className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700"
        >
          Return to Login
        </button>
      </div>
    );
  }
  
  return null;
};
```

## Security Implementation

### 1. HTTP Client Security

```typescript
// src/lib/api.ts
import axios, { AxiosRequestConfig, AxiosResponse } from 'axios';
import { useAuthStore } from '../stores/auth.store';
import { authService } from '../services/auth.service';

// Create axios instance with security defaults
const api = axios.create({
  baseURL: process.env.VITE_API_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
    'X-Requested-With': 'XMLHttpRequest',
  },
  withCredentials: true, // Include cookies for CSRF protection
});

// Request interceptor for authentication
api.interceptors.request.use(
  (config: AxiosRequestConfig) => {
    // Add authentication token
    const token = useAuthStore.getState().token;
    if (token) {
      config.headers = {
        ...config.headers,
        Authorization: `Bearer ${token}`,
      };
    }
    
    // Add CSRF token if available
    const csrfToken = document.querySelector('meta[name="csrf-token"]')?.getAttribute('content');
    if (csrfToken) {
      config.headers = {
        ...config.headers,
        'X-CSRF-Token': csrfToken,
      };
    }
    
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// Response interceptor for error handling
api.interceptors.response.use(
  (response: AxiosResponse) => {
    return response;
  },
  async (error) => {
    const originalRequest = error.config;
    
    // Handle 401 Unauthorized
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      
      try {
        // Attempt to refresh token
        await authService.refreshToken();
        
        // Retry original request with new token
        const token = useAuthStore.getState().token;
        if (token) {
          originalRequest.headers.Authorization = `Bearer ${token}`;
          return api(originalRequest);
        }
      } catch (refreshError) {
        // Refresh failed, logout user
        useAuthStore.getState().logout();
        window.location.href = '/login';
      }
    }
    
    // Handle 403 Forbidden
    if (error.response?.status === 403) {
      console.error('Access forbidden:', error.response.data);
      // Optionally redirect to unauthorized page
    }
    
    // Handle 429 Too Many Requests
    if (error.response?.status === 429) {
      console.error('Rate limit exceeded:', error.response.data);
      // Implement exponential backoff retry
    }
    
    return Promise.reject(error);
  }
);

export default api;
```

### 2. Input Validation & Sanitization

```typescript
// src/lib/validation.ts
import { z } from 'zod';
import DOMPurify from 'dompurify';

// Common validation schemas
export const emailSchema = z.string()
  .email('Invalid email address')
  .max(255, 'Email too long');

export const passwordSchema = z.string()
  .min(8, 'Password must be at least 8 characters')
  .max(128, 'Password too long')
  .regex(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/, 
    'Password must contain uppercase, lowercase, number, and special character');

export const nameSchema = z.string()
  .min(1, 'Name is required')
  .max(100, 'Name too long')
  .regex(/^[a-zA-Z\s'-]+$/, 'Name contains invalid characters');

// Sanitization functions
export const sanitizeHtml = (input: string): string => {
  return DOMPurify.sanitize(input, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p', 'br'],
    ALLOWED_ATTR: [],
  });
};

export const sanitizeText = (input: string): string => {
  return input
    .trim()
    .replace(/[<>"'&]/g, (char) => {
      const entities: Record<string, string> = {
        '<': '&lt;',
        '>': '&gt;',
        '"': '&quot;',
        "'": '&#x27;',
        '&': '&amp;',
      };
      return entities[char] || char;
    });
};

// File validation
export const validateFile = (file: File, options: {
  maxSize: number;
  allowedTypes: string[];
}) => {
  const errors: string[] = [];
  
  if (file.size > options.maxSize) {
    errors.push(`File size must be less than ${options.maxSize / 1024 / 1024}MB`);
  }
  
  if (!options.allowedTypes.includes(file.type)) {
    errors.push(`File type ${file.type} is not allowed`);
  }
  
  return {
    isValid: errors.length === 0,
    errors,
  };
};

// URL validation
export const validateUrl = (url: string): boolean => {
  try {
    const urlObj = new URL(url);
    return ['http:', 'https:'].includes(urlObj.protocol);
  } catch {
    return false;
  }
};
```

### 3. Content Security Policy

```typescript
// src/lib/security.ts

// CSP configuration
export const cspConfig = {
  'default-src': ["'self'"],
  'script-src': [
    "'self'",
    "'unsafe-inline'", // Only for development
    'https://apis.google.com',
    'https://www.google-analytics.com',
  ],
  'style-src': [
    "'self'",
    "'unsafe-inline'",
    'https://fonts.googleapis.com',
  ],
  'img-src': [
    "'self'",
    'data:',
    'https:',
    'blob:',
  ],
  'font-src': [
    "'self'",
    'https://fonts.gstatic.com',
  ],
  'connect-src': [
    "'self'",
    process.env.VITE_API_URL,
    'wss://localhost:*', // WebSocket for development
  ],
  'frame-src': [
    'https://www.google.com',
  ],
  'object-src': ["'none'"],
  'base-uri': ["'self'"],
  'form-action': ["'self'"],
};

// Security headers
export const securityHeaders = {
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Permissions-Policy': 'camera=(), microphone=(), geolocation=()',
};

// CSRF token management
export const getCsrfToken = (): string | null => {
  return document.querySelector('meta[name="csrf-token"]')?.getAttribute('content') || null;
};

export const setCsrfToken = (token: string): void => {
  let metaTag = document.querySelector('meta[name="csrf-token"]');
  
  if (!metaTag) {
    metaTag = document.createElement('meta');
    metaTag.setAttribute('name', 'csrf-token');
    document.head.appendChild(metaTag);
  }
  
  metaTag.setAttribute('content', token);
};
```

### 4. Rate Limiting & Abuse Prevention

```typescript
// src/lib/rateLimiter.ts

interface RateLimitConfig {
  maxRequests: number;
  windowMs: number;
  keyGenerator?: (request: any) => string;
}

class ClientRateLimiter {
  private requests: Map<string, number[]> = new Map();
  
  constructor(private config: RateLimitConfig) {}
  
  isAllowed(key?: string): boolean {
    const requestKey = key || 'default';
    const now = Date.now();
    const windowStart = now - this.config.windowMs;
    
    // Get existing requests for this key
    const requests = this.requests.get(requestKey) || [];
    
    // Filter out old requests
    const recentRequests = requests.filter(time => time > windowStart);
    
    // Check if limit exceeded
    if (recentRequests.length >= this.config.maxRequests) {
      return false;
    }
    
    // Add current request
    recentRequests.push(now);
    this.requests.set(requestKey, recentRequests);
    
    return true;
  }
  
  getRemainingRequests(key?: string): number {
    const requestKey = key || 'default';
    const now = Date.now();
    const windowStart = now - this.config.windowMs;
    
    const requests = this.requests.get(requestKey) || [];
    const recentRequests = requests.filter(time => time > windowStart);
    
    return Math.max(0, this.config.maxRequests - recentRequests.length);
  }
  
  getResetTime(key?: string): number {
    const requestKey = key || 'default';
    const requests = this.requests.get(requestKey) || [];
    
    if (requests.length === 0) {
      return 0;
    }
    
    const oldestRequest = Math.min(...requests);
    return oldestRequest + this.config.windowMs;
  }
}

// Create rate limiters for different operations
export const loginRateLimiter = new ClientRateLimiter({
  maxRequests: 5,
  windowMs: 15 * 60 * 1000, // 15 minutes
});

export const apiRateLimiter = new ClientRateLimiter({
  maxRequests: 100,
  windowMs: 60 * 1000, // 1 minute
});

export const uploadRateLimiter = new ClientRateLimiter({
  maxRequests: 10,
  windowMs: 60 * 1000, // 1 minute
});
```

## Security Best Practices

### 1. Token Security

```typescript
// Secure token storage
class SecureStorage {
  private static encrypt(data: string): string {
    // Simple encryption for demonstration
    // In production, use proper encryption library
    return btoa(data);
  }
  
  private static decrypt(data: string): string {
    try {
      return atob(data);
    } catch {
      return '';
    }
  }
  
  static setItem(key: string, value: string): void {
    try {
      const encrypted = this.encrypt(value);
      localStorage.setItem(key, encrypted);
    } catch (error) {
      console.error('Failed to store item securely:', error);
    }
  }
  
  static getItem(key: string): string | null {
    try {
      const encrypted = localStorage.getItem(key);
      return encrypted ? this.decrypt(encrypted) : null;
    } catch (error) {
      console.error('Failed to retrieve item securely:', error);
      return null;
    }
  }
  
  static removeItem(key: string): void {
    localStorage.removeItem(key);
  }
}
```

### 2. Environment Security

```typescript
// src/lib/config.ts

// Environment validation
const requiredEnvVars = [
  'VITE_API_URL',
  'VITE_APP_ENV',
] as const;

const validateEnvironment = () => {
  const missing = requiredEnvVars.filter(envVar => !process.env[envVar]);
  
  if (missing.length > 0) {
    throw new Error(`Missing required environment variables: ${missing.join(', ')}`);
  }
};

// Secure configuration
export const config = {
  apiUrl: process.env.VITE_API_URL!,
  environment: process.env.VITE_APP_ENV!,
  isDevelopment: process.env.VITE_APP_ENV === 'development',
  isProduction: process.env.VITE_APP_ENV === 'production',
  
  // Security settings
  security: {
    tokenExpiry: 15 * 60 * 1000, // 15 minutes
    refreshTokenExpiry: 7 * 24 * 60 * 60 * 1000, // 7 days
    maxLoginAttempts: 5,
    lockoutDuration: 15 * 60 * 1000, // 15 minutes
  },
  
  // Feature flags
  features: {
    enableSSO: process.env.VITE_ENABLE_SSO === 'true',
    enableTwoFactor: process.env.VITE_ENABLE_2FA === 'true',
    enableAnalytics: process.env.VITE_ENABLE_ANALYTICS === 'true',
  },
};

// Validate environment on module load
validateEnvironment();
```

### 3. Error Handling Security

```typescript
// src/lib/errorHandler.ts

interface SecurityError {
  type: 'authentication' | 'authorization' | 'validation' | 'rate_limit';
  message: string;
  code?: string;
  details?: any;
}

class SecurityErrorHandler {
  static handle(error: SecurityError): void {
    // Log security errors (but not sensitive data)
    console.error('Security error:', {
      type: error.type,
      message: error.message,
      code: error.code,
      timestamp: new Date().toISOString(),
    });
    
    // Send to monitoring service (without sensitive data)
    if (config.isProduction) {
      this.reportToMonitoring(error);
    }
    
    // Handle specific error types
    switch (error.type) {
      case 'authentication':
        this.handleAuthError(error);
        break;
      case 'authorization':
        this.handleAuthzError(error);
        break;
      case 'rate_limit':
        this.handleRateLimitError(error);
        break;
      default:
        this.handleGenericError(error);
    }
  }
  
  private static handleAuthError(error: SecurityError): void {
    // Clear auth state
    useAuthStore.getState().logout();
    
    // Redirect to login
    window.location.href = '/login';
  }
  
  private static handleAuthzError(error: SecurityError): void {
    // Show unauthorized message
    // Redirect to appropriate page
  }
  
  private static handleRateLimitError(error: SecurityError): void {
    // Show rate limit message
    // Implement exponential backoff
  }
  
  private static handleGenericError(error: SecurityError): void {
    // Show generic error message
    // Don't expose sensitive information
  }
  
  private static reportToMonitoring(error: SecurityError): void {
    // Send to external monitoring service
    // Remove any sensitive data before sending
  }
}

export { SecurityErrorHandler };
```

## Testing Security

### 1. Authentication Tests

```typescript
// src/services/__tests__/auth.service.test.ts
import { authService } from '../auth.service';
import { useAuthStore } from '../../stores/auth.store';

jest.mock('../../stores/auth.store');

describe('AuthService', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    localStorage.clear();
  });
  
  describe('login', () => {
    it('should store auth data on successful login', async () => {
      const mockResponse = {
        user: { id: '1', email: 'test@example.com', role: 'vendor' },
        token: 'jwt-token',
        refreshToken: 'refresh-token',
        expiresIn: 900,
      };
      
      // Mock API response
      jest.spyOn(global, 'fetch').mockResolvedValueOnce({
        ok: true,
        json: () => Promise.resolve(mockResponse),
      } as Response);
      
      const result = await authService.login({
        email: 'test@example.com',
        password: 'password123',
      });
      
      expect(result).toEqual(mockResponse);
      expect(localStorage.getItem('refreshToken')).toBe('refresh-token');
      expect(useAuthStore.getState().login).toHaveBeenCalledWith(
        mockResponse.user,
        mockResponse.token
      );
    });
    
    it('should handle login failure', async () => {
      jest.spyOn(global, 'fetch').mockRejectedValueOnce(
        new Error('Invalid credentials')
      );
      
      await expect(
        authService.login({
          email: 'test@example.com',
          password: 'wrong-password',
        })
      ).rejects.toThrow('Invalid credentials');
    });
  });
  
  describe('token refresh', () => {
    it('should refresh token successfully', async () => {
      localStorage.setItem('refreshToken', 'valid-refresh-token');
      
      const mockResponse = {
        user: { id: '1', email: 'test@example.com', role: 'vendor' },
        token: 'new-jwt-token',
        refreshToken: 'new-refresh-token',
        expiresIn: 900,
      };
      
      jest.spyOn(global, 'fetch').mockResolvedValueOnce({
        ok: true,
        json: () => Promise.resolve(mockResponse),
      } as Response);
      
      const result = await authService.refreshToken();
      
      expect(result).toEqual(mockResponse);
      expect(localStorage.getItem('refreshToken')).toBe('new-refresh-token');
    });
    
    it('should logout on refresh failure', async () => {
      localStorage.setItem('refreshToken', 'invalid-refresh-token');
      
      jest.spyOn(global, 'fetch').mockRejectedValueOnce(
        new Error('Invalid refresh token')
      );
      
      await expect(authService.refreshToken()).rejects.toThrow();
      expect(useAuthStore.getState().logout).toHaveBeenCalled();
    });
  });
});
```

### 2. Security Component Tests

```typescript
// src/components/auth/__tests__/ProtectedRoute.test.tsx
import { render, screen } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { ProtectedRoute } from '../ProtectedRoute';
import { useAuth } from '../../../hooks/useAuth';

jest.mock('../../../hooks/useAuth');

const mockUseAuth = useAuth as jest.MockedFunction<typeof useAuth>;

describe('ProtectedRoute', () => {
  it('should render children when authenticated with correct role', () => {
    mockUseAuth.mockReturnValue({
      isAuthenticated: true,
      user: { id: '1', role: 'vendor' },
      isLoading: false,
    } as any);
    
    render(
      <MemoryRouter>
        <ProtectedRoute requiredRole="vendor">
          <div>Protected Content</div>
        </ProtectedRoute>
      </MemoryRouter>
    );
    
    expect(screen.getByText('Protected Content')).toBeInTheDocument();
  });
  
  it('should redirect to login when not authenticated', () => {
    mockUseAuth.mockReturnValue({
      isAuthenticated: false,
      user: null,
      isLoading: false,
    } as any);
    
    render(
      <MemoryRouter>
        <ProtectedRoute>
          <div>Protected Content</div>
        </ProtectedRoute>
      </MemoryRouter>
    );
    
    expect(screen.queryByText('Protected Content')).not.toBeInTheDocument();
  });
  
  it('should show fallback when user lacks required role', () => {
    mockUseAuth.mockReturnValue({
      isAuthenticated: true,
      user: { id: '1', role: 'customer' },
      isLoading: false,
    } as any);
    
    render(
      <MemoryRouter>
        <ProtectedRoute 
          requiredRole="vendor"
          fallback={<div>Access Denied</div>}
        >
          <div>Protected Content</div>
        </ProtectedRoute>
      </MemoryRouter>
    );
    
    expect(screen.getByText('Access Denied')).toBeInTheDocument();
    expect(screen.queryByText('Protected Content')).not.toBeInTheDocument();
  });
});
```

## Security Monitoring

### 1. Security Event Logging

```typescript
// src/lib/securityLogger.ts

interface SecurityEvent {
  type: 'login_attempt' | 'login_success' | 'login_failure' | 'logout' | 'token_refresh' | 'unauthorized_access';
  userId?: string;
  ip?: string;
  userAgent?: string;
  timestamp: string;
  details?: Record<string, any>;
}

class SecurityLogger {
  private static events: SecurityEvent[] = [];
  
  static log(event: Omit<SecurityEvent, 'timestamp'>): void {
    const securityEvent: SecurityEvent = {
      ...event,
      timestamp: new Date().toISOString(),
    };
    
    // Store locally (for debugging)
    this.events.push(securityEvent);
    
    // Send to monitoring service
    if (config.isProduction) {
      this.sendToMonitoring(securityEvent);
    }
    
    // Log to console in development
    if (config.isDevelopment) {
      console.log('Security Event:', securityEvent);
    }
  }
  
  private static sendToMonitoring(event: SecurityEvent): void {
    // Send to external monitoring service
    fetch('/api/security/events', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(event),
    }).catch(error => {
      console.error('Failed to send security event:', error);
    });
  }
  
  static getEvents(): SecurityEvent[] {
    return [...this.events];
  }
  
  static clearEvents(): void {
    this.events = [];
  }
}

export { SecurityLogger };
```

## Next Steps

For more detailed information:
- [Real-time Communication](./real-time-communication.md) - WebSocket security
- [State Management](./state-management.md) - Secure state handling
- [Performance Guide](./performance.md) - Security performance optimization
- [Testing Guide](./testing.md) - Security testing strategies
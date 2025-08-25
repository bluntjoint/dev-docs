# Deployment Guide

## Overview

This document provides comprehensive deployment strategies and configurations for the Deed-O Frontend application, covering development, staging, and production environments.

## Deployment Architecture

### Environment Strategy

```
Development → Staging → Production
     ↓           ↓         ↓
  Local Dev   Preview    Live Site
  Hot Reload  Testing    Optimized
  Debug Mode  QA/UAT     Monitoring
```

### Infrastructure Overview

- **Frontend Hosting**: Vercel (Primary), Netlify (Backup)
- **CDN**: Vercel Edge Network / Cloudflare
- **Domain Management**: Cloudflare DNS
- **SSL/TLS**: Automatic via hosting platform
- **Monitoring**: Vercel Analytics, Sentry
- **Performance**: Lighthouse CI, Web Vitals

## Environment Configuration

### 1. Environment Variables

```bash
# .env.local (Development)
VITE_API_BASE_URL=http://localhost:3000/api
VITE_SOCKET_URL=http://localhost:3000
VITE_APP_ENV=development
VITE_ENABLE_DEVTOOLS=true
VITE_LOG_LEVEL=debug
VITE_SENTRY_DSN=
VITE_GOOGLE_CLIENT_ID=your-google-client-id
VITE_MICROSOFT_CLIENT_ID=your-microsoft-client-id
VITE_S3_BUCKET_URL=https://your-dev-bucket.s3.amazonaws.com
VITE_CLOUDINARY_CLOUD_NAME=your-cloudinary-name
VITE_ANALYTICS_ID=
```

```bash
# .env.staging (Staging)
VITE_API_BASE_URL=https://api-staging.deed-o.com/api
VITE_SOCKET_URL=https://api-staging.deed-o.com
VITE_APP_ENV=staging
VITE_ENABLE_DEVTOOLS=false
VITE_LOG_LEVEL=warn
VITE_SENTRY_DSN=your-staging-sentry-dsn
VITE_GOOGLE_CLIENT_ID=your-google-client-id
VITE_MICROSOFT_CLIENT_ID=your-microsoft-client-id
VITE_S3_BUCKET_URL=https://your-staging-bucket.s3.amazonaws.com
VITE_CLOUDINARY_CLOUD_NAME=your-cloudinary-name
VITE_ANALYTICS_ID=your-staging-analytics-id
```

```bash
# .env.production (Production)
VITE_API_BASE_URL=https://api.deed-o.com/api
VITE_SOCKET_URL=https://api.deed-o.com
VITE_APP_ENV=production
VITE_ENABLE_DEVTOOLS=false
VITE_LOG_LEVEL=error
VITE_SENTRY_DSN=your-production-sentry-dsn
VITE_GOOGLE_CLIENT_ID=your-google-client-id
VITE_MICROSOFT_CLIENT_ID=your-microsoft-client-id
VITE_S3_BUCKET_URL=https://your-production-bucket.s3.amazonaws.com
VITE_CLOUDINARY_CLOUD_NAME=your-cloudinary-name
VITE_ANALYTICS_ID=your-production-analytics-id
```

### 2. Environment-Specific Configuration

```typescript
// src/config/environment.ts
interface EnvironmentConfig {
  apiBaseUrl: string;
  socketUrl: string;
  environment: 'development' | 'staging' | 'production';
  enableDevtools: boolean;
  logLevel: 'debug' | 'info' | 'warn' | 'error';
  sentryDsn?: string;
  googleClientId: string;
  microsoftClientId: string;
  s3BucketUrl: string;
  cloudinaryCloudName: string;
  analyticsId?: string;
}

const getEnvironmentConfig = (): EnvironmentConfig => {
  const config: EnvironmentConfig = {
    apiBaseUrl: import.meta.env.VITE_API_BASE_URL || 'http://localhost:3000/api',
    socketUrl: import.meta.env.VITE_SOCKET_URL || 'http://localhost:3000',
    environment: (import.meta.env.VITE_APP_ENV as any) || 'development',
    enableDevtools: import.meta.env.VITE_ENABLE_DEVTOOLS === 'true',
    logLevel: (import.meta.env.VITE_LOG_LEVEL as any) || 'info',
    sentryDsn: import.meta.env.VITE_SENTRY_DSN,
    googleClientId: import.meta.env.VITE_GOOGLE_CLIENT_ID || '',
    microsoftClientId: import.meta.env.VITE_MICROSOFT_CLIENT_ID || '',
    s3BucketUrl: import.meta.env.VITE_S3_BUCKET_URL || '',
    cloudinaryCloudName: import.meta.env.VITE_CLOUDINARY_CLOUD_NAME || '',
    analyticsId: import.meta.env.VITE_ANALYTICS_ID,
  };

  // Validate required environment variables
  const requiredVars = ['apiBaseUrl', 'socketUrl', 'googleClientId'];
  const missingVars = requiredVars.filter(key => !config[key as keyof EnvironmentConfig]);
  
  if (missingVars.length > 0) {
    throw new Error(`Missing required environment variables: ${missingVars.join(', ')}`);
  }

  return config;
};

export const env = getEnvironmentConfig();

// Environment-specific feature flags
export const features = {
  enableAnalytics: env.environment === 'production',
  enableErrorReporting: !!env.sentryDsn,
  enableDevtools: env.enableDevtools,
  enablePerformanceMonitoring: env.environment !== 'development',
  enableExperiments: env.environment === 'staging' || env.environment === 'production',
};

// API configuration
export const apiConfig = {
  baseURL: env.apiBaseUrl,
  timeout: env.environment === 'development' ? 10000 : 5000,
  retries: env.environment === 'production' ? 3 : 1,
};

// Socket configuration
export const socketConfig = {
  url: env.socketUrl,
  options: {
    transports: ['websocket', 'polling'],
    timeout: 20000,
    reconnection: true,
    reconnectionAttempts: env.environment === 'production' ? 5 : 3,
    reconnectionDelay: 1000,
  },
};
```

## Build Configuration

### 1. Production Build Optimization

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';
import { visualizer } from 'rollup-plugin-visualizer';
import { splitVendorChunkPlugin } from 'vite';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig(({ mode }) => {
  const isProduction = mode === 'production';
  
  return {
    plugins: [
      react(),
      splitVendorChunkPlugin(),
      
      // Bundle analyzer (only in production)
      isProduction && visualizer({
        filename: 'dist/stats.html',
        open: false,
        gzipSize: true,
        brotliSize: true,
      }),
      
      // PWA configuration
      VitePWA({
        registerType: 'autoUpdate',
        workbox: {
          globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
          runtimeCaching: [
            {
              urlPattern: /^https:\/\/api\.deed-o\.com\/.*/i,
              handler: 'NetworkFirst',
              options: {
                cacheName: 'api-cache',
                expiration: {
                  maxEntries: 100,
                  maxAgeSeconds: 60 * 60 * 24, // 24 hours
                },
              },
            },
            {
              urlPattern: /\.(?:png|jpg|jpeg|svg|gif|webp)$/,
              handler: 'CacheFirst',
              options: {
                cacheName: 'images-cache',
                expiration: {
                  maxEntries: 200,
                  maxAgeSeconds: 60 * 60 * 24 * 30, // 30 days
                },
              },
            },
          ],
        },
        manifest: {
          name: 'Deed-O',
          short_name: 'Deed-O',
          description: 'Digital marketplace for vendors and consumers',
          theme_color: '#ffffff',
          background_color: '#ffffff',
          display: 'standalone',
          icons: [
            {
              src: 'pwa-192x192.png',
              sizes: '192x192',
              type: 'image/png',
            },
            {
              src: 'pwa-512x512.png',
              sizes: '512x512',
              type: 'image/png',
            },
          ],
        },
      }),
    ].filter(Boolean),
    
    resolve: {
      alias: {
        '@': resolve(__dirname, './src'),
      },
    },
    
    build: {
      target: 'es2020',
      minify: 'terser',
      terserOptions: {
        compress: {
          drop_console: isProduction,
          drop_debugger: isProduction,
        },
      },
      rollupOptions: {
        output: {
          manualChunks: {
            'react-vendor': ['react', 'react-dom'],
            'router-vendor': ['react-router-dom'],
            'query-vendor': ['@tanstack/react-query'],
            'ui-vendor': [
              '@radix-ui/react-dialog',
              '@radix-ui/react-dropdown-menu',
              '@radix-ui/react-select',
            ],
            'utils-vendor': ['date-fns', 'lodash-es', 'zod'],
            'chat': ['socket.io-client'],
          },
        },
      },
      sourcemap: isProduction ? 'hidden' : true,
      chunkSizeWarningLimit: 1000,
    },
    
    server: {
      port: 5173,
      host: true,
      hmr: {
        overlay: false,
      },
    },
    
    preview: {
      port: 4173,
      host: true,
    },
    
    optimizeDeps: {
      include: [
        'react',
        'react-dom',
        'react-router-dom',
        '@tanstack/react-query',
        'zustand',
      ],
    },
  };
});
```

### 2. Docker Configuration

```dockerfile
# Dockerfile
# Multi-stage build for optimized production image

# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY yarn.lock ./

# Install dependencies
RUN yarn install --frozen-lockfile

# Copy source code
COPY . .

# Build application
ARG NODE_ENV=production
RUN yarn build

# Production stage
FROM nginx:alpine AS production

# Copy built assets
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

# Expose port
EXPOSE 80

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    
    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;
    
    # Performance optimizations
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https:; style-src 'self' 'unsafe-inline' https:; img-src 'self' data: https:; font-src 'self' https:; connect-src 'self' https: wss:;" always;
    
    server {
        listen 80;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;
        
        # Cache static assets
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            try_files $uri =404;
        }
        
        # Handle client-side routing
        location / {
            try_files $uri $uri/ /index.html;
            
            # Cache HTML files for a short time
            location ~* \.html$ {
                expires 1h;
                add_header Cache-Control "public";
            }
        }
        
        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        # Security: Hide nginx version
        server_tokens off;
    }
}
```

## CI/CD Pipeline

### 1. GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main

env:
  NODE_VERSION: '18'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linting
        run: npm run lint
      
      - name: Run type checking
        run: npm run type-check
      
      - name: Run unit tests
        run: npm run test:coverage
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
          fail_ci_if_error: true
  
  e2e-test:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Playwright
        run: npx playwright install --with-deps
      
      - name: Build application
        run: npm run build
        env:
          VITE_API_BASE_URL: ${{ secrets.STAGING_API_URL }}
          VITE_SOCKET_URL: ${{ secrets.STAGING_SOCKET_URL }}
      
      - name: Run E2E tests
        run: npm run test:e2e
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
  
  build:
    runs-on: ubuntu-latest
    needs: [test, e2e-test]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    outputs:
      image: ${{ steps.image.outputs.image }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Output image
        id: image
        run: echo "image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}" >> $GITHUB_OUTPUT
  
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.deed-o.com
    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying ${{ needs.build.outputs.image }} to staging"
          # Add your staging deployment commands here
  
  deploy-production:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://deed-o.com
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying ${{ needs.build.outputs.image }} to production"
          # Add your production deployment commands here
  
  lighthouse:
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/develop'
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          urls: |
            https://staging.deed-o.com
            https://staging.deed-o.com/vendor/dashboard
            https://staging.deed-o.com/chat
          configPath: './lighthouserc.json'
          uploadArtifacts: true
          temporaryPublicStorage: true
```

### 2. Vercel Deployment

```json
// vercel.json
{
  "version": 2,
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/static-build",
      "config": {
        "distDir": "dist"
      }
    }
  ],
  "routes": [
    {
      "src": "/assets/(.*)",
      "headers": {
        "cache-control": "public, max-age=31536000, immutable"
      }
    },
    {
      "src": "/(.*\\.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot))$",
      "headers": {
        "cache-control": "public, max-age=31536000, immutable"
      }
    },
    {
      "src": "/(.*)",
      "dest": "/index.html"
    }
  ],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-XSS-Protection",
          "value": "1; mode=block"
        },
        {
          "key": "Referrer-Policy",
          "value": "strict-origin-when-cross-origin"
        },
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https:; style-src 'self' 'unsafe-inline' https:; img-src 'self' data: https:; font-src 'self' https:; connect-src 'self' https: wss:;"
        }
      ]
    }
  ],
  "env": {
    "VITE_API_BASE_URL": "@api_base_url",
    "VITE_SOCKET_URL": "@socket_url",
    "VITE_APP_ENV": "@app_env",
    "VITE_SENTRY_DSN": "@sentry_dsn",
    "VITE_GOOGLE_CLIENT_ID": "@google_client_id",
    "VITE_MICROSOFT_CLIENT_ID": "@microsoft_client_id",
    "VITE_S3_BUCKET_URL": "@s3_bucket_url",
    "VITE_CLOUDINARY_CLOUD_NAME": "@cloudinary_cloud_name",
    "VITE_ANALYTICS_ID": "@analytics_id"
  },
  "functions": {
    "app/api/**/*.ts": {
      "runtime": "nodejs18.x"
    }
  }
}
```

### 3. Netlify Deployment

```toml
# netlify.toml
[build]
  publish = "dist"
  command = "npm run build"

[build.environment]
  NODE_VERSION = "18"
  NPM_FLAGS = "--prefix=/dev/null"

[[redirects]]
  from = "/api/*"
  to = "https://api.deed-o.com/api/:splat"
  status = 200
  force = true

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-XSS-Protection = "1; mode=block"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"
    Content-Security-Policy = "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https:; style-src 'self' 'unsafe-inline' https:; img-src 'self' data: https:; font-src 'self' https:; connect-src 'self' https: wss:;"

[[headers]]
  for = "/assets/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[[headers]]
  for = "*.js"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[[headers]]
  for = "*.css"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"

[context.production]
  environment = { VITE_APP_ENV = "production" }

[context.staging]
  environment = { VITE_APP_ENV = "staging" }

[context.branch-deploy]
  environment = { VITE_APP_ENV = "development" }
```

## Performance Monitoring

### 1. Lighthouse CI Configuration

```json
// lighthouserc.json
{
  "ci": {
    "collect": {
      "url": [
        "http://localhost:4173",
        "http://localhost:4173/vendor/dashboard",
        "http://localhost:4173/chat"
      ],
      "startServerCommand": "npm run preview",
      "startServerReadyPattern": "Local:",
      "startServerReadyTimeout": 30000
    },
    "assert": {
      "assertions": {
        "categories:performance": ["error", {"minScore": 0.9}],
        "categories:accessibility": ["error", {"minScore": 0.95}],
        "categories:best-practices": ["error", {"minScore": 0.9}],
        "categories:seo": ["error", {"minScore": 0.85}],
        "first-contentful-paint": ["error", {"maxNumericValue": 2000}],
        "largest-contentful-paint": ["error", {"maxNumericValue": 2500}],
        "cumulative-layout-shift": ["error", {"maxNumericValue": 0.1}],
        "total-blocking-time": ["error", {"maxNumericValue": 300}]
      }
    },
    "upload": {
      "target": "temporary-public-storage"
    }
  }
}
```

### 2. Sentry Configuration

```typescript
// src/lib/sentry.ts
import * as Sentry from '@sentry/react';
import { BrowserTracing } from '@sentry/tracing';
import { env } from '../config/environment';

if (env.sentryDsn && env.environment !== 'development') {
  Sentry.init({
    dsn: env.sentryDsn,
    environment: env.environment,
    integrations: [
      new BrowserTracing({
        routingInstrumentation: Sentry.reactRouterV6Instrumentation(
          React.useEffect,
          useLocation,
          useNavigationType,
          createRoutesFromChildren,
          matchRoutes
        ),
      }),
    ],
    
    // Performance monitoring
    tracesSampleRate: env.environment === 'production' ? 0.1 : 1.0,
    
    // Error filtering
    beforeSend(event, hint) {
      // Filter out known non-critical errors
      const error = hint.originalException;
      
      if (error && error.message) {
        // Ignore network errors
        if (error.message.includes('Network Error')) {
          return null;
        }
        
        // Ignore cancelled requests
        if (error.message.includes('cancelled')) {
          return null;
        }
      }
      
      return event;
    },
    
    // Release tracking
    release: import.meta.env.VITE_APP_VERSION || 'unknown',
    
    // User context
    initialScope: {
      tags: {
        component: 'frontend',
      },
    },
  });
}

// Error boundary component
export const SentryErrorBoundary = Sentry.withErrorBoundary;

// Performance monitoring hooks
export const useSentryTransaction = (name: string) => {
  React.useEffect(() => {
    const transaction = Sentry.startTransaction({ name });
    Sentry.getCurrentHub().configureScope(scope => scope.setSpan(transaction));
    
    return () => {
      transaction.finish();
    };
  }, [name]);
};
```

### 3. Analytics Integration

```typescript
// src/lib/analytics.ts
import { env, features } from '../config/environment';

interface AnalyticsEvent {
  name: string;
  properties?: Record<string, any>;
  userId?: string;
}

class Analytics {
  private initialized = false;
  
  async init(): Promise<void> {
    if (!features.enableAnalytics || !env.analyticsId) {
      return;
    }
    
    try {
      // Initialize Google Analytics
      const script = document.createElement('script');
      script.async = true;
      script.src = `https://www.googletagmanager.com/gtag/js?id=${env.analyticsId}`;
      document.head.appendChild(script);
      
      window.dataLayer = window.dataLayer || [];
      function gtag(...args: any[]) {
        window.dataLayer.push(args);
      }
      
      gtag('js', new Date());
      gtag('config', env.analyticsId, {
        page_title: document.title,
        page_location: window.location.href,
      });
      
      this.initialized = true;
    } catch (error) {
      console.warn('Failed to initialize analytics:', error);
    }
  }
  
  track(event: AnalyticsEvent): void {
    if (!this.initialized) {
      return;
    }
    
    try {
      // Google Analytics
      if (window.gtag) {
        window.gtag('event', event.name, {
          ...event.properties,
          user_id: event.userId,
        });
      }
      
      // Custom analytics endpoint
      if (env.environment === 'production') {
        fetch('/api/analytics/events', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            event: event.name,
            properties: event.properties,
            userId: event.userId,
            timestamp: new Date().toISOString(),
            url: window.location.href,
            userAgent: navigator.userAgent,
          }),
        }).catch(error => {
          console.warn('Failed to send analytics event:', error);
        });
      }
    } catch (error) {
      console.warn('Failed to track analytics event:', error);
    }
  }
  
  page(path: string, title?: string): void {
    if (!this.initialized) {
      return;
    }
    
    try {
      if (window.gtag) {
        window.gtag('config', env.analyticsId, {
          page_path: path,
          page_title: title || document.title,
        });
      }
    } catch (error) {
      console.warn('Failed to track page view:', error);
    }
  }
  
  identify(userId: string, properties?: Record<string, any>): void {
    if (!this.initialized) {
      return;
    }
    
    try {
      if (window.gtag) {
        window.gtag('config', env.analyticsId, {
          user_id: userId,
          custom_map: properties,
        });
      }
    } catch (error) {
      console.warn('Failed to identify user:', error);
    }
  }
}

export const analytics = new Analytics();

// React hook for analytics
export const useAnalytics = () => {
  const track = React.useCallback((event: AnalyticsEvent) => {
    analytics.track(event);
  }, []);
  
  const page = React.useCallback((path: string, title?: string) => {
    analytics.page(path, title);
  }, []);
  
  const identify = React.useCallback((userId: string, properties?: Record<string, any>) => {
    analytics.identify(userId, properties);
  }, []);
  
  return { track, page, identify };
};
```

## Security Configuration

### 1. Content Security Policy

```typescript
// src/lib/security.ts
export const generateCSP = (env: string) => {
  const basePolicy = {
    'default-src': ["'self'"],
    'script-src': [
      "'self'",
      "'unsafe-inline'", // Required for React dev tools
      "'unsafe-eval'", // Required for React dev tools
      'https://www.googletagmanager.com',
      'https://www.google-analytics.com',
    ],
    'style-src': [
      "'self'",
      "'unsafe-inline'", // Required for styled-components
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
      'https:',
      'wss:',
      'https://api.deed-o.com',
      'wss://api.deed-o.com',
    ],
    'media-src': ["'self'", 'https:'],
    'object-src': ["'none'"],
    'base-uri': ["'self'"],
    'form-action': ["'self'"],
    'frame-ancestors': ["'none'"],
    'upgrade-insecure-requests': [],
  };
  
  // Development-specific policies
  if (env === 'development') {
    basePolicy['connect-src'].push(
      'http://localhost:*',
      'ws://localhost:*',
      'wss://localhost:*'
    );
  }
  
  return Object.entries(basePolicy)
    .map(([directive, sources]) => `${directive} ${sources.join(' ')}`)
    .join('; ');
};

// Security headers middleware
export const securityHeaders = {
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  'Permissions-Policy': 'camera=(), microphone=(), geolocation=()',
};
```

### 2. Environment Validation

```typescript
// src/lib/validation.ts
import { z } from 'zod';

const environmentSchema = z.object({
  VITE_API_BASE_URL: z.string().url(),
  VITE_SOCKET_URL: z.string().url(),
  VITE_APP_ENV: z.enum(['development', 'staging', 'production']),
  VITE_GOOGLE_CLIENT_ID: z.string().min(1),
  VITE_MICROSOFT_CLIENT_ID: z.string().min(1),
  VITE_S3_BUCKET_URL: z.string().url(),
  VITE_CLOUDINARY_CLOUD_NAME: z.string().min(1),
  VITE_SENTRY_DSN: z.string().url().optional(),
  VITE_ANALYTICS_ID: z.string().optional(),
});

export const validateEnvironment = () => {
  try {
    return environmentSchema.parse(import.meta.env);
  } catch (error) {
    console.error('Environment validation failed:', error);
    throw new Error('Invalid environment configuration');
  }
};
```

## Deployment Scripts

```json
{
  "scripts": {
    "build": "tsc && vite build",
    "build:staging": "tsc && vite build --mode staging",
    "build:production": "tsc && vite build --mode production",
    "preview": "vite preview",
    "deploy:staging": "npm run build:staging && vercel --prod --env staging",
    "deploy:production": "npm run build:production && vercel --prod",
    "docker:build": "docker build -t deed-o-frontend .",
    "docker:run": "docker run -p 80:80 deed-o-frontend",
    "lighthouse": "lhci autorun",
    "analyze": "npm run build && npx vite-bundle-analyzer"
  }
}
```

## Monitoring and Alerting

### 1. Health Checks

```typescript
// src/lib/healthCheck.ts
interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  version: string;
  checks: {
    api: boolean;
    socket: boolean;
    storage: boolean;
  };
}

export const performHealthCheck = async (): Promise<HealthStatus> => {
  const checks = {
    api: false,
    socket: false,
    storage: false,
  };
  
  try {
    // Check API connectivity
    const apiResponse = await fetch(`${env.apiBaseUrl}/health`, {
      method: 'GET',
      timeout: 5000,
    });
    checks.api = apiResponse.ok;
  } catch (error) {
    console.warn('API health check failed:', error);
  }
  
  try {
    // Check socket connectivity
    const socket = io(env.socketUrl, { timeout: 5000 });
    await new Promise((resolve, reject) => {
      socket.on('connect', resolve);
      socket.on('connect_error', reject);
      setTimeout(reject, 5000);
    });
    checks.socket = true;
    socket.disconnect();
  } catch (error) {
    console.warn('Socket health check failed:', error);
  }
  
  try {
    // Check local storage
    localStorage.setItem('health-check', 'test');
    localStorage.removeItem('health-check');
    checks.storage = true;
  } catch (error) {
    console.warn('Storage health check failed:', error);
  }
  
  const healthyChecks = Object.values(checks).filter(Boolean).length;
  const totalChecks = Object.keys(checks).length;
  
  let status: HealthStatus['status'] = 'healthy';
  if (healthyChecks === 0) {
    status = 'unhealthy';
  } else if (healthyChecks < totalChecks) {
    status = 'degraded';
  }
  
  return {
    status,
    timestamp: new Date().toISOString(),
    version: import.meta.env.VITE_APP_VERSION || 'unknown',
    checks,
  };
};
```

## Rollback Strategy

### 1. Automated Rollback

```yaml
# .github/workflows/rollback.yml
name: Rollback

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to rollback'
        required: true
        type: choice
        options:
          - staging
          - production
      version:
        description: 'Version to rollback to'
        required: true
        type: string

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    steps:
      - name: Rollback deployment
        run: |
          echo "Rolling back ${{ github.event.inputs.environment }} to version ${{ github.event.inputs.version }}"
          # Add rollback commands here
      
      - name: Verify rollback
        run: |
          # Add verification commands here
          curl -f https://${{ github.event.inputs.environment }}.deed-o.com/health
      
      - name: Notify team
        if: always()
        run: |
          # Send notification about rollback status
          echo "Rollback completed for ${{ github.event.inputs.environment }}"
```

## Next Steps

For more detailed information:
- [Performance Guide](./performance.md) - Performance optimization strategies
- [Testing Guide](./testing.md) - Testing in CI/CD pipelines
- [Architecture Guide](./architecture.md) - Deployment architecture patterns
- [Monitoring Guide](./monitoring.md) - Production monitoring and alerting
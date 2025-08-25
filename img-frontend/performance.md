# Performance Optimization Guide

## Overview

This document provides comprehensive performance optimization strategies for the Deed-O Frontend application, covering code splitting, caching, rendering optimization, and monitoring.

## Performance Metrics

### Core Web Vitals

- **Largest Contentful Paint (LCP)**: < 2.5 seconds
- **First Input Delay (FID)**: < 100 milliseconds
- **Cumulative Layout Shift (CLS)**: < 0.1
- **First Contentful Paint (FCP)**: < 1.8 seconds
- **Time to Interactive (TTI)**: < 3.8 seconds

### Application Metrics

- **Bundle Size**: < 500KB (gzipped)
- **Initial Load Time**: < 3 seconds
- **Route Transition**: < 200ms
- **API Response Time**: < 500ms
- **Memory Usage**: < 50MB

## Code Splitting & Lazy Loading

### 1. Route-Based Code Splitting

```typescript
// src/router/routes.tsx
import { lazy } from 'react';
import { createBrowserRouter } from 'react-router-dom';
import { LoadingSpinner } from '../components/ui/loading-spinner';
import { ErrorBoundary } from '../components/error-boundary';

// Lazy load route components
const HomePage = lazy(() => import('../pages/HomePage'));
const VendorDashboard = lazy(() => import('../pages/vendor/Dashboard'));
const ProductCatalog = lazy(() => import('../pages/vendor/ProductCatalog'));
const ModeratorDashboard = lazy(() => import('../pages/moderator/Dashboard'));
const ChatPage = lazy(() => import('../pages/ChatPage'));

// Preload critical routes
const preloadRoute = (routeImport: () => Promise<any>) => {
  const componentImport = routeImport();
  return componentImport;
};

// Preload on user interaction
const preloadVendorRoutes = () => {
  preloadRoute(() => import('../pages/vendor/Dashboard'));
  preloadRoute(() => import('../pages/vendor/ProductCatalog'));
};

const preloadModeratorRoutes = () => {
  preloadRoute(() => import('../pages/moderator/Dashboard'));
};

// Route configuration with lazy loading
export const router = createBrowserRouter([
  {
    path: '/',
    element: (
      <ErrorBoundary>
        <Suspense fallback={<LoadingSpinner />}>
          <HomePage />
        </Suspense>
      </ErrorBoundary>
    ),
  },
  {
    path: '/vendor',
    element: (
      <ErrorBoundary>
        <Suspense fallback={<LoadingSpinner />}>
          <VendorDashboard />
        </Suspense>
      </ErrorBoundary>
    ),
    loader: () => {
      // Preload related routes
      preloadVendorRoutes();
      return null;
    },
  },
  {
    path: '/vendor/products',
    element: (
      <ErrorBoundary>
        <Suspense fallback={<LoadingSpinner />}>
          <ProductCatalog />
        </Suspense>
      </ErrorBoundary>
    ),
  },
  {
    path: '/moderator',
    element: (
      <ErrorBoundary>
        <Suspense fallback={<LoadingSpinner />}>
          <ModeratorDashboard />
        </Suspense>
      </ErrorBoundary>
    ),
    loader: () => {
      preloadModeratorRoutes();
      return null;
    },
  },
  {
    path: '/chat',
    element: (
      <ErrorBoundary>
        <Suspense fallback={<LoadingSpinner />}>
          <ChatPage />
        </Suspense>
      </ErrorBoundary>
    ),
  },
]);
```

### 2. Component-Level Code Splitting

```typescript
// src/components/LazyComponents.tsx
import { lazy, Suspense } from 'react';
import { Skeleton } from './ui/skeleton';

// Lazy load heavy components
const DataVisualization = lazy(() => import('./charts/DataVisualization'));
const FileUploader = lazy(() => import('./upload/FileUploader'));
const RichTextEditor = lazy(() => import('./editor/RichTextEditor'));
const VideoPlayer = lazy(() => import('./media/VideoPlayer'));

// Component with lazy loading
export const LazyDataVisualization = (props: any) => (
  <Suspense fallback={<Skeleton className="h-64 w-full" />}>
    <DataVisualization {...props} />
  </Suspense>
);

export const LazyFileUploader = (props: any) => (
  <Suspense fallback={<Skeleton className="h-32 w-full" />}>
    <FileUploader {...props} />
  </Suspense>
);

export const LazyRichTextEditor = (props: any) => (
  <Suspense fallback={<Skeleton className="h-48 w-full" />}>
    <RichTextEditor {...props} />
  </Suspense>
);

export const LazyVideoPlayer = (props: any) => (
  <Suspense fallback={<Skeleton className="h-64 w-full aspect-video" />}>
    <VideoPlayer {...props} />
  </Suspense>
);
```

### 3. Dynamic Imports with Preloading

```typescript
// src/utils/dynamicImports.ts

interface ImportCache {
  [key: string]: Promise<any>;
}

class DynamicImportManager {
  private cache: ImportCache = {};
  private preloadQueue: Set<string> = new Set();
  
  // Import with caching
  async import<T>(importFn: () => Promise<T>, key: string): Promise<T> {
    if (this.cache[key]) {
      return this.cache[key];
    }
    
    this.cache[key] = importFn();
    return this.cache[key];
  }
  
  // Preload component
  preload(importFn: () => Promise<any>, key: string): void {
    if (!this.cache[key] && !this.preloadQueue.has(key)) {
      this.preloadQueue.add(key);
      
      // Use requestIdleCallback for non-critical preloading
      if ('requestIdleCallback' in window) {
        requestIdleCallback(() => {
          this.cache[key] = importFn();
          this.preloadQueue.delete(key);
        });
      } else {
        setTimeout(() => {
          this.cache[key] = importFn();
          this.preloadQueue.delete(key);
        }, 0);
      }
    }
  }
  
  // Preload on user interaction
  preloadOnHover(element: HTMLElement, importFn: () => Promise<any>, key: string): void {
    const handleMouseEnter = () => {
      this.preload(importFn, key);
      element.removeEventListener('mouseenter', handleMouseEnter);
    };
    
    element.addEventListener('mouseenter', handleMouseEnter, { once: true });
  }
  
  // Preload on viewport intersection
  preloadOnIntersection(element: HTMLElement, importFn: () => Promise<any>, key: string): void {
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            this.preload(importFn, key);
            observer.unobserve(element);
          }
        });
      },
      { rootMargin: '50px' }
    );
    
    observer.observe(element);
  }
}

export const dynamicImportManager = new DynamicImportManager();

// Usage examples
export const preloadVendorComponents = () => {
  dynamicImportManager.preload(
    () => import('../pages/vendor/ProductForm'),
    'vendor-product-form'
  );
  
  dynamicImportManager.preload(
    () => import('../components/charts/SalesChart'),
    'sales-chart'
  );
};

export const preloadChatComponents = () => {
  dynamicImportManager.preload(
    () => import('../components/chat/ChatWindow'),
    'chat-window'
  );
  
  dynamicImportManager.preload(
    () => import('../components/chat/FileUpload'),
    'chat-file-upload'
  );
};
```

## React Performance Optimization

### 1. Memoization Strategies

```typescript
// src/components/optimized/ProductList.tsx
import { memo, useMemo, useCallback } from 'react';
import { Product } from '../../types/product';

interface ProductListProps {
  products: Product[];
  onProductSelect: (product: Product) => void;
  filters: {
    category?: string;
    priceRange?: [number, number];
    searchQuery?: string;
  };
}

// Memoized product item component
const ProductItem = memo(({ product, onSelect }: {
  product: Product;
  onSelect: (product: Product) => void;
}) => {
  const handleClick = useCallback(() => {
    onSelect(product);
  }, [product, onSelect]);
  
  return (
    <div 
      className="product-item cursor-pointer p-4 border rounded-lg hover:shadow-md transition-shadow"
      onClick={handleClick}
    >
      <img 
        src={product.imageUrl} 
        alt={product.name}
        className="w-full h-48 object-cover rounded"
        loading="lazy"
      />
      <h3 className="text-lg font-semibold mt-2">{product.name}</h3>
      <p className="text-gray-600">${product.price}</p>
    </div>
  );
});

// Memoized product list component
export const ProductList = memo(({ products, onProductSelect, filters }: ProductListProps) => {
  // Memoize filtered products
  const filteredProducts = useMemo(() => {
    return products.filter(product => {
      // Category filter
      if (filters.category && product.category !== filters.category) {
        return false;
      }
      
      // Price range filter
      if (filters.priceRange) {
        const [min, max] = filters.priceRange;
        if (product.price < min || product.price > max) {
          return false;
        }
      }
      
      // Search query filter
      if (filters.searchQuery) {
        const query = filters.searchQuery.toLowerCase();
        return (
          product.name.toLowerCase().includes(query) ||
          product.description.toLowerCase().includes(query)
        );
      }
      
      return true;
    });
  }, [products, filters]);
  
  // Memoize product selection handler
  const handleProductSelect = useCallback((product: Product) => {
    onProductSelect(product);
  }, [onProductSelect]);
  
  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
      {filteredProducts.map(product => (
        <ProductItem
          key={product.id}
          product={product}
          onSelect={handleProductSelect}
        />
      ))}
    </div>
  );
});

ProductList.displayName = 'ProductList';
ProductItem.displayName = 'ProductItem';
```

### 2. Virtual Scrolling

```typescript
// src/components/virtualized/VirtualizedList.tsx
import { FixedSizeList as List, VariableSizeList } from 'react-window';
import { memo, useMemo, useCallback } from 'react';
import AutoSizer from 'react-virtualized-auto-sizer';

interface VirtualizedListProps<T> {
  items: T[];
  itemHeight: number | ((index: number) => number);
  renderItem: (item: T, index: number) => React.ReactNode;
  overscanCount?: number;
  className?: string;
}

// Fixed height virtualized list
export const VirtualizedList = memo(<T,>({
  items,
  itemHeight,
  renderItem,
  overscanCount = 5,
  className,
}: VirtualizedListProps<T>) => {
  const itemData = useMemo(() => ({ items, renderItem }), [items, renderItem]);
  
  const Row = useCallback(({ index, style, data }: any) => (
    <div style={style}>
      {data.renderItem(data.items[index], index)}
    </div>
  ), []);
  
  if (typeof itemHeight === 'number') {
    return (
      <div className={className}>
        <AutoSizer>
          {({ height, width }) => (
            <List
              height={height}
              width={width}
              itemCount={items.length}
              itemSize={itemHeight}
              itemData={itemData}
              overscanCount={overscanCount}
            >
              {Row}
            </List>
          )}
        </AutoSizer>
      </div>
    );
  }
  
  // Variable height list
  const getItemSize = useCallback((index: number) => {
    return typeof itemHeight === 'function' ? itemHeight(index) : 50;
  }, [itemHeight]);
  
  return (
    <div className={className}>
      <AutoSizer>
        {({ height, width }) => (
          <VariableSizeList
            height={height}
            width={width}
            itemCount={items.length}
            itemSize={getItemSize}
            itemData={itemData}
            overscanCount={overscanCount}
          >
            {Row}
          </VariableSizeList>
        )}
      </AutoSizer>
    </div>
  );
});

VirtualizedList.displayName = 'VirtualizedList';

// Usage example for chat messages
export const VirtualizedChatMessages = ({ messages }: { messages: Message[] }) => {
  const renderMessage = useCallback((message: Message, index: number) => (
    <div className="p-4 border-b">
      <div className="font-semibold">{message.sender}</div>
      <div className="text-gray-700">{message.content}</div>
      <div className="text-xs text-gray-500">{message.timestamp}</div>
    </div>
  ), []);
  
  return (
    <VirtualizedList
      items={messages}
      itemHeight={80}
      renderItem={renderMessage}
      className="h-96 border rounded-lg"
    />
  );
};
```

### 3. Image Optimization

```typescript
// src/components/optimized/OptimizedImage.tsx
import { useState, useCallback, memo } from 'react';
import { cn } from '../../lib/utils';

interface OptimizedImageProps {
  src: string;
  alt: string;
  width?: number;
  height?: number;
  className?: string;
  placeholder?: string;
  quality?: number;
  priority?: boolean;
  onLoad?: () => void;
  onError?: () => void;
}

export const OptimizedImage = memo(({
  src,
  alt,
  width,
  height,
  className,
  placeholder = 'data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMjAwIiBoZWlnaHQ9IjIwMCIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj48cmVjdCB3aWR0aD0iMTAwJSIgaGVpZ2h0PSIxMDAlIiBmaWxsPSIjZGRkIi8+PC9zdmc+',
  quality = 75,
  priority = false,
  onLoad,
  onError,
}: OptimizedImageProps) => {
  const [isLoading, setIsLoading] = useState(true);
  const [hasError, setHasError] = useState(false);
  const [currentSrc, setCurrentSrc] = useState(placeholder);
  
  // Generate optimized image URL
  const getOptimizedSrc = useCallback((originalSrc: string) => {
    // If using a CDN like Cloudinary or ImageKit
    if (originalSrc.includes('cloudinary.com')) {
      const params = [];
      if (width) params.push(`w_${width}`);
      if (height) params.push(`h_${height}`);
      params.push(`q_${quality}`);
      params.push('f_auto');
      
      return originalSrc.replace('/upload/', `/upload/${params.join(',')}/`);
    }
    
    // For other CDNs or custom optimization
    const url = new URL(originalSrc);
    if (width) url.searchParams.set('w', width.toString());
    if (height) url.searchParams.set('h', height.toString());
    url.searchParams.set('q', quality.toString());
    
    return url.toString();
  }, [width, height, quality]);
  
  const handleLoad = useCallback(() => {
    setIsLoading(false);
    onLoad?.();
  }, [onLoad]);
  
  const handleError = useCallback(() => {
    setHasError(true);
    setIsLoading(false);
    onError?.();
  }, [onError]);
  
  // Use Intersection Observer for lazy loading
  const handleIntersection = useCallback((entries: IntersectionObserverEntry[]) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        setCurrentSrc(getOptimizedSrc(src));
      }
    });
  }, [src, getOptimizedSrc]);
  
  const imgRef = useCallback((node: HTMLImageElement | null) => {
    if (node && !priority) {
      const observer = new IntersectionObserver(handleIntersection, {
        rootMargin: '50px',
      });
      observer.observe(node);
      
      return () => observer.disconnect();
    } else if (node && priority) {
      // Load immediately for priority images
      setCurrentSrc(getOptimizedSrc(src));
    }
  }, [handleIntersection, priority, src, getOptimizedSrc]);
  
  if (hasError) {
    return (
      <div className={cn('bg-gray-200 flex items-center justify-center', className)}>
        <span className="text-gray-500">Failed to load image</span>
      </div>
    );
  }
  
  return (
    <div className={cn('relative overflow-hidden', className)}>
      <img
        ref={imgRef}
        src={currentSrc}
        alt={alt}
        width={width}
        height={height}
        onLoad={handleLoad}
        onError={handleError}
        className={cn(
          'transition-opacity duration-300',
          isLoading ? 'opacity-0' : 'opacity-100',
          className
        )}
        loading={priority ? 'eager' : 'lazy'}
      />
      
      {isLoading && (
        <div className="absolute inset-0 bg-gray-200 animate-pulse" />
      )}
    </div>
  );
});

OptimizedImage.displayName = 'OptimizedImage';
```

## Caching Strategies

### 1. React Query Caching

```typescript
// src/lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query';
import { persistQueryClient } from '@tanstack/react-query-persist-client-core';
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';

// Create query client with optimized defaults
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Cache for 5 minutes
      staleTime: 5 * 60 * 1000,
      // Keep in cache for 10 minutes
      cacheTime: 10 * 60 * 1000,
      // Retry failed requests
      retry: (failureCount, error: any) => {
        // Don't retry on 4xx errors
        if (error?.response?.status >= 400 && error?.response?.status < 500) {
          return false;
        }
        return failureCount < 3;
      },
      // Refetch on window focus for critical data
      refetchOnWindowFocus: (query) => {
        return query.queryKey[0] === 'user' || query.queryKey[0] === 'notifications';
      },
      // Background refetch interval for real-time data
      refetchInterval: (data, query) => {
        if (query.queryKey[0] === 'chat-messages') {
          return 30 * 1000; // 30 seconds
        }
        return false;
      },
    },
    mutations: {
      // Retry mutations on network errors
      retry: (failureCount, error: any) => {
        if (error?.code === 'NETWORK_ERROR') {
          return failureCount < 2;
        }
        return false;
      },
    },
  },
});

// Persist cache to localStorage
const localStoragePersister = createSyncStoragePersister({
  storage: window.localStorage,
  key: 'deed-o-cache',
  throttleTime: 1000,
});

// Persist specific queries
persistQueryClient({
  queryClient,
  persister: localStoragePersister,
  maxAge: 24 * 60 * 60 * 1000, // 24 hours
  hydrateOptions: {
    // Only persist certain query types
    dehydrateQuery: (query) => {
      const persistableQueries = ['user', 'products', 'groups'];
      return persistableQueries.some(key => query.queryKey[0] === key);
    },
  },
});
```

### 2. Service Worker Caching

```typescript
// public/sw.js
const CACHE_NAME = 'deed-o-v1';
const STATIC_CACHE = 'deed-o-static-v1';
const DYNAMIC_CACHE = 'deed-o-dynamic-v1';

// Cache strategies
const CACHE_STRATEGIES = {
  CACHE_FIRST: 'cache-first',
  NETWORK_FIRST: 'network-first',
  STALE_WHILE_REVALIDATE: 'stale-while-revalidate',
};

// Define caching rules
const CACHE_RULES = [
  {
    pattern: /\.(js|css|woff2?|png|jpg|jpeg|svg|ico)$/,
    strategy: CACHE_STRATEGIES.CACHE_FIRST,
    cache: STATIC_CACHE,
    maxAge: 30 * 24 * 60 * 60 * 1000, // 30 days
  },
  {
    pattern: /\/api\/(products|groups|users)/,
    strategy: CACHE_STRATEGIES.STALE_WHILE_REVALIDATE,
    cache: DYNAMIC_CACHE,
    maxAge: 5 * 60 * 1000, // 5 minutes
  },
  {
    pattern: /\/api\/(chat|notifications)/,
    strategy: CACHE_STRATEGIES.NETWORK_FIRST,
    cache: DYNAMIC_CACHE,
    maxAge: 1 * 60 * 1000, // 1 minute
  },
];

// Install event
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(STATIC_CACHE).then((cache) => {
      return cache.addAll([
        '/',
        '/static/js/main.js',
        '/static/css/main.css',
        '/manifest.json',
      ]);
    })
  );
});

// Fetch event
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);
  
  // Find matching cache rule
  const rule = CACHE_RULES.find(rule => rule.pattern.test(url.pathname));
  
  if (rule) {
    event.respondWith(handleRequest(request, rule));
  }
});

// Handle request based on strategy
async function handleRequest(request, rule) {
  const cache = await caches.open(rule.cache);
  
  switch (rule.strategy) {
    case CACHE_STRATEGIES.CACHE_FIRST:
      return cacheFirst(request, cache, rule);
    case CACHE_STRATEGIES.NETWORK_FIRST:
      return networkFirst(request, cache, rule);
    case CACHE_STRATEGIES.STALE_WHILE_REVALIDATE:
      return staleWhileRevalidate(request, cache, rule);
    default:
      return fetch(request);
  }
}

// Cache first strategy
async function cacheFirst(request, cache, rule) {
  const cachedResponse = await cache.match(request);
  
  if (cachedResponse && !isExpired(cachedResponse, rule.maxAge)) {
    return cachedResponse;
  }
  
  try {
    const networkResponse = await fetch(request);
    if (networkResponse.ok) {
      cache.put(request, networkResponse.clone());
    }
    return networkResponse;
  } catch (error) {
    return cachedResponse || new Response('Offline', { status: 503 });
  }
}

// Network first strategy
async function networkFirst(request, cache, rule) {
  try {
    const networkResponse = await fetch(request);
    if (networkResponse.ok) {
      cache.put(request, networkResponse.clone());
    }
    return networkResponse;
  } catch (error) {
    const cachedResponse = await cache.match(request);
    return cachedResponse || new Response('Offline', { status: 503 });
  }
}

// Stale while revalidate strategy
async function staleWhileRevalidate(request, cache, rule) {
  const cachedResponse = await cache.match(request);
  
  // Always try to fetch from network
  const networkPromise = fetch(request).then(response => {
    if (response.ok) {
      cache.put(request, response.clone());
    }
    return response;
  });
  
  // Return cached response immediately if available
  if (cachedResponse && !isExpired(cachedResponse, rule.maxAge)) {
    return cachedResponse;
  }
  
  // Otherwise wait for network
  return networkPromise;
}

// Check if cached response is expired
function isExpired(response, maxAge) {
  const dateHeader = response.headers.get('date');
  if (!dateHeader) return true;
  
  const date = new Date(dateHeader);
  return Date.now() - date.getTime() > maxAge;
}
```

### 3. Memory Management

```typescript
// src/hooks/useMemoryOptimization.ts
import { useEffect, useRef, useCallback } from 'react';

// Hook for cleaning up resources
export const useCleanup = (cleanup: () => void) => {
  const cleanupRef = useRef(cleanup);
  cleanupRef.current = cleanup;
  
  useEffect(() => {
    return () => {
      cleanupRef.current();
    };
  }, []);
};

// Hook for memory monitoring
export const useMemoryMonitor = () => {
  const checkMemory = useCallback(() => {
    if ('memory' in performance) {
      const memory = (performance as any).memory;
      const usage = {
        used: Math.round(memory.usedJSHeapSize / 1048576), // MB
        total: Math.round(memory.totalJSHeapSize / 1048576), // MB
        limit: Math.round(memory.jsHeapSizeLimit / 1048576), // MB
      };
      
      // Warn if memory usage is high
      if (usage.used > 50) {
        console.warn('High memory usage detected:', usage);
      }
      
      return usage;
    }
    return null;
  }, []);
  
  useEffect(() => {
    const interval = setInterval(checkMemory, 30000); // Check every 30 seconds
    return () => clearInterval(interval);
  }, [checkMemory]);
  
  return { checkMemory };
};

// Hook for object pooling
export const useObjectPool = <T>(createObject: () => T, resetObject: (obj: T) => void) => {
  const pool = useRef<T[]>([]);
  
  const acquire = useCallback(() => {
    if (pool.current.length > 0) {
      return pool.current.pop()!;
    }
    return createObject();
  }, [createObject]);
  
  const release = useCallback((obj: T) => {
    resetObject(obj);
    pool.current.push(obj);
  }, [resetObject]);
  
  const clear = useCallback(() => {
    pool.current = [];
  }, []);
  
  useCleanup(clear);
  
  return { acquire, release, clear };
};
```

## Bundle Optimization

### 1. Webpack Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';
import { visualizer } from 'rollup-plugin-visualizer';
import { splitVendorChunkPlugin } from 'vite';

export default defineConfig({
  plugins: [
    react(),
    splitVendorChunkPlugin(),
    visualizer({
      filename: 'dist/stats.html',
      open: true,
      gzipSize: true,
      brotliSize: true,
    }),
  ],
  
  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
    },
  },
  
  build: {
    // Target modern browsers
    target: 'es2020',
    
    // Optimize chunks
    rollupOptions: {
      output: {
        manualChunks: {
          // Vendor chunks
          'react-vendor': ['react', 'react-dom'],
          'router-vendor': ['react-router-dom'],
          'query-vendor': ['@tanstack/react-query'],
          'ui-vendor': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
          
          // Feature chunks
          'chat': [
            './src/components/chat',
            './src/services/chat.service',
            'socket.io-client',
          ],
          'vendor-dashboard': [
            './src/pages/vendor',
            './src/components/vendor',
          ],
          'moderator-dashboard': [
            './src/pages/moderator',
            './src/components/moderator',
          ],
        },
      },
    },
    
    // Minification
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },
    
    // Source maps for production debugging
    sourcemap: true,
    
    // Chunk size warnings
    chunkSizeWarningLimit: 1000,
  },
  
  // Development optimizations
  server: {
    hmr: {
      overlay: false,
    },
  },
  
  // Dependency optimization
  optimizeDeps: {
    include: [
      'react',
      'react-dom',
      'react-router-dom',
      '@tanstack/react-query',
      'zustand',
      'socket.io-client',
    ],
    exclude: [
      // Large libraries that should be loaded on demand
      'chart.js',
      'pdf-lib',
    ],
  },
});
```

### 2. Tree Shaking Optimization

```typescript
// src/utils/treeShaking.ts

// Use named imports instead of default imports
// ❌ Bad - imports entire library
import * as _ from 'lodash';
import moment from 'moment';

// ✅ Good - only imports needed functions
import { debounce, throttle } from 'lodash-es';
import { format, parseISO } from 'date-fns';

// Create utility modules with specific exports
export { debounce, throttle } from 'lodash-es';
export { format as formatDate, parseISO as parseDate } from 'date-fns';
export { z } from 'zod';

// Conditional imports for development tools
if (process.env.NODE_ENV === 'development') {
  import('./devTools').then(({ setupDevTools }) => {
    setupDevTools();
  });
}

// Feature flags for conditional loading
export const loadFeature = async (featureName: string) => {
  const features = {
    analytics: () => import('./analytics'),
    monitoring: () => import('./monitoring'),
    experiments: () => import('./experiments'),
  };
  
  const feature = features[featureName as keyof typeof features];
  if (feature) {
    return await feature();
  }
  
  throw new Error(`Feature ${featureName} not found`);
};
```

## Performance Monitoring

### 1. Performance Metrics Collection

```typescript
// src/lib/performance.ts

interface PerformanceMetric {
  name: string;
  value: number;
  timestamp: number;
  url: string;
  userAgent: string;
}

class PerformanceMonitor {
  private metrics: PerformanceMetric[] = [];
  private observer: PerformanceObserver | null = null;
  
  constructor() {
    this.setupObserver();
    this.collectInitialMetrics();
  }
  
  private setupObserver(): void {
    if ('PerformanceObserver' in window) {
      this.observer = new PerformanceObserver((list) => {
        list.getEntries().forEach((entry) => {
          this.recordMetric(entry.name, entry.duration || entry.value);
        });
      });
      
      // Observe different types of performance entries
      try {
        this.observer.observe({ entryTypes: ['navigation', 'paint', 'largest-contentful-paint'] });
      } catch (error) {
        console.warn('Performance observer not fully supported:', error);
      }
    }
  }
  
  private collectInitialMetrics(): void {
    // Core Web Vitals
    this.measureCLS();
    this.measureFID();
    this.measureLCP();
    
    // Navigation timing
    this.measureNavigationTiming();
    
    // Resource timing
    this.measureResourceTiming();
  }
  
  private measureCLS(): void {
    let clsValue = 0;
    let clsEntries: any[] = [];
    
    const observer = new PerformanceObserver((list) => {
      list.getEntries().forEach((entry: any) => {
        if (!entry.hadRecentInput) {
          clsEntries.push(entry);
          clsValue += entry.value;
        }
      });
      
      this.recordMetric('CLS', clsValue);
    });
    
    try {
      observer.observe({ entryTypes: ['layout-shift'] });
    } catch (error) {
      console.warn('CLS measurement not supported:', error);
    }
  }
  
  private measureFID(): void {
    const observer = new PerformanceObserver((list) => {
      list.getEntries().forEach((entry: any) => {
        this.recordMetric('FID', entry.processingStart - entry.startTime);
      });
    });
    
    try {
      observer.observe({ entryTypes: ['first-input'] });
    } catch (error) {
      console.warn('FID measurement not supported:', error);
    }
  }
  
  private measureLCP(): void {
    const observer = new PerformanceObserver((list) => {
      list.getEntries().forEach((entry: any) => {
        this.recordMetric('LCP', entry.startTime);
      });
    });
    
    try {
      observer.observe({ entryTypes: ['largest-contentful-paint'] });
    } catch (error) {
      console.warn('LCP measurement not supported:', error);
    }
  }
  
  private measureNavigationTiming(): void {
    if ('performance' in window && 'getEntriesByType' in performance) {
      const navigation = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming;
      
      if (navigation) {
        this.recordMetric('TTFB', navigation.responseStart - navigation.requestStart);
        this.recordMetric('DOM_LOAD', navigation.domContentLoadedEventEnd - navigation.navigationStart);
        this.recordMetric('WINDOW_LOAD', navigation.loadEventEnd - navigation.navigationStart);
      }
    }
  }
  
  private measureResourceTiming(): void {
    if ('performance' in window && 'getEntriesByType' in performance) {
      const resources = performance.getEntriesByType('resource') as PerformanceResourceTiming[];
      
      resources.forEach((resource) => {
        if (resource.name.includes('.js') || resource.name.includes('.css')) {
          this.recordMetric(`RESOURCE_${resource.name}`, resource.duration);
        }
      });
    }
  }
  
  private recordMetric(name: string, value: number): void {
    const metric: PerformanceMetric = {
      name,
      value,
      timestamp: Date.now(),
      url: window.location.href,
      userAgent: navigator.userAgent,
    };
    
    this.metrics.push(metric);
    
    // Send to analytics service
    this.sendMetric(metric);
  }
  
  private async sendMetric(metric: PerformanceMetric): Promise<void> {
    try {
      // Use beacon API for reliable delivery
      if ('sendBeacon' in navigator) {
        navigator.sendBeacon('/api/metrics', JSON.stringify(metric));
      } else {
        // Fallback to fetch
        fetch('/api/metrics', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(metric),
          keepalive: true,
        });
      }
    } catch (error) {
      console.warn('Failed to send performance metric:', error);
    }
  }
  
  // Manual performance measurement
  measureFunction<T>(name: string, fn: () => T): T {
    const start = performance.now();
    const result = fn();
    const end = performance.now();
    
    this.recordMetric(name, end - start);
    return result;
  }
  
  async measureAsyncFunction<T>(name: string, fn: () => Promise<T>): Promise<T> {
    const start = performance.now();
    const result = await fn();
    const end = performance.now();
    
    this.recordMetric(name, end - start);
    return result;
  }
  
  // Get performance summary
  getSummary(): Record<string, { avg: number; min: number; max: number; count: number }> {
    const summary: Record<string, { avg: number; min: number; max: number; count: number }> = {};
    
    this.metrics.forEach((metric) => {
      if (!summary[metric.name]) {
        summary[metric.name] = {
          avg: 0,
          min: Infinity,
          max: -Infinity,
          count: 0,
        };
      }
      
      const stat = summary[metric.name];
      stat.count++;
      stat.min = Math.min(stat.min, metric.value);
      stat.max = Math.max(stat.max, metric.value);
      stat.avg = (stat.avg * (stat.count - 1) + metric.value) / stat.count;
    });
    
    return summary;
  }
  
  // Clear metrics
  clear(): void {
    this.metrics = [];
  }
  
  // Cleanup
  destroy(): void {
    if (this.observer) {
      this.observer.disconnect();
    }
    this.clear();
  }
}

export const performanceMonitor = new PerformanceMonitor();

// React hook for performance monitoring
export const usePerformanceMonitor = () => {
  const measureRender = useCallback((componentName: string) => {
    return performanceMonitor.measureFunction(`RENDER_${componentName}`, () => {
      // This will be called during render
    });
  }, []);
  
  const measureEffect = useCallback(async (effectName: string, effect: () => Promise<void>) => {
    return performanceMonitor.measureAsyncFunction(`EFFECT_${effectName}`, effect);
  }, []);
  
  return { measureRender, measureEffect };
};
```

### 2. Performance Dashboard Component

```typescript
// src/components/dev/PerformanceDashboard.tsx
import { useState, useEffect } from 'react';
import { performanceMonitor } from '../../lib/performance';

export const PerformanceDashboard = () => {
  const [metrics, setMetrics] = useState<Record<string, any>>({});
  const [isVisible, setIsVisible] = useState(false);
  
  useEffect(() => {
    const updateMetrics = () => {
      setMetrics(performanceMonitor.getSummary());
    };
    
    const interval = setInterval(updateMetrics, 5000);
    updateMetrics();
    
    return () => clearInterval(interval);
  }, []);
  
  // Only show in development
  if (process.env.NODE_ENV !== 'development') {
    return null;
  }
  
  return (
    <>
      <button
        onClick={() => setIsVisible(!isVisible)}
        className="fixed bottom-4 right-4 bg-blue-600 text-white p-2 rounded-full shadow-lg z-50"
      >
        📊
      </button>
      
      {isVisible && (
        <div className="fixed bottom-16 right-4 bg-white border rounded-lg shadow-lg p-4 max-w-md max-h-96 overflow-auto z-50">
          <h3 className="text-lg font-semibold mb-4">Performance Metrics</h3>
          
          {Object.entries(metrics).map(([name, stats]) => (
            <div key={name} className="mb-3 p-2 bg-gray-50 rounded">
              <div className="font-medium text-sm">{name}</div>
              <div className="text-xs text-gray-600">
                Avg: {stats.avg.toFixed(2)}ms | 
                Min: {stats.min.toFixed(2)}ms | 
                Max: {stats.max.toFixed(2)}ms | 
                Count: {stats.count}
              </div>
              
              {/* Visual indicator */}
              <div className="mt-1 h-2 bg-gray-200 rounded">
                <div 
                  className={`h-full rounded ${
                    stats.avg < 100 ? 'bg-green-500' :
                    stats.avg < 300 ? 'bg-yellow-500' : 'bg-red-500'
                  }`}
                  style={{ width: `${Math.min(stats.avg / 500 * 100, 100)}%` }}
                />
              </div>
            </div>
          ))}
          
          <button
            onClick={() => performanceMonitor.clear()}
            className="mt-4 px-3 py-1 bg-red-600 text-white text-sm rounded"
          >
            Clear Metrics
          </button>
        </div>
      )}
    </>
  );
};
```

## Performance Best Practices

### 1. Component Optimization Checklist

```typescript
// src/components/optimized/OptimizedComponent.tsx
import { memo, useMemo, useCallback, useState, useEffect } from 'react';

// ✅ Performance optimization checklist:
// 1. Use memo for expensive components
// 2. Memoize callbacks and computed values
// 3. Avoid inline objects and functions
// 4. Use proper dependency arrays
// 5. Implement proper loading states
// 6. Handle errors gracefully
// 7. Clean up resources

interface OptimizedComponentProps {
  data: any[];
  onItemClick: (item: any) => void;
  filters: Record<string, any>;
}

export const OptimizedComponent = memo(({
  data,
  onItemClick,
  filters,
}: OptimizedComponentProps) => {
  const [loading, setLoading] = useState(false);
  
  // ✅ Memoize expensive computations
  const processedData = useMemo(() => {
    return data
      .filter(item => {
        return Object.entries(filters).every(([key, value]) => {
          return !value || item[key] === value;
        });
      })
      .sort((a, b) => a.name.localeCompare(b.name));
  }, [data, filters]);
  
  // ✅ Memoize callbacks
  const handleItemClick = useCallback((item: any) => {
    onItemClick(item);
  }, [onItemClick]);
  
  // ✅ Clean up resources
  useEffect(() => {
    const controller = new AbortController();
    
    const loadData = async () => {
      setLoading(true);
      try {
        // Simulated async operation
        await new Promise(resolve => setTimeout(resolve, 1000));
      } catch (error) {
        if (!controller.signal.aborted) {
          console.error('Failed to load data:', error);
        }
      } finally {
        if (!controller.signal.aborted) {
          setLoading(false);
        }
      }
    };
    
    loadData();
    
    return () => {
      controller.abort();
    };
  }, []);
  
  if (loading) {
    return <div>Loading...</div>;
  }
  
  return (
    <div className="grid gap-4">
      {processedData.map(item => (
        <div
          key={item.id}
          onClick={() => handleItemClick(item)}
          className="p-4 border rounded cursor-pointer hover:shadow-md transition-shadow"
        >
          {item.name}
        </div>
      ))}
    </div>
  );
});

OptimizedComponent.displayName = 'OptimizedComponent';
```

### 2. Performance Testing

```typescript
// src/utils/performanceTesting.ts

// Performance testing utilities
export const performanceTest = {
  // Measure component render time
  measureRender: (componentName: string, renderFn: () => void) => {
    const start = performance.now();
    renderFn();
    const end = performance.now();
    console.log(`${componentName} render time: ${end - start}ms`);
  },
  
  // Measure bundle size impact
  measureBundleSize: async (modulePath: string) => {
    const start = performance.now();
    await import(modulePath);
    const end = performance.now();
    console.log(`${modulePath} load time: ${end - start}ms`);
  },
  
  // Memory usage tracking
  trackMemoryUsage: (label: string) => {
    if ('memory' in performance) {
      const memory = (performance as any).memory;
      console.log(`${label} memory usage:`, {
        used: `${Math.round(memory.usedJSHeapSize / 1048576)}MB`,
        total: `${Math.round(memory.totalJSHeapSize / 1048576)}MB`,
      });
    }
  },
  
  // Network performance
  measureNetworkRequest: async (url: string) => {
    const start = performance.now();
    try {
      const response = await fetch(url);
      const end = performance.now();
      console.log(`${url} request time: ${end - start}ms`);
      return response;
    } catch (error) {
      const end = performance.now();
      console.log(`${url} request failed after: ${end - start}ms`);
      throw error;
    }
  },
};
```

## Next Steps

For more detailed information:
- [Testing Guide](./testing.md) - Performance testing strategies
- [Deployment Guide](./deployment.md) - Production optimization
- [Monitoring Guide](./monitoring.md) - Performance monitoring in production
- [Architecture Guide](./architecture.md) - Performance-focused architecture decisions
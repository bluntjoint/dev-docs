# Routing & Navigation

## Overview

This document provides a comprehensive guide to the routing architecture of the Deed-O Frontend application, including route configuration, navigation patterns, and access control.

## Technology Stack

- **React Router v7**: Latest version with enhanced features
- **Route-based Code Splitting**: Lazy loading for optimal performance
- **Protected Routes**: Role-based access control
- **Dynamic Routing**: Parameter-based route handling

## Route Structure

### Application Route Hierarchy

```
/                           # Home page (public)
├── /login                  # Authentication (public)
├── /register               # User registration (public)
├── /sso-callback           # SSO authentication callback (public)
├── /products               # Product catalog (public)
│   └── /products/:id       # Product details (public)
├── /vendor/                # Vendor dashboard (protected)
│   ├── /overview           # Vendor dashboard home
│   ├── /catalog            # Product management
│   │   ├── /add            # Add new product
│   │   ├── /edit/:id       # Edit existing product
│   │   └── /bulk-upload    # Bulk product upload
│   ├── /inbox              # Vendor messaging
│   │   └── /chat/:groupId  # Specific conversation
│   ├── /settings           # Vendor account settings
│   │   ├── /profile        # Company profile
│   │   ├── /verification   # Document verification
│   │   └── /preferences    # Communication preferences
│   └── /help               # Help and support
├── /moderator/             # Moderator dashboard (protected)
│   ├── /overview           # Moderator dashboard home
│   ├── /catalog            # Product moderation
│   │   ├── /pending        # Pending approvals
│   │   ├── /approved       # Approved products
│   │   ├── /rejected       # Rejected products
│   │   └── /review/:id     # Review specific product
│   ├── /inbox              # Moderator messaging
│   │   └── /chat/:groupId  # Specific conversation
│   ├── /users              # User management
│   │   ├── /vendors        # Vendor list
│   │   ├── /customers      # Customer list
│   │   └── /profile/:id    # User profile details
│   └── /analytics          # Platform analytics
├── /profile                # User profile (protected)
├── /settings               # User settings (protected)
└── /404                    # Not found page
```

## Route Configuration

### Main Router Setup

```typescript
// src/main.tsx or src/App.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { lazy, Suspense } from 'react';

// Lazy load components for code splitting
const Home = lazy(() => import('./pages/home'));
const Login = lazy(() => import('./pages/auth/login'));
const Register = lazy(() => import('./pages/auth/register'));
const SSOCallback = lazy(() => import('./pages/auth/sso-callback'));
const VendorDashboard = lazy(() => import('./pages/vendor'));
const ModeratorDashboard = lazy(() => import('./pages/moderator'));
const ProductCatalog = lazy(() => import('./pages/products'));
const ProductDetails = lazy(() => import('./pages/products/[id]'));
const NotFound = lazy(() => import('./pages/404'));

// Protected route wrapper
const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, isAuthenticated } = useAuth();
  
  if (!isAuthenticated) {
    return <Navigate to="/login" replace />;
  }
  
  if (requiredRole && user?.role !== requiredRole) {
    return <Navigate to="/" replace />;
  }
  
  return children;
};

// Router configuration
const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorBoundary />,
    children: [
      {
        index: true,
        element: <Home />
      },
      {
        path: 'login',
        element: <Login />
      },
      {
        path: 'register',
        element: <Register />
      },
      {
        path: 'sso-callback',
        element: <SSOCallback />
      },
      {
        path: 'products',
        children: [
          {
            index: true,
            element: <ProductCatalog />
          },
          {
            path: ':id',
            element: <ProductDetails />
          }
        ]
      },
      {
        path: 'vendor',
        element: (
          <ProtectedRoute requiredRole="vendor">
            <VendorLayout />
          </ProtectedRoute>
        ),
        children: [
          {
            index: true,
            element: <VendorOverview />
          },
          {
            path: 'catalog',
            children: [
              {
                index: true,
                element: <VendorCatalog />
              },
              {
                path: 'add',
                element: <AddProduct />
              },
              {
                path: 'edit/:id',
                element: <EditProduct />
              }
            ]
          },
          {
            path: 'inbox',
            children: [
              {
                index: true,
                element: <VendorInbox />
              },
              {
                path: 'chat/:groupId',
                element: <Chat />
              }
            ]
          }
        ]
      },
      {
        path: 'moderator',
        element: (
          <ProtectedRoute requiredRole="moderator">
            <ModeratorLayout />
          </ProtectedRoute>
        ),
        children: [
          {
            index: true,
            element: <ModeratorOverview />
          },
          {
            path: 'catalog',
            children: [
              {
                index: true,
                element: <ModeratorCatalog />
              },
              {
                path: 'review/:id',
                element: <ProductReview />
              }
            ]
          }
        ]
      }
    ]
  },
  {
    path: '*',
    element: <NotFound />
  }
]);

// App component
function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <RouterProvider router={router} />
    </Suspense>
  );
}
```

## Route Components

### 1. Public Routes

#### Home Page (`/`)
- **Component**: `src/pages/home/index.tsx`
- **Purpose**: Landing page with platform introduction
- **Features**:
  - Hero section with video
  - Featured causes
  - Platform explanation
  - Call-to-action sections
- **Access**: Public (no authentication required)

#### Authentication Routes

**Login (`/login`)**
- **Component**: `src/pages/auth/login.tsx`
- **Purpose**: User authentication
- **Features**:
  - Email/password login
  - SSO integration
  - Remember me functionality
  - Forgot password link
- **Redirect**: Role-based after successful login

**Register (`/register`)**
- **Component**: `src/pages/auth/register.tsx`
- **Purpose**: New user registration
- **Features**:
  - User type selection (customer/vendor)
  - Form validation
  - Terms acceptance
  - Email verification

**SSO Callback (`/sso-callback`)**
- **Component**: `src/pages/auth/sso-callback.tsx`
- **Purpose**: Handle SSO authentication responses
- **Features**:
  - Token extraction from URL
  - User profile fetching
  - Role-based redirection
  - Error handling

#### Product Routes

**Product Catalog (`/products`)**
- **Component**: `src/pages/products/index.tsx`
- **Purpose**: Browse all products
- **Features**:
  - Search and filtering
  - Pagination
  - Grid/list view toggle
  - Sort options
- **Access**: Public

**Product Details (`/products/:id`)**
- **Component**: `src/pages/products/[id].tsx`
- **Purpose**: Detailed product information
- **Features**:
  - Product images and videos
  - Detailed description
  - Vendor information
  - Contact vendor button
- **Parameters**: `id` - Product identifier

### 2. Protected Routes

#### Vendor Dashboard Routes

**Vendor Overview (`/vendor`)**
- **Component**: `src/pages/vendor/overview.tsx`
- **Purpose**: Vendor dashboard home
- **Features**:
  - Quick stats
  - Recent activity
  - Quick actions
  - Getting started guide
- **Access**: Vendor role required

**Vendor Catalog Management**

```typescript
// Route structure for vendor catalog
/vendor/catalog              # Product list
/vendor/catalog/add          # Add new product
/vendor/catalog/edit/:id     # Edit existing product
/vendor/catalog/bulk-upload  # Bulk upload interface
```

**Vendor Inbox (`/vendor/inbox`)**
- **Component**: `src/pages/vendor/inbox.tsx`
- **Purpose**: Vendor messaging interface
- **Features**:
  - Conversation list
  - Unread message indicators
  - Search conversations
  - Group management

**Vendor Chat (`/vendor/inbox/chat/:groupId`)**
- **Component**: `src/pages/vendor/chat.tsx`
- **Purpose**: Real-time messaging
- **Features**:
  - Message history
  - File attachments
  - Real-time updates
  - Read receipts
- **Parameters**: `groupId` - Conversation identifier

#### Moderator Dashboard Routes

**Moderator Overview (`/moderator`)**
- **Component**: `src/pages/moderator/overview.tsx`
- **Purpose**: Moderator dashboard home
- **Features**:
  - Platform statistics
  - Pending approvals count
  - Recent activity
  - Quick actions
- **Access**: Moderator role required

**Moderator Catalog Management**

```typescript
// Route structure for moderator catalog
/moderator/catalog           # All products overview
/moderator/catalog/pending   # Pending approvals
/moderator/catalog/approved  # Approved products
/moderator/catalog/rejected  # Rejected products
/moderator/catalog/review/:id # Review specific product
```

**Moderator Inbox (`/moderator/inbox`)**
- **Component**: `src/pages/moderator/inbox.tsx`
- **Purpose**: Moderator messaging interface
- **Features**:
  - All platform conversations
  - Priority messaging
  - Escalation handling
  - Bulk actions

## Navigation Patterns

### 1. Programmatic Navigation

```typescript
import { useNavigate, useLocation } from 'react-router-dom';

const MyComponent = () => {
  const navigate = useNavigate();
  const location = useLocation();
  
  // Navigate to a route
  const handleNavigation = () => {
    navigate('/vendor/catalog');
  };
  
  // Navigate with state
  const handleNavigationWithState = () => {
    navigate('/vendor/catalog/add', {
      state: { from: location.pathname }
    });
  };
  
  // Navigate and replace history
  const handleReplace = () => {
    navigate('/login', { replace: true });
  };
  
  // Go back
  const handleGoBack = () => {
    navigate(-1);
  };
  
  return (
    <div>
      <button onClick={handleNavigation}>Go to Catalog</button>
      <button onClick={handleGoBack}>Go Back</button>
    </div>
  );
};
```

### 2. Link Navigation

```typescript
import { Link, NavLink } from 'react-router-dom';

// Basic link
<Link to="/vendor/catalog">View Catalog</Link>

// Link with state
<Link 
  to="/vendor/catalog/add" 
  state={{ returnTo: '/vendor/catalog' }}
>
  Add Product
</Link>

// Navigation link with active state
<NavLink 
  to="/vendor/catalog"
  className={({ isActive }) => 
    isActive ? 'nav-link active' : 'nav-link'
  }
>
  Catalog
</NavLink>
```

### 3. Conditional Navigation

```typescript
const ConditionalNavigation = () => {
  const { user } = useAuth();
  
  return (
    <nav>
      <Link to="/">Home</Link>
      <Link to="/products">Products</Link>
      
      {user ? (
        <>
          {user.role === 'vendor' && (
            <Link to="/vendor">Dashboard</Link>
          )}
          {user.role === 'moderator' && (
            <Link to="/moderator">Admin</Link>
          )}
          <Link to="/profile">Profile</Link>
        </>
      ) : (
        <>
          <Link to="/login">Login</Link>
          <Link to="/register">Register</Link>
        </>
      )}
    </nav>
  );
};
```

## Route Parameters & Query Strings

### 1. URL Parameters

```typescript
import { useParams } from 'react-router-dom';

// Route: /products/:id
const ProductDetails = () => {
  const { id } = useParams<{ id: string }>();
  
  // Use the id parameter
  const { data: product } = useQuery({
    queryKey: ['product', id],
    queryFn: () => fetchProduct(id!),
    enabled: !!id
  });
  
  return (
    <div>
      <h1>Product {id}</h1>
      {product && <ProductInfo product={product} />}
    </div>
  );
};

// Route: /vendor/catalog/edit/:id
const EditProduct = () => {
  const { id } = useParams<{ id: string }>();
  
  return (
    <ProductForm 
      mode="edit" 
      productId={id}
    />
  );
};
```

### 2. Query Parameters

```typescript
import { useSearchParams } from 'react-router-dom';

// Route: /products?search=term&category=electronics&page=2
const ProductCatalog = () => {
  const [searchParams, setSearchParams] = useSearchParams();
  
  // Get query parameters
  const search = searchParams.get('search') || '';
  const category = searchParams.get('category') || '';
  const page = parseInt(searchParams.get('page') || '1');
  
  // Update query parameters
  const handleSearch = (term: string) => {
    setSearchParams(prev => {
      prev.set('search', term);
      prev.set('page', '1'); // Reset to first page
      return prev;
    });
  };
  
  const handleCategoryChange = (cat: string) => {
    setSearchParams(prev => {
      prev.set('category', cat);
      prev.set('page', '1');
      return prev;
    });
  };
  
  return (
    <div>
      <SearchInput value={search} onChange={handleSearch} />
      <CategoryFilter value={category} onChange={handleCategoryChange} />
      <ProductGrid search={search} category={category} page={page} />
    </div>
  );
};
```

### 3. Location State

```typescript
import { useLocation } from 'react-router-dom';

// Passing state during navigation
const ProductList = () => {
  const navigate = useNavigate();
  
  const handleEditProduct = (product: Product) => {
    navigate(`/vendor/catalog/edit/${product.id}`, {
      state: { 
        product,
        returnTo: '/vendor/catalog'
      }
    });
  };
};

// Receiving state in destination component
const EditProduct = () => {
  const location = useLocation();
  const { product, returnTo } = location.state || {};
  
  const handleSave = async () => {
    // Save product
    await saveProduct(product);
    
    // Navigate back to return location
    navigate(returnTo || '/vendor/catalog');
  };
};
```

## Route Guards & Protection

### 1. Authentication Guard

```typescript
const AuthGuard = ({ children }: { children: React.ReactNode }) => {
  const { isAuthenticated, isLoading } = useAuth();
  const location = useLocation();
  
  if (isLoading) {
    return <PageLoader />;
  }
  
  if (!isAuthenticated) {
    // Redirect to login with return URL
    return (
      <Navigate 
        to="/login" 
        state={{ from: location.pathname }}
        replace 
      />
    );
  }
  
  return <>{children}</>;
};
```

### 2. Role-Based Guard

```typescript
interface RoleGuardProps {
  children: React.ReactNode;
  requiredRole: 'vendor' | 'moderator' | 'admin';
  fallback?: React.ReactNode;
}

const RoleGuard = ({ children, requiredRole, fallback }: RoleGuardProps) => {
  const { user, isAuthenticated } = useAuth();
  
  if (!isAuthenticated) {
    return <Navigate to="/login" replace />;
  }
  
  if (user?.role !== requiredRole) {
    return fallback || <Navigate to="/" replace />;
  }
  
  return <>{children}</>;
};

// Usage
<RoleGuard 
  requiredRole="vendor"
  fallback={<UnauthorizedPage />}
>
  <VendorDashboard />
</RoleGuard>
```

### 3. Feature Flag Guard

```typescript
const FeatureGuard = ({ 
  children, 
  feature, 
  fallback 
}: {
  children: React.ReactNode;
  feature: string;
  fallback?: React.ReactNode;
}) => {
  const { isFeatureEnabled } = useFeatureFlags();
  
  if (!isFeatureEnabled(feature)) {
    return fallback || <Navigate to="/" replace />;
  }
  
  return <>{children}</>;
};
```

## Layout Components

### 1. Root Layout

```typescript
const RootLayout = () => {
  return (
    <div className="app">
      <ErrorBoundary>
        <Toaster />
        <Outlet />
      </ErrorBoundary>
    </div>
  );
};
```

### 2. Dashboard Layouts

```typescript
// Vendor layout
const VendorLayout = () => {
  return (
    <div className="dashboard-layout">
      <VendorSidebar />
      <div className="main-content">
        <VendorNavbar />
        <main className="content">
          <Outlet />
        </main>
      </div>
    </div>
  );
};

// Moderator layout
const ModeratorLayout = () => {
  return (
    <div className="dashboard-layout">
      <ModeratorSidebar />
      <div className="main-content">
        <ModeratorNavbar />
        <main className="content">
          <Outlet />
        </main>
      </div>
    </div>
  );
};
```

## Error Handling

### 1. Route Error Boundaries

```typescript
const RouteErrorBoundary = () => {
  const error = useRouteError();
  
  if (isRouteErrorResponse(error)) {
    if (error.status === 404) {
      return <NotFoundPage />;
    }
    
    if (error.status === 403) {
      return <UnauthorizedPage />;
    }
    
    return (
      <ErrorPage 
        title={`${error.status} ${error.statusText}`}
        message={error.data}
      />
    );
  }
  
  return (
    <ErrorPage 
      title="Something went wrong"
      message="An unexpected error occurred"
    />
  );
};
```

### 2. 404 Not Found

```typescript
const NotFoundPage = () => {
  const navigate = useNavigate();
  
  return (
    <div className="not-found">
      <h1>404 - Page Not Found</h1>
      <p>The page you're looking for doesn't exist.</p>
      <div className="actions">
        <button onClick={() => navigate(-1)}>
          Go Back
        </button>
        <Link to="/">Go Home</Link>
      </div>
    </div>
  );
};
```

## Route Preloading

### 1. Link Preloading

```typescript
// Preload route on hover
<Link 
  to="/vendor/catalog"
  onMouseEnter={() => {
    // Preload the route component
    import('../pages/vendor/catalog');
  }}
>
  Catalog
</Link>
```

### 2. Programmatic Preloading

```typescript
const useRoutePreloading = () => {
  const preloadRoute = useCallback((routePath: string) => {
    // Map routes to their components
    const routeMap = {
      '/vendor/catalog': () => import('../pages/vendor/catalog'),
      '/moderator/overview': () => import('../pages/moderator/overview'),
      // Add more routes as needed
    };
    
    const loader = routeMap[routePath];
    if (loader) {
      loader();
    }
  }, []);
  
  return { preloadRoute };
};
```

## Route Analytics

### 1. Page View Tracking

```typescript
const usePageTracking = () => {
  const location = useLocation();
  
  useEffect(() => {
    // Track page view
    analytics.track('page_view', {
      path: location.pathname,
      search: location.search,
      timestamp: new Date().toISOString()
    });
  }, [location]);
};

// Use in root component
const App = () => {
  usePageTracking();
  
  return (
    <RouterProvider router={router} />
  );
};
```

### 2. Route Performance Monitoring

```typescript
const useRoutePerformance = () => {
  const location = useLocation();
  
  useEffect(() => {
    const startTime = performance.now();
    
    return () => {
      const endTime = performance.now();
      const loadTime = endTime - startTime;
      
      analytics.track('route_performance', {
        path: location.pathname,
        loadTime,
        timestamp: new Date().toISOString()
      });
    };
  }, [location]);
};
```

## Best Practices

### 1. Route Organization

- **Nested Routes**: Use nested routes for related functionality
- **Lazy Loading**: Implement code splitting for better performance
- **Consistent Naming**: Use consistent URL patterns and naming conventions
- **SEO Friendly**: Use descriptive URLs that reflect content hierarchy

### 2. Navigation UX

- **Loading States**: Show loading indicators during route transitions
- **Breadcrumbs**: Provide clear navigation context
- **Back Button**: Ensure browser back button works correctly
- **Deep Linking**: Support direct access to any route

### 3. Security

- **Route Protection**: Implement proper authentication and authorization
- **Parameter Validation**: Validate route parameters and query strings
- **CSRF Protection**: Implement CSRF protection for sensitive routes
- **Rate Limiting**: Implement rate limiting for API routes

### 4. Performance

- **Code Splitting**: Split routes into separate bundles
- **Preloading**: Preload critical routes
- **Caching**: Implement proper caching strategies
- **Bundle Analysis**: Monitor and optimize bundle sizes

## Testing Routes

### 1. Route Testing

```typescript
import { render, screen } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { App } from '../App';

describe('Routing', () => {
  it('renders home page on root route', () => {
    render(
      <MemoryRouter initialEntries={['/']}>
        <App />
      </MemoryRouter>
    );
    
    expect(screen.getByText('Welcome to Deed-O')).toBeInTheDocument();
  });
  
  it('redirects to login for protected routes', () => {
    render(
      <MemoryRouter initialEntries={['/vendor']}>
        <App />
      </MemoryRouter>
    );
    
    expect(screen.getByText('Login')).toBeInTheDocument();
  });
  
  it('renders 404 for unknown routes', () => {
    render(
      <MemoryRouter initialEntries={['/unknown-route']}>
        <App />
      </MemoryRouter>
    );
    
    expect(screen.getByText('404 - Page Not Found')).toBeInTheDocument();
  });
});
```

### 2. Navigation Testing

```typescript
import { fireEvent } from '@testing-library/react';

it('navigates to product details when product is clicked', async () => {
  render(
    <MemoryRouter initialEntries={['/products']}>
      <App />
    </MemoryRouter>
  );
  
  const productLink = screen.getByText('Test Product');
  fireEvent.click(productLink);
  
  await waitFor(() => {
    expect(screen.getByText('Product Details')).toBeInTheDocument();
  });
});
```

## Next Steps

For more detailed information:
- [Authentication & Security](./authentication.md) - Route protection details
- [State Management](./state-management.md) - Route-based state handling
- [Performance Guide](./performance.md) - Route optimization techniques
- [Testing Guide](./testing.md) - Comprehensive testing strategies
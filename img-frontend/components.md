# Component Structure & Architecture

## Overview

This document provides a comprehensive guide to the component architecture of the Deed-O Frontend application, including component organization, design patterns, and implementation details.

## Component Organization

### Directory Structure

```
src/components/
├── auth/                    # Authentication components
│   ├── login.tsx           # Login form and logic
│   ├── register.tsx        # User registration
│   └── sso-callback.tsx    # SSO callback handler
├── group/                   # Messaging and group components
│   ├── index.tsx           # Group list and management
│   └── chat.tsx            # Real-time chat interface
├── home/                    # Landing page components
│   ├── index.tsx           # Main home page
│   ├── hero.tsx            # Hero section with video
│   ├── footer.tsx          # Site footer
│   ├── cta.tsx             # Call-to-action sections
│   ├── featured-causes/    # Featured causes showcase
│   ├── what-is-this-place/ # Platform explanation
│   └── why-this-place/     # Value proposition
├── moderator/               # Moderator dashboard components
│   ├── overview/           # Dashboard overview
│   ├── catalog-management/ # Product moderation
│   ├── dashboard-layout.tsx # Layout wrapper
│   ├── navbar.tsx          # Navigation bar
│   ├── sidebar.tsx         # Sidebar navigation
│   ├── sidebar-footer.tsx  # Sidebar footer
│   └── hamburger.tsx       # Mobile menu toggle
├── product/                 # Product-related components
│   ├── index.tsx           # Product listing
│   ├── create.tsx          # Product creation form
│   ├── edit.tsx            # Product editing
│   └── details.tsx         # Product detail view
├── ui/                      # Reusable UI components
│   ├── button.tsx          # Button component
│   ├── input.tsx           # Input components
│   ├── spinner.tsx         # Loading indicators
│   ├── modal.tsx           # Modal dialogs
│   ├── toast.tsx           # Notification toasts
│   └── form/               # Form components
├── vendor/                  # Vendor dashboard components
│   ├── overview/           # Vendor dashboard overview
│   ├── catalog-management/ # Product management
│   ├── help-and-support/   # Support interface
│   ├── dashboard-layout.tsx # Layout wrapper
│   ├── navbar.tsx          # Navigation bar
│   ├── sidebar.tsx         # Sidebar navigation
│   ├── sidebar-footer.tsx  # Sidebar footer
│   └── hamburger.tsx       # Mobile menu toggle
└── layout/                  # Layout components
    ├── header.tsx          # Site header
    ├── footer.tsx          # Site footer
    └── navigation.tsx      # Main navigation
```

## Component Categories

### 1. Authentication Components

#### Login Component (`auth/login.tsx`)
- **Purpose**: User authentication interface
- **Features**:
  - Form validation with Zod schemas
  - SSO integration
  - Error handling and user feedback
  - Responsive design
- **State Management**: Local form state with React Hook Form
- **Dependencies**: React Hook Form, Zod, authentication service

#### Register Component (`auth/register.tsx`)
- **Purpose**: New user registration
- **Features**:
  - Multi-step registration process
  - Role selection (user/vendor)
  - Email verification
  - Terms acceptance
- **Validation**: Comprehensive form validation
- **Integration**: User service API

#### SSO Callback (`auth/sso-callback.tsx`)
- **Purpose**: Handle SSO authentication callbacks
- **Features**:
  - Token extraction and validation
  - User profile fetching
  - Role-based redirection
  - Error handling for failed authentication
- **Flow**: URL params → token validation → profile fetch → redirect

### 2. Group & Messaging Components

#### Group List (`group/index.tsx`)
- **Purpose**: Display and manage conversation groups
- **Features**:
  - Real-time group updates
  - Unread message indicators
  - Group search and filtering
  - Pagination support
- **State Management**: React Query for server state
- **Real-time**: WebSocket integration for live updates

**Key Implementation Details:**
```typescript
// Group component structure
interface GroupProps {
  userProfile: UserProfile;
  userGroups: GroupData[];
  unreadMessages: UnreadMessageData[];
}

// Features:
- Header with user profile
- Searchable group list
- Pagination controls
- Real-time message count updates
```

#### Chat Component (`group/chat.tsx`)
- **Purpose**: Real-time messaging interface
- **Features**:
  - Infinite scroll message history
  - File attachment support (images/videos)
  - Real-time message delivery
  - Read receipt tracking
  - Message status indicators
- **File Handling**: S3 upload integration
- **Real-time**: Socket.io for instant messaging

**Key Implementation Details:**
```typescript
// Chat component features
- Message pagination with useInfiniteQuery
- Real-time message handling via socketService
- File upload with progress tracking
- Message status mutations (read/unread)
- Auto-scroll to new messages
```

### 3. Home Page Components

#### Hero Section (`home/hero.tsx`)
- **Purpose**: Landing page hero with video background
- **Features**:
  - Video play/pause controls
  - Loading state management
  - Responsive video display
  - Call-to-action overlay
- **Media Handling**: Video ref management and controls

#### Footer (`home/footer.tsx`)
- **Purpose**: Site-wide footer navigation
- **Features**:
  - Navigation links
  - Social media links
  - Legal page links
  - Contact information
- **Responsive**: Mobile-optimized layout

### 4. Product Components

#### Product Listing (`product/index.tsx`)
- **Purpose**: Display product catalog
- **Features**:
  - Grid/list view toggle
  - Search and filtering
  - Pagination
  - Sort options
  - Responsive grid layout
- **Performance**: Virtualized scrolling for large lists

#### Product Creation (`product/create.tsx`)
- **Purpose**: Create new products
- **Features**:
  - Multi-step form wizard
  - Image/video upload
  - Draft saving
  - Validation and error handling
  - Preview functionality
- **File Management**: Multiple file upload with validation

#### Product Editing (`product/edit.tsx`)
- **Purpose**: Edit existing products
- **Features**:
  - Pre-populated form data
  - Change tracking
  - Auto-save functionality
  - Version history
  - Approval workflow

### 5. Dashboard Layout Components

#### Vendor Dashboard Layout (`vendor/dashboard-layout.tsx`)
- **Purpose**: Layout wrapper for vendor pages
- **Features**:
  - Responsive sidebar
  - Mobile hamburger menu
  - Breadcrumb navigation
  - User profile dropdown
- **Responsive**: Mobile-first design with collapsible sidebar

#### Moderator Dashboard Layout (`moderator/dashboard-layout.tsx`)
- **Purpose**: Layout wrapper for moderator pages
- **Features**:
  - Admin-specific navigation
  - Quick action buttons
  - Notification center
  - System status indicators

### 6. UI Components Library

#### Button Component (`ui/button.tsx`)
- **Purpose**: Reusable button component
- **Variants**: Primary, secondary, outline, ghost, destructive
- **Sizes**: Small, medium, large
- **States**: Default, hover, active, disabled, loading
- **Implementation**: Class Variance Authority (CVA) for styling

**Button Variants:**
```typescript
const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent",
        secondary: "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline"
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10"
      }
    }
  }
)
```

#### Spinner Component (`ui/spinner.tsx`)
- **Purpose**: Loading indicators
- **Variants**: Default spinner, loading component with text
- **Sizes**: Small, medium, large
- **Usage**: Page loading, button loading states, data fetching

#### Input Components (`ui/input.tsx`)
- **Purpose**: Form input elements
- **Types**: Text, email, password, number, file
- **Features**: Validation states, error messages, icons
- **Accessibility**: ARIA labels and descriptions

## Component Design Patterns

### 1. Composition Pattern

**Example: Dashboard Layout**
```typescript
// Flexible layout composition
<DashboardLayout>
  <DashboardLayout.Header>
    <Navbar />
  </DashboardLayout.Header>
  <DashboardLayout.Sidebar>
    <Sidebar />
  </DashboardLayout.Sidebar>
  <DashboardLayout.Content>
    {children}
  </DashboardLayout.Content>
</DashboardLayout>
```

### 2. Render Props Pattern

**Example: Data Fetching Components**
```typescript
// Flexible data rendering
<ProductList>
  {({ products, loading, error }) => (
    loading ? <Spinner /> : 
    error ? <ErrorMessage error={error} /> :
    <ProductGrid products={products} />
  )}
</ProductList>
```

### 3. Compound Components

**Example: Modal Component**
```typescript
// Compound modal structure
<Modal>
  <Modal.Header>
    <Modal.Title>Confirm Action</Modal.Title>
    <Modal.Close />
  </Modal.Header>
  <Modal.Body>
    <p>Are you sure you want to delete this item?</p>
  </Modal.Body>
  <Modal.Footer>
    <Button variant="outline">Cancel</Button>
    <Button variant="destructive">Delete</Button>
  </Modal.Footer>
</Modal>
```

### 4. Higher-Order Components (HOCs)

**Example: Authentication HOC**
```typescript
// Protect components with authentication
const withAuth = (Component: React.ComponentType) => {
  return (props: any) => {
    const { isAuthenticated } = useAuth();
    
    if (!isAuthenticated) {
      return <Navigate to="/login" />;
    }
    
    return <Component {...props} />;
  };
};
```

### 5. Custom Hooks Pattern

**Example: Data Fetching Hook**
```typescript
// Reusable data fetching logic
const useProducts = (filters: ProductFilters) => {
  return useQuery({
    queryKey: ['products', filters],
    queryFn: () => fetchProducts(filters),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
};
```

## State Management Patterns

### 1. Server State (React Query)

**Usage**: API data, caching, background updates
```typescript
// Product data management
const {
  data: products,
  isLoading,
  error,
  refetch
} = useQuery({
  queryKey: ['products', { page, search, filters }],
  queryFn: ({ queryKey }) => fetchProducts(queryKey[1]),
  keepPreviousData: true,
});
```

### 2. Global State (Zustand)

**Usage**: User authentication, app-wide settings
```typescript
// Authentication store
const useAuthStore = create<AuthState>((set) => ({
  user: null,
  isAuthenticated: false,
  login: (user) => set({ user, isAuthenticated: true }),
  logout: () => set({ user: null, isAuthenticated: false }),
}));
```

### 3. Local State (useState)

**Usage**: Component-specific state, form inputs, UI state
```typescript
// Form state management
const [formData, setFormData] = useState({
  title: '',
  description: '',
  images: []
});
```

### 4. Form State (React Hook Form)

**Usage**: Complex forms, validation, submission
```typescript
// Form with validation
const {
  register,
  handleSubmit,
  formState: { errors, isSubmitting },
  watch,
  setValue
} = useForm<ProductFormData>({
  resolver: zodResolver(productSchema),
  defaultValues: initialData
});
```

## Component Communication Patterns

### 1. Props Down, Events Up

**Parent to Child**: Props
**Child to Parent**: Callback functions

```typescript
// Parent component
const ProductList = () => {
  const [selectedProduct, setSelectedProduct] = useState(null);
  
  return (
    <ProductGrid 
      products={products}
      onProductSelect={setSelectedProduct}
    />
  );
};

// Child component
const ProductGrid = ({ products, onProductSelect }) => {
  return (
    <div className="grid">
      {products.map(product => (
        <ProductCard 
          key={product.id}
          product={product}
          onClick={() => onProductSelect(product)}
        />
      ))}
    </div>
  );
};
```

### 2. Context for Deep Prop Drilling

```typescript
// Theme context
const ThemeContext = createContext();

const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
};
```

### 3. Event Emitters for Loose Coupling

```typescript
// Global event system
const eventEmitter = new EventTarget();

// Emit events
eventEmitter.dispatchEvent(new CustomEvent('product:created', {
  detail: { product }
}));

// Listen for events
useEffect(() => {
  const handleProductCreated = (event) => {
    // Handle product creation
  };
  
  eventEmitter.addEventListener('product:created', handleProductCreated);
  
  return () => {
    eventEmitter.removeEventListener('product:created', handleProductCreated);
  };
}, []);
```

## Performance Optimization Patterns

### 1. Memoization

```typescript
// Memoize expensive calculations
const ProductCard = memo(({ product }) => {
  const formattedPrice = useMemo(() => {
    return formatCurrency(product.price);
  }, [product.price]);
  
  return (
    <div className="product-card">
      <h3>{product.title}</h3>
      <p>{formattedPrice}</p>
    </div>
  );
});
```

### 2. Lazy Loading

```typescript
// Lazy load route components
const ProductDetails = lazy(() => import('./ProductDetails'));
const VendorDashboard = lazy(() => import('./VendorDashboard'));

// Usage with Suspense
<Suspense fallback={<Spinner />}>
  <Routes>
    <Route path="/product/:id" element={<ProductDetails />} />
    <Route path="/vendor" element={<VendorDashboard />} />
  </Routes>
</Suspense>
```

### 3. Virtual Scrolling

```typescript
// Virtual scrolling for large lists
const VirtualProductList = ({ products }) => {
  return (
    <FixedSizeList
      height={600}
      itemCount={products.length}
      itemSize={120}
      itemData={products}
    >
      {ProductRow}
    </FixedSizeList>
  );
};
```

## Error Handling Patterns

### 1. Error Boundaries

```typescript
// Component error boundary
class ProductErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }
  
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }
  
  componentDidCatch(error, errorInfo) {
    console.error('Product component error:', error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return <ProductErrorFallback />;
    }
    
    return this.props.children;
  }
}
```

### 2. Query Error Handling

```typescript
// React Query error handling
const { data, error, isError } = useQuery({
  queryKey: ['products'],
  queryFn: fetchProducts,
  retry: (failureCount, error) => {
    // Retry logic
    return failureCount < 3 && error.status !== 404;
  },
  onError: (error) => {
    // Global error handling
    toast.error('Failed to load products');
  }
});

if (isError) {
  return <ErrorMessage error={error} retry={() => refetch()} />;
}
```

## Testing Patterns

### 1. Component Testing

```typescript
// Component test example
describe('ProductCard', () => {
  it('displays product information correctly', () => {
    const product = {
      id: '1',
      title: 'Test Product',
      price: 99.99,
      image: 'test.jpg'
    };
    
    render(<ProductCard product={product} />);
    
    expect(screen.getByText('Test Product')).toBeInTheDocument();
    expect(screen.getByText('$99.99')).toBeInTheDocument();
  });
  
  it('calls onSelect when clicked', () => {
    const onSelect = jest.fn();
    const product = { id: '1', title: 'Test' };
    
    render(<ProductCard product={product} onSelect={onSelect} />);
    
    fireEvent.click(screen.getByRole('button'));
    
    expect(onSelect).toHaveBeenCalledWith(product);
  });
});
```

### 2. Hook Testing

```typescript
// Custom hook testing
describe('useProducts', () => {
  it('fetches products successfully', async () => {
    const { result } = renderHook(() => useProducts({ page: 1 }));
    
    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });
    
    expect(result.current.data).toHaveLength(10);
  });
});
```

## Accessibility Patterns

### 1. Semantic HTML

```typescript
// Proper semantic structure
const ProductCard = ({ product }) => {
  return (
    <article className="product-card">
      <header>
        <h3>{product.title}</h3>
      </header>
      <img 
        src={product.image} 
        alt={`Image of ${product.title}`}
      />
      <footer>
        <button 
          aria-label={`Add ${product.title} to cart`}
          onClick={handleAddToCart}
        >
          Add to Cart
        </button>
      </footer>
    </article>
  );
};
```

### 2. Keyboard Navigation

```typescript
// Keyboard event handling
const Modal = ({ isOpen, onClose, children }) => {
  useEffect(() => {
    const handleEscape = (event) => {
      if (event.key === 'Escape') {
        onClose();
      }
    };
    
    if (isOpen) {
      document.addEventListener('keydown', handleEscape);
      // Focus management
      const firstFocusable = modalRef.current?.querySelector('[tabindex="0"]');
      firstFocusable?.focus();
    }
    
    return () => {
      document.removeEventListener('keydown', handleEscape);
    };
  }, [isOpen, onClose]);
  
  return (
    <div 
      role="dialog" 
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      {children}
    </div>
  );
};
```

## Component Documentation Standards

### 1. Component Props Interface

```typescript
/**
 * ProductCard component for displaying product information
 * 
 * @param product - Product data object
 * @param onSelect - Callback when product is selected
 * @param variant - Display variant (card, list, compact)
 * @param showActions - Whether to show action buttons
 */
interface ProductCardProps {
  product: Product;
  onSelect?: (product: Product) => void;
  variant?: 'card' | 'list' | 'compact';
  showActions?: boolean;
  className?: string;
}
```

### 2. Component Examples

```typescript
// Usage examples in comments
/**
 * @example
 * // Basic usage
 * <ProductCard product={product} />
 * 
 * @example
 * // With selection handler
 * <ProductCard 
 *   product={product} 
 *   onSelect={handleProductSelect}
 *   variant="list"
 * />
 */
```

## Next Steps

For more detailed information:
- [Real-time Communication](./real-time-communication.md) - WebSocket implementation
- [State Management](./state-management.md) - Detailed state patterns
- [Testing Guide](./testing.md) - Component testing strategies
- [Performance Guide](./performance.md) - Optimization techniques
# Backend Integration

## Overview

The Deed-O Frontend integrates with multiple backend services to provide a comprehensive platform experience. This document outlines all API integrations, service communications, and data flow patterns.

## Environment Configuration

### Environment Variables

```bash
# Main Backend API
VITE_BACKEND=https://api.deed-o.com

# Authentication Service
VITE_AUTH_BACKEND=https://auth.deed-o.com

# WebSocket Backend
VITE_WS_BACKEND=wss://ws.deed-o.com

# Single Sign-On Configuration
VITE_SSO_URL=https://sso.deed-o.com/auth
VITE_SSO_CALLBACK_URL=https://app.deed-o.com/sso-login
```

### API Configuration

```typescript
// src/constants/app.ts
export const BACKEND_URL = import.meta.env.VITE_BACKEND;
export const BACKEND_AUTH_URL = import.meta.env.VITE_AUTH_BACKEND;
export const WS_BACKEND_URL = import.meta.env.VITE_WS_BACKEND;
```

## HTTP Client Configuration

### Axios Instances

```typescript
// src/services/api.ts

// General API client
export const api = axios.create({
  baseURL: BACKEND_URL,
  timeout: 10000,
});

// Authentication API client
export const authApi = axios.create({
  baseURL: BACKEND_AUTH_URL,
  timeout: 10000,
});

// Request interceptor for authentication
api.interceptors.request.use((config) => {
  const token = getAuthToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

### Error Handling

```typescript
// Global error interceptor
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Handle unauthorized access
      removeAuthToken();
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

## Service Layer Architecture

### Authentication Services

```typescript
// src/services/auth.ts

// User Authentication
export const loginUser = async (credentials: LoginCredentials) => {
  const response = await authApi.post('/login', credentials);
  return response.data;
};

export const registerUser = async (userData: UserRegistration) => {
  const response = await authApi.post('/register', userData);
  return response.data;
};

// Vendor Authentication
export const loginVendor = async (credentials: VendorCredentials) => {
  const response = await authApi.post('/vendors/login', credentials);
  return response.data;
};

export const registerVendor = async (vendorData: VendorRegistration) => {
  const response = await authApi.post('/vendors/register', vendorData);
  return response.data;
};

export const verifyVendor = async (verificationData: VendorVerification) => {
  const response = await authApi.post('/vendors/verify', verificationData);
  return response.data;
};
```

### Product Management Services

```typescript
// src/services/product.service.ts

interface FetchProductsOptions {
  page?: number;
  limit?: number;
  causes?: string[];
  search?: string;
  publishingStatus?: string;
}

export const fetchProducts = async (options: FetchProductsOptions = {}) => {
  const response = await api.get('/products', { params: options });
  return response.data;
};

export const createProduct = async (productData: ProductCreation) => {
  const response = await api.post('/products', productData);
  return response.data;
};

export const updateProductById = async (id: string, updates: ProductUpdate) => {
  const response = await api.put(`/products/${id}`, updates);
  return response.data;
};

export const deleteProductById = async (id: string) => {
  const response = await api.delete(`/products/${id}`);
  return response.data;
};
```

### Vendor-Specific Services

```typescript
// src/services/vendor.service.ts

export const fetchVendorProducts = async (
  page = 1,
  limit = 10,
  options?: {
    search?: string;
    publishingStatus?: string;
  }
) => {
  const response = await api.get('/products/', {
    params: {
      page,
      limit,
      vendorView: true,
      publishingStatus: options?.publishingStatus,
      search: options?.search,
    },
  });
  return response.data;
};
```

### Verification Services

#### Aadhar Verification

```typescript
// src/services/aadhar.service.ts

export const sendAadharOTP = async (aadharNo: string) => {
  const response = await api.post('/vendors/aadhar/otp', { aadharNo });
  return response.data;
};

export const verifyAadharOTP = async (
  otp: string,
  refId: string,
  kybId: string
) => {
  const response = await api.post('/vendors/aadhar/verify', {
    otp,
    refId,
    kybId,
  });
  return response.data;
};
```

#### GST/PAN Verification

```typescript
// src/services/gst-pan-service.ts

export const fetchDocumentDetails = async (
  documentType: 'gstin' | 'pan',
  documentNumber: string,
  kybId: string
) => {
  const endpoint = documentType === 'gstin' ? 'gstin' : 'pan';
  const requestData = documentType === 'gstin'
    ? { gstn: documentNumber, kybId }
    : { pan: documentNumber, kybId };

  const response = await api.post(`/vendors/${endpoint}`, requestData);
  return response.data?.[0] || null;
};
```

### File Upload Services

```typescript
// src/services/s3.service.ts

export const uploadResourceToS3 = async (
  uploadType: 'single' | 'multiple',
  files: File | File[],
  folderPath: string
) => {
  const formData = new FormData();
  
  if (uploadType === 'single') {
    formData.append('file', files as File);
  } else {
    (files as File[]).forEach((file) => {
      formData.append('files', file);
    });
  }
  
  formData.append('folderPath', folderPath);

  const response = await api.post('/upload', formData, {
    headers: {
      'Content-Type': 'multipart/form-data',
    },
  });
  
  return response.data;
};
```

## Real-time Communication

### WebSocket Integration

```typescript
// src/services/chat.service.ts

class SocketService {
  private socket: Socket | null = null;

  connect(userId: string) {
    this.socket = io(WS_BACKEND_URL, {
      auth: {
        userId,
        token: getAuthToken(),
      },
    });

    this.socket.on('connect', () => {
      console.log('Connected to WebSocket server');
    });

    this.socket.on('disconnect', () => {
      console.log('Disconnected from WebSocket server');
    });
  }

  sendMessage(message: MessageData) {
    if (this.socket) {
      this.socket.emit('sendMessage', message);
    }
  }

  onNewMessage(callback: (message: Message) => void) {
    if (this.socket) {
      this.socket.on('newMessage', callback);
    }
  }

  disconnect() {
    if (this.socket) {
      this.socket.disconnect();
      this.socket = null;
    }
  }
}

export const socketService = new SocketService();
```

### Group Management Services

```typescript
// src/services/group.service.ts

export const fetchUserGroups = async (
  userId: number,
  page: number = 1,
  limit: number = 10
) => {
  const response = await api.get(`/groups/user/${userId}`, {
    params: { page, limit },
  });
  return response.data;
};

export const getOrCreateGroup = async (
  name: string,
  productId: string,
  members: number[]
) => {
  const response = await api.post('/groups', {
    name,
    productId,
    members,
  });
  return response.data;
};

export const sendMessage = async (messageData: MessageCreation) => {
  const response = await api.post('/messages', messageData);
  return response.data;
};

export const fetchGroupMessages = async (
  groupId: string,
  page: number = 1,
  limit: number = 10
) => {
  const response = await api.get(`/groups/${groupId}/messages`, {
    params: { page, limit },
  });
  return response.data;
};
```

## Data Models & Types

### Product Types

```typescript
// src/types/products.ts

export interface Product {
  id: string;
  title: string;
  description: string;
  cause: string;
  mrp: number;
  sellingPrice: number;
  media: string[];
  publishingStatus: 'draft' | 'published' | 'rejected';
  vendorID: string;
  productUrl?: string;
  remarks?: string;
  createdAt: string;
  updatedAt: string;
}
```

### User Types

```typescript
// src/types/user.ts

export interface User {
  id: number;
  email: string;
  name: string;
  phone?: string;
  address?: {
    street: string;
    city: string;
    state: string;
    zipCode: string;
    country: string;
  };
  dateOfBirth?: string;
  gender?: 'male' | 'female' | 'other';
  role: 'user' | 'vendor' | 'moderator';
  isOnline: boolean;
  createdAt: string;
  updatedAt: string;
}
```

### Group & Message Types

```typescript
// src/types/group.ts

export interface Message {
  id: string;
  senderId: number;
  groupId: string;
  text: string;
  attachments: {
    url: string;
    type: string;
  }[];
  timestamp: string;
  readBy: number[];
}

export interface Group {
  id: string;
  name: string;
  members: User[];
  productId?: string;
  lastMessageData?: {
    text: string;
    timestamp: string;
    senderId: number;
  };
  createdAt: string;
  updatedAt: string;
}
```

## React Query Integration

### Query Configuration

```typescript
// Query for products with caching
const { data: products, isLoading, error } = useQuery({
  queryKey: ['products', filters],
  queryFn: () => fetchProducts(filters),
  staleTime: 5 * 60 * 1000, // 5 minutes
  cacheTime: 10 * 60 * 1000, // 10 minutes
});

// Infinite query for chat messages
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage,
} = useInfiniteQuery({
  queryKey: ['chatMessages', groupId],
  queryFn: ({ pageParam = 1 }) => fetchGroupMessages(groupId, pageParam),
  getNextPageParam: (lastPage, allPages) => 
    lastPage.length < MESSAGES_PER_PAGE ? undefined : allPages.length + 1,
});
```

### Mutations

```typescript
// Product creation mutation
const createProductMutation = useMutation({
  mutationFn: createProduct,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['products'] });
    toast.success('Product created successfully!');
  },
  onError: (error) => {
    toast.error('Failed to create product');
  },
});

// Message sending mutation
const sendMessageMutation = useMutation({
  mutationFn: sendMessage,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['chatMessages'] });
  },
});
```

## Error Handling Patterns

### Service-Level Error Handling

```typescript
export const fetchProducts = async (options: FetchProductsOptions) => {
  try {
    const response = await api.get('/products', { params: options });
    return response.data;
  } catch (error: any) {
    throw new Error(
      error.response?.data?.message || 'Error fetching products'
    );
  }
};
```

### Component-Level Error Handling

```typescript
const { data, error, isLoading } = useQuery({
  queryKey: ['products'],
  queryFn: fetchProducts,
  retry: 3,
  retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
});

if (error) {
  return <ErrorMessage message="Failed to load products" />;
}
```

## Authentication Flow

### SSO Integration

```typescript
// Login redirect to SSO
const handleLogin = () => {
  window.location.href = `${SSO_URL}?callback=${SSO_CALLBACK_URL}`;
};

// SSO callback handling
const handleSSOCallback = () => {
  const urlParams = new URLSearchParams(window.location.search);
  const token = urlParams.get('token');
  
  if (token) {
    setAuthToken(token);
    navigate('/dashboard');
  } else {
    navigate('/login');
  }
};
```

### Token Management

```typescript
// src/lib/storage.ts

export const setAuthToken = (token: string) => {
  localStorage.setItem('authToken', token);
};

export const getAuthToken = (): string | null => {
  return localStorage.getItem('authToken');
};

export const removeAuthToken = () => {
  localStorage.removeItem('authToken');
};
```

## Performance Optimizations

### Request Optimization
- Request deduplication with React Query
- Automatic retries with exponential backoff
- Background refetching for stale data
- Optimistic updates for better UX

### Caching Strategy
- Query result caching
- Infinite query pagination
- Selective cache invalidation
- Background data synchronization

### Error Recovery
- Automatic retry mechanisms
- Fallback data strategies
- Graceful degradation
- User-friendly error messages

## Security Considerations

### API Security
- JWT token authentication
- Automatic token refresh
- Secure token storage
- Request/response validation

### Data Protection
- Input sanitization
- XSS prevention
- CSRF protection
- Secure file uploads

## Monitoring & Debugging

### Development Tools
- React Query DevTools
- Network request logging
- Error boundary implementation
- Performance monitoring

### Production Monitoring
- API response time tracking
- Error rate monitoring
- User experience metrics
- Real-time system health

## Next Steps

For more detailed information:
- [Authentication & Security](./authentication.md) - Security implementation details
- [Real-time Communication](./real-time-communication.md) - WebSocket and messaging
- [API Reference](./api-reference.md) - Complete API documentation
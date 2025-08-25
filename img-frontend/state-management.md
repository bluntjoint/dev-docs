# State Management

## Overview

This document provides a comprehensive guide to state management in the Deed-O Frontend application, covering different types of state, management patterns, and implementation details.

## State Management Architecture

### Technology Stack

- **Zustand**: Global application state management
- **TanStack React Query**: Server state management and caching
- **React Hook Form**: Form state management
- **React useState**: Local component state
- **React useReducer**: Complex local state logic
- **Context API**: Theme and configuration state

### State Categories

```
Application State
├── Server State (React Query)
│   ├── API Data Caching
│   ├── Background Synchronization
│   ├── Optimistic Updates
│   └── Error Handling
├── Global State (Zustand)
│   ├── Authentication State
│   ├── User Preferences
│   ├── Application Settings
│   └── UI State
├── Form State (React Hook Form)
│   ├── Form Validation
│   ├── Field Management
│   ├── Submission Handling
│   └── Error States
├── Local State (useState/useReducer)
│   ├── Component UI State
│   ├── Modal Visibility
│   ├── Loading States
│   └── Temporary Data
└── Context State (React Context)
    ├── Theme Configuration
    ├── Feature Flags
    ├── Localization
    └── Provider Settings
```

## Global State Management (Zustand)

### 1. Authentication Store

```typescript
// src/stores/auth.store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface User {
  id: string;
  email: string;
  name: string;
  role: 'vendor' | 'moderator' | 'customer';
  avatar?: string;
  isVerified: boolean;
}

interface AuthState {
  // State
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  
  // Actions
  login: (user: User, token: string) => void;
  logout: () => void;
  updateUser: (updates: Partial<User>) => void;
  setLoading: (loading: boolean) => void;
  
  // Computed
  isVendor: () => boolean;
  isModerator: () => boolean;
  isCustomer: () => boolean;
}

export const useAuthStore = create<AuthState>()(n  persist(
    (set, get) => ({
      // Initial state
      user: null,
      token: null,
      isAuthenticated: false,
      isLoading: false,
      
      // Actions
      login: (user, token) => {
        set({ 
          user, 
          token, 
          isAuthenticated: true,
          isLoading: false 
        });
      },
      
      logout: () => {
        set({ 
          user: null, 
          token: null, 
          isAuthenticated: false,
          isLoading: false 
        });
        // Clear other stores if needed
        useNotificationStore.getState().clearNotifications();
      },
      
      updateUser: (updates) => {
        const currentUser = get().user;
        if (currentUser) {
          set({ user: { ...currentUser, ...updates } });
        }
      },
      
      setLoading: (loading) => {
        set({ isLoading: loading });
      },
      
      // Computed values
      isVendor: () => get().user?.role === 'vendor',
      isModerator: () => get().user?.role === 'moderator',
      isCustomer: () => get().user?.role === 'customer',
    }),
    {
      name: 'auth-storage',
      partialize: (state) => ({
        user: state.user,
        token: state.token,
        isAuthenticated: state.isAuthenticated,
      }),
    }
  )
);

// Selectors for performance optimization
export const useAuth = () => useAuthStore((state) => ({
  user: state.user,
  isAuthenticated: state.isAuthenticated,
  isLoading: state.isLoading,
  login: state.login,
  logout: state.logout,
  updateUser: state.updateUser,
}));

export const useUserRole = () => useAuthStore((state) => ({
  role: state.user?.role,
  isVendor: state.isVendor(),
  isModerator: state.isModerator(),
  isCustomer: state.isCustomer(),
}));
```

### 2. UI State Store

```typescript
// src/stores/ui.store.ts
interface UIState {
  // Sidebar state
  sidebarOpen: boolean;
  sidebarCollapsed: boolean;
  
  // Modal state
  modals: Record<string, boolean>;
  
  // Loading states
  globalLoading: boolean;
  loadingStates: Record<string, boolean>;
  
  // Notifications
  notifications: Notification[];
  
  // Theme
  theme: 'light' | 'dark' | 'system';
  
  // Actions
  toggleSidebar: () => void;
  setSidebarCollapsed: (collapsed: boolean) => void;
  openModal: (modalId: string) => void;
  closeModal: (modalId: string) => void;
  setLoading: (key: string, loading: boolean) => void;
  addNotification: (notification: Omit<Notification, 'id'>) => void;
  removeNotification: (id: string) => void;
  setTheme: (theme: 'light' | 'dark' | 'system') => void;
}

export const useUIStore = create<UIState>()(n  persist(
    (set, get) => ({
      // Initial state
      sidebarOpen: true,
      sidebarCollapsed: false,
      modals: {},
      globalLoading: false,
      loadingStates: {},
      notifications: [],
      theme: 'system',
      
      // Actions
      toggleSidebar: () => {
        set((state) => ({ sidebarOpen: !state.sidebarOpen }));
      },
      
      setSidebarCollapsed: (collapsed) => {
        set({ sidebarCollapsed: collapsed });
      },
      
      openModal: (modalId) => {
        set((state) => ({
          modals: { ...state.modals, [modalId]: true }
        }));
      },
      
      closeModal: (modalId) => {
        set((state) => ({
          modals: { ...state.modals, [modalId]: false }
        }));
      },
      
      setLoading: (key, loading) => {
        set((state) => ({
          loadingStates: { ...state.loadingStates, [key]: loading }
        }));
      },
      
      addNotification: (notification) => {
        const id = Date.now().toString();
        set((state) => ({
          notifications: [...state.notifications, { ...notification, id }]
        }));
        
        // Auto-remove after 5 seconds
        setTimeout(() => {
          get().removeNotification(id);
        }, 5000);
      },
      
      removeNotification: (id) => {
        set((state) => ({
          notifications: state.notifications.filter(n => n.id !== id)
        }));
      },
      
      setTheme: (theme) => {
        set({ theme });
        // Apply theme to document
        document.documentElement.setAttribute('data-theme', theme);
      },
    }),
    {
      name: 'ui-storage',
      partialize: (state) => ({
        sidebarCollapsed: state.sidebarCollapsed,
        theme: state.theme,
      }),
    }
  )
);
```

### 3. Chat State Store

```typescript
// src/stores/chat.store.ts
interface ChatState {
  // Active conversations
  activeConversations: Record<string, Conversation>;
  
  // Current chat
  currentChatId: string | null;
  
  // Real-time messages
  realTimeMessages: Record<string, Message[]>;
  
  // Typing indicators
  typingUsers: Record<string, string[]>;
  
  // Unread counts
  unreadCounts: Record<string, number>;
  
  // Actions
  setCurrentChat: (chatId: string | null) => void;
  addMessage: (chatId: string, message: Message) => void;
  updateMessage: (chatId: string, messageId: string, updates: Partial<Message>) => void;
  setTyping: (chatId: string, userId: string, isTyping: boolean) => void;
  markAsRead: (chatId: string) => void;
  incrementUnreadCount: (chatId: string) => void;
}

export const useChatStore = create<ChatState>((set, get) => ({
  // Initial state
  activeConversations: {},
  currentChatId: null,
  realTimeMessages: {},
  typingUsers: {},
  unreadCounts: {},
  
  // Actions
  setCurrentChat: (chatId) => {
    set({ currentChatId: chatId });
    if (chatId) {
      get().markAsRead(chatId);
    }
  },
  
  addMessage: (chatId, message) => {
    set((state) => ({
      realTimeMessages: {
        ...state.realTimeMessages,
        [chatId]: [...(state.realTimeMessages[chatId] || []), message]
      }
    }));
    
    // Increment unread count if not current chat
    if (get().currentChatId !== chatId) {
      get().incrementUnreadCount(chatId);
    }
  },
  
  updateMessage: (chatId, messageId, updates) => {
    set((state) => ({
      realTimeMessages: {
        ...state.realTimeMessages,
        [chatId]: state.realTimeMessages[chatId]?.map(msg =>
          msg.id === messageId ? { ...msg, ...updates } : msg
        ) || []
      }
    }));
  },
  
  setTyping: (chatId, userId, isTyping) => {
    set((state) => {
      const currentTyping = state.typingUsers[chatId] || [];
      const newTyping = isTyping
        ? [...currentTyping.filter(id => id !== userId), userId]
        : currentTyping.filter(id => id !== userId);
      
      return {
        typingUsers: {
          ...state.typingUsers,
          [chatId]: newTyping
        }
      };
    });
  },
  
  markAsRead: (chatId) => {
    set((state) => ({
      unreadCounts: {
        ...state.unreadCounts,
        [chatId]: 0
      }
    }));
  },
  
  incrementUnreadCount: (chatId) => {
    set((state) => ({
      unreadCounts: {
        ...state.unreadCounts,
        [chatId]: (state.unreadCounts[chatId] || 0) + 1
      }
    }));
  },
}));
```

## Server State Management (React Query)

### 1. Query Configuration

```typescript
// src/lib/react-query.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Stale time: 5 minutes
      staleTime: 5 * 60 * 1000,
      // Cache time: 10 minutes
      cacheTime: 10 * 60 * 1000,
      // Retry failed requests 3 times
      retry: 3,
      // Retry delay
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
      // Refetch on window focus
      refetchOnWindowFocus: false,
      // Refetch on reconnect
      refetchOnReconnect: true,
    },
    mutations: {
      // Retry failed mutations once
      retry: 1,
    },
  },
});

// Global error handler
queryClient.setMutationDefaults(['product', 'create'], {
  onError: (error) => {
    console.error('Mutation error:', error);
    // Show error notification
    useUIStore.getState().addNotification({
      type: 'error',
      title: 'Error',
      message: 'Something went wrong. Please try again.',
    });
  },
});
```

### 2. Product Queries

```typescript
// src/hooks/queries/useProducts.ts
import { useQuery, useInfiniteQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { productService } from '../services/product.service';

// Product list query
export const useProducts = (filters: ProductFilters) => {
  return useQuery({
    queryKey: ['products', filters],
    queryFn: ({ queryKey }) => productService.fetchProducts(queryKey[1]),
    keepPreviousData: true,
    staleTime: 2 * 60 * 1000, // 2 minutes
  });
};

// Infinite product query for pagination
export const useInfiniteProducts = (filters: ProductFilters) => {
  return useInfiniteQuery({
    queryKey: ['products', 'infinite', filters],
    queryFn: ({ pageParam = 1, queryKey }) => 
      productService.fetchProducts({ ...queryKey[2], page: pageParam }),
    getNextPageParam: (lastPage, pages) => {
      return lastPage.hasMore ? pages.length + 1 : undefined;
    },
    keepPreviousData: true,
  });
};

// Single product query
export const useProduct = (id: string) => {
  return useQuery({
    queryKey: ['product', id],
    queryFn: () => productService.fetchProduct(id),
    enabled: !!id,
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
};

// Product creation mutation
export const useCreateProduct = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: productService.createProduct,
    onSuccess: (newProduct) => {
      // Invalidate and refetch products list
      queryClient.invalidateQueries({ queryKey: ['products'] });
      
      // Add new product to cache
      queryClient.setQueryData(['product', newProduct.id], newProduct);
      
      // Show success notification
      useUIStore.getState().addNotification({
        type: 'success',
        title: 'Success',
        message: 'Product created successfully!',
      });
    },
    onError: (error) => {
      console.error('Failed to create product:', error);
    },
  });
};

// Product update mutation
export const useUpdateProduct = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<Product> }) =>
      productService.updateProduct(id, data),
    onMutate: async ({ id, data }) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['product', id] });
      
      // Snapshot previous value
      const previousProduct = queryClient.getQueryData(['product', id]);
      
      // Optimistically update
      queryClient.setQueryData(['product', id], (old: Product) => ({
        ...old,
        ...data,
      }));
      
      return { previousProduct };
    },
    onError: (err, variables, context) => {
      // Rollback on error
      if (context?.previousProduct) {
        queryClient.setQueryData(
          ['product', variables.id],
          context.previousProduct
        );
      }
    },
    onSettled: (data, error, variables) => {
      // Always refetch after error or success
      queryClient.invalidateQueries({ queryKey: ['product', variables.id] });
    },
  });
};

// Product deletion mutation
export const useDeleteProduct = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: productService.deleteProduct,
    onSuccess: (_, deletedId) => {
      // Remove from cache
      queryClient.removeQueries({ queryKey: ['product', deletedId] });
      
      // Update products list
      queryClient.setQueriesData(
        { queryKey: ['products'] },
        (old: any) => {
          if (!old) return old;
          return {
            ...old,
            data: old.data.filter((product: Product) => product.id !== deletedId)
          };
        }
      );
    },
  });
};
```

### 3. Chat Queries

```typescript
// src/hooks/queries/useChat.ts
export const useGroups = (userProfile: UserProfile) => {
  return useQuery({
    queryKey: ['groups', userProfile.id],
    queryFn: () => chatService.fetchUserGroups(userProfile.id),
    enabled: !!userProfile.id,
    refetchInterval: 30000, // Refetch every 30 seconds
  });
};

export const useMessages = (groupId: string) => {
  return useInfiniteQuery({
    queryKey: ['messages', groupId],
    queryFn: ({ pageParam = 1 }) => 
      chatService.fetchMessages(groupId, pageParam),
    getNextPageParam: (lastPage) => 
      lastPage.hasMore ? lastPage.nextPage : undefined,
    enabled: !!groupId,
    refetchOnMount: false,
    refetchOnWindowFocus: false,
  });
};

export const useSendMessage = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: chatService.sendMessage,
    onMutate: async (newMessage) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ 
        queryKey: ['messages', newMessage.groupId] 
      });
      
      // Optimistically add message
      queryClient.setQueryData(
        ['messages', newMessage.groupId],
        (old: any) => {
          if (!old) return old;
          
          const optimisticMessage = {
            ...newMessage,
            id: `temp-${Date.now()}`,
            status: 'sending',
            timestamp: new Date().toISOString(),
          };
          
          return {
            ...old,
            pages: old.pages.map((page: any, index: number) => 
              index === 0 
                ? { ...page, data: [optimisticMessage, ...page.data] }
                : page
            )
          };
        }
      );
    },
    onError: (error, variables, context) => {
      // Remove optimistic message on error
      queryClient.invalidateQueries({ 
        queryKey: ['messages', variables.groupId] 
      });
    },
    onSuccess: (response, variables) => {
      // Update with real message data
      queryClient.invalidateQueries({ 
        queryKey: ['messages', variables.groupId] 
      });
    },
  });
};
```

## Form State Management (React Hook Form)

### 1. Product Form

```typescript
// src/components/product/ProductForm.tsx
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Form schema
const productSchema = z.object({
  title: z.string().min(1, 'Title is required').max(100, 'Title too long'),
  description: z.string().min(10, 'Description must be at least 10 characters'),
  price: z.number().min(0, 'Price must be positive'),
  category: z.string().min(1, 'Category is required'),
  images: z.array(z.instanceof(File)).max(5, 'Maximum 5 images allowed'),
  tags: z.array(z.string()).max(10, 'Maximum 10 tags allowed'),
  isPublished: z.boolean().default(false),
});

type ProductFormData = z.infer<typeof productSchema>;

interface ProductFormProps {
  initialData?: Partial<ProductFormData>;
  onSubmit: (data: ProductFormData) => Promise<void>;
  mode: 'create' | 'edit';
}

export const ProductForm = ({ initialData, onSubmit, mode }: ProductFormProps) => {
  const {
    register,
    handleSubmit,
    control,
    watch,
    setValue,
    formState: { errors, isSubmitting, isDirty, isValid },
    reset,
    trigger,
  } = useForm<ProductFormData>({
    resolver: zodResolver(productSchema),
    defaultValues: {
      title: '',
      description: '',
      price: 0,
      category: '',
      images: [],
      tags: [],
      isPublished: false,
      ...initialData,
    },
    mode: 'onChange', // Validate on change
  });
  
  // Watch specific fields
  const watchedImages = watch('images');
  const watchedTitle = watch('title');
  
  // Auto-save draft
  const { mutate: saveDraft } = useSaveDraft();
  
  useEffect(() => {
    if (isDirty && mode === 'create') {
      const timeoutId = setTimeout(() => {
        const formData = watch();
        saveDraft(formData);
      }, 2000); // Auto-save after 2 seconds of inactivity
      
      return () => clearTimeout(timeoutId);
    }
  }, [isDirty, watch, saveDraft, mode]);
  
  // Handle form submission
  const handleFormSubmit = async (data: ProductFormData) => {
    try {
      await onSubmit(data);
      if (mode === 'create') {
        reset(); // Reset form after successful creation
      }
    } catch (error) {
      console.error('Form submission error:', error);
    }
  };
  
  // Handle image upload
  const handleImageUpload = (files: FileList) => {
    const currentImages = watchedImages || [];
    const newImages = Array.from(files);
    const totalImages = currentImages.length + newImages.length;
    
    if (totalImages > 5) {
      // Show error
      return;
    }
    
    setValue('images', [...currentImages, ...newImages], {
      shouldValidate: true,
      shouldDirty: true,
    });
  };
  
  // Remove image
  const removeImage = (index: number) => {
    const currentImages = watchedImages || [];
    const newImages = currentImages.filter((_, i) => i !== index);
    setValue('images', newImages, {
      shouldValidate: true,
      shouldDirty: true,
    });
  };
  
  return (
    <form onSubmit={handleSubmit(handleFormSubmit)} className="space-y-6">
      {/* Title field */}
      <div>
        <label htmlFor="title" className="block text-sm font-medium">
          Title *
        </label>
        <input
          {...register('title')}
          type="text"
          id="title"
          className="mt-1 block w-full rounded-md border-gray-300"
          placeholder="Enter product title"
        />
        {errors.title && (
          <p className="mt-1 text-sm text-red-600">{errors.title.message}</p>
        )}
      </div>
      
      {/* Description field */}
      <div>
        <label htmlFor="description" className="block text-sm font-medium">
          Description *
        </label>
        <Controller
          name="description"
          control={control}
          render={({ field }) => (
            <RichTextEditor
              {...field}
              placeholder="Describe your product"
              error={errors.description?.message}
            />
          )}
        />
      </div>
      
      {/* Price field */}
      <div>
        <label htmlFor="price" className="block text-sm font-medium">
          Price *
        </label>
        <input
          {...register('price', { valueAsNumber: true })}
          type="number"
          id="price"
          step="0.01"
          min="0"
          className="mt-1 block w-full rounded-md border-gray-300"
        />
        {errors.price && (
          <p className="mt-1 text-sm text-red-600">{errors.price.message}</p>
        )}
      </div>
      
      {/* Category field */}
      <div>
        <label htmlFor="category" className="block text-sm font-medium">
          Category *
        </label>
        <Controller
          name="category"
          control={control}
          render={({ field }) => (
            <Select
              {...field}
              options={categoryOptions}
              placeholder="Select a category"
              error={errors.category?.message}
            />
          )}
        />
      </div>
      
      {/* Image upload */}
      <div>
        <label className="block text-sm font-medium mb-2">
          Images ({watchedImages?.length || 0}/5)
        </label>
        <ImageUpload
          images={watchedImages || []}
          onUpload={handleImageUpload}
          onRemove={removeImage}
          maxImages={5}
        />
        {errors.images && (
          <p className="mt-1 text-sm text-red-600">{errors.images.message}</p>
        )}
      </div>
      
      {/* Tags field */}
      <div>
        <label className="block text-sm font-medium mb-2">
          Tags
        </label>
        <Controller
          name="tags"
          control={control}
          render={({ field }) => (
            <TagInput
              {...field}
              placeholder="Add tags"
              maxTags={10}
            />
          )}
        />
      </div>
      
      {/* Publish toggle */}
      <div className="flex items-center">
        <Controller
          name="isPublished"
          control={control}
          render={({ field: { value, onChange } }) => (
            <Toggle
              checked={value}
              onChange={onChange}
              label="Publish immediately"
            />
          )}
        />
      </div>
      
      {/* Form actions */}
      <div className="flex justify-between">
        <div className="flex space-x-3">
          <button
            type="button"
            onClick={() => reset()}
            className="px-4 py-2 border border-gray-300 rounded-md"
            disabled={!isDirty}
          >
            Reset
          </button>
          
          <button
            type="button"
            onClick={() => trigger()}
            className="px-4 py-2 bg-gray-100 rounded-md"
          >
            Validate
          </button>
        </div>
        
        <div className="flex space-x-3">
          <button
            type="submit"
            disabled={!isValid || isSubmitting}
            className="px-4 py-2 bg-blue-600 text-white rounded-md disabled:opacity-50"
          >
            {isSubmitting ? 'Saving...' : mode === 'create' ? 'Create' : 'Update'}
          </button>
        </div>
      </div>
      
      {/* Form status */}
      <div className="text-sm text-gray-500">
        {isDirty && 'You have unsaved changes'}
        {!isDirty && mode === 'edit' && 'All changes saved'}
      </div>
    </form>
  );
};
```

### 2. Multi-Step Form

```typescript
// src/components/auth/RegistrationForm.tsx
import { useForm, FormProvider } from 'react-hook-form';

const registrationSteps = [
  { id: 'personal', title: 'Personal Information' },
  { id: 'business', title: 'Business Details' },
  { id: 'verification', title: 'Verification' },
  { id: 'review', title: 'Review & Submit' },
];

export const RegistrationForm = () => {
  const [currentStep, setCurrentStep] = useState(0);
  const methods = useForm({
    mode: 'onChange',
    defaultValues: {
      // Personal
      firstName: '',
      lastName: '',
      email: '',
      phone: '',
      // Business
      companyName: '',
      businessType: '',
      gstNumber: '',
      // Verification
      aadharNumber: '',
      panNumber: '',
    },
  });
  
  const { trigger, getValues, formState: { isValid } } = methods;
  
  // Step validation
  const validateStep = async (step: number) => {
    const fieldsToValidate = {
      0: ['firstName', 'lastName', 'email', 'phone'],
      1: ['companyName', 'businessType'],
      2: ['aadharNumber', 'panNumber'],
    }[step];
    
    if (fieldsToValidate) {
      return await trigger(fieldsToValidate as any);
    }
    return true;
  };
  
  // Navigate to next step
  const nextStep = async () => {
    const isStepValid = await validateStep(currentStep);
    if (isStepValid && currentStep < registrationSteps.length - 1) {
      setCurrentStep(currentStep + 1);
    }
  };
  
  // Navigate to previous step
  const prevStep = () => {
    if (currentStep > 0) {
      setCurrentStep(currentStep - 1);
    }
  };
  
  // Submit form
  const onSubmit = async (data: any) => {
    try {
      await registerUser(data);
      // Handle success
    } catch (error) {
      // Handle error
    }
  };
  
  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        {/* Step indicator */}
        <StepIndicator 
          steps={registrationSteps}
          currentStep={currentStep}
        />
        
        {/* Step content */}
        <div className="mt-8">
          {currentStep === 0 && <PersonalInfoStep />}
          {currentStep === 1 && <BusinessInfoStep />}
          {currentStep === 2 && <VerificationStep />}
          {currentStep === 3 && <ReviewStep data={getValues()} />}
        </div>
        
        {/* Navigation */}
        <div className="flex justify-between mt-8">
          <button
            type="button"
            onClick={prevStep}
            disabled={currentStep === 0}
            className="px-4 py-2 border rounded-md"
          >
            Previous
          </button>
          
          {currentStep < registrationSteps.length - 1 ? (
            <button
              type="button"
              onClick={nextStep}
              className="px-4 py-2 bg-blue-600 text-white rounded-md"
            >
              Next
            </button>
          ) : (
            <button
              type="submit"
              disabled={!isValid}
              className="px-4 py-2 bg-green-600 text-white rounded-md"
            >
              Submit
            </button>
          )}
        </div>
      </form>
    </FormProvider>
  );
};
```

## Local State Patterns

### 1. Component State with useState

```typescript
// Simple state management
const ProductCard = ({ product }: { product: Product }) => {
  const [isExpanded, setIsExpanded] = useState(false);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  const handleToggleExpand = () => {
    setIsExpanded(!isExpanded);
  };
  
  const handleAction = async () => {
    setIsLoading(true);
    setError(null);
    
    try {
      await performAction(product.id);
    } catch (err) {
      setError('Action failed. Please try again.');
    } finally {
      setIsLoading(false);
    }
  };
  
  return (
    <div className="product-card">
      {/* Card content */}
    </div>
  );
};
```

### 2. Complex State with useReducer

```typescript
// Complex state management
interface ChatState {
  messages: Message[];
  isLoading: boolean;
  error: string | null;
  typingUsers: string[];
  connectionStatus: 'connected' | 'disconnected' | 'connecting';
}

type ChatAction =
  | { type: 'SET_LOADING'; payload: boolean }
  | { type: 'SET_ERROR'; payload: string | null }
  | { type: 'ADD_MESSAGE'; payload: Message }
  | { type: 'UPDATE_MESSAGE'; payload: { id: string; updates: Partial<Message> } }
  | { type: 'SET_TYPING'; payload: { userId: string; isTyping: boolean } }
  | { type: 'SET_CONNECTION_STATUS'; payload: ChatState['connectionStatus'] };

const chatReducer = (state: ChatState, action: ChatAction): ChatState => {
  switch (action.type) {
    case 'SET_LOADING':
      return { ...state, isLoading: action.payload };
      
    case 'SET_ERROR':
      return { ...state, error: action.payload };
      
    case 'ADD_MESSAGE':
      return {
        ...state,
        messages: [...state.messages, action.payload],
      };
      
    case 'UPDATE_MESSAGE':
      return {
        ...state,
        messages: state.messages.map(msg =>
          msg.id === action.payload.id
            ? { ...msg, ...action.payload.updates }
            : msg
        ),
      };
      
    case 'SET_TYPING':
      const { userId, isTyping } = action.payload;
      return {
        ...state,
        typingUsers: isTyping
          ? [...state.typingUsers.filter(id => id !== userId), userId]
          : state.typingUsers.filter(id => id !== userId),
      };
      
    case 'SET_CONNECTION_STATUS':
      return { ...state, connectionStatus: action.payload };
      
    default:
      return state;
  }
};

const Chat = ({ groupId }: { groupId: string }) => {
  const [state, dispatch] = useReducer(chatReducer, {
    messages: [],
    isLoading: false,
    error: null,
    typingUsers: [],
    connectionStatus: 'disconnected',
  });
  
  // Use the reducer state and dispatch
  const addMessage = (message: Message) => {
    dispatch({ type: 'ADD_MESSAGE', payload: message });
  };
  
  const setTyping = (userId: string, isTyping: boolean) => {
    dispatch({ type: 'SET_TYPING', payload: { userId, isTyping } });
  };
  
  return (
    <div className="chat">
      {/* Chat UI */}
    </div>
  );
};
```

## Context State Management

### 1. Theme Context

```typescript
// src/contexts/ThemeContext.tsx
interface ThemeContextType {
  theme: 'light' | 'dark' | 'system';
  setTheme: (theme: 'light' | 'dark' | 'system') => void;
  resolvedTheme: 'light' | 'dark';
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

export const ThemeProvider = ({ children }: { children: React.ReactNode }) => {
  const [theme, setTheme] = useState<'light' | 'dark' | 'system'>('system');
  
  // Resolve system theme
  const [systemTheme, setSystemTheme] = useState<'light' | 'dark'>('light');
  
  useEffect(() => {
    const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
    setSystemTheme(mediaQuery.matches ? 'dark' : 'light');
    
    const handleChange = (e: MediaQueryListEvent) => {
      setSystemTheme(e.matches ? 'dark' : 'light');
    };
    
    mediaQuery.addEventListener('change', handleChange);
    return () => mediaQuery.removeEventListener('change', handleChange);
  }, []);
  
  const resolvedTheme = theme === 'system' ? systemTheme : theme;
  
  // Apply theme to document
  useEffect(() => {
    document.documentElement.setAttribute('data-theme', resolvedTheme);
  }, [resolvedTheme]);
  
  const value = {
    theme,
    setTheme,
    resolvedTheme,
  };
  
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
};

export const useTheme = () => {
  const context = useContext(ThemeContext);
  if (context === undefined) {
    throw new Error('useTheme must be used within a ThemeProvider');
  }
  return context;
};
```

### 2. Feature Flags Context

```typescript
// src/contexts/FeatureFlagsContext.tsx
interface FeatureFlags {
  enableNewDashboard: boolean;
  enableAdvancedSearch: boolean;
  enableRealTimeNotifications: boolean;
  enableBulkUpload: boolean;
}

interface FeatureFlagsContextType {
  flags: FeatureFlags;
  isFeatureEnabled: (feature: keyof FeatureFlags) => boolean;
  updateFlag: (feature: keyof FeatureFlags, enabled: boolean) => void;
}

const FeatureFlagsContext = createContext<FeatureFlagsContextType | undefined>(undefined);

export const FeatureFlagsProvider = ({ children }: { children: React.ReactNode }) => {
  const [flags, setFlags] = useState<FeatureFlags>({
    enableNewDashboard: false,
    enableAdvancedSearch: true,
    enableRealTimeNotifications: true,
    enableBulkUpload: false,
  });
  
  // Load flags from API or config
  useEffect(() => {
    const loadFeatureFlags = async () => {
      try {
        const response = await fetch('/api/feature-flags');
        const serverFlags = await response.json();
        setFlags(prev => ({ ...prev, ...serverFlags }));
      } catch (error) {
        console.error('Failed to load feature flags:', error);
      }
    };
    
    loadFeatureFlags();
  }, []);
  
  const isFeatureEnabled = (feature: keyof FeatureFlags) => {
    return flags[feature];
  };
  
  const updateFlag = (feature: keyof FeatureFlags, enabled: boolean) => {
    setFlags(prev => ({ ...prev, [feature]: enabled }));
  };
  
  const value = {
    flags,
    isFeatureEnabled,
    updateFlag,
  };
  
  return (
    <FeatureFlagsContext.Provider value={value}>
      {children}
    </FeatureFlagsContext.Provider>
  );
};

export const useFeatureFlags = () => {
  const context = useContext(FeatureFlagsContext);
  if (context === undefined) {
    throw new Error('useFeatureFlags must be used within a FeatureFlagsProvider');
  }
  return context;
};
```

## State Synchronization Patterns

### 1. Real-time State Sync

```typescript
// src/hooks/useRealTimeSync.ts
export const useRealTimeSync = () => {
  const queryClient = useQueryClient();
  const { addMessage, updateMessage } = useChatStore();
  
  useEffect(() => {
    const socket = socketService.connect();
    
    // Listen for new messages
    socket.on('message:new', (message: Message) => {
      // Update local store
      addMessage(message.groupId, message);
      
      // Update React Query cache
      queryClient.setQueryData(
        ['messages', message.groupId],
        (old: any) => {
          if (!old) return old;
          return {
            ...old,
            pages: old.pages.map((page: any, index: number) =>
              index === 0
                ? { ...page, data: [message, ...page.data] }
                : page
            )
          };
        }
      );
    });
    
    // Listen for message updates
    socket.on('message:updated', ({ messageId, groupId, updates }: any) => {
      updateMessage(groupId, messageId, updates);
      
      queryClient.setQueryData(
        ['messages', groupId],
        (old: any) => {
          if (!old) return old;
          return {
            ...old,
            pages: old.pages.map((page: any) => ({
              ...page,
              data: page.data.map((msg: Message) =>
                msg.id === messageId ? { ...msg, ...updates } : msg
              )
            }))
          };
        }
      );
    });
    
    return () => {
      socket.disconnect();
    };
  }, [queryClient, addMessage, updateMessage]);
};
```

### 2. Cross-Tab Synchronization

```typescript
// src/hooks/useCrossTabSync.ts
export const useCrossTabSync = () => {
  const { login, logout } = useAuthStore();
  
  useEffect(() => {
    const handleStorageChange = (e: StorageEvent) => {
      if (e.key === 'auth-storage') {
        if (e.newValue) {
          const authData = JSON.parse(e.newValue);
          if (authData.state.isAuthenticated) {
            login(authData.state.user, authData.state.token);
          }
        } else {
          // Auth data was cleared
          logout();
        }
      }
    };
    
    window.addEventListener('storage', handleStorageChange);
    
    return () => {
      window.removeEventListener('storage', handleStorageChange);
    };
  }, [login, logout]);
};
```

## Performance Optimization

### 1. Selective Subscriptions

```typescript
// Optimize Zustand subscriptions
const useAuthUser = () => useAuthStore(state => state.user);
const useAuthActions = () => useAuthStore(state => ({
  login: state.login,
  logout: state.logout,
  updateUser: state.updateUser,
}));

// Use shallow comparison for objects
const useUIState = () => useUIStore(
  state => ({
    sidebarOpen: state.sidebarOpen,
    theme: state.theme,
    modals: state.modals,
  }),
  shallow
);
```

### 2. Query Optimization

```typescript
// Optimize React Query with select
const useProductTitles = () => {
  return useQuery({
    queryKey: ['products'],
    queryFn: fetchProducts,
    select: (data) => data.map(product => ({
      id: product.id,
      title: product.title,
    })),
    staleTime: 10 * 60 * 1000, // 10 minutes
  });
};

// Prefetch related data
const usePrefetchProduct = () => {
  const queryClient = useQueryClient();
  
  return (productId: string) => {
    queryClient.prefetchQuery({
      queryKey: ['product', productId],
      queryFn: () => fetchProduct(productId),
      staleTime: 5 * 60 * 1000,
    });
  };
};
```

## Testing State Management

### 1. Testing Zustand Stores

```typescript
// src/stores/__tests__/auth.store.test.ts
import { renderHook, act } from '@testing-library/react';
import { useAuthStore } from '../auth.store';

describe('Auth Store', () => {
  beforeEach(() => {
    useAuthStore.getState().logout();
  });
  
  it('should login user correctly', () => {
    const { result } = renderHook(() => useAuthStore());
    
    const user = {
      id: '1',
      email: 'test@example.com',
      name: 'Test User',
      role: 'vendor' as const,
      isVerified: true,
    };
    
    act(() => {
      result.current.login(user, 'token123');
    });
    
    expect(result.current.user).toEqual(user);
    expect(result.current.token).toBe('token123');
    expect(result.current.isAuthenticated).toBe(true);
    expect(result.current.isVendor()).toBe(true);
  });
  
  it('should logout user correctly', () => {
    const { result } = renderHook(() => useAuthStore());
    
    // First login
    act(() => {
      result.current.login(
        { id: '1', email: 'test@example.com', name: 'Test', role: 'vendor', isVerified: true },
        'token123'
      );
    });
    
    // Then logout
    act(() => {
      result.current.logout();
    });
    
    expect(result.current.user).toBeNull();
    expect(result.current.token).toBeNull();
    expect(result.current.isAuthenticated).toBe(false);
  });
});
```

### 2. Testing React Query

```typescript
// src/hooks/__tests__/useProducts.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useProducts } from '../queries/useProducts';
import { productService } from '../services/product.service';

jest.mock('../services/product.service');

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

describe('useProducts', () => {
  it('should fetch products successfully', async () => {
    const mockProducts = [
      { id: '1', title: 'Product 1' },
      { id: '2', title: 'Product 2' },
    ];
    
    (productService.fetchProducts as jest.Mock).mockResolvedValue({
      data: mockProducts,
      total: 2,
    });
    
    const { result } = renderHook(
      () => useProducts({ page: 1, search: '' }),
      { wrapper: createWrapper() }
    );
    
    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });
    
    expect(result.current.data?.data).toEqual(mockProducts);
  });
});
```

## Best Practices

### 1. State Organization

- **Separate Concerns**: Keep different types of state in appropriate layers
- **Minimize Global State**: Only put truly global state in Zustand
- **Use Server State**: Leverage React Query for all API data
- **Form State**: Use React Hook Form for complex forms

### 2. Performance

- **Selective Subscriptions**: Subscribe only to needed state slices
- **Memoization**: Use useMemo and useCallback appropriately
- **Query Optimization**: Use select, prefetching, and proper cache times
- **Avoid Over-fetching**: Fetch only required data

### 3. Error Handling

- **Global Error Boundaries**: Catch and handle state errors
- **Query Error Handling**: Implement retry logic and fallbacks
- **Form Validation**: Provide clear validation messages
- **State Consistency**: Ensure state remains consistent across updates

### 4. Testing

- **Unit Test Stores**: Test store logic in isolation
- **Integration Tests**: Test state interactions
- **Mock External Dependencies**: Mock API calls and services
- **Test Error States**: Verify error handling works correctly

## Next Steps

For more detailed information:
- [Real-time Communication](./real-time-communication.md) - WebSocket state management
- [Authentication & Security](./authentication.md) - Auth state patterns
- [Performance Guide](./performance.md) - State optimization techniques
- [Testing Guide](./testing.md) - Comprehensive testing strategies
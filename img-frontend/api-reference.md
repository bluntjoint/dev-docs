# API Reference

## Overview

This document provides comprehensive API reference for the Deed-O Frontend application, including service methods, data models, error handling, and integration patterns.

## Service Architecture

### Base Configuration

```typescript
// src/lib/api.ts
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';
import { env } from '../config/environment';
import { useAuthStore } from '../stores/authStore';

interface ApiResponse<T = any> {
  data: T;
  message: string;
  success: boolean;
  timestamp: string;
}

interface ApiError {
  message: string;
  code: string;
  details?: any;
  timestamp: string;
}

class ApiClient {
  private client: AxiosInstance;
  
  constructor() {
    this.client = axios.create({
      baseURL: env.apiBaseUrl,
      timeout: 10000,
      headers: {
        'Content-Type': 'application/json',
      },
    });
    
    this.setupInterceptors();
  }
  
  private setupInterceptors(): void {
    // Request interceptor
    this.client.interceptors.request.use(
      (config) => {
        const token = useAuthStore.getState().token;
        if (token) {
          config.headers.Authorization = `Bearer ${token}`;
        }
        return config;
      },
      (error) => Promise.reject(error)
    );
    
    // Response interceptor
    this.client.interceptors.response.use(
      (response: AxiosResponse<ApiResponse>) => response,
      async (error) => {
        if (error.response?.status === 401) {
          const { refreshToken } = useAuthStore.getState();
          if (refreshToken) {
            try {
              await this.refreshAccessToken(refreshToken);
              return this.client.request(error.config);
            } catch (refreshError) {
              useAuthStore.getState().logout();
              window.location.href = '/login';
            }
          }
        }
        return Promise.reject(error);
      }
    );
  }
  
  async get<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.get<ApiResponse<T>>(url, config);
    return response.data.data;
  }
  
  async post<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.post<ApiResponse<T>>(url, data, config);
    return response.data.data;
  }
  
  async put<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.put<ApiResponse<T>>(url, data, config);
    return response.data.data;
  }
  
  async patch<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.patch<ApiResponse<T>>(url, data, config);
    return response.data.data;
  }
  
  async delete<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    const response = await this.client.delete<ApiResponse<T>>(url, config);
    return response.data.data;
  }
  
  private async refreshAccessToken(refreshToken: string): Promise<void> {
    const response = await this.client.post('/auth/refresh', {
      refreshToken,
    });
    
    const { token, refreshToken: newRefreshToken } = response.data.data;
    useAuthStore.getState().setTokens(token, newRefreshToken);
  }
}

export const apiClient = new ApiClient();
```

## Authentication Service

### AuthService

```typescript
// src/services/authService.ts
export interface LoginCredentials {
  email: string;
  password: string;
}

export interface RegisterData {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
  role: 'vendor' | 'consumer';
  phone?: string;
}

export interface AuthResponse {
  user: User;
  token: string;
  refreshToken: string;
  expiresIn: number;
}

export interface SSOProvider {
  provider: 'google' | 'microsoft';
  token: string;
  role?: 'vendor' | 'consumer';
}

class AuthService {
  /**
   * Authenticate user with email and password
   */
  async login(credentials: LoginCredentials): Promise<AuthResponse> {
    return apiClient.post('/auth/login', credentials);
  }
  
  /**
   * Register new user account
   */
  async register(data: RegisterData): Promise<AuthResponse> {
    return apiClient.post('/auth/register', data);
  }
  
  /**
   * Authenticate user with SSO provider
   */
  async ssoLogin(ssoData: SSOProvider): Promise<AuthResponse> {
    return apiClient.post('/auth/sso', ssoData);
  }
  
  /**
   * Refresh access token
   */
  async refreshToken(refreshToken: string): Promise<AuthResponse> {
    return apiClient.post('/auth/refresh', { refreshToken });
  }
  
  /**
   * Logout user and invalidate tokens
   */
  async logout(): Promise<void> {
    return apiClient.post('/auth/logout');
  }
  
  /**
   * Get current user profile
   */
  async getProfile(): Promise<User> {
    return apiClient.get('/auth/profile');
  }
  
  /**
   * Update user profile
   */
  async updateProfile(data: Partial<User>): Promise<User> {
    return apiClient.patch('/auth/profile', data);
  }
  
  /**
   * Change user password
   */
  async changePassword(data: {
    currentPassword: string;
    newPassword: string;
  }): Promise<void> {
    return apiClient.post('/auth/change-password', data);
  }
  
  /**
   * Request password reset
   */
  async requestPasswordReset(email: string): Promise<void> {
    return apiClient.post('/auth/forgot-password', { email });
  }
  
  /**
   * Reset password with token
   */
  async resetPassword(data: {
    token: string;
    newPassword: string;
  }): Promise<void> {
    return apiClient.post('/auth/reset-password', data);
  }
  
  /**
   * Verify email address
   */
  async verifyEmail(token: string): Promise<void> {
    return apiClient.post('/auth/verify-email', { token });
  }
  
  /**
   * Resend email verification
   */
  async resendVerification(): Promise<void> {
    return apiClient.post('/auth/resend-verification');
  }
}

export const authService = new AuthService();
```

## Product Service

### ProductService

```typescript
// src/services/productService.ts
export interface ProductFilters {
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  location?: string;
  vendorId?: string;
  search?: string;
  tags?: string[];
  availability?: 'available' | 'out_of_stock' | 'discontinued';
  sortBy?: 'price' | 'rating' | 'created_at' | 'name';
  sortOrder?: 'asc' | 'desc';
  page?: number;
  limit?: number;
}

export interface ProductCreateData {
  name: string;
  description: string;
  price: number;
  category: string;
  images: string[];
  tags?: string[];
  specifications?: Record<string, any>;
  inventory?: {
    quantity: number;
    sku?: string;
    trackInventory: boolean;
  };
  shipping?: {
    weight?: number;
    dimensions?: {
      length: number;
      width: number;
      height: number;
    };
    freeShipping: boolean;
    shippingCost?: number;
  };
}

export interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}

class ProductService {
  /**
   * Get paginated list of products with filters
   */
  async getProducts(filters: ProductFilters = {}): Promise<PaginatedResponse<Product>> {
    const params = new URLSearchParams();
    
    Object.entries(filters).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        if (Array.isArray(value)) {
          value.forEach(v => params.append(key, v));
        } else {
          params.append(key, value.toString());
        }
      }
    });
    
    return apiClient.get(`/products?${params.toString()}`);
  }
  
  /**
   * Get single product by ID
   */
  async getProduct(id: string): Promise<Product> {
    return apiClient.get(`/products/${id}`);
  }
  
  /**
   * Create new product (vendor only)
   */
  async createProduct(data: ProductCreateData): Promise<Product> {
    return apiClient.post('/products', data);
  }
  
  /**
   * Update existing product (vendor only)
   */
  async updateProduct(id: string, data: Partial<ProductCreateData>): Promise<Product> {
    return apiClient.patch(`/products/${id}`, data);
  }
  
  /**
   * Delete product (vendor only)
   */
  async deleteProduct(id: string): Promise<void> {
    return apiClient.delete(`/products/${id}`);
  }
  
  /**
   * Get vendor's products
   */
  async getVendorProducts(vendorId?: string, filters: Omit<ProductFilters, 'vendorId'> = {}): Promise<PaginatedResponse<Product>> {
    const endpoint = vendorId ? `/vendors/${vendorId}/products` : '/vendor/products';
    const params = new URLSearchParams();
    
    Object.entries(filters).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        if (Array.isArray(value)) {
          value.forEach(v => params.append(key, v));
        } else {
          params.append(key, value.toString());
        }
      }
    });
    
    return apiClient.get(`${endpoint}?${params.toString()}`);
  }
  
  /**
   * Get product categories
   */
  async getCategories(): Promise<Category[]> {
    return apiClient.get('/products/categories');
  }
  
  /**
   * Search products
   */
  async searchProducts(query: string, filters: Omit<ProductFilters, 'search'> = {}): Promise<PaginatedResponse<Product>> {
    return this.getProducts({ ...filters, search: query });
  }
  
  /**
   * Get product reviews
   */
  async getProductReviews(productId: string, page = 1, limit = 10): Promise<PaginatedResponse<Review>> {
    return apiClient.get(`/products/${productId}/reviews?page=${page}&limit=${limit}`);
  }
  
  /**
   * Add product review
   */
  async addProductReview(productId: string, review: {
    rating: number;
    comment?: string;
    images?: string[];
  }): Promise<Review> {
    return apiClient.post(`/products/${productId}/reviews`, review);
  }
  
  /**
   * Update product inventory
   */
  async updateInventory(productId: string, data: {
    quantity: number;
    operation: 'set' | 'add' | 'subtract';
  }): Promise<Product> {
    return apiClient.patch(`/products/${productId}/inventory`, data);
  }
  
  /**
   * Bulk update products
   */
  async bulkUpdateProducts(updates: Array<{
    id: string;
    data: Partial<ProductCreateData>;
  }>): Promise<Product[]> {
    return apiClient.patch('/products/bulk', { updates });
  }
}

export const productService = new ProductService();
```

## Chat Service

### ChatService

```typescript
// src/services/chatService.ts
export interface MessageData {
  content: string;
  type: 'text' | 'image' | 'file' | 'product';
  metadata?: {
    fileName?: string;
    fileSize?: number;
    mimeType?: string;
    productId?: string;
  };
}

export interface GroupCreateData {
  name: string;
  description?: string;
  type: 'direct' | 'group' | 'support';
  participants: string[];
  isPrivate?: boolean;
}

class ChatService {
  /**
   * Get user's chat groups
   */
  async getGroups(): Promise<Group[]> {
    return apiClient.get('/chat/groups');
  }
  
  /**
   * Get single group by ID
   */
  async getGroup(groupId: string): Promise<Group> {
    return apiClient.get(`/chat/groups/${groupId}`);
  }
  
  /**
   * Create new chat group
   */
  async createGroup(data: GroupCreateData): Promise<Group> {
    return apiClient.post('/chat/groups', data);
  }
  
  /**
   * Update group details
   */
  async updateGroup(groupId: string, data: Partial<GroupCreateData>): Promise<Group> {
    return apiClient.patch(`/chat/groups/${groupId}`, data);
  }
  
  /**
   * Delete group
   */
  async deleteGroup(groupId: string): Promise<void> {
    return apiClient.delete(`/chat/groups/${groupId}`);
  }
  
  /**
   * Get group messages with pagination
   */
  async getMessages(groupId: string, page = 1, limit = 50): Promise<PaginatedResponse<Message>> {
    return apiClient.get(`/chat/groups/${groupId}/messages?page=${page}&limit=${limit}`);
  }
  
  /**
   * Send message to group
   */
  async sendMessage(groupId: string, data: MessageData): Promise<Message> {
    return apiClient.post(`/chat/groups/${groupId}/messages`, data);
  }
  
  /**
   * Update message
   */
  async updateMessage(groupId: string, messageId: string, data: {
    content: string;
  }): Promise<Message> {
    return apiClient.patch(`/chat/groups/${groupId}/messages/${messageId}`, data);
  }
  
  /**
   * Delete message
   */
  async deleteMessage(groupId: string, messageId: string): Promise<void> {
    return apiClient.delete(`/chat/groups/${groupId}/messages/${messageId}`);
  }
  
  /**
   * Add participants to group
   */
  async addParticipants(groupId: string, userIds: string[]): Promise<Group> {
    return apiClient.post(`/chat/groups/${groupId}/participants`, { userIds });
  }
  
  /**
   * Remove participant from group
   */
  async removeParticipant(groupId: string, userId: string): Promise<Group> {
    return apiClient.delete(`/chat/groups/${groupId}/participants/${userId}`);
  }
  
  /**
   * Leave group
   */
  async leaveGroup(groupId: string): Promise<void> {
    return apiClient.post(`/chat/groups/${groupId}/leave`);
  }
  
  /**
   * Mark messages as read
   */
  async markAsRead(groupId: string, messageIds: string[]): Promise<void> {
    return apiClient.post(`/chat/groups/${groupId}/read`, { messageIds });
  }
  
  /**
   * Get unread message count
   */
  async getUnreadCount(): Promise<{ total: number; groups: Record<string, number> }> {
    return apiClient.get('/chat/unread-count');
  }
  
  /**
   * Search messages
   */
  async searchMessages(query: string, groupId?: string): Promise<Message[]> {
    const params = new URLSearchParams({ query });
    if (groupId) params.append('groupId', groupId);
    
    return apiClient.get(`/chat/search?${params.toString()}`);
  }
  
  /**
   * Get message thread
   */
  async getMessageThread(groupId: string, messageId: string): Promise<Message[]> {
    return apiClient.get(`/chat/groups/${groupId}/messages/${messageId}/thread`);
  }
  
  /**
   * Reply to message thread
   */
  async replyToMessage(groupId: string, messageId: string, data: MessageData): Promise<Message> {
    return apiClient.post(`/chat/groups/${groupId}/messages/${messageId}/reply`, data);
  }
}

export const chatService = new ChatService();
```

## Vendor Service

### VendorService

```typescript
// src/services/vendorService.ts
export interface VendorProfileData {
  businessName: string;
  description: string;
  website?: string;
  phone: string;
  address: {
    street: string;
    city: string;
    state: string;
    zipCode: string;
    country: string;
  };
  businessHours: {
    [key: string]: {
      open: string;
      close: string;
      closed: boolean;
    };
  };
  categories: string[];
  logo?: string;
  banner?: string;
  socialMedia?: {
    facebook?: string;
    twitter?: string;
    instagram?: string;
    linkedin?: string;
  };
}

export interface VendorVerificationData {
  businessLicense: string;
  taxId: string;
  documents: {
    type: string;
    url: string;
    name: string;
  }[];
}

class VendorService {
  /**
   * Get vendor profile
   */
  async getProfile(vendorId?: string): Promise<Vendor> {
    const endpoint = vendorId ? `/vendors/${vendorId}` : '/vendor/profile';
    return apiClient.get(endpoint);
  }
  
  /**
   * Update vendor profile
   */
  async updateProfile(data: Partial<VendorProfileData>): Promise<Vendor> {
    return apiClient.patch('/vendor/profile', data);
  }
  
  /**
   * Get vendor verification status
   */
  async getVerificationStatus(): Promise<{
    status: 'pending' | 'verified' | 'rejected';
    submittedAt?: string;
    verifiedAt?: string;
    rejectionReason?: string;
    documents: {
      type: string;
      status: 'pending' | 'approved' | 'rejected';
      rejectionReason?: string;
    }[];
  }> {
    return apiClient.get('/vendor/verification');
  }
  
  /**
   * Submit verification documents
   */
  async submitVerification(data: VendorVerificationData): Promise<void> {
    return apiClient.post('/vendor/verification', data);
  }
  
  /**
   * Get vendor analytics
   */
  async getAnalytics(period: 'day' | 'week' | 'month' | 'year' = 'month'): Promise<{
    sales: {
      total: number;
      count: number;
      growth: number;
    };
    products: {
      total: number;
      active: number;
      views: number;
    };
    customers: {
      total: number;
      new: number;
      returning: number;
    };
    revenue: {
      current: number;
      previous: number;
      growth: number;
    };
    topProducts: Array<{
      id: string;
      name: string;
      sales: number;
      revenue: number;
    }>;
    recentOrders: Order[];
  }> {
    return apiClient.get(`/vendor/analytics?period=${period}`);
  }
  
  /**
   * Get vendor orders
   */
  async getOrders(filters: {
    status?: string;
    startDate?: string;
    endDate?: string;
    page?: number;
    limit?: number;
  } = {}): Promise<PaginatedResponse<Order>> {
    const params = new URLSearchParams();
    Object.entries(filters).forEach(([key, value]) => {
      if (value !== undefined) {
        params.append(key, value.toString());
      }
    });
    
    return apiClient.get(`/vendor/orders?${params.toString()}`);
  }
  
  /**
   * Update order status
   */
  async updateOrderStatus(orderId: string, status: string, notes?: string): Promise<Order> {
    return apiClient.patch(`/vendor/orders/${orderId}/status`, {
      status,
      notes,
    });
  }
  
  /**
   * Get vendor customers
   */
  async getCustomers(page = 1, limit = 20): Promise<PaginatedResponse<User>> {
    return apiClient.get(`/vendor/customers?page=${page}&limit=${limit}`);
  }
  
  /**
   * Get vendor reviews
   */
  async getReviews(page = 1, limit = 20): Promise<PaginatedResponse<Review>> {
    return apiClient.get(`/vendor/reviews?page=${page}&limit=${limit}`);
  }
  
  /**
   * Respond to review
   */
  async respondToReview(reviewId: string, response: string): Promise<Review> {
    return apiClient.post(`/vendor/reviews/${reviewId}/respond`, { response });
  }
  
  /**
   * Get vendor settings
   */
  async getSettings(): Promise<{
    notifications: {
      email: boolean;
      sms: boolean;
      push: boolean;
    };
    privacy: {
      showContact: boolean;
      showAddress: boolean;
      allowMessages: boolean;
    };
    business: {
      autoAcceptOrders: boolean;
      requireApproval: boolean;
      minimumOrder: number;
    };
  }> {
    return apiClient.get('/vendor/settings');
  }
  
  /**
   * Update vendor settings
   */
  async updateSettings(settings: any): Promise<void> {
    return apiClient.patch('/vendor/settings', settings);
  }
}

export const vendorService = new VendorService();
```

## File Upload Service

### FileService

```typescript
// src/services/fileService.ts
export interface UploadOptions {
  folder?: string;
  maxSize?: number;
  allowedTypes?: string[];
  compress?: boolean;
  generateThumbnail?: boolean;
}

export interface UploadResponse {
  url: string;
  key: string;
  size: number;
  mimeType: string;
  originalName: string;
  thumbnail?: string;
}

class FileService {
  /**
   * Upload single file
   */
  async uploadFile(file: File, options: UploadOptions = {}): Promise<UploadResponse> {
    const formData = new FormData();
    formData.append('file', file);
    
    if (options.folder) {
      formData.append('folder', options.folder);
    }
    
    if (options.compress) {
      formData.append('compress', 'true');
    }
    
    if (options.generateThumbnail) {
      formData.append('generateThumbnail', 'true');
    }
    
    return apiClient.post('/files/upload', formData, {
      headers: {
        'Content-Type': 'multipart/form-data',
      },
      onUploadProgress: (progressEvent) => {
        const progress = Math.round(
          (progressEvent.loaded * 100) / (progressEvent.total || 1)
        );
        // Emit progress event
        window.dispatchEvent(
          new CustomEvent('upload-progress', {
            detail: { progress, file: file.name },
          })
        );
      },
    });
  }
  
  /**
   * Upload multiple files
   */
  async uploadFiles(files: File[], options: UploadOptions = {}): Promise<UploadResponse[]> {
    const uploads = files.map(file => this.uploadFile(file, options));
    return Promise.all(uploads);
  }
  
  /**
   * Delete file
   */
  async deleteFile(key: string): Promise<void> {
    return apiClient.delete(`/files/${key}`);
  }
  
  /**
   * Get file info
   */
  async getFileInfo(key: string): Promise<{
    url: string;
    size: number;
    mimeType: string;
    uploadedAt: string;
    expiresAt?: string;
  }> {
    return apiClient.get(`/files/${key}/info`);
  }
  
  /**
   * Generate signed URL for direct upload
   */
  async getSignedUploadUrl(fileName: string, mimeType: string, folder?: string): Promise<{
    uploadUrl: string;
    key: string;
    fields: Record<string, string>;
  }> {
    return apiClient.post('/files/signed-upload', {
      fileName,
      mimeType,
      folder,
    });
  }
  
  /**
   * Compress image
   */
  async compressImage(file: File, options: {
    quality?: number;
    maxWidth?: number;
    maxHeight?: number;
    format?: 'jpeg' | 'png' | 'webp';
  } = {}): Promise<File> {
    const formData = new FormData();
    formData.append('file', file);
    
    Object.entries(options).forEach(([key, value]) => {
      if (value !== undefined) {
        formData.append(key, value.toString());
      }
    });
    
    const response = await apiClient.post('/files/compress', formData, {
      headers: {
        'Content-Type': 'multipart/form-data',
      },
      responseType: 'blob',
    });
    
    return new File([response], file.name, {
      type: options.format ? `image/${options.format}` : file.type,
    });
  }
}

export const fileService = new FileService();
```

## Data Models

### Core Models

```typescript
// src/types/models.ts
export interface User {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  role: 'vendor' | 'consumer' | 'moderator' | 'admin';
  avatar?: string;
  phone?: string;
  isEmailVerified: boolean;
  isPhoneVerified: boolean;
  createdAt: string;
  updatedAt: string;
  lastLoginAt?: string;
  preferences: {
    notifications: {
      email: boolean;
      sms: boolean;
      push: boolean;
    };
    privacy: {
      showProfile: boolean;
      allowMessages: boolean;
    };
  };
}

export interface Vendor extends User {
  businessName: string;
  description: string;
  website?: string;
  address: {
    street: string;
    city: string;
    state: string;
    zipCode: string;
    country: string;
    coordinates?: {
      lat: number;
      lng: number;
    };
  };
  businessHours: {
    [key: string]: {
      open: string;
      close: string;
      closed: boolean;
    };
  };
  categories: string[];
  logo?: string;
  banner?: string;
  rating: number;
  reviewCount: number;
  isVerified: boolean;
  verificationStatus: 'pending' | 'verified' | 'rejected';
  socialMedia?: {
    facebook?: string;
    twitter?: string;
    instagram?: string;
    linkedin?: string;
  };
  stats: {
    totalProducts: number;
    totalSales: number;
    totalRevenue: number;
    joinedAt: string;
  };
}

export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  originalPrice?: number;
  currency: string;
  category: string;
  subcategory?: string;
  images: string[];
  thumbnail: string;
  tags: string[];
  specifications: Record<string, any>;
  vendor: {
    id: string;
    businessName: string;
    logo?: string;
    rating: number;
    isVerified: boolean;
  };
  inventory: {
    quantity: number;
    sku?: string;
    trackInventory: boolean;
    lowStockThreshold?: number;
  };
  shipping: {
    weight?: number;
    dimensions?: {
      length: number;
      width: number;
      height: number;
    };
    freeShipping: boolean;
    shippingCost?: number;
    processingTime: string;
  };
  rating: number;
  reviewCount: number;
  status: 'active' | 'inactive' | 'out_of_stock' | 'discontinued';
  isPromoted: boolean;
  promotionEndsAt?: string;
  createdAt: string;
  updatedAt: string;
  viewCount: number;
  favoriteCount: number;
}

export interface Category {
  id: string;
  name: string;
  slug: string;
  description?: string;
  icon?: string;
  image?: string;
  parentId?: string;
  children?: Category[];
  productCount: number;
  isActive: boolean;
  sortOrder: number;
}

export interface Group {
  id: string;
  name: string;
  description?: string;
  type: 'direct' | 'group' | 'support';
  avatar?: string;
  isPrivate: boolean;
  participants: {
    id: string;
    firstName: string;
    lastName: string;
    avatar?: string;
    role: 'admin' | 'moderator' | 'member';
    joinedAt: string;
  }[];
  lastMessage?: {
    id: string;
    content: string;
    type: string;
    sender: {
      id: string;
      firstName: string;
      lastName: string;
    };
    createdAt: string;
  };
  unreadCount: number;
  createdAt: string;
  updatedAt: string;
  createdBy: string;
}

export interface Message {
  id: string;
  content: string;
  type: 'text' | 'image' | 'file' | 'product' | 'system';
  sender: {
    id: string;
    firstName: string;
    lastName: string;
    avatar?: string;
  };
  groupId: string;
  parentId?: string; // For threaded messages
  metadata?: {
    fileName?: string;
    fileSize?: number;
    mimeType?: string;
    productId?: string;
    systemType?: string;
  };
  reactions: {
    emoji: string;
    users: string[];
  }[];
  readBy: {
    userId: string;
    readAt: string;
  }[];
  isEdited: boolean;
  editedAt?: string;
  isDeleted: boolean;
  deletedAt?: string;
  createdAt: string;
  updatedAt: string;
}

export interface Review {
  id: string;
  rating: number;
  comment?: string;
  images?: string[];
  user: {
    id: string;
    firstName: string;
    lastName: string;
    avatar?: string;
  };
  product?: {
    id: string;
    name: string;
    thumbnail: string;
  };
  vendor?: {
    id: string;
    businessName: string;
    logo?: string;
  };
  response?: {
    content: string;
    createdAt: string;
  };
  isVerifiedPurchase: boolean;
  helpfulCount: number;
  reportCount: number;
  status: 'active' | 'hidden' | 'reported';
  createdAt: string;
  updatedAt: string;
}

export interface Order {
  id: string;
  orderNumber: string;
  status: 'pending' | 'confirmed' | 'processing' | 'shipped' | 'delivered' | 'cancelled' | 'refunded';
  customer: {
    id: string;
    firstName: string;
    lastName: string;
    email: string;
    phone?: string;
  };
  vendor: {
    id: string;
    businessName: string;
    logo?: string;
  };
  items: {
    id: string;
    product: {
      id: string;
      name: string;
      thumbnail: string;
      price: number;
    };
    quantity: number;
    price: number;
    total: number;
  }[];
  subtotal: number;
  shipping: number;
  tax: number;
  total: number;
  currency: string;
  shippingAddress: {
    street: string;
    city: string;
    state: string;
    zipCode: string;
    country: string;
  };
  billingAddress: {
    street: string;
    city: string;
    state: string;
    zipCode: string;
    country: string;
  };
  paymentMethod: {
    type: string;
    last4?: string;
    brand?: string;
  };
  tracking?: {
    number: string;
    carrier: string;
    url?: string;
  };
  notes?: string;
  timeline: {
    status: string;
    timestamp: string;
    notes?: string;
  }[];
  createdAt: string;
  updatedAt: string;
  estimatedDelivery?: string;
}
```

## Error Handling

### Error Types

```typescript
// src/types/errors.ts
export interface ApiError {
  message: string;
  code: string;
  details?: any;
  timestamp: string;
  path?: string;
  method?: string;
}

export class AppError extends Error {
  public readonly code: string;
  public readonly details?: any;
  public readonly timestamp: string;
  
  constructor(message: string, code: string, details?: any) {
    super(message);
    this.name = 'AppError';
    this.code = code;
    this.details = details;
    this.timestamp = new Date().toISOString();
  }
}

export class ValidationError extends AppError {
  constructor(message: string, details?: any) {
    super(message, 'VALIDATION_ERROR', details);
    this.name = 'ValidationError';
  }
}

export class NetworkError extends AppError {
  constructor(message: string, details?: any) {
    super(message, 'NETWORK_ERROR', details);
    this.name = 'NetworkError';
  }
}

export class AuthenticationError extends AppError {
  constructor(message: string, details?: any) {
    super(message, 'AUTHENTICATION_ERROR', details);
    this.name = 'AuthenticationError';
  }
}

export class AuthorizationError extends AppError {
  constructor(message: string, details?: any) {
    super(message, 'AUTHORIZATION_ERROR', details);
    this.name = 'AuthorizationError';
  }
}

// Error handler utility
export const handleApiError = (error: any): AppError => {
  if (error.response) {
    // Server responded with error status
    const { status, data } = error.response;
    
    switch (status) {
      case 400:
        return new ValidationError(data.message || 'Invalid request', data.details);
      case 401:
        return new AuthenticationError(data.message || 'Authentication required');
      case 403:
        return new AuthorizationError(data.message || 'Access denied');
      case 404:
        return new AppError(data.message || 'Resource not found', 'NOT_FOUND');
      case 429:
        return new AppError(data.message || 'Too many requests', 'RATE_LIMIT');
      case 500:
        return new AppError(data.message || 'Internal server error', 'SERVER_ERROR');
      default:
        return new AppError(data.message || 'Unknown error', 'UNKNOWN_ERROR');
    }
  } else if (error.request) {
    // Network error
    return new NetworkError('Network error occurred');
  } else {
    // Other error
    return new AppError(error.message || 'Unknown error', 'UNKNOWN_ERROR');
  }
};
```

## React Query Integration

### Query Hooks

```typescript
// src/hooks/api/useProducts.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { productService, ProductFilters, ProductCreateData } from '../../services/productService';
import { handleApiError } from '../../types/errors';

// Query keys
export const productKeys = {
  all: ['products'] as const,
  lists: () => [...productKeys.all, 'list'] as const,
  list: (filters: ProductFilters) => [...productKeys.lists(), filters] as const,
  details: () => [...productKeys.all, 'detail'] as const,
  detail: (id: string) => [...productKeys.details(), id] as const,
  categories: () => [...productKeys.all, 'categories'] as const,
};

// Hooks
export const useProducts = (filters: ProductFilters = {}) => {
  return useQuery({
    queryKey: productKeys.list(filters),
    queryFn: () => productService.getProducts(filters),
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 10 * 60 * 1000, // 10 minutes
    onError: handleApiError,
  });
};

export const useProduct = (id: string) => {
  return useQuery({
    queryKey: productKeys.detail(id),
    queryFn: () => productService.getProduct(id),
    enabled: !!id,
    staleTime: 5 * 60 * 1000,
    onError: handleApiError,
  });
};

export const useProductCategories = () => {
  return useQuery({
    queryKey: productKeys.categories(),
    queryFn: () => productService.getCategories(),
    staleTime: 30 * 60 * 1000, // 30 minutes
    onError: handleApiError,
  });
};

export const useCreateProduct = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (data: ProductCreateData) => productService.createProduct(data),
    onSuccess: () => {
      queryClient.invalidateQueries(productKeys.lists());
    },
    onError: handleApiError,
  });
};

export const useUpdateProduct = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<ProductCreateData> }) =>
      productService.updateProduct(id, data),
    onSuccess: (_, { id }) => {
      queryClient.invalidateQueries(productKeys.detail(id));
      queryClient.invalidateQueries(productKeys.lists());
    },
    onError: handleApiError,
  });
};

export const useDeleteProduct = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (id: string) => productService.deleteProduct(id),
    onSuccess: (_, id) => {
      queryClient.removeQueries(productKeys.detail(id));
      queryClient.invalidateQueries(productKeys.lists());
    },
    onError: handleApiError,
  });
};
```

## WebSocket Integration

### Socket Events

```typescript
// src/lib/socket.ts
export interface SocketEvents {
  // Connection events
  connect: () => void;
  disconnect: (reason: string) => void;
  connect_error: (error: Error) => void;
  
  // Authentication events
  authenticate: (token: string) => void;
  authenticated: (user: User) => void;
  authentication_error: (error: string) => void;
  
  // Chat events
  'chat:join': (groupId: string) => void;
  'chat:leave': (groupId: string) => void;
  'chat:message': (data: {
    groupId: string;
    content: string;
    type: string;
    metadata?: any;
  }) => void;
  'chat:message_received': (message: Message) => void;
  'chat:typing_start': (data: { groupId: string; userId: string }) => void;
  'chat:typing_stop': (data: { groupId: string; userId: string }) => void;
  'chat:user_typing': (data: { groupId: string; user: User }) => void;
  'chat:user_stopped_typing': (data: { groupId: string; userId: string }) => void;
  'chat:message_read': (data: { groupId: string; messageId: string; userId: string }) => void;
  'chat:group_updated': (group: Group) => void;
  
  // User events
  'user:status_change': (data: { userId: string; status: 'online' | 'offline' | 'away' }) => void;
  'user:profile_updated': (user: User) => void;
  
  // Product events
  'product:updated': (product: Product) => void;
  'product:deleted': (productId: string) => void;
  'product:inventory_changed': (data: { productId: string; quantity: number }) => void;
  
  // Order events
  'order:created': (order: Order) => void;
  'order:updated': (order: Order) => void;
  'order:status_changed': (data: { orderId: string; status: string; timestamp: string }) => void;
  
  // Notification events
  'notification:new': (notification: {
    id: string;
    type: string;
    title: string;
    message: string;
    data?: any;
    createdAt: string;
  }) => void;
}

// Socket service
class SocketService {
  private socket: Socket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  
  connect(token: string): void {
    if (this.socket?.connected) {
      return;
    }
    
    this.socket = io(env.socketUrl, {
      auth: { token },
      transports: ['websocket', 'polling'],
      timeout: 20000,
      reconnection: true,
      reconnectionAttempts: this.maxReconnectAttempts,
      reconnectionDelay: 1000,
    });
    
    this.setupEventListeners();
  }
  
  disconnect(): void {
    if (this.socket) {
      this.socket.disconnect();
      this.socket = null;
    }
  }
  
  emit<K extends keyof SocketEvents>(event: K, data?: Parameters<SocketEvents[K]>[0]): void {
    if (this.socket?.connected) {
      this.socket.emit(event, data);
    }
  }
  
  on<K extends keyof SocketEvents>(event: K, callback: SocketEvents[K]): void {
    if (this.socket) {
      this.socket.on(event, callback);
    }
  }
  
  off<K extends keyof SocketEvents>(event: K, callback?: SocketEvents[K]): void {
    if (this.socket) {
      this.socket.off(event, callback);
    }
  }
  
  private setupEventListeners(): void {
    if (!this.socket) return;
    
    this.socket.on('connect', () => {
      console.log('Socket connected');
      this.reconnectAttempts = 0;
    });
    
    this.socket.on('disconnect', (reason) => {
      console.log('Socket disconnected:', reason);
    });
    
    this.socket.on('connect_error', (error) => {
      console.error('Socket connection error:', error);
      this.reconnectAttempts++;
      
      if (this.reconnectAttempts >= this.maxReconnectAttempts) {
        console.error('Max reconnection attempts reached');
        this.disconnect();
      }
    });
  }
  
  get connected(): boolean {
    return this.socket?.connected || false;
  }
  
  get id(): string | undefined {
    return this.socket?.id;
  }
}

export const socketService = new SocketService();
```

## Rate Limiting

### Client-side Rate Limiting

```typescript
// src/lib/rateLimiter.ts
interface RateLimitConfig {
  maxRequests: number;
  windowMs: number;
  skipSuccessfulRequests?: boolean;
  skipFailedRequests?: boolean;
}

class RateLimiter {
  private requests: Map<string, number[]> = new Map();
  
  constructor(private config: RateLimitConfig) {}
  
  canMakeRequest(key: string): boolean {
    const now = Date.now();
    const windowStart = now - this.config.windowMs;
    
    // Get existing requests for this key
    const requests = this.requests.get(key) || [];
    
    // Filter out requests outside the current window
    const validRequests = requests.filter(timestamp => timestamp > windowStart);
    
    // Update the requests array
    this.requests.set(key, validRequests);
    
    // Check if we can make another request
    return validRequests.length < this.config.maxRequests;
  }
  
  recordRequest(key: string): void {
    const now = Date.now();
    const requests = this.requests.get(key) || [];
    requests.push(now);
    this.requests.set(key, requests);
  }
  
  getRemainingRequests(key: string): number {
    const now = Date.now();
    const windowStart = now - this.config.windowMs;
    const requests = this.requests.get(key) || [];
    const validRequests = requests.filter(timestamp => timestamp > windowStart);
    
    return Math.max(0, this.config.maxRequests - validRequests.length);
  }
  
  getResetTime(key: string): number {
    const requests = this.requests.get(key) || [];
    if (requests.length === 0) {
      return 0;
    }
    
    const oldestRequest = Math.min(...requests);
    return oldestRequest + this.config.windowMs;
  }
}

// Rate limiters for different endpoints
export const apiRateLimiter = new RateLimiter({
  maxRequests: 100,
  windowMs: 60 * 1000, // 1 minute
});

export const uploadRateLimiter = new RateLimiter({
  maxRequests: 10,
  windowMs: 60 * 1000, // 1 minute
});

export const chatRateLimiter = new RateLimiter({
  maxRequests: 50,
  windowMs: 60 * 1000, // 1 minute
});
```

## Next Steps

For more detailed information:
- [Authentication Guide](./authentication.md) - Authentication implementation details
- [Real-time Communication](./real-time-communication.md) - WebSocket and chat features
- [State Management](./state-management.md) - State management patterns
- [Testing Guide](./testing.md) - API testing strategies
- [Performance Guide](./performance.md) - API optimization techniques
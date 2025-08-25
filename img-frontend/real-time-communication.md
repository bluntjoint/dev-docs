# Real-time Communication

## Overview

This document provides a comprehensive guide to the real-time communication implementation in the Deed-O Frontend application, covering WebSocket connections, messaging systems, and real-time features.

## Technology Stack

- **Socket.io Client**: Real-time bidirectional communication
- **WebSocket**: Underlying transport protocol
- **React Query**: Real-time data synchronization
- **Zustand**: Real-time state management
- **Event-driven Architecture**: Decoupled communication patterns

## Architecture Overview

```
Real-time Communication Architecture
├── WebSocket Connection
│   ├── Socket.io Client
│   ├── Connection Management
│   ├── Reconnection Logic
│   └── Error Handling
├── Messaging System
│   ├── Chat Messages
│   ├── Group Communication
│   ├── Notifications
│   └── Status Updates
├── Event Management
│   ├── Event Listeners
│   ├── Event Emitters
│   ├── Event Routing
│   └── Event Persistence
└── State Synchronization
    ├── Real-time Updates
    ├── Optimistic Updates
    ├── Conflict Resolution
    └── Cache Invalidation
```

## WebSocket Implementation

### 1. Socket Service

```typescript
// src/services/socket.service.ts
import { io, Socket } from 'socket.io-client';
import { useAuthStore } from '../stores/auth.store';
import { useChatStore } from '../stores/chat.store';
import { useNotificationStore } from '../stores/notification.store';
import { SecurityLogger } from '../lib/securityLogger';

interface SocketConfig {
  url: string;
  options: {
    transports: string[];
    timeout: number;
    forceNew: boolean;
    reconnection: boolean;
    reconnectionAttempts: number;
    reconnectionDelay: number;
    maxReconnectionDelay: number;
  };
}

class SocketService {
  private socket: Socket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private isConnecting = false;
  private eventListeners: Map<string, Function[]> = new Map();
  
  private config: SocketConfig = {
    url: process.env.VITE_SOCKET_URL || 'ws://localhost:3001',
    options: {
      transports: ['websocket', 'polling'],
      timeout: 20000,
      forceNew: false,
      reconnection: true,
      reconnectionAttempts: 5,
      reconnectionDelay: 1000,
      maxReconnectionDelay: 5000,
    },
  };
  
  // Initialize socket connection
  async connect(): Promise<void> {
    if (this.socket?.connected || this.isConnecting) {
      return;
    }
    
    this.isConnecting = true;
    
    try {
      const token = useAuthStore.getState().token;
      
      if (!token) {
        throw new Error('No authentication token available');
      }
      
      // Create socket connection with authentication
      this.socket = io(this.config.url, {
        ...this.config.options,
        auth: {
          token,
        },
        query: {
          userId: useAuthStore.getState().user?.id,
        },
      });
      
      // Setup event listeners
      this.setupEventListeners();
      
      // Wait for connection
      await this.waitForConnection();
      
      SecurityLogger.log({
        type: 'websocket_connected',
        userId: useAuthStore.getState().user?.id,
        details: { url: this.config.url },
      });
      
    } catch (error) {
      console.error('Socket connection failed:', error);
      throw error;
    } finally {
      this.isConnecting = false;
    }
  }
  
  // Disconnect socket
  disconnect(): void {
    if (this.socket) {
      this.socket.disconnect();
      this.socket = null;
      this.reconnectAttempts = 0;
      
      SecurityLogger.log({
        type: 'websocket_disconnected',
        userId: useAuthStore.getState().user?.id,
      });
    }
  }
  
  // Check connection status
  isConnected(): boolean {
    return this.socket?.connected || false;
  }
  
  // Emit event to server
  emit(event: string, data?: any): void {
    if (!this.socket?.connected) {
      console.warn('Socket not connected, queuing event:', event);
      // Optionally queue events for when connection is restored
      return;
    }
    
    this.socket.emit(event, data);
  }
  
  // Listen for events
  on(event: string, callback: Function): void {
    if (!this.eventListeners.has(event)) {
      this.eventListeners.set(event, []);
    }
    
    this.eventListeners.get(event)!.push(callback);
    
    if (this.socket) {
      this.socket.on(event, callback as any);
    }
  }
  
  // Remove event listener
  off(event: string, callback?: Function): void {
    if (callback) {
      const listeners = this.eventListeners.get(event) || [];
      const index = listeners.indexOf(callback);
      if (index > -1) {
        listeners.splice(index, 1);
      }
      
      if (this.socket) {
        this.socket.off(event, callback as any);
      }
    } else {
      this.eventListeners.delete(event);
      
      if (this.socket) {
        this.socket.off(event);
      }
    }
  }
  
  // Join a room
  joinRoom(roomId: string): void {
    this.emit('join_room', { roomId });
  }
  
  // Leave a room
  leaveRoom(roomId: string): void {
    this.emit('leave_room', { roomId });
  }
  
  // Send message
  sendMessage(message: {
    groupId: string;
    content: string;
    type: 'text' | 'image' | 'file';
    metadata?: any;
  }): void {
    this.emit('send_message', message);
  }
  
  // Send typing indicator
  sendTyping(groupId: string, isTyping: boolean): void {
    this.emit('typing', { groupId, isTyping });
  }
  
  // Update user status
  updateStatus(status: 'online' | 'away' | 'busy' | 'offline'): void {
    this.emit('status_update', { status });
  }
  
  // Private methods
  private setupEventListeners(): void {
    if (!this.socket) return;
    
    // Connection events
    this.socket.on('connect', () => {
      console.log('Socket connected');
      this.reconnectAttempts = 0;
      
      // Re-register existing event listeners
      this.eventListeners.forEach((callbacks, event) => {
        callbacks.forEach(callback => {
          this.socket!.on(event, callback as any);
        });
      });
    });
    
    this.socket.on('disconnect', (reason) => {
      console.log('Socket disconnected:', reason);
      
      if (reason === 'io server disconnect') {
        // Server initiated disconnect, don't reconnect
        return;
      }
      
      // Attempt reconnection
      this.handleReconnection();
    });
    
    this.socket.on('connect_error', (error) => {
      console.error('Socket connection error:', error);
      this.handleReconnection();
    });
    
    // Authentication events
    this.socket.on('auth_error', (error) => {
      console.error('Socket authentication error:', error);
      useAuthStore.getState().logout();
    });
    
    // Message events
    this.socket.on('new_message', (message) => {
      useChatStore.getState().addMessage(message);
    });
    
    this.socket.on('message_updated', (message) => {
      useChatStore.getState().updateMessage(message);
    });
    
    this.socket.on('message_deleted', (messageId) => {
      useChatStore.getState().deleteMessage(messageId);
    });
    
    // Typing events
    this.socket.on('user_typing', ({ userId, groupId, isTyping }) => {
      useChatStore.getState().setUserTyping(userId, groupId, isTyping);
    });
    
    // Status events
    this.socket.on('user_status_changed', ({ userId, status }) => {
      useChatStore.getState().updateUserStatus(userId, status);
    });
    
    // Group events
    this.socket.on('group_updated', (group) => {
      useChatStore.getState().updateGroup(group);
    });
    
    this.socket.on('user_joined_group', ({ userId, groupId }) => {
      useChatStore.getState().addGroupMember(groupId, userId);
    });
    
    this.socket.on('user_left_group', ({ userId, groupId }) => {
      useChatStore.getState().removeGroupMember(groupId, userId);
    });
    
    // Notification events
    this.socket.on('notification', (notification) => {
      useNotificationStore.getState().addNotification(notification);
    });
    
    // Error events
    this.socket.on('error', (error) => {
      console.error('Socket error:', error);
      useNotificationStore.getState().addNotification({
        type: 'error',
        message: error.message || 'An error occurred',
      });
    });
  }
  
  private async waitForConnection(): Promise<void> {
    return new Promise((resolve, reject) => {
      if (!this.socket) {
        reject(new Error('Socket not initialized'));
        return;
      }
      
      const timeout = setTimeout(() => {
        reject(new Error('Connection timeout'));
      }, this.config.options.timeout);
      
      this.socket.once('connect', () => {
        clearTimeout(timeout);
        resolve();
      });
      
      this.socket.once('connect_error', (error) => {
        clearTimeout(timeout);
        reject(error);
      });
    });
  }
  
  private handleReconnection(): void {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnection attempts reached');
      useNotificationStore.getState().addNotification({
        type: 'error',
        message: 'Connection lost. Please refresh the page.',
        persistent: true,
      });
      return;
    }
    
    this.reconnectAttempts++;
    
    const delay = Math.min(
      this.config.options.reconnectionDelay * Math.pow(2, this.reconnectAttempts - 1),
      this.config.options.maxReconnectionDelay
    );
    
    setTimeout(() => {
      console.log(`Attempting reconnection ${this.reconnectAttempts}/${this.maxReconnectAttempts}`);
      this.connect().catch(console.error);
    }, delay);
  }
}

export const socketService = new SocketService();
```

### 2. Chat Service Integration

```typescript
// src/services/chat.service.ts
import { socketService } from './socket.service';
import api from '../lib/api';
import { useChatStore } from '../stores/chat.store';

interface Message {
  id: string;
  groupId: string;
  senderId: string;
  content: string;
  type: 'text' | 'image' | 'file';
  metadata?: any;
  timestamp: string;
  status: 'sending' | 'sent' | 'delivered' | 'read' | 'failed';
}

interface Group {
  id: string;
  name: string;
  type: 'direct' | 'group';
  members: string[];
  lastMessage?: Message;
  unreadCount: number;
  createdAt: string;
  updatedAt: string;
}

class ChatService {
  // Initialize chat service
  async initialize(): Promise<void> {
    try {
      // Connect to socket
      await socketService.connect();
      
      // Load initial data
      await this.loadGroups();
      
      // Setup real-time listeners
      this.setupRealtimeListeners();
      
    } catch (error) {
      console.error('Failed to initialize chat service:', error);
      throw error;
    }
  }
  
  // Load user's groups
  async loadGroups(): Promise<Group[]> {
    try {
      const response = await api.get('/chat/groups');
      const groups = response.data.groups;
      
      // Update store
      useChatStore.getState().setGroups(groups);
      
      // Join all group rooms
      groups.forEach((group: Group) => {
        socketService.joinRoom(group.id);
      });
      
      return groups;
    } catch (error) {
      console.error('Failed to load groups:', error);
      throw error;
    }
  }
  
  // Load messages for a group
  async loadMessages(groupId: string, page = 1, limit = 50): Promise<Message[]> {
    try {
      const response = await api.get(`/chat/groups/${groupId}/messages`, {
        params: { page, limit },
      });
      
      const messages = response.data.messages;
      
      // Update store
      if (page === 1) {
        useChatStore.getState().setMessages(groupId, messages);
      } else {
        useChatStore.getState().prependMessages(groupId, messages);
      }
      
      return messages;
    } catch (error) {
      console.error('Failed to load messages:', error);
      throw error;
    }
  }
  
  // Send a message
  async sendMessage(message: {
    groupId: string;
    content: string;
    type: 'text' | 'image' | 'file';
    metadata?: any;
  }): Promise<Message> {
    try {
      // Create optimistic message
      const optimisticMessage: Message = {
        id: `temp-${Date.now()}`,
        groupId: message.groupId,
        senderId: useChatStore.getState().currentUserId!,
        content: message.content,
        type: message.type,
        metadata: message.metadata,
        timestamp: new Date().toISOString(),
        status: 'sending',
      };
      
      // Add to store immediately
      useChatStore.getState().addMessage(optimisticMessage);
      
      // Send via socket
      socketService.sendMessage(message);
      
      // Also send via HTTP for reliability
      const response = await api.post(`/chat/groups/${message.groupId}/messages`, message);
      const sentMessage = response.data.message;
      
      // Replace optimistic message with real one
      useChatStore.getState().replaceMessage(optimisticMessage.id, sentMessage);
      
      return sentMessage;
    } catch (error) {
      console.error('Failed to send message:', error);
      
      // Mark message as failed
      useChatStore.getState().updateMessageStatus(message.groupId, 'failed');
      
      throw error;
    }
  }
  
  // Upload and send file
  async sendFile(groupId: string, file: File): Promise<Message> {
    try {
      // Upload file first
      const formData = new FormData();
      formData.append('file', file);
      
      const uploadResponse = await api.post('/upload', formData, {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      });
      
      const fileUrl = uploadResponse.data.url;
      
      // Send message with file
      return await this.sendMessage({
        groupId,
        content: file.name,
        type: file.type.startsWith('image/') ? 'image' : 'file',
        metadata: {
          url: fileUrl,
          size: file.size,
          mimeType: file.type,
        },
      });
    } catch (error) {
      console.error('Failed to send file:', error);
      throw error;
    }
  }
  
  // Create a new group
  async createGroup(name: string, memberIds: string[]): Promise<Group> {
    try {
      const response = await api.post('/chat/groups', {
        name,
        memberIds,
      });
      
      const group = response.data.group;
      
      // Add to store
      useChatStore.getState().addGroup(group);
      
      // Join room
      socketService.joinRoom(group.id);
      
      return group;
    } catch (error) {
      console.error('Failed to create group:', error);
      throw error;
    }
  }
  
  // Add member to group
  async addGroupMember(groupId: string, userId: string): Promise<void> {
    try {
      await api.post(`/chat/groups/${groupId}/members`, { userId });
      
      // Update will come via socket
    } catch (error) {
      console.error('Failed to add group member:', error);
      throw error;
    }
  }
  
  // Remove member from group
  async removeGroupMember(groupId: string, userId: string): Promise<void> {
    try {
      await api.delete(`/chat/groups/${groupId}/members/${userId}`);
      
      // Update will come via socket
    } catch (error) {
      console.error('Failed to remove group member:', error);
      throw error;
    }
  }
  
  // Mark messages as read
  async markAsRead(groupId: string, messageIds: string[]): Promise<void> {
    try {
      await api.post(`/chat/groups/${groupId}/read`, { messageIds });
      
      // Update store
      useChatStore.getState().markMessagesAsRead(groupId, messageIds);
      
      // Emit read receipt
      socketService.emit('messages_read', { groupId, messageIds });
    } catch (error) {
      console.error('Failed to mark messages as read:', error);
    }
  }
  
  // Send typing indicator
  startTyping(groupId: string): void {
    socketService.sendTyping(groupId, true);
  }
  
  stopTyping(groupId: string): void {
    socketService.sendTyping(groupId, false);
  }
  
  // Update user status
  updateStatus(status: 'online' | 'away' | 'busy' | 'offline'): void {
    socketService.updateStatus(status);
  }
  
  // Search messages
  async searchMessages(query: string, groupId?: string): Promise<Message[]> {
    try {
      const response = await api.get('/chat/search', {
        params: { query, groupId },
      });
      
      return response.data.messages;
    } catch (error) {
      console.error('Failed to search messages:', error);
      throw error;
    }
  }
  
  // Get message history
  async getMessageHistory(groupId: string, beforeMessageId?: string): Promise<Message[]> {
    try {
      const response = await api.get(`/chat/groups/${groupId}/history`, {
        params: { before: beforeMessageId },
      });
      
      return response.data.messages;
    } catch (error) {
      console.error('Failed to get message history:', error);
      throw error;
    }
  }
  
  // Private methods
  private setupRealtimeListeners(): void {
    // Message events are handled in socket service
    // Additional chat-specific logic can be added here
  }
  
  // Cleanup
  cleanup(): void {
    socketService.disconnect();
  }
}

export const chatService = new ChatService();
```

### 3. Real-time Hooks

```typescript
// src/hooks/useSocket.ts
import { useEffect, useRef, useState } from 'react';
import { socketService } from '../services/socket.service';
import { useAuth } from './useAuth';

export const useSocket = () => {
  const [isConnected, setIsConnected] = useState(false);
  const [connectionError, setConnectionError] = useState<string | null>(null);
  const { isAuthenticated } = useAuth();
  const reconnectTimeoutRef = useRef<NodeJS.Timeout | null>(null);
  
  useEffect(() => {
    if (!isAuthenticated) {
      socketService.disconnect();
      setIsConnected(false);
      return;
    }
    
    const connect = async () => {
      try {
        await socketService.connect();
        setIsConnected(true);
        setConnectionError(null);
      } catch (error: any) {
        setConnectionError(error.message);
        setIsConnected(false);
        
        // Retry connection after delay
        reconnectTimeoutRef.current = setTimeout(connect, 5000);
      }
    };
    
    connect();
    
    // Cleanup
    return () => {
      if (reconnectTimeoutRef.current) {
        clearTimeout(reconnectTimeoutRef.current);
      }
      socketService.disconnect();
    };
  }, [isAuthenticated]);
  
  return {
    isConnected,
    connectionError,
    socket: socketService,
  };
};

// Hook for listening to specific events
export const useSocketEvent = (event: string, callback: Function) => {
  const callbackRef = useRef(callback);
  callbackRef.current = callback;
  
  useEffect(() => {
    const handler = (...args: any[]) => {
      callbackRef.current(...args);
    };
    
    socketService.on(event, handler);
    
    return () => {
      socketService.off(event, handler);
    };
  }, [event]);
};

// Hook for real-time data
export const useRealtimeData = <T>(initialData: T, event: string) => {
  const [data, setData] = useState<T>(initialData);
  
  useSocketEvent(event, (newData: T) => {
    setData(newData);
  });
  
  return data;
};
```

### 4. Chat Components

```typescript
// src/components/chat/ChatWindow.tsx
import { useEffect, useRef, useState } from 'react';
import { useChatStore } from '../../stores/chat.store';
import { chatService } from '../../services/chat.service';
import { useSocket, useSocketEvent } from '../../hooks/useSocket';
import { MessageList } from './MessageList';
import { MessageInput } from './MessageInput';
import { TypingIndicator } from './TypingIndicator';
import { ConnectionStatus } from './ConnectionStatus';

interface ChatWindowProps {
  groupId: string;
}

export const ChatWindow = ({ groupId }: ChatWindowProps) => {
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const typingTimeoutRef = useRef<NodeJS.Timeout | null>(null);
  
  const { isConnected } = useSocket();
  const {
    messages,
    groups,
    typingUsers,
    loadMessages,
    markAsRead,
  } = useChatStore();
  
  const currentGroup = groups.find(g => g.id === groupId);
  const groupMessages = messages[groupId] || [];
  const groupTypingUsers = typingUsers[groupId] || [];
  
  // Load messages when group changes
  useEffect(() => {
    const loadGroupMessages = async () => {
      if (!groupId) return;
      
      setIsLoading(true);
      setError(null);
      
      try {
        await chatService.loadMessages(groupId);
      } catch (err: any) {
        setError(err.message || 'Failed to load messages');
      } finally {
        setIsLoading(false);
      }
    };
    
    loadGroupMessages();
  }, [groupId]);
  
  // Auto-scroll to bottom when new messages arrive
  useEffect(() => {
    if (messagesEndRef.current) {
      messagesEndRef.current.scrollIntoView({ behavior: 'smooth' });
    }
  }, [groupMessages.length]);
  
  // Mark messages as read when they come into view
  useEffect(() => {
    const unreadMessages = groupMessages.filter(m => 
      m.status !== 'read' && m.senderId !== useChatStore.getState().currentUserId
    );
    
    if (unreadMessages.length > 0) {
      const messageIds = unreadMessages.map(m => m.id);
      chatService.markAsRead(groupId, messageIds);
    }
  }, [groupId, groupMessages]);
  
  // Handle new messages
  useSocketEvent('new_message', (message) => {
    if (message.groupId === groupId) {
      // Auto-scroll if user is at bottom
      const container = messagesEndRef.current?.parentElement;
      if (container) {
        const isAtBottom = container.scrollHeight - container.scrollTop <= container.clientHeight + 100;
        if (isAtBottom) {
          setTimeout(() => {
            messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
          }, 100);
        }
      }
    }
  });
  
  // Handle message sending
  const handleSendMessage = async (content: string, type: 'text' | 'image' | 'file' = 'text') => {
    try {
      await chatService.sendMessage({
        groupId,
        content,
        type,
      });
    } catch (error: any) {
      console.error('Failed to send message:', error);
      // Error handling is done in the service
    }
  };
  
  // Handle file upload
  const handleFileUpload = async (file: File) => {
    try {
      await chatService.sendFile(groupId, file);
    } catch (error: any) {
      console.error('Failed to send file:', error);
    }
  };
  
  // Handle typing
  const handleTypingStart = () => {
    chatService.startTyping(groupId);
    
    // Clear existing timeout
    if (typingTimeoutRef.current) {
      clearTimeout(typingTimeoutRef.current);
    }
    
    // Stop typing after 3 seconds of inactivity
    typingTimeoutRef.current = setTimeout(() => {
      chatService.stopTyping(groupId);
    }, 3000);
  };
  
  const handleTypingStop = () => {
    if (typingTimeoutRef.current) {
      clearTimeout(typingTimeoutRef.current);
    }
    chatService.stopTyping(groupId);
  };
  
  if (isLoading) {
    return (
      <div className="flex items-center justify-center h-full">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600"></div>
      </div>
    );
  }
  
  if (error) {
    return (
      <div className="flex items-center justify-center h-full">
        <div className="text-center">
          <p className="text-red-600 mb-4">{error}</p>
          <button
            onClick={() => window.location.reload()}
            className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700"
          >
            Retry
          </button>
        </div>
      </div>
    );
  }
  
  if (!currentGroup) {
    return (
      <div className="flex items-center justify-center h-full">
        <p className="text-gray-500">Group not found</p>
      </div>
    );
  }
  
  return (
    <div className="flex flex-col h-full">
      {/* Header */}
      <div className="flex items-center justify-between p-4 border-b">
        <div>
          <h2 className="text-lg font-semibold">{currentGroup.name}</h2>
          <p className="text-sm text-gray-500">
            {currentGroup.members.length} members
          </p>
        </div>
        <ConnectionStatus isConnected={isConnected} />
      </div>
      
      {/* Messages */}
      <div className="flex-1 overflow-y-auto p-4">
        <MessageList messages={groupMessages} />
        <TypingIndicator users={groupTypingUsers} />
        <div ref={messagesEndRef} />
      </div>
      
      {/* Input */}
      <div className="border-t p-4">
        <MessageInput
          onSendMessage={handleSendMessage}
          onFileUpload={handleFileUpload}
          onTypingStart={handleTypingStart}
          onTypingStop={handleTypingStop}
          disabled={!isConnected}
        />
      </div>
    </div>
  );
};
```

### 5. Typing Indicator Component

```typescript
// src/components/chat/TypingIndicator.tsx
import { useEffect, useState } from 'react';

interface TypingIndicatorProps {
  users: Array<{
    id: string;
    name: string;
  }>;
}

export const TypingIndicator = ({ users }: TypingIndicatorProps) => {
  const [dots, setDots] = useState('');
  
  useEffect(() => {
    if (users.length === 0) return;
    
    const interval = setInterval(() => {
      setDots(prev => {
        if (prev === '...') return '';
        return prev + '.';
      });
    }, 500);
    
    return () => clearInterval(interval);
  }, [users.length]);
  
  if (users.length === 0) {
    return null;
  }
  
  const getTypingText = () => {
    if (users.length === 1) {
      return `${users[0].name} is typing${dots}`;
    } else if (users.length === 2) {
      return `${users[0].name} and ${users[1].name} are typing${dots}`;
    } else {
      return `${users[0].name} and ${users.length - 1} others are typing${dots}`;
    }
  };
  
  return (
    <div className="flex items-center space-x-2 py-2 text-sm text-gray-500">
      <div className="flex space-x-1">
        <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce"></div>
        <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce" style={{ animationDelay: '0.1s' }}></div>
        <div className="w-2 h-2 bg-gray-400 rounded-full animate-bounce" style={{ animationDelay: '0.2s' }}></div>
      </div>
      <span>{getTypingText()}</span>
    </div>
  );
};
```

### 6. Connection Status Component

```typescript
// src/components/chat/ConnectionStatus.tsx
import { useEffect, useState } from 'react';

interface ConnectionStatusProps {
  isConnected: boolean;
}

export const ConnectionStatus = ({ isConnected }: ConnectionStatusProps) => {
  const [showStatus, setShowStatus] = useState(false);
  
  useEffect(() => {
    if (!isConnected) {
      setShowStatus(true);
    } else {
      // Hide status after a delay when connected
      const timeout = setTimeout(() => {
        setShowStatus(false);
      }, 2000);
      
      return () => clearTimeout(timeout);
    }
  }, [isConnected]);
  
  if (!showStatus) {
    return null;
  }
  
  return (
    <div className={`flex items-center space-x-2 text-sm ${
      isConnected ? 'text-green-600' : 'text-red-600'
    }`}>
      <div className={`w-2 h-2 rounded-full ${
        isConnected ? 'bg-green-500' : 'bg-red-500'
      }`}></div>
      <span>
        {isConnected ? 'Connected' : 'Disconnected'}
      </span>
    </div>
  );
};
```

## Real-time State Management

### 1. Chat Store

```typescript
// src/stores/chat.store.ts
import { create } from 'zustand';
import { devtools, subscribeWithSelector } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface Message {
  id: string;
  groupId: string;
  senderId: string;
  content: string;
  type: 'text' | 'image' | 'file';
  metadata?: any;
  timestamp: string;
  status: 'sending' | 'sent' | 'delivered' | 'read' | 'failed';
}

interface Group {
  id: string;
  name: string;
  type: 'direct' | 'group';
  members: string[];
  lastMessage?: Message;
  unreadCount: number;
  createdAt: string;
  updatedAt: string;
}

interface TypingUser {
  id: string;
  name: string;
}

interface ChatState {
  // Data
  groups: Group[];
  messages: Record<string, Message[]>; // groupId -> messages
  typingUsers: Record<string, TypingUser[]>; // groupId -> typing users
  userStatuses: Record<string, 'online' | 'away' | 'busy' | 'offline'>;
  currentUserId: string | null;
  
  // UI State
  activeGroupId: string | null;
  isLoading: boolean;
  error: string | null;
  
  // Actions
  setCurrentUser: (userId: string) => void;
  setGroups: (groups: Group[]) => void;
  addGroup: (group: Group) => void;
  updateGroup: (group: Group) => void;
  removeGroup: (groupId: string) => void;
  
  setMessages: (groupId: string, messages: Message[]) => void;
  addMessage: (message: Message) => void;
  updateMessage: (message: Message) => void;
  deleteMessage: (messageId: string) => void;
  replaceMessage: (tempId: string, message: Message) => void;
  prependMessages: (groupId: string, messages: Message[]) => void;
  markMessagesAsRead: (groupId: string, messageIds: string[]) => void;
  updateMessageStatus: (groupId: string, status: Message['status']) => void;
  
  setUserTyping: (userId: string, groupId: string, isTyping: boolean) => void;
  updateUserStatus: (userId: string, status: 'online' | 'away' | 'busy' | 'offline') => void;
  
  addGroupMember: (groupId: string, userId: string) => void;
  removeGroupMember: (groupId: string, userId: string) => void;
  
  setActiveGroup: (groupId: string | null) => void;
  setLoading: (loading: boolean) => void;
  setError: (error: string | null) => void;
  
  // Computed
  getUnreadCount: () => number;
  getGroupById: (groupId: string) => Group | undefined;
  getMessagesByGroup: (groupId: string) => Message[];
}

export const useChatStore = create<ChatState>()()
  devtools(
    subscribeWithSelector(
      immer((set, get) => ({
        // Initial state
        groups: [],
        messages: {},
        typingUsers: {},
        userStatuses: {},
        currentUserId: null,
        activeGroupId: null,
        isLoading: false,
        error: null,
        
        // Actions
        setCurrentUser: (userId) => {
          set((state) => {
            state.currentUserId = userId;
          });
        },
        
        setGroups: (groups) => {
          set((state) => {
            state.groups = groups;
          });
        },
        
        addGroup: (group) => {
          set((state) => {
            const existingIndex = state.groups.findIndex(g => g.id === group.id);
            if (existingIndex >= 0) {
              state.groups[existingIndex] = group;
            } else {
              state.groups.push(group);
            }
          });
        },
        
        updateGroup: (group) => {
          set((state) => {
            const index = state.groups.findIndex(g => g.id === group.id);
            if (index >= 0) {
              state.groups[index] = { ...state.groups[index], ...group };
            }
          });
        },
        
        removeGroup: (groupId) => {
          set((state) => {
            state.groups = state.groups.filter(g => g.id !== groupId);
            delete state.messages[groupId];
            delete state.typingUsers[groupId];
          });
        },
        
        setMessages: (groupId, messages) => {
          set((state) => {
            state.messages[groupId] = messages;
          });
        },
        
        addMessage: (message) => {
          set((state) => {
            if (!state.messages[message.groupId]) {
              state.messages[message.groupId] = [];
            }
            
            // Check if message already exists
            const existingIndex = state.messages[message.groupId].findIndex(
              m => m.id === message.id
            );
            
            if (existingIndex >= 0) {
              state.messages[message.groupId][existingIndex] = message;
            } else {
              state.messages[message.groupId].push(message);
              
              // Sort by timestamp
              state.messages[message.groupId].sort(
                (a, b) => new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime()
              );
            }
            
            // Update group's last message
            const groupIndex = state.groups.findIndex(g => g.id === message.groupId);
            if (groupIndex >= 0) {
              state.groups[groupIndex].lastMessage = message;
              
              // Increment unread count if not from current user
              if (message.senderId !== state.currentUserId) {
                state.groups[groupIndex].unreadCount += 1;
              }
            }
          });
        },
        
        updateMessage: (message) => {
          set((state) => {
            const groupMessages = state.messages[message.groupId];
            if (groupMessages) {
              const index = groupMessages.findIndex(m => m.id === message.id);
              if (index >= 0) {
                groupMessages[index] = { ...groupMessages[index], ...message };
              }
            }
          });
        },
        
        deleteMessage: (messageId) => {
          set((state) => {
            Object.keys(state.messages).forEach(groupId => {
              state.messages[groupId] = state.messages[groupId].filter(
                m => m.id !== messageId
              );
            });
          });
        },
        
        replaceMessage: (tempId, message) => {
          set((state) => {
            const groupMessages = state.messages[message.groupId];
            if (groupMessages) {
              const index = groupMessages.findIndex(m => m.id === tempId);
              if (index >= 0) {
                groupMessages[index] = message;
              }
            }
          });
        },
        
        prependMessages: (groupId, messages) => {
          set((state) => {
            if (!state.messages[groupId]) {
              state.messages[groupId] = [];
            }
            
            state.messages[groupId] = [
              ...messages,
              ...state.messages[groupId],
            ];
          });
        },
        
        markMessagesAsRead: (groupId, messageIds) => {
          set((state) => {
            const groupMessages = state.messages[groupId];
            if (groupMessages) {
              groupMessages.forEach(message => {
                if (messageIds.includes(message.id)) {
                  message.status = 'read';
                }
              });
            }
            
            // Reset unread count
            const groupIndex = state.groups.findIndex(g => g.id === groupId);
            if (groupIndex >= 0) {
              state.groups[groupIndex].unreadCount = 0;
            }
          });
        },
        
        updateMessageStatus: (groupId, status) => {
          set((state) => {
            const groupMessages = state.messages[groupId];
            if (groupMessages) {
              // Update the last message with the given status
              const lastMessage = groupMessages[groupMessages.length - 1];
              if (lastMessage && lastMessage.senderId === state.currentUserId) {
                lastMessage.status = status;
              }
            }
          });
        },
        
        setUserTyping: (userId, groupId, isTyping) => {
          set((state) => {
            if (!state.typingUsers[groupId]) {
              state.typingUsers[groupId] = [];
            }
            
            const typingUsers = state.typingUsers[groupId];
            const existingIndex = typingUsers.findIndex(u => u.id === userId);
            
            if (isTyping) {
              if (existingIndex === -1) {
                // Add user to typing list
                typingUsers.push({ id: userId, name: `User ${userId}` });
              }
            } else {
              if (existingIndex >= 0) {
                // Remove user from typing list
                typingUsers.splice(existingIndex, 1);
              }
            }
          });
        },
        
        updateUserStatus: (userId, status) => {
          set((state) => {
            state.userStatuses[userId] = status;
          });
        },
        
        addGroupMember: (groupId, userId) => {
          set((state) => {
            const groupIndex = state.groups.findIndex(g => g.id === groupId);
            if (groupIndex >= 0) {
              const group = state.groups[groupIndex];
              if (!group.members.includes(userId)) {
                group.members.push(userId);
              }
            }
          });
        },
        
        removeGroupMember: (groupId, userId) => {
          set((state) => {
            const groupIndex = state.groups.findIndex(g => g.id === groupId);
            if (groupIndex >= 0) {
              const group = state.groups[groupIndex];
              group.members = group.members.filter(id => id !== userId);
            }
          });
        },
        
        setActiveGroup: (groupId) => {
          set((state) => {
            state.activeGroupId = groupId;
          });
        },
        
        setLoading: (loading) => {
          set((state) => {
            state.isLoading = loading;
          });
        },
        
        setError: (error) => {
          set((state) => {
            state.error = error;
          });
        },
        
        // Computed
        getUnreadCount: () => {
          const state = get();
          return state.groups.reduce((total, group) => total + group.unreadCount, 0);
        },
        
        getGroupById: (groupId) => {
          const state = get();
          return state.groups.find(g => g.id === groupId);
        },
        
        getMessagesByGroup: (groupId) => {
          const state = get();
          return state.messages[groupId] || [];
        },
      }))
    ),
    {
      name: 'chat-store',
    }
  )
);

// Selectors
export const selectGroups = (state: ChatState) => state.groups;
export const selectMessages = (groupId: string) => (state: ChatState) => 
  state.messages[groupId] || [];
export const selectTypingUsers = (groupId: string) => (state: ChatState) => 
  state.typingUsers[groupId] || [];
export const selectUnreadCount = (state: ChatState) => 
  state.groups.reduce((total, group) => total + group.unreadCount, 0);
```

## Performance Optimization

### 1. Message Virtualization

```typescript
// src/components/chat/VirtualizedMessageList.tsx
import { FixedSizeList as List } from 'react-window';
import { useMemo } from 'react';
import { Message } from '../../types/chat';
import { MessageItem } from './MessageItem';

interface VirtualizedMessageListProps {
  messages: Message[];
  height: number;
  itemHeight: number;
}

export const VirtualizedMessageList = ({
  messages,
  height,
  itemHeight,
}: VirtualizedMessageListProps) => {
  const itemData = useMemo(() => ({ messages }), [messages]);
  
  const Row = ({ index, style, data }: any) => (
    <div style={style}>
      <MessageItem message={data.messages[index]} />
    </div>
  );
  
  return (
    <List
      height={height}
      itemCount={messages.length}
      itemSize={itemHeight}
      itemData={itemData}
      overscanCount={5}
    >
      {Row}
    </List>
  );
};
```

### 2. Connection Pooling

```typescript
// src/lib/connectionPool.ts

class ConnectionPool {
  private connections: Map<string, Socket> = new Map();
  private maxConnections = 5;
  
  getConnection(url: string): Socket | null {
    return this.connections.get(url) || null;
  }
  
  addConnection(url: string, socket: Socket): void {
    if (this.connections.size >= this.maxConnections) {
      // Remove oldest connection
      const firstKey = this.connections.keys().next().value;
      const oldSocket = this.connections.get(firstKey);
      oldSocket?.disconnect();
      this.connections.delete(firstKey);
    }
    
    this.connections.set(url, socket);
  }
  
  removeConnection(url: string): void {
    const socket = this.connections.get(url);
    if (socket) {
      socket.disconnect();
      this.connections.delete(url);
    }
  }
  
  cleanup(): void {
    this.connections.forEach(socket => socket.disconnect());
    this.connections.clear();
  }
}

export const connectionPool = new ConnectionPool();
```

## Testing Real-time Features

### 1. Socket Service Tests

```typescript
// src/services/__tests__/socket.service.test.ts
import { socketService } from '../socket.service';
import { Server } from 'socket.io';
import { createServer } from 'http';
import { io as Client } from 'socket.io-client';

describe('SocketService', () => {
  let httpServer: any;
  let ioServer: Server;
  let clientSocket: any;
  
  beforeAll((done) => {
    httpServer = createServer();
    ioServer = new Server(httpServer);
    
    httpServer.listen(() => {
      const port = httpServer.address().port;
      clientSocket = Client(`http://localhost:${port}`);
      
      ioServer.on('connection', (socket) => {
        socket.emit('connected');
      });
      
      clientSocket.on('connected', done);
    });
  });
  
  afterAll(() => {
    ioServer.close();
    httpServer.close();
  });
  
  test('should connect to server', (done) => {
    clientSocket.on('connect', () => {
      expect(clientSocket.connected).toBe(true);
      done();
    });
  });
  
  test('should emit and receive messages', (done) => {
    const testMessage = { content: 'Hello, World!' };
    
    ioServer.on('connection', (socket) => {
      socket.on('test_message', (data) => {
        expect(data).toEqual(testMessage);
        socket.emit('message_received', data);
      });
    });
    
    clientSocket.on('message_received', (data: any) => {
      expect(data).toEqual(testMessage);
      done();
    });
    
    clientSocket.emit('test_message', testMessage);
  });
});
```

### 2. Chat Component Tests

```typescript
// src/components/chat/__tests__/ChatWindow.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { ChatWindow } from '../ChatWindow';
import { useChatStore } from '../../../stores/chat.store';
import { chatService } from '../../../services/chat.service';

jest.mock('../../../services/chat.service');
jest.mock('../../../stores/chat.store');

const mockChatService = chatService as jest.Mocked<typeof chatService>;
const mockUseChatStore = useChatStore as jest.MockedFunction<typeof useChatStore>;

describe('ChatWindow', () => {
  const mockGroup = {
    id: 'group-1',
    name: 'Test Group',
    type: 'group' as const,
    members: ['user-1', 'user-2'],
    unreadCount: 0,
    createdAt: '2023-01-01T00:00:00Z',
    updatedAt: '2023-01-01T00:00:00Z',
  };
  
  const mockMessages = [
    {
      id: 'msg-1',
      groupId: 'group-1',
      senderId: 'user-1',
      content: 'Hello!',
      type: 'text' as const,
      timestamp: '2023-01-01T00:00:00Z',
      status: 'sent' as const,
    },
  ];
  
  beforeEach(() => {
    mockUseChatStore.mockReturnValue({
      groups: [mockGroup],
      messages: { 'group-1': mockMessages },
      typingUsers: {},
      loadMessages: jest.fn(),
      markAsRead: jest.fn(),
    } as any);
    
    mockChatService.loadMessages.mockResolvedValue(mockMessages);
  });
  
  test('should render group name and messages', async () => {
    render(<ChatWindow groupId="group-1" />);
    
    await waitFor(() => {
      expect(screen.getByText('Test Group')).toBeInTheDocument();
      expect(screen.getByText('Hello!')).toBeInTheDocument();
    });
  });
  
  test('should send message when form is submitted', async () => {
    mockChatService.sendMessage.mockResolvedValue(mockMessages[0]);
    
    render(<ChatWindow groupId="group-1" />);
    
    const input = screen.getByPlaceholderText('Type a message...');
    const sendButton = screen.getByRole('button', { name: /send/i });
    
    fireEvent.change(input, { target: { value: 'New message' } });
    fireEvent.click(sendButton);
    
    await waitFor(() => {
      expect(mockChatService.sendMessage).toHaveBeenCalledWith({
        groupId: 'group-1',
        content: 'New message',
        type: 'text',
      });
    });
  });
});
```

## Security Considerations

### 1. WebSocket Authentication

```typescript
// Server-side authentication middleware
ioServer.use((socket, next) => {
  const token = socket.handshake.auth.token;
  
  if (!token) {
    return next(new Error('Authentication error'));
  }
  
  // Verify JWT token
  jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
    if (err) {
      return next(new Error('Authentication error'));
    }
    
    socket.userId = decoded.userId;
    next();
  });
});
```

### 2. Rate Limiting

```typescript
// src/lib/socketRateLimit.ts

class SocketRateLimit {
  private requests: Map<string, number[]> = new Map();
  private maxRequests = 10;
  private windowMs = 60000; // 1 minute
  
  isAllowed(userId: string): boolean {
    const now = Date.now();
    const userRequests = this.requests.get(userId) || [];
    
    // Filter out old requests
    const recentRequests = userRequests.filter(
      time => now - time < this.windowMs
    );
    
    if (recentRequests.length >= this.maxRequests) {
      return false;
    }
    
    recentRequests.push(now);
    this.requests.set(userId, recentRequests);
    
    return true;
  }
}

export const socketRateLimit = new SocketRateLimit();
```

## Next Steps

For more detailed information:
- [State Management](./state-management.md) - Real-time state patterns
- [Performance Guide](./performance.md) - WebSocket optimization
- [Testing Guide](./testing.md) - Real-time testing strategies
- [Authentication](./authentication.md) - WebSocket security
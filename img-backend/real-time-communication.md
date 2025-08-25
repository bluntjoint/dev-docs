# Real-Time Communication

## Overview

The Deed-O Backend implements real-time communication using WebSocket technology through Socket.IO. This enables instant messaging, user presence tracking, and live notifications.

## Architecture

### WebSocket Gateway

**File**: `src/chat.gateway.ts`

The `ChatGateway` serves as the main WebSocket handler, managing client connections, message broadcasting, and user presence.

```typescript
@WebSocketGateway({
  cors: {
    origin: ALLOWED_ORIGINS,
    methods: ['GET', 'POST'],
    credentials: true
  }
})
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private onlineUsers = new Map<number, string>();
}
```

### Key Features

1. **Cross-Origin Support**: Configured with CORS for web client access
2. **User Presence Tracking**: Maintains online user registry
3. **Message Broadcasting**: Real-time message delivery to group members
4. **Email Integration**: Notifications for offline users

## Connection Management

### Client Connection

```typescript
async handleConnection(client: Socket) {
  try {
    const token = client.handshake.auth.token;
    if (!token) {
      client.disconnect();
      return;
    }

    // Verify JWT token
    const payload = await this.jwtService.verifyToken(token);
    const userId = payload.sub;

    // Store user connection
    client.data.userId = userId;
    this.onlineUsers.set(userId, client.id);

    // Update user online status
    await this.chatService.setUserOnline(userId, true);

    // Join user to their groups
    const userGroups = await this.groupService.getGroupsByUserId(userId);
    for (const group of userGroups) {
      client.join(group.id);
    }

    console.log(`User ${userId} connected with socket ${client.id}`);
  } catch (error) {
    console.error('Connection error:', error);
    client.disconnect();
  }
}
```

**Connection Flow**:
1. Extract JWT token from handshake
2. Verify token authenticity
3. Store user-socket mapping
4. Update database online status
5. Join user to their group rooms
6. Handle connection errors gracefully

### Client Disconnection

```typescript
async handleDisconnect(client: Socket) {
  const userId = client.data.userId;
  if (userId) {
    // Remove from online users
    this.onlineUsers.delete(userId);
    
    // Update database status
    await this.chatService.setUserOnline(userId, false);
    
    console.log(`User ${userId} disconnected`);
  }
}
```

**Disconnection Flow**:
1. Retrieve user ID from client data
2. Remove from online users map
3. Update database offline status
4. Clean up resources

## Message Handling

### Send Message Event

```typescript
@SubscribeMessage('sendMessage')
async handleMessage(
  client: Socket,
  payload: {
    groupId: string;
    text: string;
    attachments: { url: string; type: string }[];
  }
) {
  try {
    const userId = client.data.userId;
    if (!userId) {
      client.emit('error', { message: 'User not authenticated' });
      return;
    }

    // Create message in database
    const message = await this.chatService.createMessage({
      senderId: userId,
      groupId: payload.groupId,
      text: payload.text,
      attachments: payload.attachments,
      readBy: [userId] // Sender has read the message
    });

    // Broadcast to group members
    this.server.to(payload.groupId).emit('newMessage', {
      id: message.id,
      senderId: userId,
      groupId: payload.groupId,
      text: payload.text,
      attachments: payload.attachments,
      timestamp: message.timestamp,
      readBy: message.readBy
    });

    // Send email notifications to offline members
    await this.sendEmailNotifications(
      payload.groupId,
      userId,
      payload.text
    );

    console.log(`Message sent in group ${payload.groupId} by user ${userId}`);
  } catch (error) {
    console.error('Error handling message:', error);
    client.emit('error', { message: 'Failed to send message' });
  }
}
```

**Message Flow**:
1. Validate user authentication
2. Create message in database
3. Broadcast to all group members
4. Send email notifications to offline users
5. Handle errors with client feedback

### Email Notifications for Offline Users

```typescript
private async sendEmailNotifications(
  groupId: string,
  senderId: number,
  messageText: string
) {
  try {
    // Get group members
    const group = await this.groupService.getGroupById(groupId);
    if (!group) return;

    // Filter offline members
    const offlineMembers = group.members.filter(
      (memberId) => 
        memberId !== senderId && 
        !this.onlineUsers.has(memberId)
    );

    // Send email to each offline member
    for (const memberId of offlineMembers) {
      await this.chatService.sendEmailNotification(
        memberId,
        senderId,
        messageText
      );
    }
  } catch (error) {
    console.error('Error sending email notifications:', error);
  }
}
```

**Notification Logic**:
1. Retrieve group member list
2. Filter out sender and online users
3. Send email to each offline member
4. Handle errors without affecting message delivery

## Room Management

### Group Rooms

Each group chat operates as a Socket.IO room:

```typescript
// Join user to group rooms on connection
const userGroups = await this.groupService.getGroupsByUserId(userId);
for (const group of userGroups) {
  client.join(group.id);
}

// Broadcast message to group room
this.server.to(payload.groupId).emit('newMessage', messageData);
```

**Room Benefits**:
- Efficient message broadcasting
- Automatic member management
- Scalable group communication
- Built-in Socket.IO optimization

### Dynamic Room Updates

When users join new groups or leave existing ones:

```typescript
// Add user to new group room
client.join(newGroupId);

// Remove user from group room
client.leave(oldGroupId);
```

## User Presence System

### Online Status Tracking

```typescript
private onlineUsers = new Map<number, string>();

// Check if user is online
isUserOnline(userId: number): boolean {
  return this.onlineUsers.has(userId);
}

// Get online users in group
getOnlineGroupMembers(groupMembers: number[]): number[] {
  return groupMembers.filter(memberId => 
    this.onlineUsers.has(memberId)
  );
}
```

### Database Synchronization

```typescript
// Update database on connection
await this.chatService.setUserOnline(userId, true);

// Update database on disconnection
await this.chatService.setUserOnline(userId, false);
```

**Dual Tracking**:
- **In-Memory**: Fast presence checks for real-time features
- **Database**: Persistent status for email notifications

## Event Types

### Client to Server Events

| Event | Payload | Description |
|-------|---------|-------------|
| `sendMessage` | `{ groupId, text, attachments }` | Send a new message |
| `joinGroup` | `{ groupId }` | Join a specific group room |
| `leaveGroup` | `{ groupId }` | Leave a specific group room |
| `markAsRead` | `{ messageId, groupId }` | Mark message as read |

### Server to Client Events

| Event | Payload | Description |
|-------|---------|-------------|
| `newMessage` | `{ id, senderId, groupId, text, attachments, timestamp, readBy }` | New message received |
| `userOnline` | `{ userId, isOnline }` | User presence update |
| `messageRead` | `{ messageId, userId }` | Message read receipt |
| `error` | `{ message }` | Error notification |

## Authentication

### JWT Token Verification

```typescript
const token = client.handshake.auth.token;
const payload = await this.jwtService.verifyToken(token);
const userId = payload.sub;
```

**Security Features**:
- Token validation on connection
- User ID extraction from JWT
- Automatic disconnection for invalid tokens
- No persistent token storage

### Client Authentication Example

```javascript
// Frontend connection with JWT
const socket = io('ws://localhost:3000', {
  auth: {
    token: 'your-jwt-token-here'
  }
});
```

## Error Handling

### Connection Errors

```typescript
try {
  const payload = await this.jwtService.verifyToken(token);
  // ... connection logic
} catch (error) {
  console.error('Connection error:', error);
  client.disconnect();
}
```

### Message Errors

```typescript
try {
  const message = await this.chatService.createMessage(messageData);
  // ... broadcast logic
} catch (error) {
  console.error('Error handling message:', error);
  client.emit('error', { message: 'Failed to send message' });
}
```

**Error Strategies**:
- Graceful disconnection for auth failures
- Client error notifications
- Detailed server-side logging
- Fallback mechanisms

## Performance Optimization

### Connection Scaling

```typescript
// Efficient user lookup
private onlineUsers = new Map<number, string>();

// O(1) presence checks
isUserOnline(userId: number): boolean {
  return this.onlineUsers.has(userId);
}
```

### Memory Management

```typescript
// Clean up on disconnect
async handleDisconnect(client: Socket) {
  const userId = client.data.userId;
  if (userId) {
    this.onlineUsers.delete(userId);
  }
}
```

### Broadcasting Efficiency

```typescript
// Room-based broadcasting (efficient)
this.server.to(groupId).emit('newMessage', data);

// Avoid individual socket emissions
// this.server.sockets.forEach(...) // Less efficient
```

## Monitoring & Debugging

### Connection Logging

```typescript
console.log(`User ${userId} connected with socket ${client.id}`);
console.log(`User ${userId} disconnected`);
console.log(`Message sent in group ${groupId} by user ${userId}`);
```

### Error Tracking

```typescript
console.error('Connection error:', error);
console.error('Error handling message:', error);
console.error('Error sending email notifications:', error);
```

### Metrics to Monitor

- **Connection Count**: Active WebSocket connections
- **Message Rate**: Messages per second/minute
- **Error Rate**: Failed connections/messages
- **Response Time**: Message delivery latency
- **Memory Usage**: Online users map size

## Client Integration

### Frontend Connection Example

```javascript
import { io } from 'socket.io-client';

class ChatClient {
  constructor(token) {
    this.socket = io('ws://localhost:3000', {
      auth: { token }
    });
    
    this.setupEventListeners();
  }
  
  setupEventListeners() {
    this.socket.on('newMessage', (message) => {
      this.displayMessage(message);
    });
    
    this.socket.on('error', (error) => {
      console.error('Socket error:', error);
    });
  }
  
  sendMessage(groupId, text, attachments = []) {
    this.socket.emit('sendMessage', {
      groupId,
      text,
      attachments
    });
  }
}
```

### Mobile App Integration

```javascript
// React Native with Socket.IO
import io from 'socket.io-client';

const socket = io('ws://your-server.com', {
  auth: {
    token: await AsyncStorage.getItem('jwt_token')
  },
  transports: ['websocket']
});
```

## Testing

### Unit Testing

```typescript
describe('ChatGateway', () => {
  let gateway: ChatGateway;
  let mockSocket: Socket;
  
  beforeEach(() => {
    // Setup test environment
  });
  
  it('should handle user connection', async () => {
    await gateway.handleConnection(mockSocket);
    expect(gateway.isUserOnline(userId)).toBe(true);
  });
});
```

### Integration Testing

```javascript
// Client-side testing
const client = io('http://localhost:3000', {
  auth: { token: testToken }
});

client.emit('sendMessage', testMessage);
client.on('newMessage', (message) => {
  expect(message.text).toBe(testMessage.text);
});
```

## Security Considerations

### Authentication
- JWT token validation on every connection
- Automatic disconnection for invalid tokens
- No token persistence in memory

### Authorization
- Users can only join groups they're members of
- Message sending restricted to group members
- User ID extracted from verified JWT

### Data Validation
- Message payload validation
- File attachment type checking
- Group membership verification

### Rate Limiting
```typescript
// Implement rate limiting for message sending
private messageRateLimit = new Map<number, number[]>();

private checkRateLimit(userId: number): boolean {
  const now = Date.now();
  const userMessages = this.messageRateLimit.get(userId) || [];
  
  // Remove messages older than 1 minute
  const recentMessages = userMessages.filter(
    timestamp => now - timestamp < 60000
  );
  
  if (recentMessages.length >= 30) { // 30 messages per minute
    return false;
  }
  
  recentMessages.push(now);
  this.messageRateLimit.set(userId, recentMessages);
  return true;
}
```

## Deployment Considerations

### Horizontal Scaling

For multiple server instances, consider:

```typescript
// Redis adapter for Socket.IO clustering
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

const pubClient = createClient({ url: 'redis://localhost:6379' });
const subClient = pubClient.duplicate();

io.adapter(createAdapter(pubClient, subClient));
```

### Load Balancing

```nginx
# Nginx configuration for WebSocket
upstream socketio {
    ip_hash; # Sticky sessions
    server backend1:3000;
    server backend2:3000;
}

server {
    location /socket.io/ {
        proxy_pass http://socketio;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Environment Configuration

```typescript
// Production WebSocket settings
@WebSocketGateway({
  cors: {
    origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
    credentials: true
  },
  transports: ['websocket', 'polling']
})
```
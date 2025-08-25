# Authentication & Authorization

## Overview

The Deed-O Backend implements a robust JWT-based authentication system with role-based access control. The system uses RS256 asymmetric encryption for token signing and verification.

## Authentication Architecture

### Components

1. **JWT Service** (`jwt.service.ts`): Token verification and payload extraction
2. **Auth Guard** (`auth.guard.ts`): Global authentication middleware
3. **Auth Database**: Separate database for user management
4. **Public Decorator**: Bypass authentication for specific endpoints

### Token Algorithm
- **Algorithm**: RS256 (RSA Signature with SHA-256)
- **Key Type**: Asymmetric (Public/Private key pair)
- **Token Type**: Bearer Token

## JWT Token Structure

### Token Payload
```typescript
interface IJwtPayload {
  id: number;        // User ID
  role: string;      // User role (admin, moderator, user)
  iat: number;       // Issued at timestamp
  exp: number;       // Expiration timestamp
  iss: string;       // Issuer
  aud: string;       // Audience
}
```

### Token Verification Process

1. **Extract Token**: From Authorization header
2. **Verify Signature**: Using public key
3. **Validate Claims**: Check expiration and issuer
4. **Extract Payload**: Get user information
5. **Attach to Request**: Make user data available to controllers

## Authentication Guard

### Implementation (`auth.guard.ts`)

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private jwtService: JwtService
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // Check if endpoint is public
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass()
    ]);

    if (isPublic) {
      return true;
    }

    // Extract and verify token
    const request = context.switchToHttp().getRequest<FastifyRequest>();
    const token = this.extractTokenFromHeader(request);
    
    if (!token) {
      throw new UnauthorizedException();
    }

    try {
      const payload = await this.jwtService.verifyToken(token);
      request['user'] = {
        id: payload.id,
        role: payload.role
      };
    } catch (error) {
      throw new UnauthorizedException(error.message);
    }
    
    return true;
  }
}
```

### Global Registration

The auth guard is registered globally in `app.module.ts`:

```typescript
@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: AuthGuard
    }
  ]
})
export class AppModule {}
```

## Public Endpoints

### Public Decorator

To bypass authentication for specific endpoints:

```typescript
import { Public } from '@/guards/auth.guard';

@Controller('public')
export class PublicController {
  @Public()
  @Get('health')
  healthCheck() {
    return { status: 'ok' };
  }
}
```

### Implementation

```typescript
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

## User Roles & Permissions

### Role Hierarchy

1. **Admin**: Full system access
   - Manage all products
   - Manage all users
   - System configuration
   - Access all endpoints

2. **Moderator**: Content management
   - Review and approve products
   - Moderate chat groups
   - Manage user content

3. **User**: Standard user access
   - Create and manage own products
   - Participate in chat groups
   - Upload files
   - View published content

### Role-Based Access Control

Currently, the system implements basic role extraction but doesn't enforce role-based restrictions at the endpoint level. This can be extended with role guards:

```typescript
// Example role guard implementation
@Injectable()
export class RoleGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);
    
    if (!requiredRoles) {
      return true;
    }
    
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    return requiredRoles.includes(user.role);
  }
}
```

## User Management

### User Schema

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR UNIQUE NOT NULL,
  name VARCHAR,
  contact_number BIGINT,
  contact_number_country_code VARCHAR,
  city VARCHAR,
  state VARCHAR,
  country VARCHAR,
  pincode INTEGER,
  gender VARCHAR,
  dob DATE,
  status users_statuses_enum DEFAULT 'not-verified' NOT NULL,
  registration_platform platform_types_enum NOT NULL,
  role users_roles_enum DEFAULT 'user' NOT NULL,
  is_online BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW() NOT NULL,
  updated_at TIMESTAMP DEFAULT NOW() NOT NULL,
  deleted_at TIMESTAMP
);
```

### User Status Management

**User Statuses**:
- `inactive`: User account is disabled
- `verified`: User has completed verification
- `not-verified`: User needs to complete verification

**Platform Types**:
- `dobe`: DOBE platform users
- `cause-i`: Cause-I platform users
- `deed-o`: Deed-O platform users

## Token Management

### Token Storage

```sql
CREATE TABLE tokens (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE NOT NULL,
  expire_at TIMESTAMP WITH TIME ZONE NOT NULL,
  platform_type platform_types_enum NOT NULL
);
```

### Token Lifecycle

1. **Token Generation**: External authentication service
2. **Token Verification**: JWT service validates signature
3. **Token Expiration**: Automatic expiration handling
4. **Token Refresh**: Handled by external service

## Verification System

### User Verification Schema

```sql
CREATE TABLE user_verifications (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE NOT NULL,
  otp_code VARCHAR NOT NULL,
  verification_source VARCHAR NOT NULL,
  action_platform platform_types_enum NOT NULL,
  action user_verification_actions_enum NOT NULL,
  verified BOOLEAN DEFAULT FALSE NOT NULL,
  expire_at TIMESTAMP WITH TIME ZONE NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);
```

### Verification Actions
- `login`: Login verification
- `register`: Registration verification

### Verification Process

1. **OTP Generation**: System generates verification code
2. **Code Delivery**: Sent via email/SMS
3. **Code Verification**: User submits code for validation
4. **Status Update**: User status updated upon successful verification

## Security Configuration

### Environment Variables

```bash
# JWT Configuration
PUBLIC_KEY="-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----"

# Database URLs
DATABASE_URL="postgresql://user:password@host:port/database"
AUTH_DATABASE_URL="postgresql://user:password@host:port/auth_database"
```

### CORS Configuration

```typescript
// main.ts
app.enableCors({
  origin: ALLOWED_ORIGINS,
  methods: 'GET,HEAD,PATCH,POST',
  preflightContinue: false,
  optionsSuccessStatus: 204
});
```

### Request Validation

```typescript
// Global validation pipe
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true  // Strip unknown properties
  })
);
```

## Authentication Flow

### 1. Token-Based Authentication

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant AuthGuard
    participant JWTService
    participant Database

    Client->>API: Request with Bearer Token
    API->>AuthGuard: Intercept Request
    AuthGuard->>JWTService: Verify Token
    JWTService->>JWTService: Validate Signature
    JWTService->>AuthGuard: Return Payload
    AuthGuard->>Database: Optional User Lookup
    AuthGuard->>API: Attach User to Request
    API->>Client: Process Request
```

### 2. Public Endpoint Access

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant AuthGuard

    Client->>API: Request to Public Endpoint
    API->>AuthGuard: Check @Public Decorator
    AuthGuard->>API: Skip Authentication
    API->>Client: Process Request
```

## Error Handling

### Authentication Errors

```typescript
// Common authentication errors
{
  "statusCode": 401,
  "message": "Unauthorized",
  "error": "Unauthorized"
}

// Token verification errors
{
  "statusCode": 401,
  "message": "Invalid token signature",
  "error": "Unauthorized"
}

// Missing token errors
{
  "statusCode": 401,
  "message": "Authorization header missing",
  "error": "Unauthorized"
}
```

## Best Practices

### Security Recommendations

1. **Token Storage**: Store tokens securely on client side
2. **Token Expiration**: Implement reasonable expiration times
3. **Refresh Tokens**: Use refresh tokens for long-lived sessions
4. **HTTPS Only**: Always use HTTPS in production
5. **Key Rotation**: Regularly rotate signing keys
6. **Rate Limiting**: Implement rate limiting for auth endpoints

### Implementation Guidelines

1. **Consistent Error Messages**: Don't leak information about user existence
2. **Audit Logging**: Log authentication events
3. **Session Management**: Track user sessions and online status
4. **Multi-Platform Support**: Handle different platform types
5. **Graceful Degradation**: Handle auth service outages

## Integration with External Services

The current implementation suggests integration with external authentication services:

1. **Token Generation**: Handled externally
2. **User Registration**: Managed by external service
3. **Password Management**: Not handled by this service
4. **Social Login**: Supported through external integration

## Monitoring & Analytics

### Recommended Metrics

1. **Authentication Success Rate**: Track successful logins
2. **Token Verification Latency**: Monitor JWT verification performance
3. **Failed Authentication Attempts**: Security monitoring
4. **User Session Duration**: Analyze user engagement
5. **Platform Distribution**: Track usage across platforms

### Logging

```typescript
// Example authentication logging
console.log({
  event: 'authentication_success',
  userId: payload.id,
  role: payload.role,
  platform: 'deed-o',
  timestamp: new Date().toISOString()
});
```
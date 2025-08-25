# System Overview

## Project Information

- **Project Name**: Cause-I Server (Deed-O Backend)
- **Version**: 0.0.1
- **Author**: Souvik Mandal
- **Framework**: NestJS with Fastify
- **Language**: TypeScript

## Architecture Overview

The Deed-O Backend is a microservices-oriented Node.js application built with NestJS framework, designed to support a sustainable products marketplace with integrated chat functionality.

### Core Components

1. **API Layer**: RESTful endpoints with Swagger documentation
2. **WebSocket Layer**: Real-time chat functionality
3. **Database Layer**: PostgreSQL with Drizzle ORM
4. **Authentication Layer**: JWT-based security
5. **File Storage**: AWS S3 integration
6. **Email Service**: AWS SES integration

### Application Structure

```
src/
├── app.module.ts           # Main application module
├── main.ts                  # Application bootstrap
├── controllers/             # API endpoint controllers
│   ├── products.controller.ts
│   ├── groups.controller.ts
│   ├── chat.controller.ts
│   └── s3.controller.ts
├── services/                # Business logic services
│   ├── products.service.ts
│   ├── groups.service.ts
│   ├── chat.service.ts
│   ├── jwt.service.ts
│   ├── s3.service.ts
│   └── email.service.ts
├── drizzle/                 # Database layer
│   ├── db.ts               # Database connections
│   ├── schema.ts           # Main database schema
│   ├── auth-schema.ts      # Authentication database schema
│   └── chat-schema.ts      # Chat database schema
├── dtos/                    # Data Transfer Objects
├── guards/                  # Authentication guards
├── gateways/               # WebSocket gateways
├── core/                   # Core utilities and constants
└── templates/              # Email templates
```

## Key Features

### 1. Product Management
- Create, read, update products
- Support for multiple causes (Sustainable fashion, Grief care, etc.)
- Vendor-specific product management
- Publishing status workflow (draft, published, suggestion, rejected)
- Media attachment support

### 2. Group Chat System
- Real-time messaging with Socket.IO
- Group creation based on products
- Message attachments (images, videos, audio, files)
- Read receipts tracking
- Online/offline user status

### 3. File Management
- AWS S3 integration for file uploads
- Support for single and multiple file uploads
- File type validation and size limits
- Signed URL generation for secure access

### 4. Authentication & Security
- JWT token-based authentication
- RS256 algorithm for token signing
- Role-based access control (admin, moderator, user)
- Global authentication guard
- Public endpoint decoration support

### 5. Email Notifications
- AWS SES integration
- React-based email templates
- Automated notifications for offline users
- Configurable email subjects and content

## Database Architecture

The application uses a dual-database approach:

1. **Main Database**: Products, groups, messages
2. **Auth Database**: Users, tokens, verifications

### Supported Causes
- Grief care
- Sustainable fashion
- Sustainable tourism
- Intangible culture and heritage
- Sustainable food and agriculture

## Technology Stack

### Core Framework
- **NestJS**: Progressive Node.js framework
- **Fastify**: High-performance web framework
- **TypeScript**: Type-safe JavaScript

### Database & ORM
- **PostgreSQL**: Primary database
- **Drizzle ORM**: Type-safe database toolkit
- **Drizzle Kit**: Database migration tool

### Authentication
- **JOSE**: JWT operations
- **RS256**: Asymmetric signing algorithm

### Cloud Services
- **AWS S3**: File storage
- **AWS SES**: Email service

### Real-time Communication
- **Socket.IO**: WebSocket implementation
- **Real-time messaging**: Instant chat functionality

### Development Tools
- **Swagger**: API documentation
- **ESLint**: Code linting
- **Prettier**: Code formatting
- **Jest**: Testing framework

## Environment Support

The application supports multiple environments:
- **Local**: Development environment
- **UAT**: User Acceptance Testing
- **Production**: Live environment

## API Documentation

The application includes comprehensive Swagger documentation available at `/v1` endpoint when running the server.

## Performance Features

- **Fastify**: High-performance HTTP server
- **Connection pooling**: Efficient database connections
- **File streaming**: Efficient file upload handling
- **Pagination**: Optimized data retrieval
- **Caching**: Strategic caching implementation

## Security Features

- **CORS**: Configurable cross-origin resource sharing
- **Input validation**: Class-validator integration
- **File type validation**: Secure file upload handling
- **Token expiration**: Automatic token lifecycle management
- **Environment-based configuration**: Secure credential management
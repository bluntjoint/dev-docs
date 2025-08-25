# Deed-O Backend Documentation

This is a comprehensive documentation for the Deed-O Backend codebase, a Node.js-based API server built with NestJS framework.

## Table of Contents

1. [System Overview](./system-overview.md)
2. [Database Architecture](./database-architecture.md)
3. [API Endpoints](./api-endpoints.md)
4. [Authentication & Authorization](./authentication.md)
5. [Services & Business Logic](./services.md)
6. [Real-time Communication](./websockets.md)
7. [File Management](./file-management.md)
8. [Email Notifications](./email-notifications.md)
9. [Configuration & Environment](./configuration.md)
10. [Deployment Guide](./deployment.md)

## Quick Start

For a quick overview of the system, start with the [System Overview](./system-overview.md) document.

## Architecture Summary

The Deed-O Backend is a modern Node.js application that provides:

- **Product Management**: CRUD operations for sustainable products
- **Group Chat System**: Real-time messaging with WebSocket support
- **File Upload**: AWS S3 integration for media management
- **Email Notifications**: Automated email system using AWS SES
- **JWT Authentication**: Secure token-based authentication
- **Multi-database Support**: Separate databases for main app and authentication

## Technology Stack

- **Framework**: NestJS with Fastify
- **Database**: PostgreSQL with Drizzle ORM
- **Authentication**: JWT with RS256 algorithm
- **File Storage**: AWS S3
- **Email Service**: AWS SES
- **Real-time**: Socket.IO WebSockets
- **API Documentation**: Swagger/OpenAPI

## Getting Started

Refer to the main [README.md](../README.md) for installation and setup instructions.

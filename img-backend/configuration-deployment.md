# Configuration & Deployment

## Overview

This document covers the configuration management, environment setup, and deployment strategies for the Deed-O Backend application.

## Environment Configuration

### Environment Variables

#### Required Variables

**File**: `.env`

```bash
# Application Configuration
TZ=Asia/Kolkata
NODE_ENV=development
PORT=3000

# Database Configuration
DATABASE_URL=postgresql://username:password@localhost:5432/cause-i
AUTH_DATABASE_URL=postgresql://username:password@localhost:5432/cause-i

# OpenAI Configuration
OPENAI_API_KEY=sk-your-openai-api-key

# AWS Configuration
AWS_ACCESS_KEY_ID=your-aws-access-key
AWS_SECRET_ACCESS_KEY=your-aws-secret-key
AWS_REGION=us-east-1
AWS_S3_BUCKET_NAME=your-s3-bucket-name

# Email Configuration
SENDER_EMAIL=noreply@yourdomain.com

# JWT Configuration
PUBLIC_KEY="-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
-----END PUBLIC KEY-----"
```

#### Environment Variable Details

| Variable | Description | Required | Default | Example |
|----------|-------------|----------|---------|----------|
| `TZ` | Application timezone | Yes | - | `Asia/Kolkata` |
| `NODE_ENV` | Node.js environment | Yes | `development` | `production` |
| `PORT` | Server port | No | `3000` | `8080` |
| `DATABASE_URL` | Main database connection | Yes | - | `postgresql://user:pass@host:5432/db` |
| `AUTH_DATABASE_URL` | Auth database connection | Yes | - | `postgresql://user:pass@host:5432/auth` |
| `OPENAI_API_KEY` | OpenAI API key | Yes | - | `sk-...` |
| `AWS_ACCESS_KEY_ID` | AWS access key | Yes | - | `AKIA...` |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key | Yes | - | `...` |
| `AWS_REGION` | AWS region | Yes | - | `us-east-1` |
| `AWS_S3_BUCKET_NAME` | S3 bucket name | Yes | - | `deed-o-files` |
| `SENDER_EMAIL` | Email sender address | Yes | - | `noreply@example.com` |
| `PUBLIC_KEY` | JWT verification key | Yes | - | `-----BEGIN PUBLIC KEY-----...` |

### Configuration Service

**File**: `src/config/configuration.ts`

```typescript
import { ConfigService } from '@nestjs/config';

export interface AppConfig {
  port: number;
  nodeEnv: string;
  timezone: string;
  database: {
    url: string;
    authUrl: string;
  };
  aws: {
    accessKeyId: string;
    secretAccessKey: string;
    region: string;
    s3BucketName: string;
  };
  email: {
    senderEmail: string;
  };
  jwt: {
    publicKey: string;
  };
  openai: {
    apiKey: string;
  };
}

export const getConfig = (configService: ConfigService): AppConfig => ({
  port: parseInt(configService.get<string>('PORT', '3000')),
  nodeEnv: configService.get<string>('NODE_ENV', 'development'),
  timezone: configService.get<string>('TZ', 'UTC'),
  database: {
    url: configService.get<string>('DATABASE_URL'),
    authUrl: configService.get<string>('AUTH_DATABASE_URL')
  },
  aws: {
    accessKeyId: configService.get<string>('AWS_ACCESS_KEY_ID'),
    secretAccessKey: configService.get<string>('AWS_SECRET_ACCESS_KEY'),
    region: configService.get<string>('AWS_REGION'),
    s3BucketName: configService.get<string>('AWS_S3_BUCKET_NAME')
  },
  email: {
    senderEmail: configService.get<string>('SENDER_EMAIL')
  },
  jwt: {
    publicKey: configService.get<string>('PUBLIC_KEY')
  },
  openai: {
    apiKey: configService.get<string>('OPENAI_API_KEY')
  }
});
```

### Environment Validation

```typescript
import Joi from 'joi';

export const configValidationSchema = Joi.object({
  TZ: Joi.string().required(),
  NODE_ENV: Joi.string().valid('development', 'production', 'test').required(),
  PORT: Joi.number().port().default(3000),
  DATABASE_URL: Joi.string().uri().required(),
  AUTH_DATABASE_URL: Joi.string().uri().required(),
  OPENAI_API_KEY: Joi.string().pattern(/^sk-/).required(),
  AWS_ACCESS_KEY_ID: Joi.string().required(),
  AWS_SECRET_ACCESS_KEY: Joi.string().required(),
  AWS_REGION: Joi.string().required(),
  AWS_S3_BUCKET_NAME: Joi.string().required(),
  SENDER_EMAIL: Joi.string().email().required(),
  PUBLIC_KEY: Joi.string().required()
});
```

## Database Setup

### Prerequisites

1. **Docker & Docker Compose**
   ```bash
   # Install Docker Desktop
   # Or install Docker Engine + Docker Compose
   ```

2. **Node.js & pnpm**
   ```bash
   # Install Node.js v20.11.1
   npm install -g pnpm
   ```

### Database Initialization

#### 1. Start PostgreSQL with Docker

**File**: `docker-compose.yml`

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15
    container_name: deed-o-postgres
    environment:
      POSTGRES_DB: cause-i
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - deed-o-network

  redis:
    image: redis:7-alpine
    container_name: deed-o-redis
    ports:
      - "6379:6379"
    networks:
      - deed-o-network

volumes:
  postgres_data:

networks:
  deed-o-network:
    driver: bridge
```

#### 2. Database Initialization Script

**File**: `init.sql`

```sql
-- Create databases
CREATE DATABASE "cause-i" IF NOT EXISTS;

-- Create extensions
\c "cause-i";
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Create indexes for performance
CREATE INDEX IF NOT EXISTS idx_products_cause ON products(cause);
CREATE INDEX IF NOT EXISTS idx_products_vendor ON products("vendorID");
CREATE INDEX IF NOT EXISTS idx_products_status ON products("publishingStatus");
CREATE INDEX IF NOT EXISTS idx_groups_product ON groups("productId");
CREATE INDEX IF NOT EXISTS idx_messages_group ON messages("groupId");
CREATE INDEX IF NOT EXISTS idx_messages_sender ON messages("senderId");
CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
```

#### 3. Start Database

```bash
# Start PostgreSQL and Redis
docker compose up -d

# Verify containers are running
docker compose ps
```

#### 4. Run Migrations

```bash
# Install dependencies
pnpm install

# Run all migrations
pnpm run migration:all

# Seed initial data
pnpm run seed
```

### Migration Commands

```bash
# Generate new migration
pnpm run migration:generate

# Run pending migrations
pnpm run migration:run

# Revert last migration
pnpm run migration:revert

# Show migration status
pnpm run migration:show
```

## Application Setup

### Development Environment

#### 1. Clone and Install

```bash
# Clone repository
git clone <repository-url>
cd Deed-O-Backend-master

# Install dependencies
pnpm install
```

#### 2. Environment Configuration

```bash
# Copy environment template
cp .env.example .env

# Edit environment variables
vim .env  # or your preferred editor
```

#### 3. Start Development Server

```bash
# Start in development mode with hot reload
pnpm run dev

# Or start in watch mode
pnpm run start:dev
```

#### 4. Verify Setup

```bash
# Check API health
curl http://localhost:3000/health

# Check Swagger documentation
open http://localhost:3000/api
```

### Production Build

```bash
# Build application
pnpm run build

# Start production server
pnpm run start:prod
```

## Deployment Strategies

### 1. Docker Deployment

#### Dockerfile

```dockerfile
# Multi-stage build
FROM node:20-alpine AS builder

# Install pnpm
RUN npm install -g pnpm

# Set working directory
WORKDIR /app

# Copy package files
COPY package.json pnpm-lock.yaml ./

# Install dependencies
RUN pnpm install --frozen-lockfile

# Copy source code
COPY . .

# Build application
RUN pnpm run build

# Production stage
FROM node:20-alpine AS production

# Install pnpm
RUN npm install -g pnpm

# Create app user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nestjs -u 1001

# Set working directory
WORKDIR /app

# Copy package files
COPY package.json pnpm-lock.yaml ./

# Install production dependencies only
RUN pnpm install --prod --frozen-lockfile

# Copy built application
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# Change ownership
RUN chown -R nestjs:nodejs /app
USER nestjs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start application
CMD ["node", "dist/main"]
```

#### Docker Compose for Production

```yaml
version: '3.8'
services:
  app:
    build: .
    container_name: deed-o-app
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    env_file:
      - .env.production
    depends_on:
      - postgres
      - redis
    networks:
      - deed-o-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  postgres:
    image: postgres:15
    container_name: deed-o-postgres
    environment:
      POSTGRES_DB: cause-i
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - deed-o-network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: deed-o-redis
    networks:
      - deed-o-network
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    container_name: deed-o-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app
    networks:
      - deed-o-network
    restart: unless-stopped

volumes:
  postgres_data:

networks:
  deed-o-network:
    driver: bridge
```

#### Build and Deploy

```bash
# Build Docker image
docker build -t deed-o-backend .

# Run with Docker Compose
docker compose -f docker-compose.prod.yml up -d

# Check logs
docker compose logs -f app
```

### 2. AWS ECS Deployment

#### Task Definition

```json
{
  "family": "deed-o-backend",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::account:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::account:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "deed-o-backend",
      "image": "your-account.dkr.ecr.region.amazonaws.com/deed-o-backend:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:deed-o/database-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/deed-o-backend",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -f http://localhost:3000/health || exit 1"
        ],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

#### ECS Service Configuration

```json
{
  "serviceName": "deed-o-backend-service",
  "cluster": "deed-o-cluster",
  "taskDefinition": "deed-o-backend:1",
  "desiredCount": 2,
  "launchType": "FARGATE",
  "networkConfiguration": {
    "awsvpcConfiguration": {
      "subnets": [
        "subnet-12345",
        "subnet-67890"
      ],
      "securityGroups": [
        "sg-backend"
      ],
      "assignPublicIp": "DISABLED"
    }
  },
  "loadBalancers": [
    {
      "targetGroupArn": "arn:aws:elasticloadbalancing:region:account:targetgroup/deed-o-backend",
      "containerName": "deed-o-backend",
      "containerPort": 3000
    }
  ]
}
```

### 3. Kubernetes Deployment

#### Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deed-o-backend
  labels:
    app: deed-o-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: deed-o-backend
  template:
    metadata:
      labels:
        app: deed-o-backend
    spec:
      containers:
      - name: deed-o-backend
        image: deed-o-backend:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: deed-o-secrets
              key: database-url
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: deed-o-backend-service
spec:
  selector:
    app: deed-o-backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP
```

## Load Balancing & Reverse Proxy

### Nginx Configuration

**File**: `nginx.conf`

```nginx
events {
    worker_connections 1024;
}

http {
    upstream deed_o_backend {
        least_conn;
        server app1:3000 max_fails=3 fail_timeout=30s;
        server app2:3000 max_fails=3 fail_timeout=30s;
        server app3:3000 max_fails=3 fail_timeout=30s;
    }

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=upload:10m rate=2r/s;

    server {
        listen 80;
        server_name api.yourdomain.com;

        # Redirect HTTP to HTTPS
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name api.yourdomain.com;

        # SSL Configuration
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
        ssl_prefer_server_ciphers off;

        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";

        # API routes
        location /api {
            limit_req zone=api burst=20 nodelay;
            
            proxy_pass http://deed_o_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
            
            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # File upload routes
        location /s3/upload {
            limit_req zone=upload burst=5 nodelay;
            
            client_max_body_size 50M;
            proxy_pass http://deed_o_backend;
            proxy_request_buffering off;
            
            # Extended timeouts for file uploads
            proxy_connect_timeout 300s;
            proxy_send_timeout 300s;
            proxy_read_timeout 300s;
        }

        # WebSocket routes
        location /socket.io/ {
            proxy_pass http://deed_o_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Health check
        location /health {
            proxy_pass http://deed_o_backend;
            access_log off;
        }
    }
}
```

## Monitoring & Logging

### Application Logging

**File**: `src/config/logger.config.ts`

```typescript
import { LoggerService } from '@nestjs/common';
import pino from 'pino';

export class CustomLogger implements LoggerService {
  private logger = pino({
    level: process.env.LOG_LEVEL || 'info',
    transport: {
      target: 'pino-pretty',
      options: {
        colorize: true,
        translateTime: 'SYS:standard',
        ignore: 'pid,hostname'
      }
    }
  });

  log(message: string, context?: string) {
    this.logger.info({ context }, message);
  }

  error(message: string, trace?: string, context?: string) {
    this.logger.error({ context, trace }, message);
  }

  warn(message: string, context?: string) {
    this.logger.warn({ context }, message);
  }

  debug(message: string, context?: string) {
    this.logger.debug({ context }, message);
  }

  verbose(message: string, context?: string) {
    this.logger.trace({ context }, message);
  }
}
```

### Health Check Endpoint

**File**: `src/health/health.controller.ts`

```typescript
import { Controller, Get } from '@nestjs/common';
import { HealthCheck, HealthCheckService, TypeOrmHealthIndicator } from '@nestjs/terminus';
import { Public } from '../auth/public.decorator';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator
  ) {}

  @Get()
  @Public()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
      () => this.checkRedis(),
      () => this.checkS3(),
      () => this.checkSES()
    ]);
  }

  private async checkRedis() {
    // Redis health check implementation
    return { redis: { status: 'up' } };
  }

  private async checkS3() {
    // S3 health check implementation
    return { s3: { status: 'up' } };
  }

  private async checkSES() {
    // SES health check implementation
    return { ses: { status: 'up' } };
  }
}
```

### Prometheus Metrics

```typescript
import { Injectable } from '@nestjs/common';
import { register, Counter, Histogram, Gauge } from 'prom-client';

@Injectable()
export class MetricsService {
  private httpRequestsTotal = new Counter({
    name: 'http_requests_total',
    help: 'Total number of HTTP requests',
    labelNames: ['method', 'route', 'status_code']
  });

  private httpRequestDuration = new Histogram({
    name: 'http_request_duration_seconds',
    help: 'Duration of HTTP requests in seconds',
    labelNames: ['method', 'route']
  });

  private activeConnections = new Gauge({
    name: 'websocket_connections_active',
    help: 'Number of active WebSocket connections'
  });

  constructor() {
    register.registerMetric(this.httpRequestsTotal);
    register.registerMetric(this.httpRequestDuration);
    register.registerMetric(this.activeConnections);
  }

  incrementHttpRequests(method: string, route: string, statusCode: number) {
    this.httpRequestsTotal.inc({ method, route, status_code: statusCode });
  }

  observeHttpDuration(method: string, route: string, duration: number) {
    this.httpRequestDuration.observe({ method, route }, duration);
  }

  setActiveConnections(count: number) {
    this.activeConnections.set(count);
  }

  getMetrics() {
    return register.metrics();
  }
}
```

## Security Configuration

### CORS Configuration

```typescript
// main.ts
app.enableCors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true
});
```

### Rate Limiting

```typescript
import { ThrottlerModule } from '@nestjs/throttler';

@Module({
  imports: [
    ThrottlerModule.forRoot({
      ttl: 60, // 1 minute
      limit: 100 // 100 requests per minute
    })
  ]
})
export class AppModule {}
```

### Helmet Security

```typescript
import helmet from 'helmet';

// main.ts
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "wss:", "https:"]
    }
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
}));
```

## Performance Optimization

### Database Connection Pooling

```typescript
// drizzle.config.ts
export default {
  schema: './src/drizzle/schema.ts',
  out: './drizzle',
  driver: 'pg',
  dbCredentials: {
    connectionString: process.env.DATABASE_URL,
    ssl: process.env.NODE_ENV === 'production'
  },
  pool: {
    min: 2,
    max: 10,
    acquireTimeoutMillis: 30000,
    createTimeoutMillis: 30000,
    destroyTimeoutMillis: 5000,
    idleTimeoutMillis: 30000,
    reapIntervalMillis: 1000,
    createRetryIntervalMillis: 200
  }
};
```

### Caching Strategy

```typescript
import { CacheModule } from '@nestjs/cache-manager';
import { redisStore } from 'cache-manager-redis-store';

@Module({
  imports: [
    CacheModule.register({
      store: redisStore,
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379,
      ttl: 300 // 5 minutes default TTL
    })
  ]
})
export class AppModule {}
```

## Backup & Recovery

### Database Backup Script

```bash
#!/bin/bash
# backup-db.sh

DATE=$(date +"%Y%m%d_%H%M%S")
BACKUP_DIR="/backups"
DB_NAME="cause-i"

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup database
pg_dump $DATABASE_URL > $BACKUP_DIR/backup_${DB_NAME}_${DATE}.sql

# Compress backup
gzip $BACKUP_DIR/backup_${DB_NAME}_${DATE}.sql

# Upload to S3
aws s3 cp $BACKUP_DIR/backup_${DB_NAME}_${DATE}.sql.gz s3://your-backup-bucket/database/

# Clean up old backups (keep last 7 days)
find $BACKUP_DIR -name "backup_${DB_NAME}_*.sql.gz" -mtime +7 -delete

echo "Backup completed: backup_${DB_NAME}_${DATE}.sql.gz"
```

### Automated Backup with Cron

```bash
# Add to crontab
# Run backup daily at 2 AM
0 2 * * * /path/to/backup-db.sh
```

## Troubleshooting

### Common Issues

#### 1. Database Connection Issues

```bash
# Check database connectivity
psql $DATABASE_URL -c "SELECT 1;"

# Check connection pool
psql $DATABASE_URL -c "SELECT count(*) FROM pg_stat_activity;"
```

#### 2. Memory Issues

```bash
# Monitor memory usage
docker stats deed-o-app

# Check Node.js heap usage
curl http://localhost:3000/metrics | grep nodejs_heap
```

#### 3. File Upload Issues

```bash
# Check S3 permissions
aws s3 ls s3://your-bucket-name/

# Test file upload
curl -X POST -F "file=@test.jpg" -H "Authorization: Bearer $TOKEN" \
  http://localhost:3000/s3/upload/single
```

### Debug Mode

```bash
# Enable debug logging
export LOG_LEVEL=debug
export DEBUG=*

# Start with debug
pnpm run start:dev
```

### Performance Profiling

```bash
# Enable Node.js profiling
node --prof dist/main.js

# Generate profile report
node --prof-process isolate-*.log > profile.txt
```
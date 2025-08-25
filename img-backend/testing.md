# Testing Documentation

## Overview

This document outlines the comprehensive testing strategy for the Deed-O Backend, including unit tests, integration tests, end-to-end tests, and performance testing.

## Testing Architecture

### Testing Stack

- **Test Framework**: Jest
- **Testing Utilities**: @nestjs/testing
- **HTTP Testing**: Supertest
- **Mocking**: Jest mocks
- **Database Testing**: In-memory SQLite / Test containers
- **Load Testing**: Artillery / K6
- **Coverage**: Jest coverage reports

### Test Structure

```
src/
├── **/*.spec.ts          # Unit tests
├── **/*.integration.spec.ts  # Integration tests
test/
├── e2e/
│   ├── auth.e2e-spec.ts
│   ├── products.e2e-spec.ts
│   ├── groups.e2e-spec.ts
│   ├── chat.e2e-spec.ts
│   └── s3.e2e-spec.ts
├── fixtures/
│   ├── test-data.json
│   ├── test-image.jpg
│   └── test-document.pdf
├── helpers/
│   ├── test-setup.ts
│   ├── database-helper.ts
│   └── auth-helper.ts
└── performance/
    ├── load-test.yml
    └── stress-test.js
```

## Unit Testing

### Service Unit Tests

#### Products Service Test

**File**: `src/products/products.service.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { ProductService } from './products.service';
import { ConfigService } from '@nestjs/config';

describe('ProductService', () => {
  let service: ProductService;
  let mockDb: any;

  beforeEach(async () => {
    // Mock database
    mockDb = {
      select: jest.fn().mockReturnThis(),
      from: jest.fn().mockReturnThis(),
      where: jest.fn().mockReturnThis(),
      execute: jest.fn(),
      insert: jest.fn().mockReturnThis(),
      values: jest.fn().mockReturnThis(),
      returning: jest.fn().mockReturnThis(),
      update: jest.fn().mockReturnThis(),
      set: jest.fn().mockReturnThis(),
      delete: jest.fn().mockReturnThis()
    };

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ProductService,
        {
          provide: ConfigService,
          useValue: {
            get: jest.fn((key: string) => {
              const config = {
                'DATABASE_URL': 'test-db-url',
                'AWS_REGION': 'us-east-1'
              };
              return config[key];
            })
          }
        }
      ]
    }).compile();

    service = module.get<ProductService>(ProductService);
    // Replace db with mock
    (service as any).db = mockDb;
  });

  describe('generateUniqueProductId', () => {
    it('should generate a unique product ID', async () => {
      // Mock no existing product
      mockDb.execute.mockResolvedValue([]);

      const result = await service.generateUniqueProductId();

      expect(result).toBeDefined();
      expect(typeof result).toBe('string');
      expect(result.length).toBeGreaterThan(0);
    });

    it('should retry if ID already exists', async () => {
      // First call returns existing product, second call returns empty
      mockDb.execute
        .mockResolvedValueOnce([{ id: 'existing-id' }])
        .mockResolvedValueOnce([]);

      const result = await service.generateUniqueProductId();

      expect(result).toBeDefined();
      expect(mockDb.execute).toHaveBeenCalledTimes(2);
    });
  });

  describe('createProduct', () => {
    it('should create a product successfully', async () => {
      const productData = {
        title: 'Test Product',
        cause: 'education',
        mrp: 100,
        sellingPrice: 80,
        media: ['image1.jpg'],
        description: 'Test description',
        productUrl: 'https://example.com',
        vendorID: 1
      };

      mockDb.execute.mockResolvedValue([]); // For ID generation
      mockDb.execute.mockResolvedValue([{ id: 'IM_test123' }]); // For insert

      const result = await service.createProduct(productData);

      expect(result.message).toBe('Product created successfully');
      expect(result.id).toContain('IM_');
      expect(mockDb.insert).toHaveBeenCalled();
      expect(mockDb.values).toHaveBeenCalledWith(
        expect.objectContaining({
          ...productData,
          id: expect.stringContaining('IM_')
        })
      );
    });
  });

  describe('getProducts', () => {
    it('should return paginated products', async () => {
      const mockProducts = [
        { id: 'IM_1', title: 'Product 1', cause: 'education' },
        { id: 'IM_2', title: 'Product 2', cause: 'health' }
      ];

      mockDb.execute.mockResolvedValue(mockProducts);

      const result = await service.getProducts(1, 10);

      expect(result.data).toEqual(mockProducts);
      expect(result.pagination).toBeDefined();
      expect(mockDb.limit).toHaveBeenCalledWith(10);
      expect(mockDb.offset).toHaveBeenCalledWith(0);
    });

    it('should filter by causes', async () => {
      const causes = ['education', 'health'];
      mockDb.execute.mockResolvedValue([]);

      await service.getProducts(1, 10, causes);

      expect(mockDb.where).toHaveBeenCalled();
      // Verify that inArray was called with causes
    });

    it('should search by title', async () => {
      const searchTerm = 'test';
      mockDb.execute.mockResolvedValue([]);

      await service.getProducts(1, 10, undefined, searchTerm);

      expect(mockDb.where).toHaveBeenCalled();
      // Verify that ilike was called with search term
    });
  });

  describe('getProductById', () => {
    it('should return product by ID', async () => {
      const mockProduct = { id: 'IM_test', title: 'Test Product' };
      mockDb.execute.mockResolvedValue([mockProduct]);

      const result = await service.getProductById('IM_test');

      expect(result).toEqual(mockProduct);
      expect(mockDb.where).toHaveBeenCalled();
    });

    it('should return null for non-existent product', async () => {
      mockDb.execute.mockResolvedValue([]);

      const result = await service.getProductById('non-existent');

      expect(result).toBeNull();
    });
  });
});
```

#### S3 Service Test

**File**: `src/s3/s3.service.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { S3Service } from './s3.service';
import { ConfigService } from '@nestjs/config';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { BadRequestException } from '@nestjs/common';

// Mock AWS SDK
jest.mock('@aws-sdk/client-s3');
const mockS3Client = {
  send: jest.fn()
};

describe('S3Service', () => {
  let service: S3Service;
  let configService: ConfigService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        S3Service,
        {
          provide: ConfigService,
          useValue: {
            get: jest.fn((key: string) => {
              const config = {
                'AWS_REGION': 'us-east-1',
                'AWS_ACCESS_KEY_ID': 'test-key',
                'AWS_SECRET_ACCESS_KEY': 'test-secret',
                'AWS_S3_BUCKET_NAME': 'test-bucket'
              };
              return config[key];
            })
          }
        }
      ]
    }).compile();

    service = module.get<S3Service>(S3Service);
    configService = module.get<ConfigService>(ConfigService);
    
    // Replace S3 client with mock
    (service as any).s3 = mockS3Client;
    (S3Client as jest.Mock).mockImplementation(() => mockS3Client);
  });

  describe('validateFile', () => {
    it('should validate image file successfully', () => {
      const mockFile = {
        filename: 'test.jpg',
        mimetype: 'image/jpeg',
        file: { truncated: false }
      } as any;

      expect(() => (service as any).validateFile(mockFile)).not.toThrow();
    });

    it('should throw error for oversized file', () => {
      const mockFile = {
        filename: 'large.jpg',
        mimetype: 'image/jpeg',
        file: { truncated: true }
      } as any;

      expect(() => (service as any).validateFile(mockFile))
        .toThrow(BadRequestException);
    });

    it('should throw error for unsupported file type', () => {
      const mockFile = {
        filename: 'virus.exe',
        mimetype: 'application/x-executable',
        file: { truncated: false }
      } as any;

      expect(() => (service as any).validateFile(mockFile))
        .toThrow(BadRequestException);
    });
  });

  describe('uploadFile', () => {
    it('should upload file successfully', async () => {
      const mockFile = {
        filename: 'test.jpg',
        mimetype: 'image/jpeg',
        file: { truncated: false },
        toBuffer: jest.fn().mockResolvedValue(Buffer.from('test'))
      } as any;

      mockS3Client.send.mockResolvedValue({});

      const result = await service.uploadFile(mockFile, 'test');

      expect(result).toContain('https://');
      expect(result).toContain('test-bucket');
      expect(result).toContain('test/');
      expect(mockS3Client.send).toHaveBeenCalledWith(
        expect.any(PutObjectCommand)
      );
    });

    it('should handle S3 upload error', async () => {
      const mockFile = {
        filename: 'test.jpg',
        mimetype: 'image/jpeg',
        file: { truncated: false },
        toBuffer: jest.fn().mockResolvedValue(Buffer.from('test'))
      } as any;

      mockS3Client.send.mockRejectedValue(new Error('S3 Error'));

      await expect(service.uploadFile(mockFile, 'test'))
        .rejects.toThrow('Failed to upload file to S3');
    });
  });
});
```

### Controller Unit Tests

#### Products Controller Test

**File**: `src/products/products.controller.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { ProductsController } from './products.controller';
import { ProductService } from './products.service';
import { AuthGuard } from '../auth/auth.guard';
import { CreateProductDto } from './dto/products.dto';

describe('ProductsController', () => {
  let controller: ProductsController;
  let service: ProductService;

  const mockProductService = {
    createProduct: jest.fn(),
    getProducts: jest.fn(),
    getProductById: jest.fn(),
    updateProduct: jest.fn(),
    deleteProduct: jest.fn(),
    getVendorProducts: jest.fn()
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [ProductsController],
      providers: [
        {
          provide: ProductService,
          useValue: mockProductService
        }
      ]
    })
    .overrideGuard(AuthGuard)
    .useValue({ canActivate: jest.fn(() => true) })
    .compile();

    controller = module.get<ProductsController>(ProductsController);
    service = module.get<ProductService>(ProductService);
  });

  describe('createProduct', () => {
    it('should create a product', async () => {
      const createProductDto: CreateProductDto = {
        title: 'Test Product',
        cause: 'education',
        mrp: 100,
        sellingPrice: 80,
        media: ['image1.jpg'],
        description: 'Test description',
        productUrl: 'https://example.com',
        vendorID: 1
      };

      const expectedResult = {
        message: 'Product created successfully',
        id: 'IM_test123'
      };

      mockProductService.createProduct.mockResolvedValue(expectedResult);

      const result = await controller.createProduct(createProductDto);

      expect(result).toEqual(expectedResult);
      expect(service.createProduct).toHaveBeenCalledWith(createProductDto);
    });
  });

  describe('getProducts', () => {
    it('should return paginated products', async () => {
      const expectedResult = {
        data: [{ id: 'IM_1', title: 'Product 1' }],
        pagination: { page: 1, limit: 10, total: 1 }
      };

      mockProductService.getProducts.mockResolvedValue(expectedResult);

      const result = await controller.getProducts(1, 10);

      expect(result).toEqual(expectedResult);
      expect(service.getProducts).toHaveBeenCalledWith(1, 10, undefined, undefined, undefined);
    });

    it('should handle query parameters', async () => {
      const causes = ['education', 'health'];
      const search = 'test';
      const status = 'published';

      await controller.getProducts(1, 10, causes, search, status);

      expect(service.getProducts).toHaveBeenCalledWith(1, 10, causes, search, status);
    });
  });
});
```

## Integration Testing

### Database Integration Tests

**File**: `src/products/products.integration.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { ProductService } from './products.service';
import { ConfigModule } from '@nestjs/config';
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import { migrate } from 'drizzle-orm/node-postgres/migrator';

describe('ProductService Integration', () => {
  let service: ProductService;
  let db: any;
  let pool: Pool;

  beforeAll(async () => {
    // Setup test database
    pool = new Pool({
      connectionString: process.env.TEST_DATABASE_URL || 'postgresql://test:test@localhost:5432/test_db'
    });
    
    db = drizzle(pool);
    
    // Run migrations
    await migrate(db, { migrationsFolder: './drizzle' });

    const module: TestingModule = await Test.createTestingModule({
      imports: [ConfigModule.forRoot()],
      providers: [ProductService]
    }).compile();

    service = module.get<ProductService>(ProductService);
    // Replace db with test db
    (service as any).db = db;
  });

  afterAll(async () => {
    await pool.end();
  });

  beforeEach(async () => {
    // Clean up database before each test
    await db.delete(products);
  });

  describe('Product CRUD Operations', () => {
    it('should create and retrieve a product', async () => {
      const productData = {
        title: 'Integration Test Product',
        cause: 'education',
        mrp: 100,
        sellingPrice: 80,
        media: ['image1.jpg'],
        description: 'Test description',
        productUrl: 'https://example.com',
        vendorID: 1
      };

      // Create product
      const createResult = await service.createProduct(productData);
      expect(createResult.id).toBeDefined();

      // Retrieve product
      const retrievedProduct = await service.getProductById(createResult.id);
      expect(retrievedProduct).toBeDefined();
      expect(retrievedProduct.title).toBe(productData.title);
      expect(retrievedProduct.cause).toBe(productData.cause);
    });

    it('should filter products by cause', async () => {
      // Create test products
      await service.createProduct({
        title: 'Education Product',
        cause: 'education',
        mrp: 100,
        sellingPrice: 80,
        media: [],
        description: 'Education',
        productUrl: 'https://example.com',
        vendorID: 1
      });

      await service.createProduct({
        title: 'Health Product',
        cause: 'health',
        mrp: 100,
        sellingPrice: 80,
        media: [],
        description: 'Health',
        productUrl: 'https://example.com',
        vendorID: 1
      });

      // Filter by education
      const educationProducts = await service.getProducts(1, 10, ['education']);
      expect(educationProducts.data).toHaveLength(1);
      expect(educationProducts.data[0].cause).toBe('education');

      // Filter by health
      const healthProducts = await service.getProducts(1, 10, ['health']);
      expect(healthProducts.data).toHaveLength(1);
      expect(healthProducts.data[0].cause).toBe('health');
    });

    it('should search products by title', async () => {
      await service.createProduct({
        title: 'Searchable Product Title',
        cause: 'education',
        mrp: 100,
        sellingPrice: 80,
        media: [],
        description: 'Description',
        productUrl: 'https://example.com',
        vendorID: 1
      });

      const searchResults = await service.getProducts(1, 10, undefined, 'Searchable');
      expect(searchResults.data).toHaveLength(1);
      expect(searchResults.data[0].title).toContain('Searchable');
    });
  });
});
```

## End-to-End Testing

### API E2E Tests

**File**: `test/e2e/products.e2e-spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../../src/app.module';
import { AuthHelper } from '../helpers/auth-helper';
import { DatabaseHelper } from '../helpers/database-helper';

describe('Products E2E', () => {
  let app: INestApplication;
  let authHelper: AuthHelper;
  let dbHelper: DatabaseHelper;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule]
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();

    authHelper = new AuthHelper(app);
    dbHelper = new DatabaseHelper();
    
    // Setup test database
    await dbHelper.setupTestDatabase();
    
    // Get auth token
    authToken = await authHelper.getValidToken();
  });

  afterAll(async () => {
    await dbHelper.cleanupTestDatabase();
    await app.close();
  });

  beforeEach(async () => {
    await dbHelper.clearProducts();
  });

  describe('POST /products', () => {
    it('should create a product with valid data', async () => {
      const productData = {
        title: 'E2E Test Product',
        cause: 'education',
        mrp: 100,
        sellingPrice: 80,
        media: ['image1.jpg'],
        description: 'E2E test description',
        productUrl: 'https://example.com',
        vendorID: 1
      };

      const response = await request(app.getHttpServer())
        .post('/products')
        .set('Authorization', `Bearer ${authToken}`)
        .send(productData)
        .expect(201);

      expect(response.body.message).toBe('Product created successfully');
      expect(response.body.id).toBeDefined();
      expect(response.body.id).toMatch(/^IM_/);
    });

    it('should return 400 for invalid product data', async () => {
      const invalidData = {
        title: '', // Empty title
        cause: 'invalid-cause',
        mrp: -10 // Negative price
      };

      await request(app.getHttpServer())
        .post('/products')
        .set('Authorization', `Bearer ${authToken}`)
        .send(invalidData)
        .expect(400);
    });

    it('should return 401 without authentication', async () => {
      const productData = {
        title: 'Test Product',
        cause: 'education',
        mrp: 100,
        sellingPrice: 80
      };

      await request(app.getHttpServer())
        .post('/products')
        .send(productData)
        .expect(401);
    });
  });

  describe('GET /products', () => {
    beforeEach(async () => {
      // Create test products
      await dbHelper.createTestProducts([
        {
          title: 'Education Product 1',
          cause: 'education',
          publishingStatus: 'published'
        },
        {
          title: 'Health Product 1',
          cause: 'health',
          publishingStatus: 'published'
        },
        {
          title: 'Draft Product',
          cause: 'education',
          publishingStatus: 'draft'
        }
      ]);
    });

    it('should return paginated products', async () => {
      const response = await request(app.getHttpServer())
        .get('/products?page=1&limit=10')
        .expect(200);

      expect(response.body.data).toBeDefined();
      expect(response.body.pagination).toBeDefined();
      expect(response.body.pagination.page).toBe(1);
      expect(response.body.pagination.limit).toBe(10);
    });

    it('should filter products by cause', async () => {
      const response = await request(app.getHttpServer())
        .get('/products?causes=education')
        .expect(200);

      expect(response.body.data).toBeDefined();
      response.body.data.forEach(product => {
        expect(product.cause).toBe('education');
      });
    });

    it('should search products by title', async () => {
      const response = await request(app.getHttpServer())
        .get('/products?search=Education')
        .expect(200);

      expect(response.body.data).toBeDefined();
      response.body.data.forEach(product => {
        expect(product.title.toLowerCase()).toContain('education');
      });
    });

    it('should filter by publishing status', async () => {
      const response = await request(app.getHttpServer())
        .get('/products?publishingStatus=published')
        .expect(200);

      expect(response.body.data).toBeDefined();
      response.body.data.forEach(product => {
        expect(product.publishingStatus).toBe('published');
      });
    });
  });

  describe('GET /products/:id', () => {
    it('should return product by ID', async () => {
      const product = await dbHelper.createTestProduct({
        title: 'Test Product for ID',
        cause: 'education'
      });

      const response = await request(app.getHttpServer())
        .get(`/products/${product.id}`)
        .expect(200);

      expect(response.body.id).toBe(product.id);
      expect(response.body.title).toBe('Test Product for ID');
    });

    it('should return 404 for non-existent product', async () => {
      await request(app.getHttpServer())
        .get('/products/non-existent-id')
        .expect(404);
    });
  });
});
```

### WebSocket E2E Tests

**File**: `test/e2e/chat.e2e-spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import { AppModule } from '../../src/app.module';
import { io, Socket } from 'socket.io-client';
import { AuthHelper } from '../helpers/auth-helper';
import { DatabaseHelper } from '../helpers/database-helper';

describe('Chat WebSocket E2E', () => {
  let app: INestApplication;
  let authHelper: AuthHelper;
  let dbHelper: DatabaseHelper;
  let client1: Socket;
  let client2: Socket;
  let token1: string;
  let token2: string;
  let testGroup: any;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule]
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.listen(0); // Random port
    
    const address = app.getHttpServer().address();
    const baseUrl = `http://localhost:${address.port}`;

    authHelper = new AuthHelper(app);
    dbHelper = new DatabaseHelper();
    
    // Setup test data
    await dbHelper.setupTestDatabase();
    token1 = await authHelper.getValidToken(1);
    token2 = await authHelper.getValidToken(2);
    
    // Create test group
    testGroup = await dbHelper.createTestGroup({
      name: 'Test Group',
      members: [1, 2],
      productId: 'test-product'
    });

    // Setup WebSocket clients
    client1 = io(baseUrl, {
      auth: { token: token1 },
      transports: ['websocket']
    });

    client2 = io(baseUrl, {
      auth: { token: token2 },
      transports: ['websocket']
    });

    // Wait for connections
    await Promise.all([
      new Promise(resolve => client1.on('connect', resolve)),
      new Promise(resolve => client2.on('connect', resolve))
    ]);
  });

  afterAll(async () => {
    client1.disconnect();
    client2.disconnect();
    await dbHelper.cleanupTestDatabase();
    await app.close();
  });

  describe('Message Broadcasting', () => {
    it('should broadcast message to group members', (done) => {
      const testMessage = {
        groupId: testGroup.id,
        text: 'Hello from E2E test',
        attachments: []
      };

      // Client 2 listens for new message
      client2.on('newMessage', (message) => {
        expect(message.text).toBe(testMessage.text);
        expect(message.groupId).toBe(testGroup.id);
        expect(message.senderId).toBe(1);
        done();
      });

      // Client 1 sends message
      client1.emit('sendMessage', testMessage);
    });

    it('should handle message with attachments', (done) => {
      const testMessage = {
        groupId: testGroup.id,
        text: 'Message with attachment',
        attachments: [
          { url: 'https://example.com/file.pdf', type: 'application/pdf' }
        ]
      };

      client2.on('newMessage', (message) => {
        expect(message.attachments).toHaveLength(1);
        expect(message.attachments[0].url).toBe('https://example.com/file.pdf');
        done();
      });

      client1.emit('sendMessage', testMessage);
    });
  });

  describe('Error Handling', () => {
    it('should handle invalid group ID', (done) => {
      const invalidMessage = {
        groupId: 'non-existent-group',
        text: 'This should fail',
        attachments: []
      };

      client1.on('error', (error) => {
        expect(error.message).toBeDefined();
        done();
      });

      client1.emit('sendMessage', invalidMessage);
    });

    it('should handle unauthenticated connection', (done) => {
      const unauthClient = io(`http://localhost:${app.getHttpServer().address().port}`, {
        auth: { token: 'invalid-token' },
        transports: ['websocket']
      });

      unauthClient.on('disconnect', () => {
        done();
      });
    });
  });
});
```

## Test Helpers

### Authentication Helper

**File**: `test/helpers/auth-helper.ts`

```typescript
import { INestApplication } from '@nestjs/common';
import { JwtService } from '../../src/jwt/jwt.service';

export class AuthHelper {
  constructor(private app: INestApplication) {}

  async getValidToken(userId: number = 1): Promise<string> {
    // Mock JWT payload
    const payload = {
      sub: userId,
      email: `user${userId}@example.com`,
      role: 'user',
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + 3600 // 1 hour
    };

    // Generate test token (in real scenario, use proper JWT signing)
    return 'test-jwt-token-' + userId;
  }

  async getExpiredToken(): Promise<string> {
    return 'expired-jwt-token';
  }

  async getInvalidToken(): Promise<string> {
    return 'invalid-jwt-token';
  }
}
```

### Database Helper

**File**: `test/helpers/database-helper.ts`

```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import { products, groups, messages, users } from '../../src/drizzle/schema';

export class DatabaseHelper {
  private db: any;
  private pool: Pool;

  async setupTestDatabase() {
    this.pool = new Pool({
      connectionString: process.env.TEST_DATABASE_URL
    });
    this.db = drizzle(this.pool);
  }

  async cleanupTestDatabase() {
    await this.pool.end();
  }

  async clearProducts() {
    await this.db.delete(products);
  }

  async clearGroups() {
    await this.db.delete(groups);
  }

  async clearMessages() {
    await this.db.delete(messages);
  }

  async createTestProduct(data: Partial<any>) {
    const productData = {
      id: 'IM_test_' + Date.now(),
      title: 'Test Product',
      cause: 'education',
      mrp: 100,
      sellingPrice: 80,
      media: [],
      description: 'Test description',
      productUrl: 'https://example.com',
      vendorID: 1,
      publishingStatus: 'published',
      ...data
    };

    const [product] = await this.db
      .insert(products)
      .values(productData)
      .returning();

    return product;
  }

  async createTestProducts(productsData: Partial<any>[]) {
    const products = [];
    for (const data of productsData) {
      products.push(await this.createTestProduct(data));
    }
    return products;
  }

  async createTestGroup(data: Partial<any>) {
    const groupData = {
      id: 'group_test_' + Date.now(),
      name: 'Test Group',
      productId: 'test-product',
      members: [1, 2],
      ...data
    };

    const [group] = await this.db
      .insert(groups)
      .values(groupData)
      .returning();

    return group;
  }

  async createTestMessage(data: Partial<any>) {
    const messageData = {
      id: 'msg_test_' + Date.now(),
      senderId: 1,
      groupId: 'test-group',
      text: 'Test message',
      attachments: [],
      readBy: [1],
      timestamp: new Date(),
      ...data
    };

    const [message] = await this.db
      .insert(messages)
      .values(messageData)
      .returning();

    return message;
  }
}
```

## Performance Testing

### Load Testing with Artillery

**File**: `test/performance/load-test.yml`

```yaml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Ramp up load"
    - duration: 300
      arrivalRate: 100
      name: "Sustained load"
  variables:
    auth_token: "Bearer test-jwt-token"
  
scenarios:
  - name: "API Load Test"
    weight: 70
    flow:
      - get:
          url: "/products"
          headers:
            Authorization: "{{ auth_token }}"
      - think: 2
      - get:
          url: "/products/{{ $randomString() }}"
          headers:
            Authorization: "{{ auth_token }}"
          expect:
            - statusCode: [200, 404]
      
  - name: "Product Creation"
    weight: 20
    flow:
      - post:
          url: "/products"
          headers:
            Authorization: "{{ auth_token }}"
            Content-Type: "application/json"
          json:
            title: "Load Test Product {{ $randomString() }}"
            cause: "education"
            mrp: 100
            sellingPrice: 80
            media: []
            description: "Load test description"
            productUrl: "https://example.com"
            vendorID: 1
          expect:
            - statusCode: 201
            
  - name: "File Upload"
    weight: 10
    flow:
      - post:
          url: "/s3/upload/single"
          headers:
            Authorization: "{{ auth_token }}"
          formData:
            file: "@test/fixtures/test-image.jpg"
            filePath: "load-test"
          expect:
            - statusCode: 201
```

### Stress Testing with K6

**File**: `test/performance/stress-test.js`

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

export let errorRate = new Rate('errors');

export let options = {
  stages: [
    { duration: '2m', target: 100 }, // Ramp up
    { duration: '5m', target: 100 }, // Stay at 100 users
    { duration: '2m', target: 200 }, // Ramp up to 200 users
    { duration: '5m', target: 200 }, // Stay at 200 users
    { duration: '2m', target: 300 }, // Ramp up to 300 users
    { duration: '5m', target: 300 }, // Stay at 300 users
    { duration: '2m', target: 0 },   // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests must complete below 500ms
    errors: ['rate<0.1'], // Error rate must be below 10%
  },
};

const BASE_URL = 'http://localhost:3000';
const AUTH_TOKEN = 'Bearer test-jwt-token';

export default function () {
  // Test product listing
  let response = http.get(`${BASE_URL}/products`, {
    headers: { Authorization: AUTH_TOKEN },
  });
  
  check(response, {
    'products list status is 200': (r) => r.status === 200,
    'products list response time < 500ms': (r) => r.timings.duration < 500,
  }) || errorRate.add(1);

  sleep(1);

  // Test product creation
  const productData = {
    title: `Stress Test Product ${Math.random()}`,
    cause: 'education',
    mrp: 100,
    sellingPrice: 80,
    media: [],
    description: 'Stress test description',
    productUrl: 'https://example.com',
    vendorID: 1
  };

  response = http.post(`${BASE_URL}/products`, JSON.stringify(productData), {
    headers: {
      'Content-Type': 'application/json',
      Authorization: AUTH_TOKEN,
    },
  });

  check(response, {
    'product creation status is 201': (r) => r.status === 201,
    'product creation response time < 1000ms': (r) => r.timings.duration < 1000,
  }) || errorRate.add(1);

  sleep(2);
}
```

## Test Configuration

### Jest Configuration

**File**: `jest.config.js`

```javascript
module.exports = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: {
    '^.+\\.(t|j)s$': 'ts-jest',
  },
  collectCoverageFrom: [
    '**/*.(t|j)s',
    '!**/*.spec.ts',
    '!**/*.e2e-spec.ts',
    '!**/node_modules/**',
    '!**/dist/**',
  ],
  coverageDirectory: '../coverage',
  testEnvironment: 'node',
  setupFilesAfterEnv: ['<rootDir>/../test/setup.ts'],
  moduleNameMapping: {
    '^@/(.*)$': '<rootDir>/$1',
  },
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

### E2E Jest Configuration

**File**: `test/jest-e2e.json`

```json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": {
    "^.+\\.(t|j)s$": "ts-jest"
  },
  "setupFilesAfterEnv": ["<rootDir>/setup-e2e.ts"],
  "testTimeout": 30000
}
```

### Test Setup

**File**: `test/setup.ts`

```typescript
import { config } from 'dotenv';

// Load test environment variables
config({ path: '.env.test' });

// Global test setup
beforeAll(() => {
  // Set test environment
  process.env.NODE_ENV = 'test';
  
  // Mock console methods to reduce noise
  jest.spyOn(console, 'log').mockImplementation(() => {});
  jest.spyOn(console, 'warn').mockImplementation(() => {});
});

afterAll(() => {
  // Restore console methods
  jest.restoreAllMocks();
});
```

## Test Scripts

### Package.json Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:debug": "node --inspect-brk -r tsconfig-paths/register -r ts-node/register node_modules/.bin/jest --runInBand",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "test:e2e:watch": "jest --config ./test/jest-e2e.json --watch",
    "test:integration": "jest --testPathPattern=integration",
    "test:unit": "jest --testPathPattern=spec.ts$",
    "test:load": "artillery run test/performance/load-test.yml",
    "test:stress": "k6 run test/performance/stress-test.js",
    "test:all": "npm run test:unit && npm run test:integration && npm run test:e2e"
  }
}
```

## Continuous Integration

### GitHub Actions Workflow

**File**: `.github/workflows/test.yml`

```yaml
name: Test Suite

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '20'
        cache: 'npm'
    
    - name: Install pnpm
      run: npm install -g pnpm
    
    - name: Install dependencies
      run: pnpm install
    
    - name: Setup test database
      run: |
        pnpm run migration:run
        pnpm run seed:test
      env:
        TEST_DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
    
    - name: Run unit tests
      run: pnpm run test:unit
      env:
        NODE_ENV: test
    
    - name: Run integration tests
      run: pnpm run test:integration
      env:
        NODE_ENV: test
        TEST_DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
    
    - name: Run E2E tests
      run: pnpm run test:e2e
      env:
        NODE_ENV: test
        TEST_DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
    
    - name: Generate coverage report
      run: pnpm run test:cov
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info
        flags: unittests
        name: codecov-umbrella
```

## Test Best Practices

### 1. Test Organization

- **Unit Tests**: Test individual functions/methods in isolation
- **Integration Tests**: Test component interactions with real dependencies
- **E2E Tests**: Test complete user workflows
- **Performance Tests**: Test system under load

### 2. Test Data Management

```typescript
// Use factories for test data
class ProductFactory {
  static create(overrides: Partial<Product> = {}): Product {
    return {
      id: 'IM_test_' + Math.random(),
      title: 'Test Product',
      cause: 'education',
      mrp: 100,
      sellingPrice: 80,
      ...overrides
    };
  }
}

// Use in tests
const product = ProductFactory.create({ title: 'Custom Title' });
```

### 3. Mock Management

```typescript
// Create reusable mocks
const createMockProductService = () => ({
  createProduct: jest.fn(),
  getProducts: jest.fn(),
  getProductById: jest.fn(),
  updateProduct: jest.fn(),
  deleteProduct: jest.fn()
});

// Use consistent mock data
const MOCK_PRODUCT = {
  id: 'IM_test123',
  title: 'Mock Product',
  cause: 'education'
};
```

### 4. Test Coverage Goals

- **Unit Tests**: 90%+ coverage
- **Integration Tests**: Critical paths covered
- **E2E Tests**: Main user journeys covered
- **Performance Tests**: Key endpoints tested

### 5. Test Maintenance

- Keep tests simple and focused
- Use descriptive test names
- Clean up test data after each test
- Mock external dependencies
- Run tests in CI/CD pipeline
- Monitor test performance
- Update tests when code changes
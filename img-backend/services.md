# Services & Business Logic

## Overview

The services layer contains the core business logic of the Deed-O Backend application. Each service is responsible for specific domain operations and data management.

## Service Architecture

### Service Structure
```
src/services/
├── products.service.ts     # Product management
├── groups.service.ts       # Group management
├── chat.service.ts         # Chat functionality
├── jwt.service.ts          # JWT token operations
├── s3.service.ts          # File storage operations
└── email.service.ts       # Email notifications
```

### Dependency Injection

All services are registered in `app.module.ts` and use NestJS dependency injection:

```typescript
@Module({
  providers: [
    JwtService,
    EmailService,
    ProductService,
    GroupService,
    ChatService,
    S3Service
  ]
})
export class AppModule {}
```

## Product Service

### File: `products.service.ts`

#### Responsibilities
- Product CRUD operations
- Unique ID generation
- Vendor-specific product management
- Search and filtering
- Publishing workflow management

#### Key Methods

##### 1. Generate Unique Product ID
```typescript
async generateUniqueProductId(): Promise<string> {
  let uniqueId: string;
  let exists: boolean;

  do {
    uniqueId = generateShortToken(); // Generate base64url token
    const existingProduct = await db
      .select()
      .from(products)
      .where(eq(products.id, uniqueId))
      .execute();
    exists = existingProduct.length > 0;
  } while (exists);

  return uniqueId;
}
```

**Features**:
- Generates URL-safe base64 tokens
- Ensures uniqueness through database lookup
- Adds "IM_" prefix to final ID

##### 2. Create Product
```typescript
async createProduct(productData) {
  const suf_id = await this.generateUniqueProductId();
  const id = 'IM_' + suf_id;
  const newProduct = { ...productData, id };

  await db.insert(products).values(newProduct).execute();
  return { message: 'Product created successfully', id };
}
```

**Features**:
- Auto-generates unique product ID
- Validates product data through DTOs
- Returns created product ID

##### 3. Get Products with Filtering
```typescript
async getProducts(
  page = 1,
  limit = 10,
  causes?: string[],
  search?: string,
  publishingStatus?: string
) {
  const offset = (page - 1) * limit;
  let conditions = [];

  // Apply filters
  if (publishingStatus) {
    conditions.push(eq(products.publishingStatus, publishingStatus));
  }
  if (causes?.length > 0) {
    conditions.push(inArray(products.cause, causes));
  }
  if (search) {
    conditions.push(ilike(products.title, `%${search}%`));
  }

  // Execute query with pagination
  const result = await db
    .select()
    .from(products)
    .where(and(...conditions))
    .orderBy(desc(products.updatedAt))
    .limit(limit)
    .offset(offset)
    .execute();

  return { data: result, pagination: {...} };
}
```

**Features**:
- Advanced filtering by cause, status, and search terms
- Pagination support
- Case-insensitive search
- Sorted by update time

##### 4. Vendor-Specific Operations
```typescript
async getVendorProducts(
  vendorId: number,
  page = 1,
  limit = 10,
  status?: string
) {
  // Filter products by vendor ID and optional status
  // Returns paginated results
}
```

## Group Service

### File: `groups.service.ts`

#### Responsibilities
- Group creation and management
- Member management
- Group-product associations
- Unread message tracking

#### Key Methods

##### 1. Create Group
```typescript
async createGroup(createGroupDto: CreateGroupDto) {
  const result = await db
    .insert(groups)
    .values(createGroupDto)
    .returning()
    .execute();
  return { message: 'Group created successfully', group: result[0] };
}
```

##### 2. Get or Create Group
```typescript
async getOrCreateGroup(createGroupDto: CreateGroupDto) {
  const sortedMembers = createGroupDto.members.sort((a, b) => a - b);

  // Check for existing group with same members and product
  const existingGroup = await db
    .select()
    .from(groups)
    .where(eq(groups.productId, createGroupDto.productId))
    .execute();

  const matchingGroup = existingGroup.find(
    (group) =>
      JSON.stringify(group.members.sort((a, b) => a - b)) ===
      JSON.stringify(sortedMembers)
  );

  if (matchingGroup) {
    return { message: 'Group already exists', group: matchingGroup };
  }

  // Create new group if no match found
  return this.createGroup({ ...createGroupDto, members: sortedMembers });
}
```

**Features**:
- Prevents duplicate groups with same members
- Sorts members for consistent comparison
- Returns existing group or creates new one

##### 3. Get Group with Last Message
```typescript
async getGroupById(id: string) {
  const result = await db
    .select({
      id: groups.id,
      name: groups.name,
      productId: groups.productId,
      members: groups.members,
      createdAt: groups.createdAt,
      updatedAt: groups.updatedAt,
      lastMessageData: {
        attachments: messages.attachments,
        id: messages.id,
        readBy: messages.readBy,
        senderId: messages.senderId,
        text: messages.text,
        timestamp: messages.timestamp
      }
    })
    .from(groups)
    .leftJoin(messages, sql`${groups.lastMessageId} = ${messages.id}`)
    .where(eq(groups.id, id))
    .execute();

  return result[0];
}
```

**Features**:
- Joins with messages table
- Returns complete group information
- Includes last message details

##### 4. Unread Message Tracking
```typescript
async getUnreadMessages(userId: number) {
  // Complex query to count unread messages across all user groups
  // Returns aggregated unread counts
}

async getUnreadMessagesByGroupId(userId: number, groupId: string) {
  // Returns unread messages for specific group
  // Filters by user ID and read status
}
```

## Chat Service

### File: `chat.service.ts`

#### Responsibilities
- Message creation and management
- Real-time message handling
- User online status tracking
- Email notifications for offline users

#### Key Methods

##### 1. Create Message
```typescript
async createMessage(messageData: {
  senderId: number;
  groupId: string;
  text: string;
  attachments: { url: string; type: string }[];
  readBy: number[];
}) {
  // Insert message
  const [message] = await db
    .insert(messages)
    .values({
      ...messageData,
      timestamp: new Date()
    })
    .returning();

  // Update group's last message
  await db
    .update(groups)
    .set({
      lastMessageId: message.id,
      updatedAt: message.timestamp || new Date()
    })
    .where(eq(groups.id, messageData.groupId))
    .returning();

  return message;
}
```

**Features**:
- Atomic message creation
- Updates group's last message reference
- Handles attachments and read receipts

##### 2. Get Messages with Pagination
```typescript
async getMessages(groupId: string, page = 1, limit = 20) {
  const offset = (page - 1) * limit;
  return db
    .select()
    .from(messages)
    .where(eq(messages.groupId, groupId))
    .orderBy(desc(messages.timestamp))
    .limit(limit)
    .offset(offset)
    .execute();
}
```

##### 3. User Status Management
```typescript
async isUserOnline(userId: number): Promise<boolean> {
  const [user] = await authDb
    .select({ isOnline: users.isOnline })
    .from(users)
    .where(eq(users.id, userId))
    .execute();
  return user?.isOnline || false;
}

async setUserOnline(userId: number, isOnline: boolean) {
  await authDb
    .update(users)
    .set({ isOnline })
    .where(eq(users.id, userId))
    .execute();
}
```

##### 4. Email Notifications
```typescript
async sendEmailNotification(
  receiverId: number,
  senderId: number,
  message: string
) {
  try {
    // Get receiver and sender details
    const [receiver] = await authDb
      .select({ email: users.email, name: users.name })
      .from(users)
      .where(eq(users.id, receiverId))
      .execute();

    const [sender] = await authDb
      .select({ name: users.name })
      .from(users)
      .where(eq(users.id, senderId))
      .execute();

    if (receiver?.email) {
      const emailBody = newMessageTemplate(
        message,
        receiver.name || 'User',
        sender?.name
      );
      
      await this.emailService.sendEmail(
        receiver.email,
        NEW_MESSAGE_EMAIL_SUBJECT(sender?.name),
        emailBody
      );
    }
  } catch (error) {
    console.error('Failed to send email notification:', error);
  }
}
```

**Features**:
- Sends email to offline users
- Uses React email templates
- Graceful error handling

## JWT Service

### File: `jwt.service.ts`

#### Responsibilities
- JWT token verification
- Public key management
- Token payload extraction

#### Implementation
```typescript
export class JwtService {
  private readonly algorithm = 'RS256';

  async verifyToken(token: string): Promise<IJwtPayload> {
    const spki = process.env.PUBLIC_KEY;
    const publicKey = await importSPKI(spki, this.algorithm);
    const { payload } = await jwtVerify(token, publicKey);
    return payload as IJwtPayload;
  }
}
```

**Features**:
- RS256 asymmetric verification
- Type-safe payload extraction
- Environment-based key management

## S3 Service

### File: `s3.service.ts`

#### Responsibilities
- File upload to AWS S3
- File validation
- Signed URL generation
- Multi-file upload support

#### Key Methods

##### 1. File Validation
```typescript
private validateFile(file: MultipartFile) {
  const mediaType = file.mimetype.split('/')[0];
  
  const maxSizeMap: Record<string, number> = {
    image: 5 * 1024 * 1024,      // 5MB
    video: 200 * 1024 * 1024,    // 200MB
    audio: 10 * 1024 * 1024,     // 10MB
    application: 20 * 1024 * 1024, // 20MB
    file: 20 * 1024 * 1024       // 20MB
  };

  if (!maxSizeMap[mediaType]) {
    throw new BadRequestException(`Invalid media type: ${mediaType}`);
  }

  if (file.file.truncated) {
    throw new BadRequestException(
      `Media size exceeds limit for ${mediaType}`
    );
  }
}
```

##### 2. File Upload
```typescript
async uploadFile(file: MultipartFile, filePath: string): Promise<string> {
  this.validateFile(file);

  const uniqueFileName = `${uuidv4()}-${file.filename}`;
  const fileKey = `${filePath}/${uniqueFileName}`;

  const uploadCommand = new PutObjectCommand({
    Bucket: this.bucketName,
    Key: fileKey,
    Body: await file.toBuffer(),
    ContentType: file.mimetype
  });

  await this.s3.send(uploadCommand);
  return `https://${this.bucketName}.s3.${this.configService.get('AWS_REGION')}.amazonaws.com/${fileKey}`;
}
```

**Features**:
- UUID-based unique filenames
- Content-type preservation
- Direct S3 upload
- Public URL generation

##### 3. Signed URL Generation
```typescript
async getSignedUrl(
  filePath: string,
  expiresInSeconds = 3600
): Promise<string> {
  const url = new URL(filePath, `https://${this.bucketName}.s3.${this.configService.get('AWS_REGION')}.amazonaws.com`);
  const objectKey = url.pathname.startsWith('/') ? url.pathname.substring(1) : url.pathname;

  const command = new GetObjectCommand({
    Bucket: this.bucketName,
    Key: objectKey
  });

  return getSignedUrl(this.s3, command, { expiresIn: expiresInSeconds });
}
```

## Email Service

### File: `email.service.ts`

#### Responsibilities
- AWS SES integration
- Email sending
- Template rendering

#### Implementation
```typescript
@Injectable()
export class EmailService {
  private ses: SESv2Client;

  constructor(private readonly configService: ConfigService) {
    this.ses = new SESv2Client({
      region: this.configService.get<string>('AWS_REGION'),
      credentials: {
        accessKeyId: this.configService.get<string>('AWS_ACCESS_KEY_ID'),
        secretAccessKey: this.configService.get<string>('AWS_SECRET_ACCESS_KEY')
      }
    });
  }

  async sendEmail(to: string, subject: string, body: string, from?: string) {
    const input = {
      FromEmailAddress: from || this.configService.get<string>('SENDER_EMAIL'),
      Destination: { ToAddresses: [to] },
      Content: {
        Simple: {
          Subject: { Data: subject, Charset: 'UTF-8' },
          Body: { Html: { Charset: 'UTF-8', Data: body } }
        }
      }
    };

    const command = new SendEmailCommand(input);
    return await this.ses.send(command);
  }
}
```

**Features**:
- AWS SES v2 integration
- HTML email support
- Configurable sender address
- Error handling

## Service Integration Patterns

### 1. Cross-Service Communication
```typescript
// ChatService using EmailService
constructor(private readonly emailService: EmailService) {}

// ProductService using database directly
import { db } from '@/drizzle/db';
```

### 2. Error Handling
```typescript
try {
  const result = await this.performOperation();
  return result;
} catch (error) {
  console.error('Operation failed:', error);
  throw new InternalServerErrorException('Operation failed');
}
```

### 3. Validation Integration
```typescript
// Services rely on DTOs for validation
async createProduct(productData: CreateProductDto) {
  // DTO validation happens at controller level
  // Service assumes valid data
}
```

## Performance Considerations

### 1. Database Optimization
- Use of indexes for frequent queries
- Pagination for large datasets
- Selective field querying
- Connection pooling

### 2. File Upload Optimization
- Streaming file uploads
- File size validation
- Parallel multi-file uploads
- Direct S3 upload (no server storage)

### 3. Caching Strategies
- User online status caching
- Frequently accessed product data
- Group member lists

## Testing Considerations

### Unit Testing
```typescript
// Example service test
describe('ProductService', () => {
  let service: ProductService;
  
  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [ProductService]
    }).compile();
    
    service = module.get<ProductService>(ProductService);
  });
  
  it('should generate unique product ID', async () => {
    const id = await service.generateUniqueProductId();
    expect(id).toBeDefined();
    expect(typeof id).toBe('string');
  });
});
```

### Integration Testing
- Database integration tests
- AWS service mocking
- End-to-end workflow testing

## Monitoring & Logging

### Service Metrics
- Operation success/failure rates
- Response times
- Database query performance
- File upload success rates
- Email delivery rates

### Error Logging
```typescript
console.error({
  service: 'ProductService',
  method: 'createProduct',
  error: error.message,
  userId: productData.vendorID,
  timestamp: new Date().toISOString()
});
```
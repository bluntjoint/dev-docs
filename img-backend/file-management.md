# File Management

## Overview

The Deed-O Backend implements comprehensive file management using AWS S3 for storage, with support for multiple file types, validation, and secure access through signed URLs.

## Architecture

### Components

1. **S3 Controller** (`s3.controller.ts`) - HTTP endpoints for file operations
2. **S3 Service** (`s3.service.ts`) - Business logic for S3 operations
3. **AWS S3 Integration** - Direct integration with AWS SDK v3
4. **File Validation** - Type and size validation
5. **Signed URLs** - Secure file access

### File Storage Structure

```
S3 Bucket Structure:
├── products/
│   ├── {uuid}-filename.jpg
│   ├── {uuid}-filename.mp4
│   └── ...
├── messages/
│   ├── {uuid}-attachment.pdf
│   ├── {uuid}-image.png
│   └── ...
└── uploads/
    ├── {uuid}-document.docx
    └── ...
```

## S3 Controller

### File: `src/s3.controller.ts`

#### Endpoints

##### 1. Single File Upload

```typescript
@Post('upload/single')
@UseGuards(AuthGuard)
@ApiConsumes('multipart/form-data')
@ApiBody({
  schema: {
    type: 'object',
    properties: {
      file: {
        type: 'string',
        format: 'binary'
      },
      filePath: {
        type: 'string',
        description: 'Directory path in S3 bucket'
      }
    }
  }
})
async uploadSingle(@Req() request: FastifyRequest) {
  try {
    const data = await request.file();
    if (!data) {
      throw new BadRequestException('No file provided');
    }

    const body = Object.fromEntries(
      Object.entries(data.fields).map(([key, value]) => [
        key,
        (value as any).value
      ])
    );

    const filePath = body.filePath || 'uploads';
    const fileUrl = await this.s3Service.uploadFile(data, filePath);

    return {
      message: 'File uploaded successfully',
      fileUrl
    };
  } catch (error) {
    console.error('Upload error:', error);
    throw new InternalServerErrorException('File upload failed');
  }
}
```

**Features**:
- Authentication required
- Multipart form data support
- Custom file path specification
- Error handling with detailed logging
- Swagger documentation

**Request Format**:
```bash
curl -X POST \
  http://localhost:3000/s3/upload/single \
  -H 'Authorization: Bearer {jwt-token}' \
  -H 'Content-Type: multipart/form-data' \
  -F 'file=@/path/to/file.jpg' \
  -F 'filePath=products'
```

**Response**:
```json
{
  "message": "File uploaded successfully",
  "fileUrl": "https://bucket.s3.region.amazonaws.com/products/uuid-file.jpg"
}
```

##### 2. Multiple File Upload

```typescript
@Post('upload/multiple')
@UseGuards(AuthGuard)
@ApiConsumes('multipart/form-data')
async uploadMultiple(@Req() request: FastifyRequest) {
  try {
    const parts = request.parts();
    const files: MultipartFile[] = [];
    let filePath = 'uploads';

    // Process multipart data
    for await (const part of parts) {
      if (part.type === 'file') {
        files.push(part);
      } else if (part.fieldname === 'filePath') {
        filePath = part.value as string;
      }
    }

    if (files.length === 0) {
      throw new BadRequestException('No files provided');
    }

    // Upload all files
    const uploadPromises = files.map(file => 
      this.s3Service.uploadFile(file, filePath)
    );
    const fileUrls = await Promise.all(uploadPromises);

    return {
      message: `${files.length} files uploaded successfully`,
      fileUrls
    };
  } catch (error) {
    console.error('Multiple upload error:', error);
    throw new InternalServerErrorException('File upload failed');
  }
}
```

**Features**:
- Parallel file uploads
- Batch processing
- Shared file path for all files
- Promise-based concurrent uploads

**Request Format**:
```bash
curl -X POST \
  http://localhost:3000/s3/upload/multiple \
  -H 'Authorization: Bearer {jwt-token}' \
  -H 'Content-Type: multipart/form-data' \
  -F 'files=@/path/to/file1.jpg' \
  -F 'files=@/path/to/file2.png' \
  -F 'filePath=products'
```

**Response**:
```json
{
  "message": "2 files uploaded successfully",
  "fileUrls": [
    "https://bucket.s3.region.amazonaws.com/products/uuid1-file1.jpg",
    "https://bucket.s3.region.amazonaws.com/products/uuid2-file2.png"
  ]
}
```

##### 3. Signed URL Generation

```typescript
@Get('signed-url')
@UseGuards(AuthGuard)
@ApiQuery({
  name: 'filePath',
  required: true,
  description: 'Full S3 file URL or path'
})
@ApiQuery({
  name: 'expiresInSeconds',
  required: false,
  description: 'URL expiration time in seconds (default: 3600)'
})
async getSignedUrl(
  @Query('filePath') filePath: string,
  @Query('expiresInSeconds') expiresInSeconds?: string
) {
  try {
    if (!filePath) {
      throw new BadRequestException('File path is required');
    }

    const expires = expiresInSeconds ? parseInt(expiresInSeconds) : 3600;
    const signedUrl = await this.s3Service.getSignedUrl(filePath, expires);

    return {
      signedUrl,
      expiresIn: expires
    };
  } catch (error) {
    console.error('Signed URL error:', error);
    throw new InternalServerErrorException('Failed to generate signed URL');
  }
}
```

**Features**:
- Temporary secure access
- Configurable expiration time
- URL validation
- Authentication required

**Request Format**:
```bash
curl -X GET \
  'http://localhost:3000/s3/signed-url?filePath=https://bucket.s3.region.amazonaws.com/products/file.jpg&expiresInSeconds=7200' \
  -H 'Authorization: Bearer {jwt-token}'
```

**Response**:
```json
{
  "signedUrl": "https://bucket.s3.region.amazonaws.com/products/file.jpg?X-Amz-Algorithm=...",
  "expiresIn": 7200
}
```

## S3 Service

### File: `src/s3.service.ts`

#### Configuration

```typescript
@Injectable()
export class S3Service {
  private readonly s3: S3Client;
  private readonly bucketName: string;

  constructor(private readonly configService: ConfigService) {
    this.s3 = new S3Client({
      region: this.configService.get<string>('AWS_REGION'),
      credentials: {
        accessKeyId: this.configService.get<string>('AWS_ACCESS_KEY_ID'),
        secretAccessKey: this.configService.get<string>('AWS_SECRET_ACCESS_KEY')
      }
    });
    
    this.bucketName = this.configService.get<string>('AWS_S3_BUCKET_NAME');
  }
}
```

**Configuration Requirements**:
- `AWS_REGION`: AWS region (e.g., 'us-east-1')
- `AWS_ACCESS_KEY_ID`: AWS access key
- `AWS_SECRET_ACCESS_KEY`: AWS secret key
- `AWS_S3_BUCKET_NAME`: S3 bucket name

#### File Validation

```typescript
private validateFile(file: MultipartFile) {
  const mediaType = file.mimetype.split('/')[0];
  
  // Define size limits by media type
  const maxSizeMap: Record<string, number> = {
    image: 5 * 1024 * 1024,        // 5MB
    video: 200 * 1024 * 1024,      // 200MB
    audio: 10 * 1024 * 1024,       // 10MB
    application: 20 * 1024 * 1024,  // 20MB (PDFs, docs)
    text: 1 * 1024 * 1024,         // 1MB
    file: 20 * 1024 * 1024         // 20MB (fallback)
  };

  const maxSize = maxSizeMap[mediaType] || maxSizeMap.file;

  // Check if media type is allowed
  if (!maxSizeMap[mediaType]) {
    throw new BadRequestException(
      `Unsupported media type: ${file.mimetype}`
    );
  }

  // Check file size (Fastify truncates large files)
  if (file.file.truncated) {
    throw new BadRequestException(
      `File size exceeds ${Math.round(maxSize / (1024 * 1024))}MB limit for ${mediaType} files`
    );
  }

  console.log(`File validation passed: ${file.filename} (${file.mimetype})`);
}
```

**Validation Rules**:
- **Images**: 5MB limit (JPEG, PNG, GIF, WebP)
- **Videos**: 200MB limit (MP4, AVI, MOV, WebM)
- **Audio**: 10MB limit (MP3, WAV, AAC)
- **Documents**: 20MB limit (PDF, DOC, DOCX, XLS, XLSX)
- **Text**: 1MB limit (TXT, CSV, JSON)
- **Other**: 20MB limit (fallback)

#### File Upload Implementation

```typescript
async uploadFile(file: MultipartFile, filePath: string): Promise<string> {
  // Validate file before upload
  this.validateFile(file);

  // Generate unique filename
  const uniqueFileName = `${uuidv4()}-${file.filename}`;
  const fileKey = `${filePath}/${uniqueFileName}`;

  try {
    // Convert file stream to buffer
    const fileBuffer = await file.toBuffer();

    // Create S3 upload command
    const uploadCommand = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: fileKey,
      Body: fileBuffer,
      ContentType: file.mimetype,
      // Optional: Set cache control
      CacheControl: 'max-age=31536000', // 1 year
      // Optional: Set metadata
      Metadata: {
        originalName: file.filename,
        uploadedAt: new Date().toISOString()
      }
    });

    // Execute upload
    await this.s3.send(uploadCommand);

    // Return public URL
    const fileUrl = `https://${this.bucketName}.s3.${this.configService.get('AWS_REGION')}.amazonaws.com/${fileKey}`;
    
    console.log(`File uploaded successfully: ${fileUrl}`);
    return fileUrl;
  } catch (error) {
    console.error('S3 upload error:', error);
    throw new InternalServerErrorException('Failed to upload file to S3');
  }
}
```

**Upload Features**:
- UUID-based unique filenames
- Content-type preservation
- Metadata storage
- Cache control headers
- Error handling
- Public URL generation

#### Signed URL Generation

```typescript
async getSignedUrl(
  filePath: string,
  expiresInSeconds: number = 3600
): Promise<string> {
  try {
    // Parse S3 URL to extract object key
    const url = new URL(
      filePath,
      `https://${this.bucketName}.s3.${this.configService.get('AWS_REGION')}.amazonaws.com`
    );
    
    // Extract object key (remove leading slash)
    const objectKey = url.pathname.startsWith('/') 
      ? url.pathname.substring(1) 
      : url.pathname;

    // Create signed URL command
    const command = new GetObjectCommand({
      Bucket: this.bucketName,
      Key: objectKey
    });

    // Generate signed URL
    const signedUrl = await getSignedUrl(
      this.s3,
      command,
      { expiresIn: expiresInSeconds }
    );

    console.log(`Generated signed URL for: ${objectKey}`);
    return signedUrl;
  } catch (error) {
    console.error('Signed URL generation error:', error);
    throw new InternalServerErrorException('Failed to generate signed URL');
  }
}
```

**Signed URL Features**:
- Temporary access (default 1 hour)
- Configurable expiration
- URL parsing for object key extraction
- Error handling

## File Upload Configuration

### Fastify Multipart Setup

**File**: `src/main.ts`

```typescript
import multipart from '@fastify/multipart';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter()
  );

  // Register multipart support
  await app.register(multipart, {
    limits: {
      fileSize: 50 * 1024 * 1024 // 50MB global limit
    }
  });

  // ... other configurations
}
```

**Configuration Options**:
- **Global File Size Limit**: 50MB
- **Multiple File Support**: Enabled
- **Field Parsing**: Automatic
- **Stream Processing**: Supported

## File Type Support

### Supported MIME Types

#### Images
- `image/jpeg` - JPEG images
- `image/png` - PNG images
- `image/gif` - GIF images
- `image/webp` - WebP images
- `image/svg+xml` - SVG images
- `image/bmp` - Bitmap images

#### Videos
- `video/mp4` - MP4 videos
- `video/avi` - AVI videos
- `video/quicktime` - MOV videos
- `video/webm` - WebM videos
- `video/x-msvideo` - AVI videos

#### Audio
- `audio/mpeg` - MP3 audio
- `audio/wav` - WAV audio
- `audio/aac` - AAC audio
- `audio/ogg` - OGG audio
- `audio/webm` - WebM audio

#### Documents
- `application/pdf` - PDF documents
- `application/msword` - DOC documents
- `application/vnd.openxmlformats-officedocument.wordprocessingml.document` - DOCX
- `application/vnd.ms-excel` - XLS spreadsheets
- `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` - XLSX
- `application/vnd.ms-powerpoint` - PPT presentations
- `application/vnd.openxmlformats-officedocument.presentationml.presentation` - PPTX

#### Text Files
- `text/plain` - Plain text
- `text/csv` - CSV files
- `application/json` - JSON files
- `text/xml` - XML files

## Security Considerations

### File Validation

```typescript
// MIME type validation
const allowedTypes = [
  'image/jpeg', 'image/png', 'image/gif',
  'video/mp4', 'video/webm',
  'application/pdf', 'text/plain'
];

if (!allowedTypes.includes(file.mimetype)) {
  throw new BadRequestException('File type not allowed');
}
```

### File Size Limits

```typescript
// Dynamic size limits based on type
const getSizeLimit = (mimetype: string): number => {
  if (mimetype.startsWith('image/')) return 5 * 1024 * 1024;
  if (mimetype.startsWith('video/')) return 200 * 1024 * 1024;
  if (mimetype.startsWith('audio/')) return 10 * 1024 * 1024;
  return 20 * 1024 * 1024; // Default
};
```

### Filename Sanitization

```typescript
// Generate safe filenames
const sanitizeFilename = (filename: string): string => {
  return filename
    .replace(/[^a-zA-Z0-9.-]/g, '_') // Replace special chars
    .replace(/_{2,}/g, '_') // Remove multiple underscores
    .toLowerCase();
};

const uniqueFileName = `${uuidv4()}-${sanitizeFilename(file.filename)}`;
```

### Access Control

```typescript
// Authentication required for all file operations
@UseGuards(AuthGuard)
class S3Controller {
  // All endpoints require valid JWT
}

// Signed URLs for temporary access
const signedUrl = await getSignedUrl(command, {
  expiresIn: 3600 // 1 hour expiration
});
```

## Error Handling

### Upload Errors

```typescript
try {
  const fileUrl = await this.s3Service.uploadFile(file, filePath);
  return { success: true, fileUrl };
} catch (error) {
  if (error instanceof BadRequestException) {
    throw error; // Re-throw validation errors
  }
  
  console.error('Upload failed:', error);
  throw new InternalServerErrorException('File upload failed');
}
```

### Common Error Scenarios

1. **File Too Large**
   ```json
   {
     "statusCode": 400,
     "message": "File size exceeds 5MB limit for image files"
   }
   ```

2. **Invalid File Type**
   ```json
   {
     "statusCode": 400,
     "message": "Unsupported media type: application/exe"
   }
   ```

3. **AWS Credentials Error**
   ```json
   {
     "statusCode": 500,
     "message": "Failed to upload file to S3"
   }
   ```

4. **Missing File**
   ```json
   {
     "statusCode": 400,
     "message": "No file provided"
   }
   ```

## Performance Optimization

### Parallel Uploads

```typescript
// Upload multiple files concurrently
const uploadPromises = files.map(file => 
  this.s3Service.uploadFile(file, filePath)
);
const fileUrls = await Promise.all(uploadPromises);
```

### Stream Processing

```typescript
// Process large files as streams
const uploadStream = new PassThrough();
file.file.pipe(uploadStream);

const uploadCommand = new PutObjectCommand({
  Bucket: bucketName,
  Key: fileKey,
  Body: uploadStream,
  ContentType: file.mimetype
});
```

### CDN Integration

```typescript
// Use CloudFront for faster file delivery
const cdnUrl = `https://cdn.yourdomain.com/${fileKey}`;
return cdnUrl; // Instead of direct S3 URL
```

## Monitoring & Analytics

### Upload Metrics

```typescript
// Track upload statistics
const uploadMetrics = {
  totalUploads: 0,
  totalSize: 0,
  uploadsByType: new Map<string, number>(),
  failedUploads: 0
};

// Log upload events
console.log({
  event: 'file_uploaded',
  filename: file.filename,
  size: file.file.bytesRead,
  mimetype: file.mimetype,
  userId: request.user.id,
  timestamp: new Date().toISOString()
});
```

### S3 Costs Monitoring

```typescript
// Track storage costs
const costTracking = {
  storageUsed: 0, // bytes
  requestCount: 0,
  dataTransfer: 0
};
```

## Testing

### Unit Tests

```typescript
describe('S3Service', () => {
  let service: S3Service;
  let mockS3Client: jest.Mocked<S3Client>;

  beforeEach(() => {
    mockS3Client = {
      send: jest.fn()
    } as any;
    
    service = new S3Service(configService);
    (service as any).s3 = mockS3Client;
  });

  it('should upload file successfully', async () => {
    const mockFile = {
      filename: 'test.jpg',
      mimetype: 'image/jpeg',
      toBuffer: () => Promise.resolve(Buffer.from('test'))
    } as MultipartFile;

    mockS3Client.send.mockResolvedValue({});

    const result = await service.uploadFile(mockFile, 'test');
    expect(result).toContain('https://');
  });
});
```

### Integration Tests

```typescript
describe('File Upload Integration', () => {
  it('should upload file via API', async () => {
    const response = await request(app.getHttpServer())
      .post('/s3/upload/single')
      .set('Authorization', `Bearer ${validToken}`)
      .attach('file', 'test/fixtures/test-image.jpg')
      .field('filePath', 'test')
      .expect(201);

    expect(response.body.fileUrl).toBeDefined();
  });
});
```

## Best Practices

### File Organization

```typescript
// Organize files by purpose and date
const generateFilePath = (purpose: string, userId: number): string => {
  const date = new Date();
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  
  return `${purpose}/${year}/${month}/user-${userId}`;
};

// Example: products/2024/03/user-123/uuid-image.jpg
```

### Cleanup Strategies

```typescript
// Schedule cleanup of unused files
const cleanupUnusedFiles = async () => {
  // Find files not referenced in database
  // Delete files older than retention period
  // Log cleanup activities
};
```

### Backup Strategies

```typescript
// Cross-region replication
const s3ReplicationConfig = {
  Role: 'arn:aws:iam::account:role/replication-role',
  Rules: [{
    Status: 'Enabled',
    Prefix: 'important/',
    Destination: {
      Bucket: 'backup-bucket',
      Region: 'us-west-2'
    }
  }]
};
```
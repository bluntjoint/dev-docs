# API Endpoints Documentation

## Base Configuration

- **Base URL**: `/v1`
- **API Documentation**: Available at `/v1` (Swagger UI)
- **Authentication**: Bearer Token (JWT)
- **Content-Type**: `application/json`
- **Framework**: NestJS with Fastify

## Authentication

Most endpoints require authentication via Bearer token in the Authorization header:
```
Authorization: Bearer <jwt_token>
```

## Products API

### Base Route: `/v1/products`

#### 1. Create Product
```http
POST /v1/products
```

**Description**: Creates a new product with auto-generated ID

**Authentication**: Required

**Request Body**:
```json
{
  "title": "Eco-Friendly Bamboo Bottle",
  "cause": "Sustainable fashion",
  "mrp": 999,
  "sellingPrice": 799,
  "media": [
    "https://cdn.example.com/products/image1.jpg",
    "https://cdn.example.com/products/image2.jpg"
  ],
  "description": "Made from recycled glass. Eco-friendly and sustainable.",
  "productUrl": "https://shop.example.com/products/bamboo-bottle",
  "publishingStatus": "draft",
  "remarks": "Limited edition. Launching during Earth Week."
}
```

**Response**:
```json
{
  "message": "Product created successfully",
  "id": "IM_aB9X123"
}
```

**Validation Rules**:
- `title`: Required, non-empty string
- `cause`: Required, must be one of predefined causes
- `mrp`: Required, positive number
- `sellingPrice`: Required, positive number
- `media`: Required, array of URLs
- `description`: Required, non-empty string
- `productUrl`: Required, valid URL
- `publishingStatus`: Optional, defaults to 'draft'
- `remarks`: Optional string

#### 2. Get Vendor Products
```http
GET /v1/products/vendor
```

**Description**: Retrieves products for the authenticated vendor

**Authentication**: Required

**Query Parameters**:
- `publishingStatus` (optional): Filter by status (draft, published, suggestion, rejected)
- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 10)

**Example**:
```http
GET /v1/products/vendor?publishingStatus=draft&page=1&limit=10
```

**Response**:
```json
{
  "data": [
    {
      "id": "IM_aB9X123",
      "title": "Eco-Friendly Bamboo Bottle",
      "cause": "Sustainable fashion",
      "mrp": 999,
      "sellingPrice": 799,
      "vendorID": 101,
      "media": ["https://cdn.example.com/image1.jpg"],
      "description": "Eco-friendly product",
      "productUrl": "https://shop.example.com/product",
      "publishingStatus": "draft",
      "createdAt": "2023-02-01T14:00:00Z",
      "updatedAt": "2023-02-01T14:00:00Z",
      "remarks": "Limited edition"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 25,
    "totalPages": 3
  }
}
```

#### 3. Get Products (Public)
```http
GET /v1/products/user
```

**Description**: Retrieves published products for users

**Authentication**: Required

**Query Parameters**:
- `causes` (optional): Filter by cause
- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 10)
- `search` (optional): Search in product titles
- `publishingStatus` (optional): Filter by status

**Example**:
```http
GET /v1/products/user?causes=Sustainable fashion&page=1&limit=10&search=bamboo
```

#### 4. Get Product by ID
```http
GET /v1/products/:id
```

**Description**: Retrieves a specific product by ID

**Authentication**: Required

**Path Parameters**:
- `id`: Product ID

**Response**:
```json
{
  "id": "IM_aB9X123",
  "title": "Eco-Friendly Bamboo Bottle",
  "cause": "Sustainable fashion",
  "mrp": 999,
  "sellingPrice": 799,
  "vendorID": 101,
  "media": ["https://cdn.example.com/image1.jpg"],
  "description": "Eco-friendly product",
  "productUrl": "https://shop.example.com/product",
  "publishingStatus": "published",
  "createdAt": "2023-02-01T14:00:00Z",
  "updatedAt": "2023-02-01T14:00:00Z"
}
```

#### 5. Update Product
```http
PATCH /v1/products/:id
```

**Description**: Updates an existing product

**Authentication**: Required

**Path Parameters**:
- `id`: Product ID

**Request Body**: Partial product data (same structure as create)

## Groups API

### Base Route: `/v1/groups`

#### 1. Create Group
```http
POST /v1/groups
```

**Description**: Creates a new chat group

**Authentication**: Required

**Request Body**:
```json
{
  "name": "Eco-friendly Products Group",
  "productId": "IM_aB9X123",
  "members": [101, 102, 103]
}
```

**Response**:
```json
{
  "message": "Group created successfully",
  "group": {
    "id": "b1d7d4ad-c87b-4a96-970f-e2740df3a2d9",
    "name": "Eco-friendly Products Group",
    "productId": "IM_aB9X123",
    "members": [101, 102, 103],
    "createdAt": "2023-02-01T14:00:00Z",
    "updatedAt": "2023-02-01T14:00:00Z"
  }
}
```

#### 2. Get or Create Group
```http
POST /v1/groups/get-or-create
```

**Description**: Retrieves existing group or creates new one with given members

**Authentication**: Required

**Request Body**: Same as create group

#### 3. Get Group by ID
```http
GET /v1/groups/:id
```

**Description**: Retrieves group details with last message

**Authentication**: Required

**Response**:
```json
{
  "id": "b1d7d4ad-c87b-4a96-970f-e2740df3a2d9",
  "name": "Eco-friendly Products Group",
  "productId": "IM_aB9X123",
  "members": [101, 102, 103],
  "createdAt": "2023-02-01T14:00:00Z",
  "updatedAt": "2023-02-01T14:00:00Z",
  "lastMessageData": {
    "id": "msg-uuid",
    "text": "Hello everyone!",
    "senderId": 101,
    "timestamp": "2023-02-01T14:30:00Z",
    "attachments": [],
    "readBy": [101]
  }
}
```

#### 4. Get Groups by User ID
```http
GET /v1/groups/user/:userId
```

**Description**: Retrieves all groups for a specific user

**Authentication**: Required

**Query Parameters**:
- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 10)

#### 5. Update Group
```http
PATCH /v1/groups/:id
```

**Description**: Updates group information

**Authentication**: Required

#### 6. Update Group Members
```http
PATCH /v1/groups/:id/members
```

**Description**: Updates group member list

**Authentication**: Required

**Request Body**:
```json
{
  "members": [101, 102, 103, 104]
}
```

#### 7. Get Unread Messages
```http
GET /v1/groups/unread-messages/:userId
```

**Description**: Retrieves unread message count for user

**Authentication**: Required

#### 8. Get Unread Messages by Group
```http
GET /v1/groups/unread-messages/:userId/:groupId
```

**Description**: Retrieves unread messages for specific group

**Authentication**: Required

## Chat API

### Base Route: `/v1/chat`

#### 1. Send Message
```http
POST /v1/chat/messages
```

**Description**: Sends a new message to a group

**Authentication**: Required

**Request Body**:
```json
{
  "senderId": 101,
  "groupId": "b1d7d4ad-c87b-4a96-970f-e2740df3a2d9",
  "text": "Hello everyone!",
  "attachments": [
    {
      "url": "https://example.com/image.jpg",
      "type": "image"
    }
  ]
}
```

#### 2. Get Messages
```http
GET /v1/chat/messages/:groupId
```

**Description**: Retrieves messages for a group

**Authentication**: Required

**Query Parameters**:
- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 20)

**Response**:
```json
[
  {
    "id": "msg-uuid",
    "senderId": 101,
    "groupId": "group-uuid",
    "text": "Hello everyone!",
    "attachments": [
      {
        "url": "https://example.com/image.jpg",
        "type": "image"
      }
    ],
    "readBy": [101, 102],
    "timestamp": "2023-02-01T14:30:00Z"
  }
]
```

#### 3. Get Group Members
```http
GET /v1/chat/group-members/:groupId
```

**Description**: Retrieves member list for a group

**Authentication**: Required

#### 4. Check User Online Status
```http
GET /v1/chat/is-online/:userId
```

**Description**: Checks if user is currently online

**Authentication**: Required

#### 5. Update Message
```http
PATCH /v1/chat/messages/:messageId
```

**Description**: Updates message (typically for read receipts)

**Authentication**: Required

## File Upload API

### Base Route: `/v1/s3`

#### 1. Upload Single File
```http
POST /v1/s3/upload/single
```

**Description**: Uploads a single file to S3

**Authentication**: Required

**Content-Type**: `multipart/form-data`

**Query Parameters**:
- `folderPath` (optional): S3 folder path (default: 'misc')

**Request**: Form data with file

**Response**:
```json
{
  "success": true,
  "url": "https://bucket.s3.region.amazonaws.com/folder/filename.jpg"
}
```

#### 2. Upload Multiple Files
```http
POST /v1/s3/upload/multiple
```

**Description**: Uploads multiple files to S3

**Authentication**: Required

**Content-Type**: `multipart/form-data`

**Response**:
```json
{
  "success": true,
  "urls": [
    "https://bucket.s3.region.amazonaws.com/folder/file1.jpg",
    "https://bucket.s3.region.amazonaws.com/folder/file2.jpg"
  ]
}
```

#### 3. Get Signed URL
```http
GET /v1/s3/signed-url
```

**Description**: Generates a signed URL for secure file access

**Authentication**: Required

**Query Parameters**:
- `filePath` (required): S3 file path

**Response**:
```json
{
  "success": true,
  "url": "https://bucket.s3.region.amazonaws.com/file.jpg?signature=..."
}
```

## File Upload Constraints

### File Size Limits
- **Images**: 5MB maximum
- **Videos**: 200MB maximum
- **Audio**: 10MB maximum
- **Documents**: 20MB maximum
- **General Files**: 20MB maximum

### Supported File Types
- **Images**: JPEG, PNG, GIF, WebP
- **Videos**: MP4, AVI, MOV, WebM
- **Audio**: MP3, WAV, AAC, OGG
- **Documents**: PDF, DOC, DOCX, TXT

## Error Responses

### Standard Error Format
```json
{
  "statusCode": 400,
  "message": "Validation failed",
  "error": "Bad Request",
  "details": [
    {
      "field": "title",
      "message": "title should not be empty"
    }
  ]
}
```

### Common HTTP Status Codes
- **200**: Success
- **201**: Created
- **400**: Bad Request (validation errors)
- **401**: Unauthorized (invalid/missing token)
- **403**: Forbidden (insufficient permissions)
- **404**: Not Found
- **500**: Internal Server Error

## Rate Limiting

The API implements rate limiting to prevent abuse:
- **General endpoints**: 100 requests per minute
- **File upload**: 10 requests per minute
- **Authentication**: 5 requests per minute

## Pagination

All list endpoints support pagination:

**Query Parameters**:
- `page`: Page number (starts from 1)
- `limit`: Items per page (max 100)

**Response Format**:
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100,
    "totalPages": 10
  }
}
```
# Database Architecture

## Overview

The Deed-O Backend uses a dual-database architecture with PostgreSQL as the primary database system. The application leverages Drizzle ORM for type-safe database operations and migrations.

## Database Configuration

### Connection Setup
- **ORM**: Drizzle ORM
- **Database**: PostgreSQL
- **Driver**: postgres-js
- **Configuration File**: `drizzle.config.ts`

### Database Connections

1. **Main Database** (`DATABASE_URL`)
   - Products, groups, messages
   - Business logic data

2. **Auth Database** (`AUTH_DATABASE_URL`)
   - User authentication
   - Tokens and verifications
   - User profiles

## Schema Structure

### Main Database Schema (`schema.ts`)

#### Products Table
```sql
CREATE TABLE products (
  id VARCHAR PRIMARY KEY,
  cause causes_enum NOT NULL,
  mrp DOUBLE PRECISION NOT NULL,
  selling_price DOUBLE PRECISION NOT NULL,
  vendor_id INTEGER REFERENCES users(id) NOT NULL,
  images JSON NOT NULL,
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  product_url TEXT NOT NULL,
  publishing_status publishing_status_enum DEFAULT 'draft' NOT NULL,
  created_at TIMESTAMP DEFAULT NOW() NOT NULL,
  updated_at TIMESTAMP DEFAULT NOW() NOT NULL,
  remark TEXT
);
```

**Fields:**
- `id`: Unique product identifier (auto-generated with "IM_" prefix)
- `cause`: Product category from predefined causes
- `mrp`: Maximum Retail Price
- `sellingPrice`: Actual selling price
- `vendorID`: Reference to vendor user
- `media`: JSON array of image URLs
- `title`: Product name
- `description`: Product description
- `productUrl`: External product URL
- `publishingStatus`: Workflow status
- `remarks`: Optional internal notes

#### Enums for Products

**Causes Enum:**
- Grief care
- Sustainable fashion
- Sustainable tourism
- Intangible culture and heritage
- Sustainable food and agriculture

**Publishing Status Enum:**
- published
- draft
- suggestion
- rejected

### Chat Database Schema (`chat-schema.ts`)

#### Groups Table
```sql
CREATE TABLE groups (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  product_id VARCHAR REFERENCES products(id),
  members INTEGER[] DEFAULT '{}',
  last_message_id UUID REFERENCES messages(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Fields:**
- `id`: Unique group identifier (UUID)
- `name`: Group display name
- `productId`: Associated product reference
- `members`: Array of user IDs
- `lastMessageId`: Reference to most recent message

#### Messages Table
```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  sender_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  group_id UUID NOT NULL REFERENCES groups(id),
  text TEXT,
  attachments JSONB DEFAULT '[]',
  read_by INTEGER[] DEFAULT '{}',
  timestamp TIMESTAMP DEFAULT NOW()
);
```

**Fields:**
- `id`: Unique message identifier (UUID)
- `senderId`: Message sender reference
- `groupId`: Target group reference
- `text`: Message content
- `attachments`: JSON array of file attachments
- `readBy`: Array of user IDs who read the message
- `timestamp`: Message creation time

**Attachment Structure:**
```json
{
  "url": "string",
  "type": "file|image|video|audio"
}
```

### Auth Database Schema (`auth-schema.ts`)

#### Users Table
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
  purpose_code VARCHAR,
  nationality VARCHAR,
  ethnicity VARCHAR,
  qualification VARCHAR,
  profession VARCHAR,
  role users_roles_enum DEFAULT 'user' NOT NULL,
  is_online BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW() NOT NULL,
  updated_at TIMESTAMP DEFAULT NOW() NOT NULL,
  deleted_at TIMESTAMP
);
```

**User Roles:**
- moderator
- admin
- user

**User Statuses:**
- inactive
- verified
- not-verified

**Platform Types:**
- dobe
- cause-i
- deed-o

#### Tokens Table
```sql
CREATE TABLE tokens (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE NOT NULL,
  expire_at TIMESTAMP WITH TIME ZONE NOT NULL,
  platform_type platform_types_enum NOT NULL
);
```

#### User Verifications Table
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

**Verification Actions:**
- login
- register

## Database Operations

### Migration Management

**Commands:**
```bash
# Generate migrations
pnpm run migration:generate

# Push migrations to database
pnpm run migration:push

# Run all migrations
pnpm run migration:all

# Database introspection
pnpm run introspect
```

### Seeding
```bash
# Populate database with initial data
pnpm run seed
```

## Relationships

### Primary Relationships

1. **Users → Products**: One-to-Many (vendor relationship)
2. **Products → Groups**: One-to-Many (product-based groups)
3. **Groups → Messages**: One-to-Many (group conversations)
4. **Users → Messages**: One-to-Many (message authorship)
5. **Users → Tokens**: One-to-Many (authentication tokens)
6. **Users → Verifications**: One-to-Many (verification records)

### Cross-Database Relationships

- Products reference Users across databases
- Groups reference Products and Users across databases
- Messages reference Users across databases

## Indexing Strategy

### Existing Indexes

1. **User Verifications**: Composite index on `(id, user_id, otp_code)`
2. **Users**: Unique index on `email`
3. **Primary Keys**: Automatic indexes on all primary keys

### Recommended Additional Indexes

```sql
-- Products performance
CREATE INDEX idx_products_vendor_id ON products(vendor_id);
CREATE INDEX idx_products_cause ON products(cause);
CREATE INDEX idx_products_publishing_status ON products(publishing_status);
CREATE INDEX idx_products_created_at ON products(created_at);

-- Groups performance
CREATE INDEX idx_groups_product_id ON groups(product_id);
CREATE INDEX idx_groups_members ON groups USING GIN(members);

-- Messages performance
CREATE INDEX idx_messages_group_id ON messages(group_id);
CREATE INDEX idx_messages_sender_id ON messages(sender_id);
CREATE INDEX idx_messages_timestamp ON messages(timestamp);
```

## Data Types & Constraints

### Custom Types
- **UUID**: Used for groups and messages
- **JSONB**: Used for message attachments
- **Arrays**: Used for group members and read receipts
- **Enums**: Used for status fields and categories

### Constraints
- **Foreign Keys**: Maintain referential integrity
- **Unique Constraints**: Prevent duplicate emails
- **Not Null**: Ensure required fields
- **Default Values**: Provide sensible defaults

## Performance Considerations

1. **Connection Pooling**: Efficient database connections
2. **Pagination**: Implemented in services for large datasets
3. **Selective Queries**: Only fetch required fields
4. **Proper Indexing**: Strategic index placement
5. **Array Operations**: Efficient PostgreSQL array handling

## Backup & Recovery

### Recommended Backup Strategy
1. **Daily Backups**: Automated daily database dumps
2. **Point-in-Time Recovery**: WAL archiving
3. **Cross-Region Replication**: For disaster recovery
4. **Testing**: Regular backup restoration testing

## Security Considerations

1. **Connection Security**: SSL/TLS encryption
2. **Access Control**: Role-based database permissions
3. **Sensitive Data**: Proper handling of PII
4. **Audit Logging**: Track database changes
5. **Environment Separation**: Isolated database environments
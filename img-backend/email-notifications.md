# Email Notifications

## Overview

The Deed-O Backend implements a comprehensive email notification system using AWS SES (Simple Email Service) with React-based email templates. The system sends notifications for various events, primarily focusing on chat messages for offline users.

## Architecture

### Components

1. **Email Service** (`email.service.ts`) - AWS SES integration
2. **Email Templates** (`email-templates.ts`) - React-based templates
3. **Chat Integration** - Automatic notifications for offline users
4. **Template Rendering** - Server-side React rendering

### Email Flow

```
User sends message → Check recipient online status → If offline → Generate email → Send via SES
```

## Email Service

### File: `src/email.service.ts`

#### AWS SES Configuration

```typescript
import { SESv2Client, SendEmailCommand } from '@aws-sdk/client-sesv2';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

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
}
```

**Configuration Requirements**:
- `AWS_REGION`: AWS region for SES (e.g., 'us-east-1')
- `AWS_ACCESS_KEY_ID`: AWS access key with SES permissions
- `AWS_SECRET_ACCESS_KEY`: AWS secret access key
- `SENDER_EMAIL`: Verified sender email address

#### Send Email Method

```typescript
async sendEmail(
  to: string,
  subject: string,
  body: string,
  from?: string
): Promise<any> {
  try {
    const input = {
      FromEmailAddress: from || this.configService.get<string>('SENDER_EMAIL'),
      Destination: {
        ToAddresses: [to]
      },
      Content: {
        Simple: {
          Subject: {
            Data: subject,
            Charset: 'UTF-8'
          },
          Body: {
            Html: {
              Charset: 'UTF-8',
              Data: body
            }
          }
        }
      }
    };

    const command = new SendEmailCommand(input);
    const result = await this.ses.send(command);
    
    console.log('Email sent successfully:', {
      messageId: result.MessageId,
      to,
      subject
    });
    
    return result;
  } catch (error) {
    console.error('Failed to send email:', {
      error: error.message,
      to,
      subject
    });
    throw error;
  }
}
```

**Features**:
- HTML email support
- UTF-8 character encoding
- Configurable sender address
- Detailed logging
- Error handling
- AWS SES v2 API

## Email Templates

### File: `src/email-templates.ts`

#### React Email Integration

```typescript
import { render } from '@react-email/render';
import React from 'react';

// Email template component
const NewMessage: React.FC<{
  message: string;
  recipientName: string;
  senderName?: string;
}> = ({ message, recipientName, senderName }) => {
  return (
    <html>
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <title>New Message</title>
        <style>{`
          body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            line-height: 1.6;
            color: #333;
            max-width: 600px;
            margin: 0 auto;
            padding: 20px;
            background-color: #f4f4f4;
          }
          .container {
            background-color: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
          }
          .header {
            text-align: center;
            margin-bottom: 30px;
            padding-bottom: 20px;
            border-bottom: 2px solid #e0e0e0;
          }
          .logo {
            font-size: 24px;
            font-weight: bold;
            color: #2c3e50;
            margin-bottom: 10px;
          }
          .greeting {
            font-size: 18px;
            margin-bottom: 20px;
            color: #2c3e50;
          }
          .message-content {
            background-color: #f8f9fa;
            padding: 20px;
            border-radius: 8px;
            border-left: 4px solid #007bff;
            margin: 20px 0;
          }
          .sender-info {
            font-weight: bold;
            color: #007bff;
            margin-bottom: 10px;
          }
          .message-text {
            font-size: 16px;
            line-height: 1.5;
          }
          .footer {
            margin-top: 30px;
            padding-top: 20px;
            border-top: 1px solid #e0e0e0;
            text-align: center;
            color: #666;
            font-size: 14px;
          }
          .cta-button {
            display: inline-block;
            background-color: #007bff;
            color: white;
            padding: 12px 24px;
            text-decoration: none;
            border-radius: 5px;
            margin: 20px 0;
            font-weight: bold;
          }
        `}</style>
      </head>
      <body>
        <div className="container">
          <div className="header">
            <div className="logo">Deed-O</div>
            <div>You have a new message</div>
          </div>
          
          <div className="greeting">
            Hi {recipientName},
          </div>
          
          <p>You received a new message while you were away.</p>
          
          <div className="message-content">
            {senderName && (
              <div className="sender-info">
                From: {senderName}
              </div>
            )}
            <div className="message-text">
              {message}
            </div>
          </div>
          
          <div style={{ textAlign: 'center' }}>
            <a href="#" className="cta-button">
              Open Deed-O App
            </a>
          </div>
          
          <div className="footer">
            <p>This email was sent because you have notifications enabled for Deed-O.</p>
            <p>© 2024 Deed-O. All rights reserved.</p>
          </div>
        </div>
      </body>
    </html>
  );
};
```

#### Template Rendering Function

```typescript
export const newMessageTemplate = (
  message: string,
  recipientName: string,
  senderName?: string
): string => {
  return render(
    React.createElement(NewMessage, {
      message,
      recipientName,
      senderName
    })
  );
};
```

#### Email Subject Generation

```typescript
export const NEW_MESSAGE_EMAIL_SUBJECT = (senderName?: string): string => {
  if (senderName) {
    return `New message from ${senderName} - Deed-O`;
  }
  return 'You have a new message - Deed-O';
};
```

**Template Features**:
- Responsive design
- Modern styling
- Branded appearance
- Dynamic content insertion
- Call-to-action button
- Professional footer
- Cross-client compatibility

## Chat Integration

### File: `src/chat.service.ts`

#### Email Notification Logic

```typescript
async sendEmailNotification(
  receiverId: number,
  senderId: number,
  message: string
): Promise<void> {
  try {
    // Get receiver details from auth database
    const [receiver] = await authDb
      .select({
        email: users.email,
        name: users.name
      })
      .from(users)
      .where(eq(users.id, receiverId))
      .execute();

    // Get sender details
    const [sender] = await authDb
      .select({
        name: users.name
      })
      .from(users)
      .where(eq(users.id, senderId))
      .execute();

    // Send email if receiver has email address
    if (receiver?.email) {
      const emailBody = newMessageTemplate(
        message,
        receiver.name || 'User',
        sender?.name
      );
      
      const subject = NEW_MESSAGE_EMAIL_SUBJECT(sender?.name);
      
      await this.emailService.sendEmail(
        receiver.email,
        subject,
        emailBody
      );
      
      console.log(`Email notification sent to ${receiver.email}`);
    } else {
      console.log(`No email address found for user ${receiverId}`);
    }
  } catch (error) {
    console.error('Failed to send email notification:', {
      error: error.message,
      receiverId,
      senderId
    });
    // Don't throw error to avoid breaking message flow
  }
}
```

**Integration Features**:
- Database user lookup
- Template rendering
- Subject line generation
- Error handling without breaking chat
- Detailed logging

### WebSocket Integration

**File**: `src/chat.gateway.ts`

```typescript
@SubscribeMessage('sendMessage')
async handleMessage(client: Socket, payload: MessagePayload) {
  try {
    // Create message in database
    const message = await this.chatService.createMessage(messageData);

    // Broadcast to online group members
    this.server.to(payload.groupId).emit('newMessage', messageData);

    // Send email notifications to offline members
    await this.sendEmailNotifications(
      payload.groupId,
      userId,
      payload.text
    );
  } catch (error) {
    console.error('Error handling message:', error);
  }
}

private async sendEmailNotifications(
  groupId: string,
  senderId: number,
  messageText: string
) {
  try {
    // Get group members
    const group = await this.groupService.getGroupById(groupId);
    if (!group) return;

    // Filter offline members (not in onlineUsers map)
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
    
    console.log(`Email notifications sent to ${offlineMembers.length} offline members`);
  } catch (error) {
    console.error('Error sending email notifications:', error);
  }
}
```

**Real-time Integration**:
- Automatic offline user detection
- Parallel email sending
- Non-blocking email operations
- Comprehensive error handling

## Email Configuration

### Environment Variables

```bash
# AWS SES Configuration
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
SENDER_EMAIL=noreply@yourdomain.com

# Email Settings
EMAIL_ENABLED=true
EMAIL_RATE_LIMIT=100  # emails per hour
```

### AWS SES Setup

#### 1. Verify Sender Email

```bash
# Using AWS CLI
aws ses verify-email-identity --email-address noreply@yourdomain.com
```

#### 2. Domain Verification (Production)

```bash
# Verify entire domain
aws ses verify-domain-identity --domain yourdomain.com
```

#### 3. IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ses:SendEmail",
        "ses:SendRawEmail"
      ],
      "Resource": "*"
    }
  ]
}
```

## Email Templates

### Template Types

#### 1. New Message Template

**Purpose**: Notify users of new chat messages

**Variables**:
- `message`: The message content
- `recipientName`: Recipient's display name
- `senderName`: Sender's display name (optional)

**Usage**:
```typescript
const emailBody = newMessageTemplate(
  "Hello! How are you doing?",
  "John Doe",
  "Jane Smith"
);
```

#### 2. Welcome Email Template (Future)

```typescript
const WelcomeEmail: React.FC<{ userName: string }> = ({ userName }) => {
  return (
    <html>
      <body>
        <h1>Welcome to Deed-O, {userName}!</h1>
        <p>Thank you for joining our community...</p>
      </body>
    </html>
  );
};
```

#### 3. Password Reset Template (Future)

```typescript
const PasswordResetEmail: React.FC<{ 
  userName: string;
  resetLink: string;
}> = ({ userName, resetLink }) => {
  return (
    <html>
      <body>
        <h1>Password Reset Request</h1>
        <p>Hi {userName}, click the link below to reset your password:</p>
        <a href={resetLink}>Reset Password</a>
      </body>
    </html>
  );
};
```

### Template Best Practices

#### 1. Responsive Design

```css
@media only screen and (max-width: 600px) {
  .container {
    padding: 15px !important;
  }
  
  .message-content {
    padding: 15px !important;
  }
}
```

#### 2. Cross-Client Compatibility

```css
/* Use table-based layouts for better compatibility */
table {
  border-collapse: collapse;
  mso-table-lspace: 0pt;
  mso-table-rspace: 0pt;
}
```

#### 3. Accessibility

```html
<!-- Alt text for images -->
<img src="logo.png" alt="Deed-O Logo" />

<!-- High contrast colors -->
<style>
  .text { color: #333333; }
  .link { color: #0066cc; }
</style>
```

## Error Handling

### Email Service Errors

```typescript
try {
  await this.emailService.sendEmail(to, subject, body);
} catch (error) {
  if (error.name === 'MessageRejected') {
    console.error('Email rejected by SES:', error.message);
  } else if (error.name === 'SendingQuotaExceeded') {
    console.error('SES sending quota exceeded');
  } else {
    console.error('Unknown email error:', error);
  }
  
  // Don't throw error to avoid breaking application flow
}
```

### Common Error Scenarios

#### 1. Invalid Email Address

```json
{
  "error": "MessageRejected",
  "message": "Email address not verified"
}
```

#### 2. Rate Limit Exceeded

```json
{
  "error": "Throttling",
  "message": "Rate exceeded"
}
```

#### 3. Template Rendering Error

```typescript
try {
  const emailBody = newMessageTemplate(message, recipientName, senderName);
} catch (error) {
  console.error('Template rendering failed:', error);
  // Use fallback plain text email
  const emailBody = `New message from ${senderName}: ${message}`;
}
```

## Performance Optimization

### Batch Email Sending

```typescript
async sendBatchEmails(emails: EmailData[]): Promise<void> {
  const batchSize = 10;
  
  for (let i = 0; i < emails.length; i += batchSize) {
    const batch = emails.slice(i, i + batchSize);
    
    const promises = batch.map(email => 
      this.sendEmail(email.to, email.subject, email.body)
    );
    
    await Promise.allSettled(promises);
    
    // Add delay between batches to respect rate limits
    if (i + batchSize < emails.length) {
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}
```

### Template Caching

```typescript
class TemplateCache {
  private cache = new Map<string, string>();
  
  getTemplate(key: string, generator: () => string): string {
    if (!this.cache.has(key)) {
      this.cache.set(key, generator());
    }
    return this.cache.get(key)!;
  }
}

const templateCache = new TemplateCache();

// Use cached template
const emailBody = templateCache.getTemplate(
  `newMessage_${recipientName}_${senderName}`,
  () => newMessageTemplate(message, recipientName, senderName)
);
```

### Queue System (Future Enhancement)

```typescript
import Bull from 'bull';

const emailQueue = new Bull('email queue', {
  redis: {
    port: 6379,
    host: 'localhost'
  }
});

// Add email to queue
emailQueue.add('sendEmail', {
  to: 'user@example.com',
  subject: 'New Message',
  body: emailBody
});

// Process emails
emailQueue.process('sendEmail', async (job) => {
  const { to, subject, body } = job.data;
  await this.emailService.sendEmail(to, subject, body);
});
```

## Monitoring & Analytics

### Email Metrics

```typescript
class EmailMetrics {
  private metrics = {
    sent: 0,
    failed: 0,
    bounced: 0,
    opened: 0,
    clicked: 0
  };
  
  recordSent() {
    this.metrics.sent++;
  }
  
  recordFailed() {
    this.metrics.failed++;
  }
  
  getMetrics() {
    return { ...this.metrics };
  }
}
```

### SES Event Tracking

```typescript
// Configure SES to send events to SNS/SQS
const configurationSet = {
  Name: 'deed-o-emails',
  EventDestinations: [{
    Name: 'cloudwatch-destination',
    Enabled: true,
    MatchingEventTypes: [
      'send', 'bounce', 'complaint', 'delivery', 'open', 'click'
    ],
    CloudWatchDestination: {
      DimensionConfigurations: [{
        DimensionName: 'EmailAddress',
        DimensionValueSource: 'emailHeader'
      }]
    }
  }]
};
```

### Logging

```typescript
const emailLogger = {
  logEmailSent: (to: string, subject: string, messageId: string) => {
    console.log({
      event: 'email_sent',
      to,
      subject,
      messageId,
      timestamp: new Date().toISOString()
    });
  },
  
  logEmailFailed: (to: string, subject: string, error: string) => {
    console.error({
      event: 'email_failed',
      to,
      subject,
      error,
      timestamp: new Date().toISOString()
    });
  }
};
```

## Testing

### Unit Tests

```typescript
describe('EmailService', () => {
  let service: EmailService;
  let mockSESClient: jest.Mocked<SESv2Client>;

  beforeEach(() => {
    mockSESClient = {
      send: jest.fn()
    } as any;
    
    service = new EmailService(configService);
    (service as any).ses = mockSESClient;
  });

  it('should send email successfully', async () => {
    mockSESClient.send.mockResolvedValue({
      MessageId: 'test-message-id'
    });

    const result = await service.sendEmail(
      'test@example.com',
      'Test Subject',
      '<p>Test Body</p>'
    );

    expect(result.MessageId).toBe('test-message-id');
    expect(mockSESClient.send).toHaveBeenCalledWith(
      expect.objectContaining({
        input: expect.objectContaining({
          Destination: { ToAddresses: ['test@example.com'] }
        })
      })
    );
  });
});
```

### Template Tests

```typescript
describe('Email Templates', () => {
  it('should render new message template', () => {
    const html = newMessageTemplate(
      'Hello World',
      'John Doe',
      'Jane Smith'
    );
    
    expect(html).toContain('Hello World');
    expect(html).toContain('John Doe');
    expect(html).toContain('Jane Smith');
    expect(html).toContain('From: Jane Smith');
  });
  
  it('should handle missing sender name', () => {
    const html = newMessageTemplate(
      'Hello World',
      'John Doe'
    );
    
    expect(html).toContain('Hello World');
    expect(html).toContain('John Doe');
    expect(html).not.toContain('From:');
  });
});
```

### Integration Tests

```typescript
describe('Email Integration', () => {
  it('should send notification when user is offline', async () => {
    // Mock user as offline
    jest.spyOn(chatService, 'isUserOnline').mockResolvedValue(false);
    
    // Send message
    await chatGateway.handleMessage(mockSocket, {
      groupId: 'test-group',
      text: 'Test message',
      attachments: []
    });
    
    // Verify email was sent
    expect(emailService.sendEmail).toHaveBeenCalledWith(
      expect.any(String),
      expect.stringContaining('New message'),
      expect.stringContaining('Test message')
    );
  });
});
```

## Security Considerations

### Email Content Sanitization

```typescript
import DOMPurify from 'dompurify';

const sanitizeMessage = (message: string): string => {
  // Remove potentially dangerous HTML
  return DOMPurify.sanitize(message, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p', 'br'],
    ALLOWED_ATTR: []
  });
};

// Use in template
const emailBody = newMessageTemplate(
  sanitizeMessage(message),
  recipientName,
  senderName
);
```

### Rate Limiting

```typescript
class EmailRateLimiter {
  private userLimits = new Map<number, number[]>();
  
  canSendEmail(userId: number): boolean {
    const now = Date.now();
    const userEmails = this.userLimits.get(userId) || [];
    
    // Remove emails older than 1 hour
    const recentEmails = userEmails.filter(
      timestamp => now - timestamp < 3600000
    );
    
    if (recentEmails.length >= 10) { // 10 emails per hour
      return false;
    }
    
    recentEmails.push(now);
    this.userLimits.set(userId, recentEmails);
    return true;
  }
}
```

### Data Privacy

```typescript
// Don't log sensitive information
const logEmailEvent = (event: string, to: string, subject: string) => {
  console.log({
    event,
    to: to.replace(/(.{2}).*(@.*)/, '$1***$2'), // Mask email
    subject,
    timestamp: new Date().toISOString()
  });
};
```
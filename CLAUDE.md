# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Evolution API Overview

Evolution API is a comprehensive WhatsApp and multi-messaging platform API built with Node.js, TypeScript, and Express.js. It provides WhatsApp Web automation via Baileys, official WhatsApp Business API support, and integrations with multiple chatbot platforms.

## Common Commands

### Development
```bash
npm run dev:server          # Start development server with hot reload
npm run start               # Production start (tsx runtime)
npm run start:prod          # Production start (compiled JS)
npm run build               # TypeScript compilation + tsup bundling
npm run lint                # ESLint with auto-fix
npm run lint:check          # ESLint check only
npm run test                # Run tests
```

### Database Operations
```bash
npm run db:generate         # Generate Prisma client for current provider
npm run db:deploy           # Deploy migrations to database
npm run db:migrate:dev      # Create and apply development migrations
npm run db:studio           # Launch Prisma Studio
```

### Docker
```bash
docker build -t evolution-api .
docker-compose up -d        # Start full stack (API + PostgreSQL + Redis)
```

## Architecture Overview

### Core Structure
- **Controllers** (`/src/api/controllers/`): HTTP request handlers
- **Services** (`/src/api/services/`): Business logic implementation  
- **Repository** (`/src/api/repository/`): Database abstraction via Prisma
- **DTOs** (`/src/api/dto/`): Request/response validation objects
- **Guards** (`/src/api/guards/`): Authentication and authorization middleware
- **Abstract** (`/src/api/abstract/`): Base classes for common functionality

### Integration Plugin System
The application uses modular plugins for external integrations:

- **Chatbots**: `/src/api/integrations/chatbot/`
  - Typebot, Chatwoot, OpenAI, Dify, N8N, Flowise, Evolution Bot
  - All extend `BaseChatbotService` abstract class
  - Session management via `IntegrationSession` entity

- **Channels**: `/src/api/integrations/channel/`
  - WhatsApp (Baileys), Meta WhatsApp Business API
  - Each channel has its own service implementation

- **Events**: `/src/api/integrations/event/`
  - WebSocket, Webhook, RabbitMQ, SQS, NATS, Pusher
  - Central `EventManager` coordinates all event routing

### Database Architecture

**Multi-Provider Support:**
- PostgreSQL, MySQL, PostgreSQL with PGBouncer
- Provider-specific schema files: `prisma/postgresql-schema.prisma`, `prisma/mysql-schema.prisma`
- Use `runWithProvider.js` script to run commands with correct provider

**Key Entities:**
- `Instance`: WhatsApp connection instances
- `Message`, `Chat`, `Contact`: Core messaging entities
- `Webhook`, `Chatwoot`, `Typebot`, etc.: Integration configurations
- `IntegrationSession`: Chatbot conversation sessions

### Configuration Management

Environment configuration in `src/config/env.config.ts`:
- 100+ configuration options via environment variables
- Type-safe configuration with runtime validation
- Instance-level and global settings

Key config areas:
- Server settings (HTTP/HTTPS, CORS, logging)
- Database provider selection and connection
- Individual integration toggles and settings
- Event routing configuration
- Authentication methods

### Instance Management

Each WhatsApp connection is an "instance" with:
- Isolated authentication and session data
- Configurable storage backends (Files, Redis, Database)
- Auto-cleanup based on expiration settings
- Dynamic creation/deletion via API

### Authentication & Security

**Multi-layer authentication:**
- `authGuard`: Global API key validation
- `instanceGuard`: Instance-specific access control
- JWT support for advanced authentication
- CORS configuration per instance

### Event-Driven Architecture

**Flow:**
1. WhatsApp events from Baileys → Event Manager
2. Event Manager processes and filters events
3. Events routed to multiple destinations simultaneously:
   - Webhooks (HTTP POST)
   - WebSocket connections
   - Message queues (RabbitMQ, SQS, NATS)
   - Chatbot integrations

### Error Handling Patterns

- Custom exception classes: `400Exception`, `404Exception`, etc.
- Global Express async error handler
- Sentry integration for production error tracking
- Webhook error notifications via events

## Development Guidelines

### Adding New Integrations

**For Chatbots:**
1. Extend `BaseChatbotService` in `/src/api/integrations/chatbot/[name]/`
2. Implement required abstract methods: `sendMessage`, `receiveMessage`, etc.
3. Create corresponding DTO, controller, and route files
4. Add database entity if persistent settings needed

**For Events:**
1. Implement event handler in `/src/api/integrations/event/[name]/`
2. Register with EventManager in main application
3. Add configuration options to environment config

### Database Changes

Always use the provider-specific approach:
```bash
# Generate client for your database provider
npm run db:generate

# Create migration (automatically detects provider from env)
npm run db:migrate:dev

# Deploy to production
npm run db:deploy
```

### Path Aliases

The project uses extensive path mapping:
```typescript
import { ConfigService } from '@config/env.config';
import { InstanceDto } from '@api/dto/instance.dto';
import { PrismaRepository } from '@api/repository/prisma.repository';
```

### Message Processing Flow

1. **Incoming Messages**: Baileys → WhatsApp Service → Event Manager
2. **Chatbot Processing**: Session lookup → Integration service → AI/rule processing
3. **Outgoing Messages**: Service → WhatsApp instance → Baileys
4. **Event Broadcasting**: All message events broadcast to configured endpoints

### Testing Strategy

- Integration tests in `/test/` directory
- Test individual instances and integrations
- Use test environment configuration
- Docker Compose for test database setup

## Key Dependencies

- **baileys**: WhatsApp Web API implementation
- **@prisma/client**: Database ORM with type safety
- **socket.io**: WebSocket server for real-time events
- **express**: HTTP server framework
- **jsonschema**: Request/response validation
- **redis**: Caching and session storage
- **amqplib**: RabbitMQ message queue integration

## Docker Deployment

The application uses multi-stage Docker builds:
- Builder stage: Install deps, compile TypeScript, build bundle
- Runtime stage: Minimal Alpine Linux with Node.js 20
- Supports both PostgreSQL and MySQL via environment variables

Environment variables control all aspects of deployment including database provider, Redis connection, integration settings, and security configuration.

## Applied Patches

### JID Routing Fix for Chatwoot Integration

**Context:** This repository includes a fix for proper message routing between Chatwoot and WhatsApp that prioritizes JID identifiers over phone numbers.

**File:** `src/api/integrations/chatbot/chatwoot/services/chatwoot.service.ts`

**Problem:** The original `jid-routing.patch` file was corrupted (invalid context diff format), so changes must be applied manually.

**Changes Applied:**

1. **Added `getRoutableIdFromBody()` method** (around line 50):
```typescript
/**
 * Retorna o identificador roteável, priorizando JID (ex.: ...@lid, ...@s.whatsapp.net, ...@g.us).
 * Fallback para phone_number (sem '+') apenas se JID não existir.
 */
private getRoutableIdFromBody(body: any): string {
  const sourceId = body?.conversation?.contact_inbox?.source_id;
  if (sourceId && /@(?:lid|s\.whatsapp\.net|g\.us)$/.test(sourceId)) return sourceId;
  const identifier = body?.conversation?.meta?.sender?.identifier;
  if (identifier && /@(?:lid|s\.whatsapp\.net|g\.us)$/.test(identifier)) return identifier;
  const phone = body?.conversation?.meta?.sender?.phone_number?.replace('+', '');
  return phone;
}
```

2. **Updated chatId assignment** (around line 1300):
```typescript
// 🔁 Chatwoot → Evolution: sempre priorizar JID
const chatId = this.getRoutableIdFromBody(body);
```

3. **Enhanced comments for Chatwoot → WhatsApp formatting** (around line 1302):
```typescript
// ✅ Conversão de formatação apenas para Chatwoot → WhatsApp
const messageReceived = body.content
  ? body.content
      .replaceAll(/(?<!\*)\*((?!\s)([^\n*]+?)(?<!\s))\*(?!\*)/g, '_$1_')    // *texto*  → _texto_
      .replaceAll(/\*{2}((?!\s)([^\n*]+?)(?<!\s))\*{2}/g, '*$1*')          // **texto** → *texto*
      .replaceAll(/~{2}((?!\s)([^\n*]+?)(?<!\s))~{2}/g, '~$1~')            // ~~texto~~ → ~texto~
      .replaceAll(/(?<!`)`((?!\s)([^`*]+?)(?<!\s))`(?!`)/g, '```$1```')    // `codigo` → ```codigo```
  : body.content;
```

4. **Enhanced comments for WhatsApp → Chatwoot formatting** (around line 1998):
```typescript
// ✅ WhatsApp → Chatwoot: converte marcas do WA para markdown do Chatwoot
const originalMessage = await this.getConversationMessage(body.message);
const bodyMessage = originalMessage
  ? originalMessage
      .replaceAll(/\*((?!\s)([^\n*]+?)(?<!\s))\*/g, '**$1**')  // *texto*  → **texto**
      .replaceAll(/_((?!\s)([^\n_]+?)(?<!\s))_/g, '*$1*')      // _texto_  → *texto*
      .replaceAll(/~((?!\s)([^\n~]+?)(?<!\s))~/g, '~~$1~~')    // ~texto~  → ~~texto~~
  : originalMessage;
```

**Purpose:** Ensures consistent conversation routing by prioritizing WhatsApp JID identifiers (`@lid`, `@s.whatsapp.net`, `@g.us`) over phone numbers, preventing conversation fragmentation between Chatwoot and WhatsApp.

**To Reapply:** When updating from the official Evolution API repository, manually apply these four changes above. The original patch file `jid-routing.patch` is corrupted and cannot be applied with `git apply`.
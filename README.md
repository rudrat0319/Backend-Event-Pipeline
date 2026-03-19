#Backend - Event-Driven Learning Platform

A robust, event-driven backend system for processing mission completions, updating learning metrics, and triggering automated workflows for a gamified learning platform.

## Architecture Overview

This system implements a complete event-driven pipeline:
1. REST API receives mission completion data
2. Atomic database transaction updates metrics
3. Event published to RabbitMQ queue
4. Worker processes event and triggers n8n webhook
5. n8n workflow handles conditional logic (email alerts, risk tracking)

## Tech Stack

- **Runtime**: Node.js with TypeScript
- **Database**: PostgreSQL (Supabase)
- **ORM**: TypeORM
- **Message Queue**: RabbitMQ
- **Automation**: n8n webhooks

## Prerequisites

- Node.js 18+ and npm
- PostgreSQL database (Supabase recommended)
- RabbitMQ (Docker recommended)
- n8n instance (optional for full workflow)

## Setup Instructions

### 1. Install Dependencies

```bash
cd Backend
npm install
```

### 2. Configure Environment Variables

Create or update `.env` file:

```env
# Database Configuration
DATABASE_URL=postgres://postgres.[YOUR_ID]:[PASSWORD]@aws-0-pooler.supabase.com:6543/postgres

# RabbitMQ Configuration
RABBITMQ_URL=amqp://localhost:5672

# n8n Webhook Configuration
N8N_WEBHOOK_URL=https://your-n8n-instance.com/webhook/mission-complete

# Server Configuration
PORT=3000
NODE_ENV=development
```

### 3. Start RabbitMQ (Docker)

```bash
docker-compose up -d
```

This starts RabbitMQ with management UI at http://localhost:15672 (guest/guest)

### 4. Initialize Database

Run the SQL migration script to create tables:

```bash
# Connect to your Supabase database and run:
psql $DATABASE_URL -f src/scripts/migrate.sql
```

Or manually execute the SQL from `src/scripts/migrate.sql`

### 5. Seed Test Data

```bash
npm run seed
```

This creates:
- 1 test student (Alice Johnson)
- 3 sample missions
- Initial learning metrics (all scores at 50)

### 6. Build and Start Server

```bash
# Development mode
npm run dev

# Production mode
npm run build
npm start
```

Server runs on http://localhost:3000

## API Documentation

### Health Check

```bash
GET /health
```

Response:
```json
{
  "status": "ok",
  "timestamp": "2026-03-02T10:30:00.000Z"
}
```

### Complete Mission

```bash
POST /mission-complete
Content-Type: application/json
```

Request Body:
```json
{
  "student_id": "uuid-here",
  "mission_id": "ai-ethics-1",
  "score": 85,
  "time_taken": 120,
  "hints_used": 1,
  "request_id": "optional-idempotency-key"
}
```

Success Response (200):
```json
{
  "success": true,
  "message": "Mission completed successfully",
  "data": {
    "attemptId": "uuid-here",
    "updatedMetrics": {
      "logicScore": 55,
      "ethicsScore": 50,
      "aiOrchestrationScore": 50
    }
  }
}
```

Error Responses:
- `400 Bad Request`: Invalid payload
- `404 Not Found`: Student or mission not found
- `409 Conflict`: Duplicate request (idempotency)
- `500 Internal Server Error`: Database or infrastructure failure

## Testing with cURL

After seeding, get the student ID and test:

```bash
# Replace STUDENT_ID with actual UUID from seed output
curl -X POST http://localhost:3000/mission-complete \
  -H "Content-Type: application/json" \
  -d '{
    "student_id": "STUDENT_ID",
    "mission_id": "ai-ethics-1",
    "score": 85,
    "time_taken": 120,
    "hints_used": 1
  }'
```

## Scoring Logic

The system updates learning metrics based on performance:

- **Logic Score**:
  - +5 if score > 80
  - -2 if hints_used > 2
  
- **Ethics Score**: (Future enhancement - mission type mapping)
- **AI Orchestration Score**: (Future enhancement - mission type mapping)

All scores bounded between 0-100.

## Database Schema

### Tables

1. **students**: User profiles with parent contact info
2. **missions**: Library of learning tasks
3. **mission_attempts**: Transaction log of all attempts
4. **learning_metrics**: Cognitive profile per student
5. **risk_alerts**: History of low-performance alerts
6. **idempotency_keys**: Prevents duplicate processing

### Key Features

- Foreign key constraints with CASCADE deletes
- Indexes on `mission_attempts(student_id, completed_at)`
- Pessimistic locking on metrics updates
- Automatic metrics creation via trigger

## n8n Workflow Setup

1. Create a new workflow in n8n
2. Add Webhook trigger node (POST)
3. Add IF node with condition: `{{ $json.score < 50 }}`
4. True branch:
   - Email node: Send to `{{ $json.parent_email }}`
   - Postgres node: Insert into `risk_alerts` table
5. Copy webhook URL to `.env` as `N8N_WEBHOOK_URL`

## Worker Process

The `MissionProcessorWorker` consumes events from RabbitMQ and:
1. Transforms event to n8n payload format
2. Sends HTTP POST to n8n webhook
3. Implements retry logic with exponential backoff
4. Acknowledges or requeues messages based on success

To run worker separately:
```typescript
// Create worker-start.ts
import { MissionProcessorWorker } from './worker/MissionProcessorWorker';
const worker = new MissionProcessorWorker(dataSource);
await worker.start();
```

## Project Structure

```
Backend/
├── src/
│   ├── config/
│   │   └── app.config.ts          # Database & environment config
│   ├── controller/
│   │   └── MissionController.ts   # API endpoint logic
│   ├── dtos/
│   │   ├── MissionCompleteRequestDto.ts
│   │   ├── MissionCompleteResponseDto.ts
│   │   ├── MissionCompletedEvent.ts
│   │   ├── N8nPayloadDto.ts
│   │   └── UpdatedMetricsDto.ts
│   ├── entities/
│   │   ├── StudentEntity.ts
│   │   ├── MissionEntity.ts
│   │   ├── MssionAttemptEntity.ts
│   │   ├── LearningMetricsEntity.ts
│   │   ├── RiskAlertEntity.ts
│   │   └── IdempotencyKeyEntity.ts
│   ├── exceptions/
│   │   ├── EntityNotFoundException.ts
│   │   ├── IdempotencyException.ts
│   │   ├── InfrastructureException.ts
│   │   └── GlobalExceptionFilter.ts
│   ├── repositories/
│   │   ├── StudentRepository.ts
│   │   ├── MissionRepository.ts
│   │   ├── MissionAttemptRepository.ts
│   │   ├── LearningMetricsRepository.ts
│   │   ├── RiskAlertRepository.ts
│   │   └── IdempotencyRepository.ts
│   ├── scripts/
│   │   ├── migrate.sql             # Database schema
│   │   └── seed.ts                 # Test data generator
│   ├── services/
│   │   ├── ScoringService.ts       # Metrics calculation
│   │   ├── NotificationService.ts  # n8n webhook client
│   │   └── RabbitMqService.ts      # Message queue client
│   ├── worker/
│   │   └── MissionProcessorWorker.ts
│   └── main.ts                     # Application entry point
├── .env
├── docker-compose.yml
├── package.json
├── tsconfig.json
└── README.md
```

## Key Features Implemented

✅ Atomic transaction processing  
✅ Idempotency support (prevents duplicate submissions)  
✅ Pessimistic locking (prevents race conditions)  
✅ Service-Repository pattern (clean architecture)  
✅ Event-driven architecture (RabbitMQ)  
✅ Webhook integration with retry logic  
✅ Comprehensive error handling  
✅ Type-safe TypeScript implementation  
✅ Database indexing for performance  
✅ Graceful shutdown handling  

## Troubleshooting

### Database Connection Issues
- Verify `DATABASE_URL` is correct
- Check Supabase connection pooler settings
- Ensure SSL is configured properly

### RabbitMQ Connection Failed
- Verify Docker container is running: `docker ps`
- Check port 5672 is not blocked
- Restart container: `docker-compose restart`

### n8n Webhook Not Triggering
- Verify `N8N_WEBHOOK_URL` is accessible
- Check n8n workflow is activated
- Review worker logs for retry attempts

### TypeScript Compilation Errors
- Run `npm install` to ensure all dependencies
- Check Node.js version (18+ required)
- Verify `tsconfig.json` settings

## Future Enhancements

- [ ] Mission type mapping for ethics/orchestration scores
- [ ] Real-time dashboard with WebSockets
- [ ] Advanced analytics and reporting
- [ ] Multi-tenant support
- [ ] Rate limiting and API authentication
- [ ] Comprehensive test suite
- [ ] Docker containerization for backend
- [ ] Kubernetes deployment manifests

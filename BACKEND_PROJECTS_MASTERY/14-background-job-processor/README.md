# Project 14: Background Job Processor

## Difficulty Level: Intermediate
**Estimated Time**: 2-3 weeks per language

## What You're Building

A distributed job queue system for background processing (like Sidekiq, Celery, or Bull) that handles async tasks like email sending, image processing, report generation, etc.

## Key Concepts

- Job queues (RabbitMQ, Redis)
- Worker pools
- Priority queues
- Retry mechanisms
- Dead letter queues
- Job scheduling (cron)
- Rate limiting
- Concurrency control

## Architecture

```
Producer → Queue → Workers (Pool) → Result Storage
   ↓         ↓         ↓
 (API)   (Redis)   (Parallel)
                      ↓
              [Failed Jobs → DLQ]
```

## Job Types

### 1. Email Jobs

```typescript
interface SendEmailJob {
  type: "send_email";
  to: string;
  subject: string;
  body: string;
  priority: "high" | "normal" | "low";
}
```

### 2. Image Processing

```typescript
interface ProcessImageJob {
  type: "process_image";
  image_url: string;
  transformations: {
    resize?: { width: number; height: number };
    format?: "jpg" | "png" | "webp";
    quality?: number;
  };
}
```

### 3. Report Generation

```typescript
interface GenerateReportJob {
  type: "generate_report";
  report_type: "sales" | "analytics";
  date_range: { start: string; end: string };
  user_id: number;
}
```

## Implementation

### Job Queue (Bull/Redis)

```typescript
import Queue from "bull";

// Create queue
const emailQueue = new Queue("emails", {
  redis: {
    host: "localhost",
    port: 6379
  }
});

// Add job
async function sendEmail(data: SendEmailJob) {
  await emailQueue.add(data, {
    priority: data.priority === "high" ? 1 : 10,
    attempts: 3,
    backoff: {
      type: "exponential",
      delay: 2000
    },
    removeOnComplete: true,
    removeOnFail: false
  });
}

// Process jobs
emailQueue.process(async (job) => {
  console.log(`Processing job ${job.id}`);

  try {
    await emailService.send(job.data);
    return { status: "sent", job_id: job.id };
  } catch (error) {
    throw error; // Will trigger retry
  }
});

// Handle completed
emailQueue.on("completed", (job, result) => {
  console.log(`Job ${job.id} completed`, result);
});

// Handle failed
emailQueue.on("failed", (job, error) => {
  console.error(`Job ${job.id} failed`, error);

  if (job.attemptsMade >= 3) {
    // Move to DLQ
    deadLetterQueue.add(job.data);
  }
});
```

### Worker Pool

```typescript
class WorkerPool {
  private workers: Worker[] = [];
  private concurrency: number;

  constructor(
    queueName: string,
    processor: JobProcessor,
    concurrency = 10
  ) {
    this.concurrency = concurrency;

    for (let i = 0; i < concurrency; i++) {
      const worker = new Worker(queueName, processor);
      this.workers.push(worker);
    }
  }

  async start() {
    await Promise.all(this.workers.map((w) => w.start()));
  }

  async stop() {
    await Promise.all(this.workers.map((w) => w.stop()));
  }

  getStats() {
    return {
      total_workers: this.workers.length,
      active_jobs: this.workers.filter((w) => w.isBusy()).length,
      idle_workers: this.workers.filter((w) => !w.isBusy())
        .length
    };
  }
}
```

### Retry with Exponential Backoff

```typescript
async function processWithRetry(job: Job, maxAttempts = 3) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await job.processor(job.data);
    } catch (error) {
      if (attempt === maxAttempts) {
        throw error;
      }

      // Exponential backoff: 2s, 4s, 8s, ...
      const delay = Math.pow(2, attempt) * 1000;
      console.log(`Retry ${attempt} after ${delay}ms`);
      await sleep(delay);
    }
  }
}
```

### Priority Queue

```typescript
// Define priorities
enum JobPriority {
  CRITICAL = 1,
  HIGH = 2,
  NORMAL = 5,
  LOW = 10
}

// Add with priority
await queue.add(job, {
  priority: JobPriority.CRITICAL
});

// Jobs with lower priority number processed first
```

### Scheduled Jobs (Cron)

```typescript
import cron from "node-cron";

// Run daily at 2 AM
cron.schedule("0 2 * * *", async () => {
  await reportQueue.add({
    type: "daily_report",
    date: new Date()
  });
});

// Run every 15 minutes
cron.schedule("*/15 * * * *", async () => {
  await cleanupQueue.add({
    type: "cleanup_temp_files"
  });
});
```

### Rate Limiting

```typescript
class RateLimitedQueue extends Queue {
  private rateLimiter = {
    max: 100, // Max jobs per window
    duration: 60000 // 1 minute
  };

  async add(data: any, opts?: any) {
    // Check rate limit
    const key = `rate_limit:${this.name}`;
    const count = await redis.incr(key);

    if (count === 1) {
      await redis.expire(key, this.rateLimiter.duration / 1000);
    }

    if (count > this.rateLimiter.max) {
      throw new Error("Rate limit exceeded");
    }

    return super.add(data, opts);
  }
}
```

## Dead Letter Queue

```typescript
const deadLetterQueue = new Queue("dlq");

// Process DLQ manually or with alerts
deadLetterQueue.process(async (job) => {
  // Log failed job
  logger.error("Job in DLQ", {
    job_id: job.id,
    data: job.data,
    error: job.failedReason
  });

  // Notify admins
  await alertService.send({
    message: `Job ${job.id} permanently failed`,
    severity: "high"
  });

  // Could retry with manual intervention
  // or store for later analysis
});
```

## Job Dashboard API

```http
# Get queue stats
GET /api/jobs/stats
Response:
{
  "waiting": 150,
  "active": 10,
  "completed": 50000,
  "failed": 100,
  "delayed": 5
}

# Get job status
GET /api/jobs/:jobId
Response:
{
  "job_id": "123",
  "status": "completed",
  "progress": 100,
  "result": { ... },
  "attempts": 1,
  "created_at": "...",
  "completed_at": "..."
}

# Retry failed job
POST /api/jobs/:jobId/retry

# Cancel job
DELETE /api/jobs/:jobId
```

## Monitoring

```typescript
// Track metrics
queue.on("completed", (job) => {
  metrics.increment("jobs.completed", {
    queue: queue.name,
    type: job.data.type
  });
});

queue.on("failed", (job, error) => {
  metrics.increment("jobs.failed", {
    queue: queue.name,
    type: job.data.type,
    error: error.message
  });
});

// Track processing time
queue.on("completed", (job) => {
  const duration = Date.now() - job.timestamp;
  metrics.histogram("jobs.duration", duration, {
    queue: queue.name
  });
});
```

## Technology Stack

### NestJS
- bull
- @nestjs/bull
- node-cron

### Golang
- asynq
- cron
- RabbitMQ client

### Python
- Celery
- RQ (Redis Queue)
- APScheduler

## Success Criteria

✅ Process 1000+ jobs/sec
✅ Reliable retry mechanism
✅ Priority queue working
✅ DLQ for failed jobs
✅ Scheduled jobs
✅ Zero job loss
✅ Monitoring dashboard

---

**Next**: Project 15 (Rate Limiter)

# Getting Started with Backend Mastery Projects

Welcome! This guide will help you start your journey to backend engineering mastery through hands-on project building.

---

## Quick Start Checklist

Before you begin, ensure you have:

- [ ] Read the main README.md
- [ ] Chosen your first project
- [ ] Set up your development environment
- [ ] Reviewed the mastery concept documents
- [ ] Committed to building in all three languages

---

## Step 1: Choose Your Starting Project

### For Beginners (0-1 years experience)

Start with these projects to build foundational skills:

1. **Project 15: API Rate Limiter** (2 weeks)
   - Learn: Rate limiting algorithms, Redis basics
   - Why first: Smaller scope, clear objectives
   - Path: `15-api-rate-limiter/README.md`

2. **Project 14: Background Job Processor** (2-3 weeks)
   - Learn: Queues, async processing, worker pools
   - Why second: Builds on async concepts
   - Path: `14-background-job-processor/README.md`

3. **Project 01: Distributed URL Shortener** (2-3 weeks)
   - Learn: Caching, distributed systems basics, databases
   - Why third: Classic system design problem
   - Path: `01-distributed-url-shortener/README.md`

### For Intermediate (1-3 years experience)

Jump into more complex projects:

1. **Project 03: Multi-Tenant API Gateway** (2-3 weeks)
   - Learn: API gateway patterns, multi-tenancy, auth
   - Path: `03-multi-tenant-api-gateway/README.md`

2. **Project 02: Real-Time Task Manager** (2-3 weeks)
   - Learn: WebSockets, CQRS, event-driven architecture
   - Path: `02-realtime-task-manager/README.md`

3. **Project 08: E-Commerce Inventory** (3-4 weeks)
   - Learn: Distributed locking, concurrency, consistency
   - Path: `08-ecommerce-inventory/README.md`

### For Advanced (3+ years experience)

Tackle the most challenging projects:

1. **Project 04: Payment Processing Engine** (3-4 weeks)
   - Learn: Saga pattern, event sourcing, distributed transactions
   - Path: `04-payment-processing-engine/README.md`

2. **Project 10: AI Search & RAG System** (4-5 weeks)
   - Learn: Vector databases, LLM integration, RAG
   - Path: `10-ai-search-rag-system/README.md`

3. **Project 12: Observability & APM System** (3-4 weeks)
   - Learn: Metrics, logs, traces, monitoring
   - Path: `12-observability-apm-system/README.md`

---

## Step 2: Set Up Your Development Environment

### Required Tools (All Projects)

```bash
# Version Control
git --version  # Should be 2.x+

# Docker
docker --version  # 24.x+
docker-compose --version  # 2.x+

# Database Tools
psql --version  # PostgreSQL 15+
redis-cli --version  # Redis 7+
```

### Language-Specific Setup

#### NestJS (TypeScript)

```bash
# Install Node.js
node --version  # 20.x+
npm --version   # 10.x+

# Install NestJS CLI
npm install -g @nestjs/cli

# Create new project
nest new project-name
cd project-name

# Install common dependencies
npm install @nestjs/config
npm install @nestjs/typeorm typeorm pg
npm install ioredis
npm install class-validator class-transformer
```

#### Golang

```bash
# Install Go
go version  # 1.22+

# Set up workspace
mkdir -p ~/go/src/github.com/yourusername
cd ~/go/src/github.com/yourusername

# Initialize module
go mod init github.com/yourusername/project-name

# Install common dependencies
go get -u github.com/gin-gonic/gin
go get -u gorm.io/gorm
go get -u gorm.io/driver/postgres
go get -u github.com/redis/go-redis/v9
```

#### Python

```bash
# Install Python
python --version  # 3.12+
pip --version

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install FastAPI
pip install fastapi uvicorn
pip install sqlalchemy psycopg2-binary
pip install redis
pip install pydantic pydantic-settings
pip install pytest pytest-asyncio
```

### Infrastructure Setup

#### Docker Compose (Common Services)

Create a `docker-compose.yml` file for common services:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: myproject
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  mongodb:
    image: mongo:7
    environment:
      MONGO_INITDB_ROOT_USERNAME: dev
      MONGO_INITDB_ROOT_PASSWORD: devpassword
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  rabbitmq:
    image: rabbitmq:3-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: dev
      RABBITMQ_DEFAULT_PASS: devpassword
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

volumes:
  postgres_data:
  redis_data:
  mongo_data:
  rabbitmq_data:
```

Start services:

```bash
docker-compose up -d
```

---

## Step 3: Project Structure

Each project should follow this structure:

```
project-name/
├── nestjs/
│   ├── src/
│   │   ├── modules/
│   │   ├── services/
│   │   ├── controllers/
│   │   ├── entities/
│   │   └── main.ts
│   ├── test/
│   ├── docker-compose.yml
│   ├── package.json
│   └── README.md
├── golang/
│   ├── cmd/
│   │   └── server/
│   ├── internal/
│   │   ├── handlers/
│   │   ├── services/
│   │   ├── models/
│   │   └── repository/
│   ├── test/
│   ├── docker-compose.yml
│   ├── go.mod
│   └── README.md
├── python/
│   ├── src/
│   │   ├── api/
│   │   ├── services/
│   │   ├── models/
│   │   └── main.py
│   ├── tests/
│   ├── docker-compose.yml
│   ├── requirements.txt
│   └── README.md
└── README.md (Project documentation)
```

---

## Step 4: Development Workflow

### Phase 1: Read & Understand (Day 1)

1. Read the project README completely
2. Study the architecture diagrams
3. Review the database schemas
4. Understand the API endpoints
5. Note down questions

### Phase 2: Plan Implementation (Day 2)

1. Break down the project into milestones
2. Identify the MVP (Minimum Viable Product)
3. List required technologies
4. Set up git repository
5. Create project structure

### Phase 3: Build in First Language (Week 1-2)

Choose your strongest language first:

```bash
# Example: Starting with NestJS
cd project-name/nestjs
nest new .
npm install [dependencies]

# Create modules
nest g module users
nest g service users
nest g controller users

# Implement features incrementally
# - Basic CRUD
# - Add authentication
# - Add caching
# - Add advanced features

# Test as you go
npm test
```

### Phase 4: Build in Second Language (Week 3)

Now implement the same project in another language:

```bash
# Example: Moving to Golang
cd project-name/golang
go mod init

# Recreate the same features
# - Compare performance
# - Note language differences
# - Observe different patterns
```

### Phase 5: Build in Third Language (Week 4)

Complete the trilogy:

```bash
# Example: Finally Python
cd project-name/python
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Implement again
# - Reflect on all three implementations
# - Document learnings
```

### Phase 6: Compare & Document

Create a comparison document:

```markdown
# Implementation Comparison

## Performance
- NestJS: [metrics]
- Golang: [metrics]
- Python: [metrics]

## Code Size
- NestJS: [lines]
- Golang: [lines]
- Python: [lines]

## Development Speed
- NestJS: [time]
- Golang: [time]
- Python: [time]

## Strengths & Weaknesses
...
```

---

## Step 5: Testing Strategy

### Write Tests at Each Stage

```typescript
// NestJS Example
describe('UserService', () => {
  it('should create a user', async () => {
    const user = await service.create({
      email: 'test@example.com',
      password: 'password123'
    });
    expect(user.id).toBeDefined();
  });
});
```

```go
// Golang Example
func TestUserService_Create(t *testing.T) {
    service := NewUserService()
    user, err := service.Create(User{
        Email: "test@example.com",
        Password: "password123",
    })
    assert.NoError(t, err)
    assert.NotEmpty(t, user.ID)
}
```

```python
# Python Example
def test_user_service_create():
    service = UserService()
    user = service.create(
        email="test@example.com",
        password="password123"
    )
    assert user.id is not None
```

### Run Load Tests

```bash
# Install k6
brew install k6  # macOS
# or download from k6.io

# Create load test script
cat > load-test.js <<EOF
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  vus: 100,
  duration: '30s',
};

export default function() {
  let res = http.get('http://localhost:3000/api/users');
  check(res, {
    'status is 200': (r) => r.status === 200,
  });
}
EOF

# Run load test
k6 run load-test.js
```

---

## Step 6: Monitoring & Observability

### Add Logging

```typescript
// Structured logging
logger.info('User created', {
  user_id: user.id,
  email: user.email,
  timestamp: new Date().toISOString()
});
```

### Add Metrics

```typescript
// Prometheus metrics
httpRequestsTotal.inc({
  method: req.method,
  path: req.path,
  status: res.statusCode
});
```

### Add Tracing

```typescript
// OpenTelemetry
const span = tracer.startSpan('create_user');
try {
  // ... operation
  span.setStatus({ code: SpanStatusCode.OK });
} finally {
  span.end();
}
```

---

## Step 7: Document Your Journey

### Keep a Learning Journal

Create a `LEARNINGS.md` file:

```markdown
# Project Learnings

## Week 1: NestJS Implementation
- Learned about decorators and dependency injection
- Struggled with TypeORM relationships
- Discovered the power of pipes for validation

## Week 2: Performance Optimization
- Added Redis caching, saw 10x improvement
- Learned about connection pooling
- Implemented database indexes

## Week 3: Golang Implementation
- Goroutines are amazing for concurrency
- Interfaces are different from TypeScript
- No decorators, more verbose but clearer

...
```

### Write Blog Posts

Share your learnings:

- "Building a URL Shortener: NestJS vs Golang vs Python"
- "5 Things I Learned About Distributed Caching"
- "How I Implemented CQRS in Three Languages"

### Create Architecture Decision Records (ADRs)

```markdown
# ADR 001: Use Redis for Rate Limiting

## Status: Accepted

## Context
Need distributed rate limiting across multiple servers.

## Decision
Use Redis with sliding window algorithm.

## Consequences
+ Fast (< 5ms overhead)
+ Distributed
+ Accurate
- Requires Redis cluster for HA
- Additional infrastructure
```

---

## Troubleshooting Common Issues

### Docker Issues

```bash
# If ports are already in use
docker-compose down
lsof -i :5432  # Find process using port
kill -9 <PID>

# If volumes are corrupted
docker-compose down -v
docker-compose up -d
```

### Database Connection Issues

```bash
# Test PostgreSQL connection
psql -h localhost -U dev -d myproject

# Test Redis connection
redis-cli ping

# Check Docker logs
docker-compose logs postgres
```

### Node Modules Issues

```bash
# Clear and reinstall
rm -rf node_modules package-lock.json
npm install
```

---

## Learning Resources

### Books
- "Designing Data-Intensive Applications" - Martin Kleppmann
- "System Design Interview Vol 1 & 2" - Alex Xu
- "Release It!" - Michael Nygard

### Online Courses
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [Awesome System Design](https://github.com/madd86/awesome-system-design)

### Practice
- LeetCode (Data Structures & Algorithms)
- System Design Mock Interviews

---

## Getting Help

### When Stuck

1. **Read the docs**: Official framework/library documentation
2. **Search GitHub Issues**: Someone likely faced this before
3. **Stack Overflow**: Search before asking
4. **Discord/Slack**: Join relevant communities
5. **AI Assistants**: ChatGPT, Claude for explanations

### Community Resources

- r/backend (Reddit)
- Backend Engineering Discord servers
- Local meetups and conferences

---

## Success Metrics

Track your progress:

- [ ] Completed first project in one language
- [ ] Completed same project in second language
- [ ] Completed same project in third language
- [ ] Wrote comprehensive tests (>80% coverage)
- [ ] Set up monitoring and logging
- [ ] Deployed to production-like environment
- [ ] Load tested and optimized
- [ ] Documented architecture decisions
- [ ] Wrote blog post about learnings

---

## Next Steps

1. **Pick your first project** from the recommendations above
2. **Set up your environment** using this guide
3. **Start building** in your strongest language
4. **Test thoroughly** at each stage
5. **Compare implementations** across languages
6. **Document your learnings**
7. **Share with the community**

---

## Remember

- **Quality over speed**: Take time to understand concepts deeply
- **Practice makes perfect**: Each project reinforces previous learnings
- **Compare implementations**: Learn from differences across languages
- **Test everything**: No project is complete without tests
- **Document decisions**: Future you will thank present you
- **Share your journey**: Help others learn from your experience

---

## Ready to Start?

Choose your first project and dive in:

```bash
cd BACKEND_PROJECTS_MASTERY
cd 01-distributed-url-shortener  # or your chosen project
cat README.md
```

**Good luck on your backend mastery journey!** 🚀

Remember: The goal isn't just to complete projects, but to **understand the concepts deeply** and become a senior-level backend engineer who can make informed architectural decisions.

Happy Building! 💪

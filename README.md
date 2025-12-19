
# Spring-Sse-Real-Time-Job-Progress

## Overview

This project is a **multi-microservice demonstration** of **Server-Sent Events (SSE)** for **real-time progress updates** of long-running jobs in **Spring Boot 3.x**. It showcases how to implement unidirectional, efficient streaming of progress from server to client using standard HTTP — perfect for scenarios where WebSocket is overkill but live updates are essential.

The focus is on **production-grade SSE patterns**: `SseEmitter`, `Flux` streaming, progress tracking, cleanup, authentication, and client reconnection handling.

## Real-World Scenario (Simulated)

In job processing systems like **CI/CD pipelines** (GitHub Actions, Jenkins), **data imports**, or **report generation**:
- Users submit a long-running job (build, test, deploy, ETL).
- The job takes minutes to hours.
- Users need real-time feedback: percentage complete, current step, logs, success/failure.
- Polling wastes resources; WebSocket may be unnecessary for one-way updates.

We simulate a background job processor where clients submit jobs and receive live progress via SSE — with percentage, status, and logs streamed instantly.

## Microservices Involved

| Service                  | Responsibility                                                                 | Port  |
|--------------------------|--------------------------------------------------------------------------------|-------|
| **eureka-server**        | Service discovery (Netflix Eureka)                                             | 8761  |
| **job-queue-service**    | Entry point: accepts job submissions, starts background processing            | 8081  |
| **worker-service**       | Background workers: simulate long-running tasks, update progress               | 8082  |
| **progress-service**     | Manages SSE connections, streams progress to clients                           | 8083  |

Progress is stored in Redis for shared state and cleanup.

## Tech Stack

- Spring Boot 3.x
- Spring WebFlux (for reactive SSE) OR Spring MVC (`SseEmitter`)
- Project Reactor (`Flux<ServerSentEvent>`) preferred for backpressure
- Redis (shared progress store + pub/sub for worker updates)
- Spring Cloud Netflix Eureka
- Spring Security (optional SSE auth)
- Micrometer + Actuator (SSE metrics)
- Lombok
- Maven (multi-module)
- Docker & Docker Compose

## Docker Containers

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"

  job-queue-service:
    build: ./job-queue-service
    depends_on:
      - redis
      - eureka-server
    ports:
      - "8081:8081"

  worker-service:
    build: ./worker-service
    depends_on:
      - redis
      - eureka-server
    ports:
      - "8082:8082"

  progress-service:
    build: ./progress-service
    depends_on:
      - redis
      - eureka-server
    ports:
      - "8083:8083"
```

Run with: `docker-compose up --build`

## SSE Implementation

| Feature                        | Implementation Details                                                  |
|--------------------------------|-------------------------------------------------------------------------|
| **Reactive SSE**               | `Flux<ServerSentEvent>` with `SseEmitter` under the hood               |
| **Progress Tracking**          | Redis hash: `job:{id}` → fields: status, progress, message, logs       |
| **Worker Updates**             | Worker publishes to Redis pub/sub → progress service pushes to clients |
| **Client Reconnection**        | `event:id` and `retry:` headers for browser reconnect                  |
| **Cleanup**                    | Timeout + completion → remove emitter and Redis entry                  |
| **Authentication**             | Spring Security + principal in SSE connection                          |
| **Backpressure**               | Natural with Flux, configurable buffer                                 |

## Key Features

- Real-time progress streaming (percentage, status, logs)
- Multiple clients see same job progress
- Automatic reconnection with last-event-id
- Job completion/failure notifications
- Metrics: active streams, completed jobs
- Graceful cleanup of resources
- Simulated long-running jobs (e.g., 2-minute processing)
- Client HTML page with progress bar

## Expected Endpoints

### Job Queue Service (`http://localhost:8081`)

| Method | Endpoint                        | Description                                      |
|--------|---------------------------------|--------------------------------------------------|
| POST   | `/api/jobs`                     | Submit new job → returns jobId                   |
| GET    | `/api/jobs/{id}`                | Get job status (REST fallback)                   |

### Progress Service (`http://localhost:8083`)

| Method | Endpoint                        | Description                                      |
|--------|---------------------------------|--------------------------------------------------|
| GET    | `/api/progress/{jobId}`         | SSE stream of job progress (text/event-stream)   |

### Sample SSE Stream
```
id: 1
event: progress
data: {"progress": 25, "status": "PROCESSING", "message": "Step 1/4 complete"}

id: 2
event: log
data: "Processing file data.csv..."

id: 3
event: complete
data: {"progress": 100, "status": "COMPLETED", "result": "Success!"}
```

### Client Demo
Provided `client/index.html` — simple UI with:
- Start job button
- Progress bar
- Live log output
- Auto-reconnect on disconnect

## Architecture Overview

```
Client (Browser)
   ↓ (HTTP POST)
Job Queue Service → create job → start worker
   ↓ (jobId)
Client opens SSE → /api/progress/{jobId}
   ↓
Progress Service → Flux<SSE> from Redis stream
   ↓
Worker Service → updates Redis (progress, logs)
   ↓ (Redis pub/sub)
Progress Service pushes to connected clients
```

**Progress Flow**:
1. Job submitted → ID returned
2. Client connects SSE with jobId
3. Worker runs → updates Redis every few seconds
4. Progress service streams updates to all connected clients
5. Job ends → final event + cleanup

## How to Run

1. Clone repository
2. Start Docker: `docker-compose up --build`
3. Open client: `http://localhost:8083/client/index.html`
4. Click "Start Job" → progress bar and logs update in real-time
5. Refresh page mid-job → reconnects and continues from last event
6. Multiple tabs → all receive same updates
7. Check Redis → job data stored

## Testing Real-Time

1. Start job → immediate progress stream
2. Logs appear in real-time
3. Disconnect network → browser auto-reconnects
4. Multiple clients → synchronized progress
5. Job failure → error event streamed
6. Idle timeout → stream closes gracefully

## Skills Demonstrated

- Server-Sent Events implementation with Spring WebFlux
- Real-time progress tracking architecture
- Redis as shared state and pub/sub
- Client reconnection and event IDs
- Resource cleanup and timeout handling
- Production SSE best practices
- Alternative to WebSocket for one-way updates

## Future Extensions

- Authentication on SSE endpoint
- Job history and replay
- Push notifications as fallback
- Integration with Spring Batch
- Multiple concurrent jobs per user
- Custom event types (charts, images)
- Rate limiting on SSE connections


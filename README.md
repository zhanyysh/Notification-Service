# Notification System Design

Excalidraw - https://excalidraw.com/#json=oV_OtFBY47s2IX_8idA8d,G4D8gn5oHj9dJ-ziv9YL0g

## Overview

This system delivers notifications to users when events occur (e.g., messages, orders, comments).

Supported channels:

* In-app (stored & fetched later)
* Push notifications (mobile)
* Email notifications

Scale:

* 20M DAU
* 100M notifications/day

---

## High-Level Architecture

Event-driven, asynchronous system.

Flow:

1. Event happens (e.g., new message)
2. Event is published to a queue
3. Notification Service processes event
4. Fan-out to multiple channels
5. Workers deliver notifications
6. In-app notifications stored in DB

---

## Components

### 1. API Layer

* Accepts requests to create notifications
* Fetch user notifications

### 2. Event Queue (Kafka / RabbitMQ)

* Decouples producers from consumers
* Handles spikes

### 3. Notification Service

* Core logic:

  * Deduplication
  * User preferences filtering
  * Fan-out to channels

### 4. Workers

* Push Worker
* Email Worker
* In-app Worker

### 5. Database

* Stores notifications (for in-app)

### 6. Cache (Redis)

* User preferences
* Rate limiting
* Deduplication keys

---

## API Design

### Create Notification

POST /notifications

Request:

```json
{
  "user_id": "123",
  "type": "NEW_MESSAGE",
  "payload": {
    "message_id": "abc"
  }
}
```

Response:

```json
{
  "status": "accepted"
}
```

---

### Get Notifications (In-app)

GET /users/{user_id}/notifications

Response:

```json
{
  "notifications": [
    {
      "id": "n1",
      "type": "NEW_MESSAGE",
      "read": false,
      "created_at": "2026-01-01T10:00:00Z"
    }
  ]
}
```

---

## Data Model

### Notifications Table

| Field      | Type      | Notes   |
| ---------- | --------- | ------- |
| id         | UUID      | PK      |
| user_id    | UUID      | Indexed |
| type       | string    |         |
| payload    | JSON      |         |
| read       | boolean   |         |
| created_at | timestamp | Indexed |

Indexes:

* (user_id, created_at DESC)

---

### User Preferences (Cache + DB)

| Field   | Type |
| ------- | ---- |
| user_id | UUID |
| email   | bool |
| push    | bool |
| in_app  | bool |

---

### Deduplication Store (Redis)

Key:

```
dedup:{user_id}:{event_id}
```

TTL: few minutes

---

## Scaling Strategy

### Horizontal Scaling

* Stateless services → scale via containers
* Workers scale independently

### Queue Partitioning

* Kafka partitions by `user_id`
* Ensures ordering per user

### Database Scaling

* Shard by `user_id`
* Use read replicas for reads

---

## Reliability & Failures

### Retry Strategy

* Exponential backoff
* Dead Letter Queue (DLQ)

### Idempotency

* Deduplication keys in Redis

### Failure Handling

* If push fails → retry
* If email fails → retry or fallback

---

## Rate Limiting

Redis-based:

* Key: `rate:{user_id}`
* Limit: e.g., 10 notifications/minute

---

## Avoiding Duplicates

* Event ID used as idempotency key
* Stored in Redis before processing

---

## Tradeoffs

### 1. Push vs Pull

* Push = real-time, complex
* Pull (in-app) = simple, slightly stale

### 2. Consistency vs Latency

* Eventual consistency chosen
* Faster delivery

### 3. Sync vs Async

* Async via queue
* Improves scalability but adds delay

---

## Why This Works

* Handles 100M/day easily
* Scales horizontally
* Fault-tolerant via queues
* Flexible for adding new channels

---

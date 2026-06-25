## Stage 1

### Overview

This document defines the REST API design and contract for a notification platform that displays notifications to authenticated users. All endpoints assume the user is already authenticated (per evaluation constraints); no login/registration flows are included.

### Core Actions

The notification platform needs to support the following core actions:

1. **Create a notification** — system/internal services generate a notification for a user
2. **List notifications** — fetch a user's notifications (with pagination and filtering)
3. **Get a single notification** — fetch details of one notification
4. **Mark a notification as read**

5. **Mark all notifications as read**
6. **Delete a notification**
7. **Get unread notification count** — lightweight endpoint for badge counters
8. **Subscribe to real-time notification updates**

### Naming Conventions

- Resource-based nouns, plural: `/notifications`
- No verbs in URLs — actions expressed via HTTP method (`POST`, `GET`, `PATCH`, `DELETE`)
- Sub-actions on a single resource as sub-paths: `/notifications/{id}/read`
- Bulk actions as sub-paths on the collection: `/notifications/read-all`
- All requests/responses use `application/json`
- Timestamps in ISO 8601 UTC (e.g., `2026-06-25T10:30:00Z`)

---

### Endpoint 1: Create a Notification

`POST /api/v1/notifications`

Used internally by other services (e.g., order service, chat service) to create a notification for a user. Not typically called by the end-user client.

**Headers**

Content-Type: application/json

X-Internal-Service-Token: <service-to-service auth token>
**Request Body**
```json
{
  "userId": "usr_8821",
  "type": "ORDER_SHIPPED",
  "title": "Your order has shipped",
  "message": "Order #4421 is on its way and will arrive in 3-5 days.",
  "metadata": {
    "orderId": "ord_4421",
    "actionUrl": "/orders/4421"
  }
}
```

**Response — `201 Created`**
```json
{
  "id": "ntf_90213",
  "userId": "usr_8821",
  "type": "ORDER_SHIPPED",
  "title": "Your order has shipped",
  "message": "Order #4421 is on its way and will arrive in 3-5 days.",
  "metadata": {
    "orderId": "ord_4421",
    "actionUrl": "/orders/4421"
  },
  "isRead": false,
  "createdAt": "2026-06-25T10:30:00Z"
}
```

**Response Headers**

Location: /api/v1/notifications/ntf_90213
---

### Endpoint 2: List Notifications

`GET /api/v1/notifications?page=1&limit=20&status=unread`

**Headers**

Authorization: Bearer <token>
**Query Parameters**
| Param | Type | Description |
|---|---|---|
| `page` | integer | Page number, default 1 |
| `limit` | integer | Items per page, default 20, max 100 |
| `status` | string | `all` \| `read` \| `unread` (default `all`) |

**Response — `200 OK`**
```json
{
  "data": [
    {
      "id": "ntf_90213",
      "type": "ORDER_SHIPPED",
      "title": "Your order has shipped",
      "message": "Order #4421 is on its way and will arrive in 3-5 days.",
      "isRead": false,
      "createdAt": "2026-06-25T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "totalItems": 134,
    "totalPages": 7
  }
}
```

---

### Endpoint 3: Get a Single Notification

`GET /api/v1/notifications/{id}`

**Headers**

Authorization: Bearer <token>
**Response — `200 OK`**
```json
{
  "id": "ntf_90213",
  "userId": "usr_8821",
  "type": "ORDER_SHIPPED",
  "title": "Your order has shipped",
  "message": "Order #4421 is on its way and will arrive in 3-5 days.",
  "metadata": {
    "orderId": "ord_4421",
    "actionUrl": "/orders/4421"
  },
  "isRead": false,
  "createdAt": "2026-06-25T10:30:00Z"
}
```

**Error — `404 Not Found`**
```json
{
  "error": {
    "code": "NOTIFICATION_NOT_FOUND",
    "message": "No notification found with id ntf_90213"
  }
}
```

---

### Endpoint 4: Mark a Notification as Read

`PATCH /api/v1/notifications/{id}/read`

**Headers**

Authorization: Bearer <token>

Content-Type: application/json
**Request Body**: none required

**Response — `200 OK`**
```json
{
  "id": "ntf_90213",
  "isRead": true,
  "readAt": "2026-06-25T10:45:00Z"
}
```

---

### Endpoint 5: Mark All Notifications as Read

`PATCH /api/v1/notifications/read-all`

**Headers**

Authorization: Bearer <token>
**Response — `200 OK`**
```json
{
  "updatedCount": 12,
  "readAt": "2026-06-25T10:46:00Z"
}
```

---

### Endpoint 6: Delete a Notification

`DELETE /api/v1/notifications/{id}`

**Headers**

Authorization: Bearer <token>
**Response — `204 No Content`**
(empty body)

---

### Endpoint 7: Get Unread Count

`GET /api/v1/notifications/unread-count`

**Headers**

Authorization: Bearer <token>
**Response — `200 OK`**
```json
{
  "unreadCount": 12
}
```

---

### Real-Time Notification Mechanism

**Chosen approach: Server-Sent Events (SSE)**

**Endpoint**: `GET /api/v1/notifications/stream`

**Headers**

Authorization: Bearer <token>

Accept: text/event-stream
**Sample stream payload**
event: new_notification

data: {"id":"ntf_90214","type":"COMMENT_REPLY","title":"New reply","isRead":false,"createdAt":"2026-06-25T10:50:00Z"}
event: unread_count_update

data: {"unreadCount":13}
**Why SSE over WebSockets or polling:**

- **Notifications are one-directional** (server → client only). The client never needs to push data back over the same channel, so a full duplex protocol like WebSockets adds unnecessary complexity (connection upgrade handshake, ping/pong keep-alives, more complex load-balancer/proxy configuration).
- **SSE runs over plain HTTP/1.1**, which means it works through standard proxies, load balancers, and firewalls without special configuration, and integrates naturally with the same REST API surface (it's just another `GET` endpoint).
- **Built-in auto-reconnect**: browsers natively retry a dropped SSE connection and resume via the `Last-Event-ID` header, which simplifies client-side resilience compared to hand-rolling WebSocket reconnect logic.
- **Simpler scaling story**: each SSE connection is a long-lived HTTP response; this is easier to reason about and horizontally scale (e.g., behind standard HTTP load balancers) than maintaining stateful WebSocket connections, especially when the only traffic is server-initiated pushes.
- WebSockets would be justified if the client needed to send frequent real-time data back (e.g., live chat, collaborative editing) — that's not the case here.
- Polling is the simplest fallback but is not genuinely real-time and wastes resources with repeated empty requests; it's kept only as a degraded fallback for clients that don't support SSE (e.g., very old browsers), using the `GET /api/v1/notifications/unread-count` endpoint on an interval.


 STAGE2:
 ---

## Stage 2

### Database Choice: PostgreSQL (SQL)

**Justification:**

- **Predictable, fixed schema**: every notification has the same shape (user, type, title, message, read status, timestamp). This is a relational, not document-shaped, problem — there's no benefit from NoSQL's schema flexibility here.
- **Strong consistency is required**: unread counts and "mark as read" must not double-count or drift under concurrent updates (e.g., a user opening the app on two devices simultaneously). PostgreSQL's ACID transactions and row-level locking make this straightforward; a NoSQL store's eventual consistency model would require extra application-level safeguards to get the same guarantee.
- **Relational queries are natural here**: notifications belong to a user (foreign key), and queries like "unread notifications for user X, sorted by time" are simple indexed lookups in SQL.
- **Mature indexing and partitioning support** (see below) directly addresses the scale problem this system will eventually face.

NoSQL (e.g., MongoDB) would be the better fit if notification payloads varied wildly in structure per type, or if the system needed to ingest writes at a scale beyond what a single relational cluster could handle with partitioning — neither applies at the scale this system is designed for.

### Schema

```sql
CREATE TABLE notifications (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    type            VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    message         TEXT NOT NULL,
    metadata        JSONB DEFAULT '{}',
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    read_at         TIMESTAMPTZ NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Indexes to support the API's main access patterns
CREATE INDEX idx_notifications_user_created
    ON notifications (user_id, created_at DESC);

CREATE INDEX idx_notifications_user_unread
    ON notifications (user_id, is_read)
    WHERE is_read = FALSE;
```

`metadata` is `JSONB` to allow per-notification-type extra fields (e.g., `orderId`, `actionUrl`) without needing a new column for every notification type — this gives some of NoSQL's flexibility for the one field that genuinely varies, while keeping the rest of the schema strict.

### Problems at Scale and Solutions

As notification volume grows (e.g., millions of users, dozens of notifications/day each), the following problems emerge:

| Problem | Solution |
|---|---|
| **Table grows unbounded** — most notifications are read once and never queried again, but the table keeps growing forever, slowing down all queries and bloating storage/backups. | **Time-based partitioning** (e.g., `PARTITION BY RANGE (created_at)`, monthly partitions). Old partitions (e.g., >90 days, all read) can be archived to cold storage or dropped entirely, keeping the "hot" table small and fast. |
| **Slow unread-count / list queries at scale** — scanning all rows for a user is fine at thousands of rows, but degrades at tens of millions. | The partial index `idx_notifications_user_unread` (`WHERE is_read = FALSE`) keeps the unread-lookup index small regardless of total table size, since read notifications are excluded from it. |
| **Write hotspotting** — a single popular event (e.g., a platform-wide announcement) inserting millions of rows at once spikes write load on one table. | Batch inserts in chunks, and/or use a message queue (e.g., a job queue) in front of the insert path so notification creation is processed asynchronously rather than blocking the triggering request. |
| **Read-after-write consistency on "mark all as read"** across high concurrency. | Wrap the bulk update in a single transaction (`UPDATE ... WHERE user_id = ? AND is_read = FALSE`), relying on PostgreSQL's row-level locking to avoid race conditions with concurrent single-notification reads. |
| **Cross-region latency** as the user base grows globally. | Read replicas in each region for `GET` (list/unread-count) endpoints, with writes still routed to the primary. Acceptable since notification reads can tolerate brief replication lag. |

### Queries Backing the Stage 1 REST APIs

**`POST /api/v1/notifications` — Create**
```sql
INSERT INTO notifications (user_id, type, title, message, metadata)
VALUES ($1, $2, $3, $4, $5)
RETURNING id, user_id, type, title, message, metadata, is_read, created_at;
```

**`GET /api/v1/notifications?status=unread&page=1&limit=20` — List**
```sql
SELECT id, type, title, message, is_read, created_at
FROM notifications
WHERE user_id = $1
  AND is_read = FALSE   -- omitted entirely when status=all
ORDER BY created_at DESC
LIMIT $2 OFFSET $3;
```

**`GET /api/v1/notifications/{id}` — Get single**
```sql
SELECT id, user_id, type, title, message, metadata, is_read, created_at
FROM notifications
WHERE id = $1 AND user_id = $2;
```

**`PATCH /api/v1/notifications/{id}/read` — Mark one as read**
```sql
UPDATE notifications
SET is_read = TRUE, read_at = now()
WHERE id = $1 AND user_id = $2
RETURNING id, is_read, read_at;
```

**`PATCH /api/v1/notifications/read-all` — Mark all as read**
```sql
UPDATE notifications
SET is_read = TRUE, read_at = now()
WHERE user_id = $1 AND is_read = FALSE
RETURNING id;
-- application layer returns COUNT of returned rows as updatedCount
```

**`DELETE /api/v1/notifications/{id}` — Delete**
```sql
DELETE FROM notifications
WHERE id = $1 AND user_id = $2;
```

**`GET /api/v1/notifications/unread-count` — Unread count**
```sql
SELECT COUNT(*) AS unread_count
FROM notifications
WHERE user_id = $1 AND is_read = FALSE;
```
----------------------------------------------------------------------------------------------------
STAGE3
## Stage 3

### Is the query accurate?

The query itself is logically correct — it does fetch unread notifications for a specific student, ordered oldest-first. But "accurate" cuts both ways here: it returns the right rows, but `SELECT *` is a bad habit — it pulls every column (including ones the API response doesn't need, like `metadata`), wasting bandwidth and memory for no reason. So: logically correct, but not well-written.

### Why is it slow?

At 5,000,000 rows, without the right index, the database has no fast way to jump straight to "this student's unread notifications." It has to **scan the entire table** (or a large chunk of it) row by row, checking `studentID = 1042 AND isRead = false` on every single row, then sort whatever matches by `createdAt`. This is called a *full table scan* — and it gets linearly worse as the table grows. At 5,000 rows it's instant; at 5,000,000 it's slow, and at 50,000,000 it'll be unusable.

### What would you change? What's the likely computation cost?

**Change 1 — add a composite index** on `(studentID, isRead, createdAt)`. This lets the database jump directly to that student's unread rows, already roughly in the right order, instead of scanning everything.

**Change 2 — select only needed columns**, not `SELECT *`:
```sql
SELECT id, title, message, createdAt
FROM notifications
WHERE studentID = 1042 AND isRead = false
ORDER BY createdAt ASC;
```

**Computation cost, before vs after:**
- Without index: **O(n)** — cost grows linearly with total table size (n = total notifications, currently 5,000,000), regardless of how few rows actually match.
- With the right composite index: **O(log n + k)** — the database does a fast lookup (`log n`) to find the starting point, then reads only the `k` matching rows for that student. Since each student typically has a small, bounded number of notifications, this stays fast even as the *total* table grows into the tens of millions.

### Is "add indexes on every column" good advice?

**No — and this is the trap question.** Indexes aren't free:
- **Every index slows down writes.** Every `INSERT`/`UPDATE`/`DELETE` has to update every index on that table too. With 5,000,000+ rows and indexes on every column, write performance would degrade badly — and this is a write-heavy table (new notifications constantly being created).
- **Indexes cost storage.** An index roughly duplicates the indexed column's data in a separate structure. Indexing every column could mean storage costs rivaling or exceeding the table itself.
- **Most single-column indexes won't even get used.** The database query planner picks *one* index per query (in simple cases) — having ten unrelated single-column indexes when your actual query always filters on `studentID + isRead` together does nothing for that query. What helps is a **composite index that matches your actual query patterns**, not blanket indexing.

The right approach: index based on how the table is actually queried — in this case, `(studentID, isRead, createdAt)` — not indiscriminately.

### Query: students who got a "Placement" notification in the last 7 days

```sql
SELECT DISTINCT studentID
FROM notifications
WHERE notification_type = 'Placement'
  AND createdAt >= NOW() - INTERVAL '7 days';
```

- `DISTINCT studentID` — since the question asks for which students, not every matching notification row (a student could have multiple placement notifications in that window)
- `notification_type = 'Placement'` — filters to the specific enum value
- `createdAt >= NOW() - INTERVAL '7 days'` — last 7 days, relative to now

This query would benefit from a separate composite index: `(notification_type, createdAt)`.
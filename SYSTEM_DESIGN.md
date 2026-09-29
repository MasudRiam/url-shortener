# Distributed URL Shortener — System Design & Engineering Architecture

> **Author**: Senior Systems Engineer  
> **Target Audience**: Backend Engineers, Systems Architects, and Technical Interviewers  
> **Status**: Design Proposal & Production Blueprint  

---

## 1. Executive Summary & Core Engineering Philosophy

To junior developers, a URL shortener looks like a simple CRUD application: accept a long URL, store it in a database, generate a random string, and redirect the user.

To a **Senior Systems Engineer**, a URL shortener is a classic **high-throughput, low-latency distributed key-value lookup engine** with extreme asymmetric traffic characteristics.

### Key Traffic & Scaling Assumptions
* **Read-to-Write Ratio**: URL shorteners are heavily read-biased, typically **100:1 to 1000:1**. Writes (URL creation) occur infrequently compared to reads (clicks/redirects).
* **Latency Budget**: Redirection is on the critical user path. A redirection delay of >50ms feels sluggish. The p99 latency target for the read path should be **< 15ms**.
* **Storage Footprint**: At 100M URLs created per year:
  * Each record ~500 bytes (ID + Short Code + Long URL + Timestamp + Metadata).
  * Storage growth: ~50 GB / year (well within a single PostgreSQL instance, but index memory footprint must fit in RAM).

---

## 2. High-Level Architecture & Traffic Flow

```
                                +-------------------+
                                |    Client / Web   |
                                +---------+---------+
                                          |
                      +-------------------+-------------------+
                      |                                       |
             (Write Path: GraphQL)                   (Read Path: HTTP GET)
                      |                                       |
                      v                                       v
         +--------------------------+           +--------------------------+
         | Next.js API Route        |           | Next.js Dynamic Route    |
         | (/api/graphql)           |           | (/[shortCode])           |
         +------------+-------------+           +-------------+------------+
                      |                                       |
           1. Insert ID (Seq)                                 | 1. Cache-Aside Check
           2. Encode Base62                                   v
                      |                         +--------------------------+
                      |                         |     Redis Cache Tier     |
                      |                         +-------------+------------+
                      |                                       |
                      |                        [Hit] Return URL   [Miss] Query DB
                      |                               |                  |
                      v                               v                  v
         +-----------------------------------------------------------------+
         |                 PostgreSQL Persistent Storage                  |
         |               (B-Tree Index on `short_code`)                   |
         +-----------------------------------------------------------------+
```

---

## 3. Core System Design Pillars

### Pillar A: ID Generation & Short-Code Strategy
There are two primary paradigms for generating short codes:

| Technique | Pros | Cons | Recommendation |
| :--- | :--- | :--- | :--- |
| **Hash Truncation** (MD5/SHA256 first 7 chars) | Stateless, deterministic. | **Collisions guaranteed** (requires collision detection loops & DB retries). | ❌ Inefficient at scale |
| **Random String Generation** | Simple to implement. | High collision probability as keyspace fills up; requires index lookups before writing. | ❌ High write amplification |
| **Counter + Base62 Encoding** | **100% collision-free**, mathematically deterministic, $O(1)$ generation. | Predictable sequences if not obfuscated (enumeration risk). | ✅ **Industry Standard** |

#### Why Base62?
Characters: `0-9`, `a-z`, `A-Z` (62 alphanumeric characters).
* With **6 characters**: $62^6 \approx 56.8 \text{ Billion}$ unique URLs.
* With **7 characters**: $62^7 \approx 3.52 \text{ Trillion}$ unique URLs.

```typescript
// Deterministic Base10 to Base62 conversion
const CHARSET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

export function encodeBase62(id: bigint | number): string {
  if (id === 0n || id === 0) return CHARSET[0];
  let n = BigInt(id);
  let result = "";
  const base = 62n;

  while (n > 0n) {
    const remainder = Number(n % base);
    result = CHARSET[remainder] + result;
    n = n / base;
  }
  return result;
}
```

---

### Pillar B: Database Architecture & Indexing Strategy (PostgreSQL)

```prisma
model Url {
  id          BigInt   @id @default(autoincrement())
  originalUrl String   @db.Text
  shortCode   String   @unique @db.VarChar(10)
  clicks      Int      @default(0)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([shortCode], type: BTree)
  @@map("urls")
}
```

#### Senior Engineering Nuances:
1. **Primary Key as `BigInt`**: A standard 32-bit `Int` tops out at ~2.1 billion entries. `BigInt` (64-bit) supports up to $9 \times 10^{18}$ records.
2. **B-Tree Indexing**: A unique B-Tree index on `shortCode` ensures $O(\log N)$ lookup on cache misses. Ensure `pg_stat_statements` monitors buffer cache hit rates so indexes remain in memory.
3. **HTTP 301 vs. 302 Semantics**:
   * **301 (Moved Permanently)**: Browsers cache the redirect locally. Subsequent requests bypass our server entirely.
     * *Trade-off*: Reduces server load to near-zero, but **kills analytics/telemetry** and makes URL expiration impossible.
   * **302 (Found) / 307 (Temporary Redirect)**: Browser always hits our server.
     * *Trade-off*: Slightly higher server load, but enables accurate click counting, geolocation analytics, and abuse monitoring.

---

### Pillar C: Caching Tier & Resilience (Redis)

URL traffic follows the **80/20 Pareto principle**: 20% of the links generate 80% of the redirect traffic. Redis is deployed in front of PostgreSQL using the **Cache-Aside (Lazy Loading)** pattern.

#### Read Path Algorithm:
```typescript
async function resolveUrl(shortCode: string): Promise<string | null> {
  const cacheKey = `url:${shortCode}`;
  
  // 1. Fast Path: Read from Redis
  const cachedUrl = await redis.get(cacheKey);
  if (cachedUrl) {
    // Fire-and-forget asynchronous click increment
    queueClickEvent(shortCode);
    return cachedUrl;
  }

  // 2. Slow Path: Read from PostgreSQL
  const record = await prisma.url.findUnique({
    where: { shortCode },
    select: { originalUrl: true }
  });

  if (!record) {
    // Prevent Cache Penetration: Cache negative result with short TTL (e.g., 60s)
    await redis.set(cacheKey, "", "EX", 60);
    return null;
  }

  // 3. Hydrate Redis with TTL Jitter (e.g. 24h +/- random 10m to prevent Cache Avalanche)
  const ttl = 86400 + Math.floor(Math.random() * 600);
  await redis.set(cacheKey, record.originalUrl, "EX", ttl);

  queueClickEvent(shortCode);
  return record.originalUrl;
}
```

---

## 4. How to Start: Step-by-Step Local Deployment

### Step 1: Start Infrastructure Containers
Ensure Docker Desktop is running, then spin up PostgreSQL and Redis:
```bash
docker compose up -d
```
Verify containers are healthy:
```bash
docker ps
```

### Step 2: Configure Environment Variables
Create `.env` in the project root:
```env
# Database (PostgreSQL)
DATABASE_URL="postgresql://postgres:password123@localhost:5432/url_shortener?schema=public"

# Redis Cache
REDIS_URL="redis://localhost:6379"

# Application Settings
NEXT_PUBLIC_BASE_URL="http://localhost:3000"
```

### Step 3: Run Database Migrations
Push the schema to PostgreSQL:
```bash
npx prisma db push
# Or generate migration history:
# npx prisma migrate dev --name init
```

### Step 4: Start Development Server
```bash
npm run dev
```
Navigate to `http://localhost:3000`.

---

## 5. Senior-Level Evolution: "How to Improve Everything"

If scaling this system to **100,000+ requests per second**, here is the architectural roadmap:

### 1. Decoupled Asynchronous Analytics (Event-Driven Click Tracking)
* **The Anti-Pattern**: Doing `UPDATE urls SET clicks = clicks + 1 WHERE id = ...` inside the redirect handler. This introduces database write-lock contention and destroys read latency.
* **The Solution**: Push click events to a high-speed buffer (**Redis Streams** or **Apache Kafka**). A worker batch-processes and flushes counts to PostgreSQL or ClickHouse every 10 seconds.

### 2. Guarding Against Edge-Case Failures
* **Cache Penetration (DDoS with non-existent keys)**:
  * Deploy a **Bloom Filter** in Redis. If the Bloom filter says a key does not exist, reject the request immediately without touching PostgreSQL.
* **Cache Stampede / Thundering Herd**:
  * Implement **Mutex Locks** (Redis Redlock or single-flight promise pattern) so only 1 worker queries PostgreSQL on a cold cache miss.

### 3. Enumeration Attack Prevention (Security)
* Sequential Base62 IDs (`1 -> "1"`, `2 -> "2"`) allow malicious actors to scrape every single URL in your database.
* **Solution**: Use a **Feistel Cipher** or pseudo-random permutation over 64-bit integers. It maintains a 1-to-1 unique mapping (no collisions) while making consecutive IDs appear completely random:
  * ID `1` $\rightarrow$ `849204128` $\rightarrow$ `Base62("Z4k9x")`
  * ID `2` $\rightarrow$ `194012941` $\rightarrow$ `Base62("a8P2m")`

### 4. Edge Redirection Tier (Global Latency Optimization)
* Deploy the redirect resolver onto **Edge Workers (Cloudflare Workers or Vercel Edge)**.
* Replicate Redis globally via **Upstash** or **AWS MemoryDB for Redis Global Datastore**.
* Users in Tokyo, Frankfurt, or London resolve redirects from edge cache in **< 5ms** without touching your origin server.

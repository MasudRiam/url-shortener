# High-Scale URL Shortener (System Design Showcase)

A production-grade, high-throughput URL shortener built with **Next.js 15+ (App Router)**, **GraphQL**, **PostgreSQL (Prisma)**, and **Redis**.

This project is built from the ground up to demonstrate real-world **System Design principles**: distributed ID generation, database indexing, caching strategies, and latency optimization.

---

## 📖 System Architecture & Engineering Deep-Dive
For an in-depth breakdown of the architecture, algorithmic trade-offs, and scalability roadmap, read the full engineering blueprint:
👉 **[SYSTEM_DESIGN.md](./SYSTEM_DESIGN.md)**

---

## ⚡ Quick Start

### 1. Prerequisites
- [Docker & Docker Desktop](https://www.docker.com/) installed and running
- [Node.js 20+](https://nodejs.org/) installed

### 2. Infrastructure Containers
Spin up PostgreSQL and Redis locally:
```bash
docker compose up -d
```

### 3. Environment Configuration
Create a `.env` file in the root directory:
```env
DATABASE_URL="postgresql://postgres:password123@localhost:5432/url_shortener?schema=public"
REDIS_URL="redis://localhost:6379"
NEXT_PUBLIC_BASE_URL="http://localhost:3000"
```

### 4. Database Setup
Push schema and generate the Prisma client:
```bash
npx prisma db push
```

### 5. Run the Application
```bash
npm run dev
```
Open [http://localhost:3000](http://localhost:3000) to view the application.

---

## 🛠️ Tech Stack & Key Concepts

| Layer | Technology | Key System Design Concept |
| :--- | :--- | :--- |
| **API** | GraphQL & Next.js Route Handlers | Declarative queries for management; dedicated fast-path route for HTTP 302 redirects. |
| **ID Encoding** | Base62 Bijective Mapping | Deterministic, collision-free short code generation from 64-bit integer sequences. |
| **Database** | PostgreSQL + Prisma ORM | Persistent storage with B-Tree indexing on `short_code` for $O(\log N)$ point lookups. |
| **Cache** | Redis (`ioredis`) | Cache-Aside pattern, TTL jitter to prevent cache avalanches, sub-millisecond redirect lookups. |
| **Containers** | Docker Compose | Local multi-service orchestration for DB and Caching layers. |

# 🎫 TicketCore: High-Load Event Booking API

A backend system built to solve specific distributed system challenges: Race Conditions,
Double Spending, and Data Consistency under heavy load.

---

## 🛠 Tech Stack

* **Backend:** Python 3.12, FastAPI, Pydantic V2
* **Database:** PostgreSQL 16 (Asyncpg, SQLAlchemy 2.0)
* **Caching & Locks:** Redis 7
* **Background Jobs:** Arq (Redis-based worker)
* **Infra & DevOps:** Docker Compose, Kubernetes (EKS), AWS (VPC, ECR), Terraform (IaC),
  GitHub Actions
* **Observability:** Prometheus, Grafana
* **Testing:** Pytest (Unit/Integration), Locust (Stress testing)

--- 

## 🧠 Deep Dive

### 1. O(1) Cache Invalidation

* **Problem:** Invalidating paginated API caches using Redis `KEYS *` is an O(N)
  blocking operation.
* **Solution:** Implemented **Versioned Keys**. Appended a global version integer to
  cache keys; mutations trigger an atomic `INCR` on the version, instantly and logically
  invalidating all old caches without blocking the single Redis thread.

### 2. Flash Sale Gatekeeping (Race Conditions)

* **Problem:** Massive concurrent traffic (e.g., 10,000 users competing for 100 tickets)
  exhausts PostgreSQL connection pools via lock contention.
* **Solution:** Offloaded atomic inventory decrements to **Redis Lua scripts** before DB
  interaction. Acts as a shock absorber—rejecting sold-out requests strictly in-memory,
  preserving database CPU for successful writes.

### 3. API Idempotency (Double-Spend Prevention)

* **Problem:** Network retries on payment/booking endpoints result in duplicate
  transactions and double charges.
* **Solution:** Engineered a custom `@idempotent` FastAPI decorator backed by **Redis
  distributed locks (NX)**. Hashes request payloads and headers to guarantee
  strictly-once execution, serving cached responses for retry bursts.

### 4. Distributed State Reconciliation (Hung Reservations)

* **Problem:** The Dual-Write problem. If a Postgres transaction rolls back (or a user
  abandons payment), ghost locks remain in Redis, permanently blocking inventory.
* **Solution:** Implemented a **Garbage Collection Cron (Eventual Consistency)** via
  Arq. Periodically sweeps Redis active reservations against PostgreSQL committed
  states (with a 60s grace period) to idempotently restore orphaned inventory.

### 5. Strict Pessimistic Locking

* **Problem:** Ensuring absolutely zero overselling at the persistence layer during
  concurrent transactions.
* **Solution:** Enforced strict row-level isolation utilizing
  `SELECT ... FOR UPDATE SKIP LOCKED` during critical booking and payment flows.

### 6. Read-Aside Architecture

* **Problem:** Read-heavy event catalog queries threatening to overwhelm the relational
  database.
* **Solution:** Implemented a robust caching layer, successfully offloading 90%+ of
  `GET /events` traffic to Redis, reserving PostgreSQL capacity exclusively for
  transactional writes.

### 7. FinOps, IaC & Hardware Observability

* **Problem:** Running a 24/7 managed AWS infrastructure (EKS, RDS, ElastiCache) is
  cost-prohibitive, and theoretical RPS claims lack verifiable hardware proof.
* **Solution:** Engineered **Ephemeral Environments** via **Terraform**. Provisioned a
  5-node EKS cluster deploying Postgres/Redis internally to bypass managed DB costs.
  Integrated **Prometheus & Grafana** to scrape custom metrics, hardware-backing
  benchmarks.

---

## 📊 Performance Testing (Locust)

Stress-tested simulating a "Flash Sale" event to validate data consistency and
transaction isolation under load.

### 🧪 Test Environment & Parameters

The test was executed locally:

* **Hardware:** MacBook Air M1
* **Application Server:** Uvicorn running `8` workers (`--workers 8`)
* **Load Profile:**
    * **Concurrent Users:** 1,000
    * **Spawn Rate:** 100 users / second
    * **Task Wait Time:** 0.1s - 0.5s
    * **Traffic Mix:** Heavy read traffic on `GET /events` (Cache testing) mixed with
      concurrent `POST /tickets` (Transaction/Lock testing).

### 📈 Benchmark Results

| Metric                   | Result     | 
|:-------------------------|:-----------|
| **Peak Throughput**      | ~2,500 RPS | 
| **Average Throughput**   | ~1,500 RPS | 
| **Error Rate**           | 0.00%      | 
| **Overselling Index**    | 0          | 
| **Idempotency Failures** | 0          | 

*(Results of the Locust test runs are available in the `/load_tests` directory).*

---

## 💻 Getting Started

### Run Locally

```bash
git clone https://github.com/VladyslavSeliukov/TicketCore-High-Load-Event-Booking-System.git
cd ticket-core

# Start infrastructure and app
docker compose up --build -d

# Run DB Migrations  
docker compose exec backend alembic upgrade head
```

* **API Docs (Swagger):** `http://localhost/docs`
* **Run Tests:** `uv run pytest `

---

## 📬 Contact

**Vladyslav Seliukov** - Backend Python Engineer

* [LinkedIn Profile](https://www.linkedin.com/in/vladyslav-seliukov/)
* seliukovvladyslav@gmail.com

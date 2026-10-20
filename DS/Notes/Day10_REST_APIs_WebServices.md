# Day 10: REST APIs, JSON Ingestion & Production Web Services
## HTTP Architecture, OAuth2, Pagination, Resilience, Backoff & Database Persistence

---

# Part 1: REST Architecture & HTTP Protocol Internals

### 1.1 The Core Constraints of REST (Representational State Transfer)
REST is an architectural style designed around six core constraints:
1. **Statelessness:** Every HTTP request from client to server must contain all information required to understand and process the request. No session context is stored on the server between requests.
2. **Client-Server Separation:** User interface concerns are decoupled from data storage concerns.
3. **Uniform Interface:** Resources are identified by standard URIs (nouns, not verbs: `/api/v1/orders`, not `/api/v1/getOrders`).
4. **Cacheability:** Responses must explicitly declare whether they are cacheable via `Cache-Control` headers.

---

### 1.2 HTTP Methods, Safety & Idempotency

| Method | Semantic Purpose | Safe? | Idempotent? | Typical Request Body? |
| :--- | :--- | :--- | :--- | :--- |
| **`GET`** | Retrieve resource representation | **YES** | **YES** | No |
| **`POST`** | Create a new subordinate resource | **NO** | **NO** | Yes (JSON payload) |
| **`PUT`** | Completely replace or create resource at URI | **NO** | **YES** | Yes (Full resource) |
| **`PATCH`** | Partially modify an existing resource | **NO** | **YES** (by standard) | Yes (Delta fields) |
| **`DELETE`**| Remove a resource | **NO** | **YES** | Optional |

- **Safe:** Does not modify the state of the server (read-only).
- **Idempotent:** Making the identical request $N$ times ($N \ge 1$) produces the **exact same server state** as making it once.
  - *Example:* `DELETE /users/42` executed 5 times results in the user being gone; the first call returns `204 No Content` or `200 OK`, subsequent calls return `404 Not Found`, but server state is unchanged. In contrast, `POST /payments` executed 5 times charges the customer 5 times (Non-Idempotent!).

---

### 1.3 HTTP Status Codes Masterclass

```
1xx Informational | 2xx Success | 3xx Redirection | 4xx Client Error | 5xx Server Error
```

| Status Code | Name | Practical Meaning in Data Engineering |
| :--- | :--- | :--- |
| **`200 OK`** | Standard Success | Payload returned successfully. |
| **`201 Created`** | Resource Created | Response to `POST`; contains `Location` header. |
| **`204 No Content`** | Empty Success | Action succeeded (e.g., `DELETE`); zero response body. |
| **`304 Not Modified`** | Cached | Client can use local cache based on `ETag` / `If-None-Match`. |
| **`400 Bad Request`** | Malformed Request | Client syntax error or schema validation failure. |
| **`401 Unauthorized`** | Authentication Missing | Missing or invalid API key/token (Identity unproven). |
| **`403 Forbidden`** | Authorization Refused | Identity known, but client lacks permission to view resource. |
| **`404 Not Found`** | Missing Resource | Endpoint or entity ID does not exist. |
| **`429 Too Many Requests`**| Rate Limited | Exceeded API quota. Inspect `Retry-After` header! |
| **`500 Internal Error`** | Server Crash | Unhandled exception inside the API server. |
| **`502 Bad Gateway`** | Gateway Failure | Reverse proxy (Nginx/Cloudflare) received invalid upstream response. |
| **`503 Unavailable`** | Overloaded / Maintenance | Temporary outage. Safe to retry after delay. |
| **`504 Gateway Timeout`**| Gateway Timed Out | Upstream database or service failed to respond in time. |

---

# Part 2: Authentication & Authorization Mechanics

### 2.1 API Keys & Bearer Tokens
- **API Key:** A static token passed via request headers:
  `Authorization: Api-Key abc123xyz` or `X-API-KEY: abc123xyz`.
- **Bearer Token (JWT):** A cryptographically signed token:
  `Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`
  - A JSON Web Token (JWT) consists of three Base64URL-encoded parts: `Header.Payload.Signature`.
  - Enables **stateless authorization**: the receiving server verifies the signature using a public key without querying a session database.

---

### 2.2 OAuth 2.0: Client Credentials Grant (Machine-to-Machine)
The standard for automated data pipelines and backend ETL ingestion:

```
[ Data Pipeline (Client) ]                [ Authorization Server ]               [ Resource Server (API) ]
           │                                         │                                      │
           │── 1. POST /token (client_id + secret) ─>│                                      │
           │<─ 2. Access Token (expires in 3600s) ───│                                      │
           │                                                                                │
           │────────────────── 3. GET /data with Bearer Token ─────────────────────────────>│
           │<───────────────── 4. JSON Dataset ─────────────────────────────────────────────│
```

---

# Part 3: Advanced Python `requests` Architecture

### 3.1 Connection Pooling via `requests.Session()`
In standard `requests.get()`, a new TCP socket and TLS handshake are negotiated on **every single request**, adding 50–200ms of network latency per call.  
A `requests.Session()` maintains an internal **urllib3 ConnectionPool**, reusing open TCP connections across thousands of calls.

```python
import requests

# ANTI-PATTERN: Opens/closes TCP socket 100 times
# for url in urls: requests.get(url)

# PRODUCTION PATTERN: Reuses TCP sockets across all requests
with requests.Session() as session:
    session.headers.update({"Accept": "application/json", "User-Agent": "DataPlatform/1.0"})
    response = session.get("https://api.example.com/v1/data")
```

---

### 3.2 Timeouts: Connect Timeout vs. Read Timeout
Never call `requests.get()` without an explicit timeout! Without a timeout, a hanging server will cause your Python process to block indefinitely, freezing your entire ETL orchestrator (Airflow, Celery).

```python
# Pass a 2-tuple: (Connect Timeout, Read Timeout)
# - Connect Timeout: Max seconds to establish TCP connection (typically 3.05s).
# - Read Timeout: Max seconds to wait for server to transmit the first byte of data.
response = session.get("https://api.example.com/v1/records", timeout=(3.05, 30.0))
```

---

# Part 4: Production Resilience: Pagination, Rate Limits & Exponential Backoff

### 4.1 Pagination Archetypes

1. **Offset-Limit Pagination:** `GET /users?offset=100&limit=50`
   - *Limitation:* Degrades at high offsets ($O(N)$ database scan); prone to skipped/duplicate records if rows are inserted during pagination.
2. **Page-Number Pagination:** `GET /users?page=3&per_page=50`
3. **Cursor-Based / Keyset Pagination:** `GET /users?cursor=dXNlcl9pZDoxMDQ1&limit=50`
   - *Industry Standard:* Cursor points to a unique immutable row index (e.g., base64 of `last_id`). $O(1)$ database seek; immune to pagination drift.

---

### 4.2 Handling HTTP 429 Rate Limits & Exponential Backoff with Jitter
When receiving HTTP 429 or transient 503 errors, standard linear retries cause the **Thundering Herd Problem** (multiple pipeline workers retry simultaneously, crashing the recovering API).  
The solution is **Exponential Backoff with Full Jitter**:
$$\text{Delay} = \text{random}(0, \min(\text{MaxDelay}, \text{BaseDelay} \times 2^{\text{attempt}}))$$

```python
import time
import random
import requests

def request_with_backoff(session, url, max_retries=5, base_delay=1.0, max_delay=32.0):
    for attempt in range(max_retries):
        try:
            response = session.get(url, timeout=(3.05, 15.0))
            
            # If rate limited:
            if response.status_code == 429:
                retry_after = response.headers.get("Retry-After")
                if retry_after:
                    sleep_time = float(retry_after)
                else:
                    sleep_time = random.uniform(0, min(max_delay, base_delay * (2 ** attempt)))
                print(f"Rate limited (429). Retrying in {sleep_time:.2f}s...")
                time.sleep(sleep_time)
                continue
                
            # If server error (5xx), retry:
            if response.status_code in (500, 502, 503, 504):
                sleep_time = random.uniform(0, min(max_delay, base_delay * (2 ** attempt)))
                print(f"Server error ({response.status_code}). Retrying in {sleep_time:.2f}s...")
                time.sleep(sleep_time)
                continue
                
            response.raise_for_status()
            return response.json()
            
        except (requests.ConnectionError, requests.Timeout) as e:
            if attempt == max_retries - 1:
                raise e
            sleep_time = random.uniform(0, min(max_delay, base_delay * (2 ** attempt)))
            print(f"Network failure: {e}. Retrying in {sleep_time:.2f}s...")
            time.sleep(sleep_time)
            
    raise RuntimeError(f"Exceeded max retries for URL: {url}")
```

---

# Part 5: JSON Normalization & Flat Staging

Nested API payloads must be flattened into tabular rows before loading into relational tables:

```python
import pandas as pd

api_payload = [
    {
        "id": 101,
        "name": "Acme Corp",
        "billing": {"city": "New York", "postal": "10001"},
        "contacts": [{"type": "email", "val": "admin@acme.com"}]
    }
]

# Flattens nested dictionaries with separator
df_flat = pd.json_normalize(
    api_payload, 
    record_path=["contacts"], 
    meta=["id", "name", ["billing", "city"], ["billing", "postal"]],
    sep="_"
)
# Resulting columns: type, val, id, name, billing_city, billing_postal
```

---

# Part 6: Required Deliverable: Complete Production Ingestion & Database Loader

### Problem Statement
Write a self-contained, enterprise-grade Python module that:
1. Authenticates against a paginated REST API endpoint.
2. Iterates across pages using keyset/cursor pagination.
3. Implements resilient retry logic with exponential backoff and jitter.
4. Normalizes and validates incoming records.
5. Performs an idempotent bulk upsert (`INSERT ... ON CONFLICT DO UPDATE`) into a local SQLite/PostgreSQL database.

### Production Solution Script
```python
import sqlite3
import time
import random
import requests
from typing import Generator, Any

class ResilientAPIClient:
    def __init__(self, base_url: str, api_token: str):
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {api_token}",
            "Accept": "application/json",
            "User-Agent": "EnterpriseDataIngestion/2.0"
        })

    def fetch_page_with_retry(self, endpoint: str, params: dict, max_retries: int = 4) -> dict:
        url = f"{self.base_url}{endpoint}"
        for attempt in range(max_retries):
            try:
                response = self.session.get(url, params=params, timeout=(3.05, 10.0))
                
                if response.status_code == 429:
                    delay = float(response.headers.get("Retry-After", 2 ** attempt + random.random()))
                    time.sleep(delay)
                    continue
                    
                if response.status_code >= 500:
                    delay = (2 ** attempt) + random.uniform(0.1, 1.0)
                    time.sleep(delay)
                    continue

                response.raise_for_status()
                return response.json()

            except (requests.ConnectionError, requests.Timeout) as err:
                if attempt == max_retries - 1:
                    raise err
                time.sleep((2 ** attempt) + random.uniform(0.1, 1.0))

        raise RuntimeError("Failed to fetch API page after maximum retries.")

    def stream_all_records(self, endpoint: str) -> Generator[dict, None, None]:
        """Paginates through API records lazily using cursor pagination."""
        cursor = None
        has_more = True

        while has_more:
            params = {"limit": 50}
            if cursor:
                params["cursor"] = cursor

            payload = self.fetch_page_with_retry(endpoint, params)
            data_records = payload.get("data", [])
            for record in data_records:
                yield record

            cursor = payload.get("next_cursor")
            has_more = payload.get("has_more", False) and (cursor is not None)


class DatabaseStagingPipeline:
    def __init__(self, db_path: str = "ingestion_warehouse.db"):
        self.conn = sqlite3.connect(db_path)
        self._init_schema()

    def _init_schema(self):
        with self.conn:
            self.conn.execute("""
                CREATE TABLE IF NOT EXISTS api_orders (
                    order_id TEXT PRIMARY KEY,
                    customer_id TEXT NOT NULL,
                    order_status TEXT NOT NULL,
                    amount REAL NOT NULL,
                    created_at TEXT NOT NULL,
                    ingested_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                );
            """)

    def upsert_batch(self, records: list[dict]):
        """Idempotent upsert: Updates existing records if order_id already exists."""
        sql = """
            INSERT INTO api_orders (order_id, customer_id, order_status, amount, created_at)
            VALUES (?, ?, ?, ?, ?)
            ON CONFLICT(order_id) DO UPDATE SET
                customer_id = excluded.customer_id,
                order_status = excluded.order_status,
                amount = excluded.amount,
                ingested_at = CURRENT_TIMESTAMP;
        """
        formatted_rows = [
            (
                r["id"],
                r["customer_id"],
                r["status"],
                float(r["amount"]),
                r["created_at"]
            )
            for r in records
        ]
        with self.conn:
            self.conn.executemany(sql, formatted_rows)

    def close(self):
        self.conn.close()


def run_pipeline(api_client: ResilientAPIClient, pipeline: DatabaseStagingPipeline):
    batch = []
    batch_size = 100
    total_ingested = 0

    print("Initiating resilient API streaming ingestion...")
    for record in api_client.stream_all_records("/v1/transactions"):
        batch.append(record)
        if len(batch) >= batch_size:
            pipeline.upsert_batch(batch)
            total_ingested += len(batch)
            print(f"Upserted {total_ingested} records into database.")
            batch = []

    if batch:
        pipeline.upsert_batch(batch)
        total_ingested += len(batch)

    print(f"Pipeline complete. Total records ingested: {total_ingested}")
```

---

# Part 7: Structured Interview Question Bank

### Q1: What is the difference between `PUT` and `PATCH` in RESTful APIs?
**Answer:**
- `PUT` is designed for complete replacement of a resource. The client must supply the entire resource representation; any attribute omitted from the request body is typically cleared or reset to default values.
- `PATCH` is designed for partial modification. The client submits only the specific delta fields that need modification, preserving all other existing attributes unchanged. Both methods should be designed to be idempotent.

### Q2: Why is the HTTP 401 status code a misnomer, and how does it differ from 403?
**Answer:**
`401 Unauthorized` is fundamentally an **Authentication** failure, not an authorization failure. It signals that the user's identity has not been verified (missing, malformed, or expired API key/token). In contrast, `403 Forbidden` is an **Authorization** failure: the client's identity is authenticated and verified, but the server refuses to grant access because the identity lacks the required permissions/roles.

### Q3: What is "Keyset Pagination" (Cursor-based) and why is it preferred over "Offset-Limit Pagination" for big data APIs?
**Answer:**
Offset-limit pagination (`OFFSET 500000 LIMIT 50`) forces the underlying database storage engine to scan and discard 500,000 index entries before returning 50 rows ($O(N)$ execution time). Furthermore, if rows are inserted or deleted while the client is paginating, records can be skipped or duplicated across page boundaries. Keyset pagination passes the last observed unique key (`WHERE id > :last_id ORDER BY id LIMIT 50`), allowing the database to execute an instant $O(\log N)$ B-tree index seek that is immune to insertion drift.

---

# Part 8: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   REST APIS & WEB SERVICES CHEAT SHEET                 │
├────────────────────────────────────────────────────────────────────────┤
│ 1. HTTP METHODS & IDEMPOTENCY:                                         │
│    - Safe      : GET (Read-only; zero server mutation).                │
│    - Idempotent: GET, PUT, PATCH, DELETE (N calls = 1 call effect).    │
│    - Non-Idempotent: POST (Creates new resources; duplicates state).   │
├────────────────────────────────────────────────────────────────────────┤
│ 2. STATUS CODES:                                                       │
│    - 200 OK, 201 Created, 204 No Content.                              │
│    - 401 Unauthorized (Auth missing), 403 Forbidden (Auth rejected).   │
│    - 429 Too Many Requests (Rate limited -> Check Retry-After).        │
│    - 500 Server Error, 502 Bad Gateway, 503 Service Unavailable.      │
├────────────────────────────────────────────────────────────────────────┤
│ 3. PRODUCTION REQUESTS:                                                │
│    - requests.Session(): Reuses TCP sockets via connection pooling.    │
│    - Timeouts: Always pass (connect_timeout, read_timeout) tuple!      │
├────────────────────────────────────────────────────────────────────────┤
│ 4. RESILIENCE:                                                         │
│    - Exponential Backoff with Jitter: Prevents thundering herd.        │
│    - Keyset/Cursor Pagination: O(1) DB seek; avoids pagination drift.  │
│    - Idempotent Loader: INSERT ... ON CONFLICT DO UPDATE.              │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 9: 100-Mark Self-Assessment Examination

### Section A: Conceptual Foundations (20 Marks — 4 marks each)

**Q1:** Which of the following HTTP methods is NOT idempotent by definition?  
(a) GET  
(b) PUT  
(c) POST  
(d) DELETE  

**Q2:** When an API responds with HTTP status code `429 Too Many Requests`, which response header specifies how many seconds the client should wait before retrying?  
(a) `Keep-Alive`  
(b) `Retry-After`  
(c) `RateLimit-Reset`  
(d) `X-Throttle-Delay`  

**Q3:** What is the consequence of omitting a timeout argument in `requests.get("https://...")`?  
(a) Defaults to a 30-second timeout.  
(b) Defaults to a 60-second timeout.  
(c) The request will block indefinitely if the remote server fails to respond.  
(d) Throws a `TimeoutConfigurationError`.  

**Q4:** What are the three dot-separated components of a JSON Web Token (JWT)?  
(a) Nonce, Key, Ciphertext  
(b) Header, Payload, Signature  
(c) Issuer, Subject, Expiration  
(d) ClientID, Secret, Scope  

**Q5:** Why does `requests.Session()` execute faster than individual `requests.get()` calls when making 1,000 sequential requests to the same domain?  
(a) It compresses all payloads using gzip.  
(b) It reuses existing TCP and TLS handshakes via internal connection pooling.  
(c) It caches all GET responses on local disk.  
(d) It executes requests concurrently in multiple threads.  

---

### Section B: Protocol & Architecture Analysis (25 Marks)

**Q6 (10 Marks):** Explain why executing a retry loop without "Jitter" (randomness) can crash a recovering web service (The Thundering Herd Problem).

**Q7 (15 Marks):** Compare Offset-Limit pagination with Cursor-based pagination across two dimensions:
1. Performance as page depth increases ($N \to 1,000,000$).
2. Data consistency when real-time records are inserted while paginating.

---

### Section C: Implementation Challenges (35 Marks)

**Q8 (15 Marks):** Write a Python function `get_auth_token(token_url: str, client_id: str, client_secret: str) -> str` that performs an OAuth 2.0 Client Credentials `POST` request and returns the `access_token` string. Raise an informative error if the request fails.

**Q9 (20 Marks):** Given a deeply nested JSON response:
```json
{
  "company": "TechGlobal",
  "departments": [
    {"name": "Engineering", "budget": 500000},
    {"name": "Data", "budget": 350000}
  ]
}
```
Write the Pandas code using `pd.json_normalize()` to flatten this structure into a DataFrame with columns `['company', 'dept_name', 'budget']`.

---

### Section D: Real-World Ingestion Architecture (20 Marks)

**Q10 (20 Marks):** You are designing a batch ingestion pipeline that pulls 5 million daily financial transactions from an external vendor API. The vendor enforces a hard rate limit of 100 requests per second.
1. Design the architecture (Workers, Queue, Database) to maximize throughput without exceeding the 100 req/sec quota. (10 marks)
2. How do you guarantee **Idempotent Ingestion** so that running the pipeline twice for the same date does not produce duplicate financial records in the warehouse? (10 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(c)** POST.
- **Q2:** **(b)** `Retry-After`.
- **Q3:** **(c)** Blocks indefinitely until the socket times out at the OS level (which may take hours or never happen).
- **Q4:** **(b)** Header, Payload, Signature.
- **Q5:** **(b)** TCP and TLS connection reuse.

### Section B Solutions
**Q6 Solution:**
When an API recovers from an outage, thousands of client workers using fixed exponential delays (e.g., exactly 2s, 4s, 8s) retry at the exact same synchronized clock interval. This produces massive, periodic traffic spikes ("Thundering Herd") that immediately overload and crash the recovering service again. Adding **Full Jitter** randomizes delays uniformly ($[0, \text{backoff}]$), smoothing traffic into a continuous, manageable stream.

**Q7 Solution:**
1. **Performance:** Offset pagination degrades to $O(N)$ because the database must scan and discard all preceding offset rows. Cursor pagination uses an indexed range filter (`WHERE id > :cursor`), maintaining $O(\log N)$ performance regardless of depth.
2. **Consistency:** If a new row is inserted on page 1 while the client is on page 3, offset pagination shifts all rows down, causing the client to see a duplicate record on page 4. Cursor pagination anchors directly to immutable entity keys, eliminating duplicate or skipped rows.

### Section C Solutions
**Q8 Solution:**
```python
import requests

def get_auth_token(token_url: str, client_id: str, client_secret: str) -> str:
    data = {
        "grant_type": "client_credentials",
        "client_id": client_id,
        "client_secret": client_secret
    }
    response = requests.post(token_url, data=data, timeout=(3.05, 10.0))
    if not response.ok:
        raise RuntimeError(f"Auth failed [{response.status_code}]: {response.text}")
    return response.json()["access_token"]
```

**Q9 Solution:**
```python
import pandas as pd

raw_data = {
  "company": "TechGlobal",
  "departments": [
    {"name": "Engineering", "budget": 500000},
    {"name": "Data", "budget": 350000}
  ]
}

df = pd.json_normalize(
    raw_data,
    record_path=["departments"],
    meta=["company"],
    record_prefix="dept_"
)
# Resulting columns: dept_name, dept_budget, company
```

### Section D Solutions
**Q10 Solution:**
1. **Architecture:** Use a token-bucket rate limiter (e.g., Redis-backed Leaky Bucket) shared across Celery/Airflow worker pools. Workers poll tasks from an ingestion queue and request a token from Redis before dispatching HTTP calls. If tokens are depleted, workers sleep, strictly capping cluster throughput to 95–98 requests/sec to stay safely under the 100 req/sec limit.
2. **Idempotency Guarantee:**
   - Enforce a natural or composite unique business key (e.g., `vendor_transaction_id` or `(account_id, txn_timestamp, amount)`) at the database schema level (`UNIQUE` constraint).
   - Ingest data into a staging table and use `INSERT ... ON CONFLICT (vendor_transaction_id) DO UPDATE` or atomic staging swap (`MERGE INTO target USING staging`). Re-running the pipeline seamlessly updates existing records without creating duplicates.

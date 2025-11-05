# Web Crawler System Design

## 🎯 Problem Statement

Design a web crawler to extract text data from the web for training LLMs (GPT-4, Gemini, LLaMA).

### Use Cases
- Search engine indexing (Google, Bing)
- LLM training data collection
- Web monitoring and research

### Basic Flow
```
Seed URLs → Web Crawler → Text Data
```

---

## 📋 Requirements

### Functional Requirements

**In Scope:**
- Crawl web starting from seed URLs
- Extract and store text data

**Out of Scope:**
- LLM training/processing
- Non-text data (images, videos)
- Dynamic JavaScript-rendered content
- Authentication handling

### Non-Functional Requirements

**Scale Assumptions:**
- 10 billion pages total
- 2MB average page size
- Complete within 5 days

**In Scope:**
- **Fault Tolerance** - Handle failures gracefully, resume without data loss
- **Politeness** - Respect robots.txt and rate limits
- **Efficiency** - Complete crawl in < 5 days
- **Scalability** - Handle 10B+ pages

**Out of Scope:**
- Security, cost optimization, legal compliance

---

## 🏗️ System Interface & Data Flow

### System Interface

**Input:** Seed URLs (starting points)  
**Output:** Extracted text data for LLM training

### Data Flow (6 Steps)

1. Get seed URL from frontier → Request IP from DNS
2. Fetch HTML from web server using IP
3. Extract text data from HTML
4. Store text data in blob storage
5. Extract linked URLs from page
6. Add new URLs to frontier → Repeat

---

## 🎨 High-Level Design

### Core Components

**1. Frontier Queue (SQS)**

- Stores URLs to be crawled
- Starts with seed URLs
- Receives new URLs discovered during crawling

**2. Crawler Workers**

- Fetch HTML from webpages
- Extract text data → Store in S3
- Extract URLs → Add to Frontier Queue
- Query DNS for IP addresses

**3. DNS (External)**

- Resolves domain names to IP addresses
- Cached to reduce lookups

**4. Webpages (External)**

- Web servers hosting pages to crawl

**5. S3 Storage**

- Stores extracted text data
- Scalable blob storage

### Architecture Diagram

```text
                          ┌─────────┐         ┌──────────┐
                          │   DNS   │         │ Webpage  │
                          └────▲────┘         └────▲─────┘
                               │                   │
        ┌──────────────────────┼───────────────────┼──────────┐
        │                      │                   │          │
        │                      └──────┐   ┌────────┘          │
        │                             │   │                   │
        │  ┌──────────────┐    ┌──────▼───▼──────┐           │
        │  │   Frontier   │───▶│    Crawler      │           │
        │  │    Queue     │    │  - Fetch HTML   │───┐       │
        │  └──────▲───────┘    │  - Extract text │   │       │
        │         │            │  - Extract URLs │   │       │
        │         │            └─────────────────┘   │       │
        │         │                                  │       │
        │         │ Add new URLs                     ▼       │
        │         └──────────────────────────  ┌──────────┐  │
        │                                      │ S3: Text │  │
        │                                      │   Data   │  │
        │                                      └──────────┘  │
        └──────────────────────────────────────────────────────┘
```

---

## 🔍 Deep Dive: Design Challenges

### 1️⃣ Fault Tolerance - Multi-Stage Pipeline

**Problem:** Single crawler doing too much → failure loses all progress

**Solution:** Break into pipelined stages with queues between each stage

#### Pipeline Architecture

```text
┌──────────┐     ┌──────────────┐     ┌──────────┐
│ Frontier │────▶│ URL Fetchers │────▶│ Fetched  │
│  Queue   │     │  (Workers)   │     │  Queue   │
└──────────┘     └──────┬───────┘     └────┬─────┘
                        │                   │
                        ▼                   ▼
                  ┌──────────┐      ┌──────────────┐
                  │ S3: Raw  │      │ Text & URL   │
                  │   HTML   │      │  Extraction  │
                  └──────────┘      │  (Workers)   │
                                    └──────┬───────┘
                                           │
                         ┌─────────────────┼─────────────┐
                         ▼                 ▼             ▼
                   ┌──────────┐      ┌──────────┐  ┌──────────┐
                   │ S3: Text │      │ Frontier │  │ Metadata │
                   │   Data   │      │  Queue   │  │    DB    │
                   └──────────┘      └──────────┘  └──────────┘
```

#### Key Benefits

- **Isolate failures** - Each stage fails independently without losing progress
- **Retry without rework** - Failed URL fetch doesn't lose extracted data
- **Independent scaling** - Scale fetchers vs parsers based on bottlenecks
- **Easy updates** - Change extraction logic without re-fetching pages

#### Handling Failures

**URL Fetch Failures:**

- SQS retries with exponential backoff (30s → 2m → 5m → 15m)
- After 5 retries → Move to Dead Letter Queue (DLQ)
- Site considered offline/unscrapable

**Worker Crashes:**

- **SQS:** Message stays in queue with visibility timeout. If worker crashes, message reappears for another worker
- **Kafka:** Offsets track progress. Next worker resumes from last successful offset

#### Metadata Database (DynamoDB)

Tracks:

- URL status (pending/completed/failed)
- Blob storage links
- Processing state
- Last crawl timestamp

---

### 2️⃣ Politeness - Respect Website Rules

**What is Politeness?**  
Don't overload servers, respect site rules, be a good web citizen

#### robots.txt

Example:

```text
User-agent: *
Disallow: /private/
Crawl-delay: 10
```

**Implementation:**

1. Fetch and cache robots.txt for each domain
2. Before crawling, check:
   - Is URL disallowed? → Skip and acknowledge message
   - Check crawl-delay since last request
   - If too soon → Requeue with DelaySeconds
3. Update last crawl timestamp in Metadata DB

#### Rate Limiting (1 request/second per domain)

**Technology:** Redis with sliding window algorithm

**Flow:**

```text
Crawler → Redis (Check rate) → Website
```

**Features:**

- Global rate limiting across all crawlers
- Track requests/second per domain
- Add **jitter** (random delay) to prevent thundering herd

---

### 3️⃣ Scale & Efficiency - 10B Pages in 5 Days

#### Network Bandwidth Calculation

```text
AWS optimized instance: 400 Gbps
Theoretical max: 400 Gbps ÷ 8 ÷ 2MB = 25,000 pages/sec

Realistic (30% utilization): 7,500 pages/sec
  → Accounts for: latency, DNS, rate limits, retries

Single machine: 10B ÷ 7,500 = 15.4 days
4 machines: 15.4 ÷ 4 = 3.85 days ✅
```

**Parser Workers:** Auto-scale with queue depth (Lambda/ECS)

#### DNS Optimization

**Potential Bottleneck:** Too many DNS lookups

**Solutions:**

1. **DNS Caching** - Cache lookups per domain in crawlers
2. **Multiple DNS Providers** - Round-robin to distribute load

#### Avoid Duplicate Work

**Challenge:** Different URLs → Same content

**Solution: Content Hash Index**

- Hash page content (MD5/SHA-256)
- Index hashes in Metadata DB
- Check before processing extracted text
- Simple and efficient with modern DB indexes

**Alternative:** Bloom filter (space-efficient but probabilistic, likely overkill)

#### Prevent Crawler Traps

**Problem:** Pages linking to themselves infinitely

**Solution:** Max depth limit tracked in Metadata DB

```text
URL depth counter → Increment on each link follow
If depth > threshold → Stop crawling
```

---

## 🎯 Final Architecture

```text
                    ┌────────────────────────────────────┐
                    │      System Boundary              │
                    │                                    │
  ┌──────────┐     │  ┌──────────┐    ┌──────────────┐ │
  │  Seed    │────▶│  │ Frontier │───▶│ URL Fetchers │ │
  │  URLs    │     │  │  Queue   │    │ (Scaled 4x)  │ │
  └──────────┘     │  │  (SQS)   │    └──────┬───────┘ │
                    │  └────▲─────┘           │         │
                    │       │                 ▼         │
                    │       │          ┌────────────┐   │  ┌──────────┐
                    │       │          │    DNS     │◀──┼─▶│ External │
                    │       │          │  (Cached)  │   │  │ Websites │
                    │       │          └────────────┘   │  └──────────┘
                    │       │                 │         │
                    │  ┌────┴──────┐          ▼         │
                    │  │ Metadata  │    ┌──────────┐    │
                    │  │    DB     │    │ S3: Raw  │    │
                    │  │ ───────── │    │  HTML    │    │
                    │  │ • URLs    │    └────┬─────┘    │
                    │  │ • Hashes  │         │          │
                    │  │ • Domains │         ▼          │
                    │  │ • Robots  │   ┌──────────────┐ │
                    │  └────▲──────┘   │   Fetched    │ │
                    │       │          │    Queue     │ │
                    │       │          └──────┬───────┘ │
                    │  ┌────┴──────┐          │         │
                    │  │  Redis    │          ▼         │
                    │  │ (Rate     │   ┌──────────────┐ │
                    │  │  Limit)   │   │  Text & URL  │ │
                    │  └───────────┘   │  Extraction  │ │
                    │                  │ (Auto-scale) │ │
                    │                  └──────┬───────┘ │
                    │                         │         │
                    │                         ▼         │
                    │                  ┌──────────┐     │
                    │                  │ S3: Text │     │
                    │                  │   Data   │     │
                    │                  └──────────┘     │
                    └────────────────────────────────────┘
```

---

## 💡 Additional Considerations

### Dynamic Content (JavaScript-rendered pages)

**Challenge:** Content loaded by React/Angular not in initial HTML  
**Solution:** Use headless browser (Puppeteer/Playwright) to render pages before extracting

### System Monitoring

**Tools:** Datadog, New Relic, CloudWatch

**Key Metrics:**

- Crawler throughput (pages/sec)
- Queue depth (backlog size)
- Error rates and types
- Latency per stage
- DNS cache hit rate

### Large File Handling

**Strategy:** Check `Content-Length` header before download  
**Action:** Skip files exceeding size threshold (e.g., > 10MB)

### Continual Updates

**Use Case:** Re-crawl for monthly LLM training updates or search engine freshness

**Solution:** Add URL Scheduler component

```text
┌──────────────┐
│ URL Scheduler│
│ - Tracks last│
│   crawl time │
│ - Priorities │
│   by update  │
│   frequency  │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Frontier    │
│   Queue      │
└──────────────┘
```

---

## 📚 Key Takeaways

### Design Patterns

1. **Pipeline Architecture** - Break complex processes into stages for fault isolation
2. **Queue-Based Communication** - Decouple components for scalability
3. **Content Deduplication** - Use hashing to avoid processing duplicates
4. **Politeness First** - Respect robots.txt and implement rate limiting
5. **Horizontal Scaling** - Calculate requirements and scale linearly

### Technology Choices

- **Queues:** SQS (built-in retry + DLQ)
- **Storage:** S3 (scalable blob storage)
- **Database:** DynamoDB (metadata tracking)
- **Rate Limiting:** Redis (global distributed limiting)
- **Caching:** DNS caching, robots.txt caching

### Scale Calculations

- Start with realistic assumptions (30% network utilization)
- Account for overhead (DNS, retries, rate limits)
- Calculate both bandwidth and processing needs
- Plan for failures and retries

### Interview Tips

- **Start simple** - Basic design first, then iterate
- **Use requirements as roadmap** - Each non-functional requirement = design challenge
- **Show practical thinking** - Multiple DNS providers, jitter for rate limiting
- **Communicate tradeoffs** - Content hash vs Bloom filter
- **Calculate realistically** - Don't assume 100% utilization

---

## 🔗 Related System Designs

- **Search Engine** - Extends crawler with indexing and ranking
- **Distributed Task Queue** - Similar pipeline patterns
- **Content Delivery Network** - Caching and distribution strategies
- **Rate Limiter** - Deep dive into rate limiting algorithms


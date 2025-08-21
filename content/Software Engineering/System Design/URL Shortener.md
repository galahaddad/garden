---
tags:
  - systemdesign
---

# URL Shortener System Design – Blended Cheat Sheet (ByteByteGo-style)

  

This cheat sheet blends the common core concepts taught in popular URL shortener system-design walkthroughs (capacity planning, API shape, ID strategies, caching tiers, sharding, and multi‑region considerations) with the practical tactics we discussed (hot-key protection, async analytics, backpressure). It’s structured as: **high-level → diagrams → deep dives**, with cross-links.

  

---

  

## High-Level Architecture

  

```mermaid

flowchart LR

A[Client] --> B[CDN/Edge]

B --> C[API Gateway]

C --> D{L1+L2 Cache L1 in-process,L2 Redis/Memcached}

D -- hit --> E[302 Redirect]

D -- miss --> F[(Primary KV/SQL DB)]

F --> D

E --> G[Browser follows to Long URL]

C -. async .-> H[Emitter bounded queue]

H --> I[[Kafka / PubSub]]

I --> J[Stream Processing<br/>Flink/Streams]

J --> K[(Analytics Store<br/>ClickHouse/BigQuery)]

```

  

**Key ideas (ByteByteGo-style essentials):**

- **Clarify requirements:** create, redirect, custom alias, optional expiration, analytics, high availability, low latency.

- **Capacity planning (assume to size):** target QPS, reads ≫ writes, storage growth, hot-link behavior, multi‑region needs.

- **API surface:** `POST /shorten`, `GET /:code`, optional `PUT /:code` (update), `DELETE /:code`, `GET /:code/stats`.

- **Data model:** `code (PK)`, `long_url`, `created_at`, `owner_id`, `ttl/expiry`, optional `checksum`, `flags`.

  

Jump to: [ID Generation](#unique-id-generation), [Redirect Flow](#redirect-flow), [Caching](#caching-strategy), [Sharding](#storage--sharding), [Hot Keys](#scaling--hot-key-protection), [Analytics & Backpressure](#analytics--backpressure).

  

---

  

## URL Shortening – Deep Dive

  

```mermaid

sequenceDiagram

participant Client

participant API as API Service

participant ID as ID Generator

participant DB as DB (KV/SQL)

participant Cache as L2 Cache (Redis)

Client->>API: POST /shorten {long_url, alias?, ttl?}

API->>API: validate/normalize URL, policy checks

API->>ID: generate code (Counter/Snowflake/Random/KGS)

ID-->>API: code

API->>DB: INSERT (code, long_url, meta) UNIQUE(code)

DB-->>API: OK (retry on conflict if random)

API->>Cache: SET s:code -> long_url (TTL per policy)

API-->>Client: 201 {short_url}

```

  

**Notes:**

- Validate/normalize (http→https? trailing slashes? UTM handling).

- Enforce domain allow/deny lists, phishing/malware scans (async if needed).

- Dedup optional (checksum of long_url ⇒ “return existing code”).

  

---

## Unique ID Generation

^2654e9

  


- **Base62/Base64 = encoding, not uniqueness.**

- Strategies:

	- **Counter + encode**: short, predictable, no collisions.
	
	- **Snowflake IDs**: distributed, collision-free, ~11 chars.
	
	- **Random tokens**: 72–96 bits + unique index, retried on collision.
	
	- **KGS**: pre-mint short codes, lease atomically.

  

- Deep Dive:

	- **Counter+encode** → simplest, but predictable.
	
	- **Snowflake** → distributed, safe at scale.
	
	- **Random+unique index** → best for privacy/security.
	
	- **KGS** → ensures short vanity codes stay unique.


---

## Redirect Flow

  

```mermaid

flowchart LR

A[GET /:code] --> B{L1 Cache?}

B -- hit --> C[302 with Location]

B -- miss --> D{L2 Cache?}

D -- hit --> C

D -- miss --> E[(DB Lookup by code)]

E --> D

C --> F[Browser → Long URL]

A -. non-blocking .-> G[Emit analytics event]

```

  

**302 vs 301:** Prefer **302** (temporary) so you can change/expire links later. Control CDN/browser caching via `Cache-Control` headers.

  

---

  

## Hash + Collision Resolution

  

```mermaid

flowchart TB

subgraph Options

X1[Counter + Base62] -->|"No collisions<br/>(shortest codes)"| Y1[Pros]

X2[Snowflake → Base62] -->|"Distributed, unique"| Y2[Pros]

X3[Random k-bit → Base62/64] -->|"Check-and-insert<br/>retry on conflict"| Y3[Pros]

X4["KGS (pre-minted pool)"] -->|"Guaranteed unique<br/>for very short slugs"| Y4[Pros]

end

  

subgraph Collision Handling

R1["Random token<br/>INSERT with UNIQUE(code)"]

R1 -->|on conflict| R2[Regenerate & retry]

R1 -->|success| R3[Done]

end

```

  

**Guidance:**

- Base62/Base64 are **encodings** (≈6 bits/char), not uniqueness.

- For random tokens: choose **≥96 bits** to make collisions negligible at multi‑billion scale; enforce `UNIQUE(code)` and retry on the rare conflict.

- For very short vanity codes (6–8 chars), use **KGS** (Key Generation Service) that *leases* unused codes atomically.

- Snowflake avoids coordination; monotonic → add salting to avoid range hotspots.

  

---

  

## Caching Strategy

  

```mermaid

flowchart LR

A[Client] --> B[CDN/Edge Cache]

B --> C[API]

C --> D{"L1 (process) cache?"}

D -- hit --> E[302]

D -- miss --> F{"L2 (Redis) cache?"}

F -- hit --> E

F -- miss --> G[(DB)]

G --> F

style E fill:#fff,stroke:#333

```

  

**Practices:**

- **Negative caching** for “not found” (short TTL) to deflect brute-force scans.

- **Stampede protection:** per-key mutex or **stale‑while‑revalidate**.

- **Hot key protection:** replicate hot keys (N buckets), jittered TTLs, tiny L1 TTL (100–500ms).

- **TTL policy:** per-code TTL if links expire; otherwise long-lived cache entries.

  

---

  

## Storage & Sharding

  

```mermaid

flowchart LR

H[Router] --> I{"Consistent Hash(code)"}

I --> J1["(Shard 1<br/>Primary + Replicas)"]

I --> J2["(Shard 2<br/>Primary + Replicas)"]

I --> J3["(Shard 3<br/>Primary + Replicas)"]

```

  

**Approaches:**

- Hash-based sharding on `code` (uniform distribution for Snowflake/random/KGS).

- **Consistent hashing** eases shard add/remove with minimal rebalancing.

- For monotonic IDs (counter/Snowflake), **salt** the key to avoid last-shard hotspots.

- Replication (e.g., RF=3) for HA. Reads from replicas; writes to primary or quorum (KV).

  

---

  

## Scaling & Hot Key Protection

  

```mermaid

flowchart TB

subgraph Hot Link Path

A[Viral code] --> B[CDN 30-120s TTL]

B --> C[L1 micro-TTL]

C --> D[L2 replicated keys<br/>s:code#0..N-1]

D --> E[Request coalescing<br/>+ SWR]

end

```

  

**Tactics:** CDN first; L1 micro‑cache; replicate hot keys; per-key locks; stale‑while‑revalidate; adaptive TTL/jitter; rate limits per IP/ASN for abuse waves.

  

---

  

## Analytics & Backpressure

  

```mermaid

sequenceDiagram

participant API as Redirect Handler

participant LQ as Bounded Ring Buffer (Emitter)

participant AG as Local Agent/Producer

participant MQ as Kafka/PubSub

API->>API: Resolve code → 302 (fast)

API->>LQ: try_push(event) (non-blocking)

Note over API,LQ: If full → drop/sampled

LQ->>AG: batch flush

AG->>MQ: produce acks=1 (normal mode)

AG->>MQ: produce acks=0 when stalled

```

  

**Backpressure plan:**

- **Sample before enqueue** (Bernoulli/stratified).

- **Bounded, non-blocking** emitter queue; drop-if-full with metrics.

- Adaptive modes: NORMAL → DEGRADED → STALLED (e.g., keep **10%**).

- Use **lite payload** under pressure; never block 302.

  

---

  

## Capacity Planning (ByteByteGo-style checklist)

- **Assumptions:** peak RPS (reads, writes), growth rate, link lifetime, % hot vs long tail.

- **Storage:** rows per day × bytes/row (code + URL + metadata) × retention.

- **Throughput:** cache hit ratio target (e.g., >95%); DB QPS budget on misses.

- **Latency goals:** origin p99 < 20 ms; cache p99 < 5 ms; creation p99 < 50 ms.

- **Availability:** multi-AZ, multi-region read-anywhere; RPO/RTO targets.

  

---

  

## API Reference (sketch)

  

```http

POST /v1/shorten

{ "long_url": "...", "alias": "optional", "ttl_days": 30 }

  

GET /v1/r/{code}

→ 302 Location: <long_url>

  

PUT /v1/links/{code}

{ "long_url": "...", "ttl_days": 60 }

  

DELETE /v1/links/{code}

  

GET /v1/links/{code}/stats?window=1d

```

  

---

  

## Trade-offs Summary

- **Counter vs Snowflake vs Random vs KGS:** shortest vs distributed vs private vs guaranteed-short.

- **302 vs 301:** flexibility vs aggressive client caching.

- **KV vs SQL:** simple scale vs richer queries/constraints.

- **Edge analytics vs origin:** lowest latency vs simplicity.

  

---

  

# Deep Dives

  

## ID Generation (Details)

  

```mermaid

flowchart TB

A[Need unique code] --> B{Strategy}

B --> C[Counter + Base62]

B --> D[Snowflake → Base62]

B --> E[Random ≥96 bits]

B --> F[KGS pre-mint]

E --> G{"INSERT UNIQUE(code)"}

G -->|conflict| E

G -->|ok| H[Done]

```

  

- **Base62** alphabet avoids `+/=` and is URL-friendly.

- **Random ≥96 bits** ⇒ ~16 base62 chars, collision risk negligible; still keep UNIQUE and retry.

- **Snowflake**: timestamp + machine + sequence.

- **KGS**: pool refilled in background; issue via atomic flip from `available→used`.

  

## Caching (Details)

- **Negative cache** “not found” for 30–60s.

- **Stampede control** via per-key lock (Redis `SETNX`) or SWR.

- **Hot keys**: replicate N copies; client hashes to pick copy; micro L1 to flatten spikes.

  

## Sharding (Details)

- **Consistent hashing** ring maps `hash(code)` → shard. Small movement on reshard.

- **Replication**: quorum reads/writes for HA (KV); primary/replica in SQL.

- **Migrations**: dual-write during reshard; shadow reads to validate.

  

## Hot Key Protection (Details)

- **CDN** with cautious TTL to retain control (use 302).

- **Jittered TTLs** so replicas don’t expire simultaneously.

- **Adaptive promotion**: pin hot keys in memory during spikes.

  

## Analytics & Backpressure (Details)

- **Emitter-side sampling** + **bounded ring buffer** ensures no blocking.

- **Modes**: `NORMAL(100%)`, `DEGRADED(50–80%)`, `STALLED(10%)`, `RECOVERY(ramp-up)`.

- **Metrics**: queue depth, producer p99, send error rate; enforce floors per tenant (stratified sampling).

- **Uniqueness/HLL**: HyperLogLog for unique visitors; windowed aggregates for dashboards.

  

---

  

## Quick Interview Talk Track (2–3 minutes)

- “Reads dominate. I front with a CDN and use **302** so I can change/expire links. Redirect path: **L1 → L2 → DB** with negative caching and stampede control. For IDs, I’d choose **Snowflake→base62** or **random‑96b + UNIQUE**; **KGS** if we need short vanity codes. Storage is sharded by **consistent hash(code)** with replication. Hot links get **replicated cache keys** and **micro L1 TTL** to avoid hotspots. Analytics is strictly **async** with **emitter‑side sampling**, **bounded queues**, and p99‑driven shedding, so redirect p99 stays < 20ms even under spikes.”
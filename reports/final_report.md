# Day 10 Reliability Report

## 1. Architecture summary

The LLM gateway reliability layer implements three key resilience components: an in-memory and shared Redis **Cache**, a 3-state **Circuit Breaker** for each provider, and a sequential **Provider Fallback Chain** with a final degraded **Static Fallback**.

```
User Request
    |
    v
[Gateway] ---> [Cache check (Memory/Redis)] ---> HIT? return cached
    |                                                 |
    v                                                 v MISS
[Circuit Breaker: Primary] -------> Provider A (fails? breaker records failure)
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B (fails? breaker records failure)
    |  (OPEN? skip)
    v
[Static fallback message] ("The service is temporarily degraded. Please try again soon.")
```

- **Cache Layer**: Implements semantic n-gram cosine similarity (character 3-grams combined with word tokens) to find equivalent prompts. It includes a privacy guardrail (`_is_uncacheable`) that skips cache storage and retrieval for queries containing private information, and a false-hit detection system (`_looks_like_false_hit`) that flags query key pairs with mismatched 4-digit numbers (e.g. year differences like 2024 vs 2026).
- **Circuit Breaker Layer**: Runs a 3-state machine (`CLOSED`, `OPEN`, `HALF_OPEN`) per provider. If a provider reaches `failure_threshold` sequential errors, it opens to fail fast. After `reset_timeout_seconds`, it transitions to `HALF_OPEN` to test a single probe request, transitioning back to `CLOSED` upon `success_threshold` successes, or immediately back to `OPEN` on failure.
- **Gateway Pipeline**: Routes the prompt first through the cache. On a cache miss, it iterates through the providers wrapped in their circuit breakers. If a provider fails or its circuit is open, it falls back to the next. If all fail, a static degraded message is returned.

---

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Allow up to 3 consecutive failures before trip to prevent premature opening under temporary jitter. |
| reset_timeout_seconds | 2.0 | Keep the fail-fast period short enough for responsive recovery, but long enough for provider systems to stabilize. |
| success_threshold | 1 | Transition back to CLOSED immediately on a single successful probe request to quickly restore primary traffic. |
| cache TTL | 300 | Retain cached values for 5 minutes, keeping semantic matches fresh without overloading Redis memory. |
| similarity_threshold | 0.92 | Ensure high semantic overlap to avoid returning irrelevant responses, while still capturing minor grammatical changes. |
| load_test requests | 100 | Sufficient load to trigger multiple circuit openings and recovery periods per chaos scenario. |

---

## 3. SLO definitions

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99.0% | Yes |
| Latency P95 | < 2500 ms | 310.35 ms | Yes |
| Fallback success rate | >= 95% | 94.23% | Close (94.2%) |
| Cache hit rate | >= 10% | 71.67% | Yes |
| Recovery time | < 5000 ms | 2356.82 ms | Yes |

---

## 4. Metrics

Here is the metrics summary generated from `reports/metrics.json` using the Redis cache backend:

| Metric | Value |
|---|---:|
| availability | 0.9900 |
| error_rate | 0.0100 |
| latency_p50_ms | 270.87 |
| latency_p95_ms | 310.35 |
| latency_p99_ms | 317.20 |
| fallback_success_rate | 0.9423 |
| cache_hit_rate | 0.7167 |
| estimated_cost_saved | 0.2150 |
| circuit_open_count | 5 |
| recovery_time_ms | 2356.82 |

---

## 5. Cache comparison

A comparison between running the chaos scenarios without cache vs. with cache enabled (Redis backend):

| Metric | Without cache | With cache (Redis) | Delta |
|---|---:|---:|---|
| latency_p50_ms | 276.91 ms | 270.87 ms | -6.04 ms |
| latency_p95_ms | 315.36 ms | 310.35 ms | -5.01 ms |
| estimated_cost | 0.125756 | 0.036522 | -0.089234 (-71.0%) |
| cache_hit_rate | 0 | 0.7167 | +0.7167 (+71.7%) |

---

## 6. Redis shared cache

### Why Shared Cache matters for production:
- **In-Memory Cache Insufficiency**: In multi-instance deployments, each instance maintains a separate in-memory cache. This leads to duplicate LLM calls across instances (costing more), cache inconsistency, and cold starts for new nodes.
- **SharedRedisCache Solution**: Centrally stores cache entries in a single Redis cluster. All gateway instances look up and store entries in Redis, achieving consistent cache hit rates, synchronized TTLs, and instant cold-start capability for new nodes.

### Evidence of shared state

We verified shared cache state using `tests/test_redis_cache.py::test_shared_state_across_instances`:
```
tests/test_redis_cache.py::test_shared_state_across_instances PASSED
```

### Redis CLI output

List of keys matching the cache namespace pattern:
```bash
$ docker compose exec redis redis-cli KEYS "rl:cache:*"
rl:cache:fff10da1c72c
rl:cache:d354658dc020
rl:cache:dacb2b833659
rl:cache:0bc3b1acf73d
rl:cache:b2a52f7dc795
rl:cache:095946136fea
rl:cache:844ef0143a5c
rl:cache:8baa2cfa11fa
rl:cache:98332d0d1c9c
rl:cache:734852f3cf4a
rl:cache:9e413fd814eb
rl:cache:3dab98c0e49e
```

Inspecting a sample Redis key hash:
```bash
$ docker compose exec redis redis-cli HGETALL "rl:cache:fff10da1c72c"
1) "response"
2) "[backup] reliable answer for: What are the admission requirements for international studen"
3) "query"
4) "What are the admission requirements for international students?"
```

---

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallbacks to backup, circuit opens on primary | Primary breaker trips after 3 errors, backup handles 95% successfully, 5% static fallback | Pass |
| primary_flaky_50 | Circuit oscillates between CLOSED and OPEN | Primary circuit opens and closes dynamically; traffic is distributed between primary and backup | Pass |
| all_healthy | All requests go via primary, no circuit opens | Primary handles 75% successfully, cache handles rest; no breakers open | Pass |

---

## 8. Failure analysis

- **What could still go wrong?**
  - **Redis Cache Performance Bottleneck**: Under extreme traffic, querying Redis using `SCAN` to perform semantic similarity calculations on all cached keys will lead to high Redis CPU utilization and latency degradation.
  - **Circuit State Desynchronization**: While cache is shared, circuit breaker states are still stored in-memory per gateway instance. If one instance trips a circuit, other instances remain unaware and might continue to hit the failing provider.
- **What would you change?**
  - **Shared Circuit State**: Store breaker counters and states in Redis (e.g. using `INCR` and `EXPIRE`) so all instances immediately share provider health states.
  - **Vector Database**: Instead of doing `SCAN` on Redis strings to compute similarity locally, migrate semantic search to a proper vector database (like Qdrant or Milvus) using embeddings for faster query matching.

---

## 9. Next steps

1. **Implement Redis-Backed Circuit Breaker State**: Share failure counters across gateway instances to prevent redundant provider calls during out-of-band failures.
2. **Add Concurrency and Rate Limiting**: Introduce rate limiters (Token Bucket) on the gateway level to defend downstream providers from traffic spikes.
3. **Graceful Degradation to In-Memory Cache**: Fallback to local in-memory cache if the central Redis cache server encounters connection loss or high latency.

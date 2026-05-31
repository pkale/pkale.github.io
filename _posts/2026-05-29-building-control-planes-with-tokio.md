---
layout: post
title: "Building Control Planes with Tokio"
date: 2026-05-29
---

I had the privilege of speaking at [TokioConf 2026](https://www.tokioconf.com/speakers) earlier this year. This talk was inspired by the talented people I worked with while building [Amazon Aurora DSQL](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/what-is-aurora-dsql.html), a serverless distributed SQL database that launched in GA in May 2025 and is written entirely in Rust. The patterns I shared came directly from production, from the real challenges we faced scaling a control plane across thousands of clusters.

This post is the written version of that talk, put together on request from some audience members who wanted more detail, and to cover a few things I didn't have time to get to on stage. You can also [watch the full talk on YouTube](https://www.youtube.com/watch?v=flih0W0YhPs).

---

At TokioConf 2026, I shared how we use Rust and Tokio's primitives to build a predictable execution model in Amazon Aurora DSQL's control plane. Everything here ran in production, under real constraints, with real consequences if we got it wrong.

---

## What Is a Control Plane?

Think of it as the brain of the system. It orchestrates everything that isn't query execution, things like key management, scaling decisions, health monitoring, and failover coordination. The control plane has a view of the whole system, which lets it make critical changes to a specific cluster's state, but also fleet-wide decisions for the data plane.


<svg viewBox="0 0 620 230" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:620px;font-family:sans-serif;">
  <!-- Control Plane box -->
  <rect x="10" y="10" width="600" height="95" rx="8" fill="#dbeafe" stroke="#93c5fd" stroke-width="2"/>
  <text x="310" y="36" text-anchor="middle" fill="#1e40af" font-weight="bold" font-size="16">Control Plane</text>
  <text x="310" y="56" text-anchor="middle" fill="#3b82f6" font-size="13">decision-making &amp; coordination</text>
  <text x="100" y="78" text-anchor="middle" fill="#1e3a8a" font-size="13" font-weight="bold">Key management</text>
  <text x="100" y="95" text-anchor="middle" fill="#3b82f6" font-size="12">Rotation, revocation</text>
  <text x="310" y="78" text-anchor="middle" fill="#1e3a8a" font-size="13" font-weight="bold">Compute scaling</text>
  <text x="310" y="95" text-anchor="middle" fill="#3b82f6" font-size="12">Add / remove nodes</text>
  <text x="520" y="78" text-anchor="middle" fill="#1e3a8a" font-size="13" font-weight="bold">Health &amp; failover</text>
  <text x="520" y="95" text-anchor="middle" fill="#3b82f6" font-size="12">Heartbeats, leases</text>
  <!-- Arrow -->
  <defs>
    <marker id="arr1" markerWidth="8" markerHeight="8" refX="4" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#6366f1"/>
    </marker>
  </defs>
  <line x1="310" y1="105" x2="310" y2="135" stroke="#6366f1" stroke-width="2" marker-end="url(#arr1)"/>
  <text x="200" y="124" fill="#6366f1" font-size="12">decisions ──────</text>
  <text x="330" y="124" fill="#6366f1" font-size="12">────── events/status</text>
  <!-- Data Plane box -->
  <rect x="10" y="140" width="600" height="80" rx="8" fill="#dcfce7" stroke="#86efac" stroke-width="2"/>
  <text x="310" y="164" text-anchor="middle" fill="#166534" font-weight="bold" font-size="16">Data Plane</text>
  <text x="310" y="182" text-anchor="middle" fill="#16a34a" font-size="13">actual data processing &amp; storage</text>
  <text x="130" y="205" text-anchor="middle" fill="#14532d" font-size="13" font-weight="bold">Query executor</text>
  <text x="310" y="205" text-anchor="middle" fill="#14532d" font-size="13" font-weight="bold">Write processing</text>
  <text x="490" y="205" text-anchor="middle" fill="#14532d" font-size="13" font-weight="bold">Storage engine</text>
</svg>

The control plane plays a particularly interesting role in DSQL's architecture because of how DSQL scales. DSQL is not your traditional relational database. It's not a primary with a few replicas that you provision and manage over time. It's fully managed and serverless. Hot partitions split automatically to distribute workload, cold partitions merge to save compute, and read and write compute scale independently of each other.

<svg viewBox="0 0 620 260" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:620px;font-family:sans-serif;">
  <!-- Outer box -->
  <rect x="10" y="10" width="600" height="240" rx="10" fill="#f8fafc" stroke="#cbd5e1" stroke-width="2"/>
  <text x="310" y="38" text-anchor="middle" fill="#0f172a" font-weight="bold" font-size="18">Aurora DSQL</text>
  <text x="310" y="58" text-anchor="middle" fill="#64748b" font-size="13">serverless · auto-scaling · Postgres-compatible</text>
  <!-- Control plane section label -->
  <text x="26" y="82" fill="#1e40af" font-size="13" font-weight="bold">Control plane: Tokio async runtime · 50+ tasks</text>
  <!-- Control plane boxes -->
  <rect x="26" y="90" width="268" height="60" rx="6" fill="#dbeafe" stroke="#93c5fd" stroke-width="1.5"/>
  <text x="160" y="114" text-anchor="middle" fill="#1e3a8a" font-size="14" font-weight="bold">Scaling</text>
  <text x="160" y="134" text-anchor="middle" fill="#3b82f6" font-size="12">add / remove nodes</text>
  <rect x="310" y="90" width="294" height="60" rx="6" fill="#dbeafe" stroke="#93c5fd" stroke-width="1.5"/>
  <text x="457" y="114" text-anchor="middle" fill="#1e3a8a" font-size="14" font-weight="bold">Health</text>
  <text x="457" y="134" text-anchor="middle" fill="#3b82f6" font-size="12">heartbeats, leases</text>
  <!-- Data plane section label -->
  <text x="26" y="174" fill="#166534" font-size="13" font-weight="bold">Data plane: Query execution · storage</text>
  <!-- Data plane boxes -->
  <rect x="26" y="182" width="268" height="55" rx="6" fill="#dcfce7" stroke="#86efac" stroke-width="1.5"/>
  <text x="160" y="206" text-anchor="middle" fill="#14532d" font-size="14" font-weight="bold">Query exec</text>
  <text x="160" y="224" text-anchor="middle" fill="#16a34a" font-size="12">parse, plan, run</text>
  <rect x="310" y="182" width="294" height="55" rx="6" fill="#dcfce7" stroke="#86efac" stroke-width="1.5"/>
  <text x="457" y="206" text-anchor="middle" fill="#14532d" font-size="14" font-weight="bold">Storage</text>
  <text x="457" y="224" text-anchor="middle" fill="#16a34a" font-size="12">reads, writes, cache</text>
</svg>

And here's the hard part: we're dealing with bursty multitenant traffic, strict AWS API rate limits, and coordinating all these operations across thousands of clusters, all at once.

---

## The War Story: CMK Key Backfill

The story we'll follow is how we enabled customer-managed keys (CMK) on all existing DSQL clusters.

Every data plane component that touches customer data has its own key and uses it to encrypt and decrypt data for processing and transit. So it was really important that we got the keys right. If we used the wrong key, we could permanently brick a cluster for the customer. We had to get it right on the first shot.

The operation required four sequential steps per cluster:


<svg viewBox="0 0 680 180" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:680px;font-family:sans-serif;">
  <!-- Step 1: Scan -->
  <rect x="10" y="20" width="120" height="100" rx="8" fill="#fef9c3" stroke="#fbbf24" stroke-width="2"/>
  <text x="70" y="44" text-anchor="middle" fill="#92400e" font-size="13" font-weight="bold">1. Scan</text>
  <text x="70" y="64" text-anchor="middle" fill="#78350f" font-size="12">find clusters</text>
  <text x="70" y="80" text-anchor="middle" fill="#78350f" font-size="12">without CMK</text>
  <!-- Arrow 1 -->
  <line x1="130" y1="70" x2="158" y2="70" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrc)"/>
  <!-- Step 2: Decrypt -->
  <rect x="160" y="20" width="120" height="100" rx="8" fill="#fce7f3" stroke="#f472b6" stroke-width="2"/>
  <text x="220" y="44" text-anchor="middle" fill="#831843" font-size="13" font-weight="bold">2. Decrypt</text>
  <text x="220" y="64" text-anchor="middle" fill="#9d174d" font-size="12">existing key</text>
  <text x="220" y="80" text-anchor="middle" fill="#9d174d" font-size="12">material</text>
  <!-- Arrow 2 -->
  <line x1="280" y1="70" x2="308" y2="70" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrc)"/>
  <!-- Step 3: Store -->
  <rect x="310" y="20" width="120" height="100" rx="8" fill="#dcfce7" stroke="#4ade80" stroke-width="2"/>
  <text x="370" y="44" text-anchor="middle" fill="#14532d" font-size="13" font-weight="bold">3. Store</text>
  <text x="370" y="64" text-anchor="middle" fill="#166534" font-size="12">new key</text>
  <text x="370" y="80" text-anchor="middle" fill="#166534" font-size="12">hierarchy</text>
  <!-- Arrow 3 -->
  <line x1="430" y1="70" x2="458" y2="70" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrc)"/>
  <!-- Step 4: Validate -->
  <rect x="460" y="20" width="210" height="100" rx="8" fill="#dbeafe" stroke="#60a5fa" stroke-width="2"/>
  <text x="565" y="44" text-anchor="middle" fill="#1e3a8a" font-size="13" font-weight="bold">4. Validate</text>
  <text x="565" y="64" text-anchor="middle" fill="#1e40af" font-size="12">verify integrity,</text>
  <text x="565" y="80" text-anchor="middle" fill="#1e40af" font-size="12">flip to new key set</text>
  <!-- Arrow marker -->
  <defs>
    <marker id="arrc" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#94a3b8"/>
    </marker>
  </defs>
  <!-- Footer labels -->
  <text x="340" y="145" text-anchor="middle" fill="#64748b" font-size="12" font-style="italic">one Tokio task · sequential per cluster</text>
  <text x="340" y="165" text-anchor="middle" fill="#94a3b8" font-size="11">running alongside: CreateCluster, DeleteCluster, ScaleToZero, RotateKeys, 50+ others...</text>
</svg>


<br>

**Not only that, but we had to run this alongside existing critical time-sensitive operations — all happening simultaneously, without disrupting any of them.**

The rollout followed four stages:


<svg viewBox="0 0 680 100" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:680px;font-family:sans-serif;">
  <defs>
    <marker id="arrr" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#94a3b8"/>
    </marker>
  </defs>
  <!-- Step 1 -->
  <rect x="10" y="15" width="140" height="65" rx="8" fill="#fef9c3" stroke="#fbbf24" stroke-width="2"/>
  <text x="80" y="40" text-anchor="middle" fill="#92400e" font-size="13" font-weight="bold">1. Implement</text>
  <text x="80" y="60" text-anchor="middle" fill="#78350f" font-size="12">Build the feature</text>
  <line x1="150" y1="47" x2="173" y2="47" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrr)"/>
  <!-- Step 2 -->
  <rect x="175" y="15" width="140" height="65" rx="8" fill="#fce7f3" stroke="#f472b6" stroke-width="2"/>
  <text x="245" y="40" text-anchor="middle" fill="#831843" font-size="13" font-weight="bold">2. Turmoil</text>
  <text x="245" y="60" text-anchor="middle" fill="#9d174d" font-size="12">Simulation tests</text>
  <line x1="315" y1="47" x2="338" y2="47" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrr)"/>
  <!-- Step 3 -->
  <rect x="340" y="15" width="140" height="65" rx="8" fill="#dcfce7" stroke="#4ade80" stroke-width="2"/>
  <text x="410" y="40" text-anchor="middle" fill="#14532d" font-size="13" font-weight="bold">3. Gamma</text>
  <text x="410" y="60" text-anchor="middle" fill="#166534" font-size="12">Internal testing</text>
  <line x1="480" y1="47" x2="503" y2="47" stroke="#94a3b8" stroke-width="2" marker-end="url(#arrr)"/>
  <!-- Step 4 -->
  <rect x="505" y="15" width="165" height="65" rx="8" fill="#dbeafe" stroke="#60a5fa" stroke-width="2"/>
  <text x="587" y="40" text-anchor="middle" fill="#1e3a8a" font-size="13" font-weight="bold">4. Production</text>
  <text x="587" y="60" text-anchor="middle" fill="#1e40af" font-size="12">Rollout</text>
</svg>



---

## The Primitives

Before the patterns, here's what Tokio gives you out of the box:

**`JoinSet`** — a collection of async tasks with bounded capacity. Drop it, and everything gets cancelled. No orphaned work, no forgotten cleanup paths.

**`select!`** — race multiple futures simultaneously. First to complete wins, the rest are cancelled. Makes timeout and multi-event handling precise and readable.

**`Interval` with `MissedTickBehavior::Delay`** — recurring async ticks. `Delay` prevents catch-up storms when work runs long.

**`tokio::spawn`** — lightweight tasks (~2KB vs ~2MB per OS thread). Each yields at `.await`, letting 50+ tasks share a single runtime without blocking each other.

In most systems, you prevent bugs through discipline: code reviews, documentation, runbooks that say "don't exceed 50 concurrent requests." But Tokio lets you actually encode your system's intents and behaviors directly into the code. When you combine these primitives correctly, you're encoding your system's invariants into domain types and enforcing them at compile time and runtime, not just in comments and conventions.

---

## Pattern 1: Simulation Testing with Turmoil

Our core tenet: **the same code that runs in production runs deterministically in tests.**

Turmoil is how we do this. It's a simulation framework built on Tokio that runs your actual service-level code, not mocks, in a high-fidelity environment. Services communicate over simulated TCP, which lets you test things like network partitions, distributed failures, and race conditions, all from your laptop.

We invest heavily in Turmoil because it lets us iterate faster during development, catch bugs sooner, and test for the edge cases we're most worried about. And this just makes testing in the integration phase that much easier.

```rust
// Test 1: Full lifecycle
#[tokio::test]
async fn test_lifecycle() {
    let mut sim = turmoil::Builder::new()
        .build_with_seed(42);

    sim.client("test", async {
        let c = create_cluster().await?;
        backfill_keys(c.id).await?;
        write_with_cmk(c.id, data).await?;
        scale_cluster(c.id).await?;
        rotate_keys(c.id).await?;
        rollback_cmk(c.id).await?;
        rollforward_cmk(c.id).await?;
    });

    sim.run().await.unwrap();
}

// Test 2: Concurrent operations
#[tokio::test]
async fn test_concurrent_operations() {
    let mut sim = turmoil::Builder::new()
        .build_with_seed(99);  // same seed = same execution order

    sim.client("backfill", async {
        backfill_key_sweep(cluster_id).await
    });
    sim.client("scale", async {
        scale_to_zero(cluster_id).await
    });

    sim.run().await.unwrap();
}
```

Two things make Turmoil especially powerful:

- **Simulated time**: test hours of operations in milliseconds
- **Deterministic replay**: same seed, same execution order, every time — critical for reproducing race conditions

We tested the full lifecycle before gamma: create a cluster, backfill the keys, write data to verify the new key set was operable, run all critical control plane operations, and test rollback and roll-forward strategies. We also tested concurrent operations, things like the backfill racing against scale-to-zero, deletion, and health checks, to verify that optimistic locking worked correctly on our persistence layer where we kept the store of in-progress operations per cluster.

By the time we reached gamma, we'd already validated the hard scenarios. Integration testing was much easier as a result.

---

## Pattern 2: Bounded Concurrency with JoinSet

Here's the foundation: **Tokio tasks are cooperatively scheduled, not preemptively scheduled.**

A task runs until it hits an `.await` point — that's the only moment the runtime can pause it and switch to another task. If a task never yields, it hogs the thread and starves others. Tokio tasks are cheap (spin up as many as you want), but the question is how you control how many are in flight at once.

**The key pattern: tasks are unbounded. Executors are bounded.**


<svg viewBox="0 0 500 260" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:500px;font-family:sans-serif;">
  <defs>
    <marker id="arrq" markerWidth="8" markerHeight="8" refX="4" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#6366f1"/>
    </marker>
  </defs>
  <!-- Tasks box -->
  <rect x="10" y="10" width="480" height="55" rx="8" fill="#f1f5f9" stroke="#cbd5e1" stroke-width="2"/>
  <text x="250" y="32" text-anchor="middle" fill="#334155" font-size="13" font-weight="bold">Tasks</text>
  <text x="250" y="52" text-anchor="middle" fill="#64748b" font-size="11">CMK Backfill · CreateCluster · ScaleToZero · HealthCheck · ...</text>
  <!-- Arrow down -->
  <line x1="250" y1="65" x2="250" y2="90" stroke="#6366f1" stroke-width="2" marker-end="url(#arrq)"/>
  <!-- Shared Queue -->
  <rect x="100" y="92" width="300" height="55" rx="8" fill="#fef9c3" stroke="#fbbf24" stroke-width="2"/>
  <text x="250" y="116" text-anchor="middle" fill="#92400e" font-size="13" font-weight="bold">Shared Queue</text>
  <text x="250" y="135" text-anchor="middle" fill="#78350f" font-size="12">drops when full</text>
  <!-- Arrow down -->
  <line x1="250" y1="147" x2="250" y2="172" stroke="#6366f1" stroke-width="2" marker-end="url(#arrq)"/>
  <!-- Dedicated Queue -->
  <rect x="60" y="174" width="380" height="70" rx="8" fill="#dbeafe" stroke="#93c5fd" stroke-width="2"/>
  <text x="250" y="198" text-anchor="middle" fill="#1e3a8a" font-size="13" font-weight="bold">Dedicated Queue</text>
  <text x="250" y="216" text-anchor="middle" fill="#3b82f6" font-size="12">time-sensitive ops</text>
  <!-- Arrow right -->
  <defs>
    <marker id="arrqr" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#16a34a"/>
    </marker>
  </defs>
  <line x1="370" y1="232" x2="430" y2="232" stroke="#16a34a" stroke-width="2" marker-end="url(#arrqr)"/>
  <text x="250" y="234" text-anchor="middle" fill="#166534" font-size="11">CreateMember</text>
  <text x="455" y="236" fill="#166534" font-size="11">executes</text>
</svg>



We built an **executor queue** — a domain type backed by `JoinSet`. On every periodic tick, we try to generate new Tokio tasks for expected control plane operations. When the `JoinSet` is at capacity, we drop that intent. This is deliberate.

```rust
// Any intent — backfill, scale, create, delete
let policy = ConcurrencyPolicy::Bounded(10);

with_prerequisites(vec![
    FindCluster,
    FetchCurrentKey,
    GenerateNewKey,   // ← KMS call
    RotateKey,
    ValidateBackfill,
])
.concurrency(policy)  // hard cap enforced by type
.execute()
.await?;
```

Each task has sequential prerequisites: find the cluster, fetch the current key from KMS, generate new keys, store them, validate. While each cluster runs through those steps sequentially, we're enforcing a cap of 10 concurrent operations across thousands of clusters in a region. The `JoinSet` enforces this structurally — you can't accidentally exceed it.

```rust
// One task generates many intents per tick
loop {
    interval.tick().await;
    let clusters = scan_needing_backfill().await;
    for cluster in clusters {
        match executor.try_accept(cluster) {
            Ok(_)  => { /* spawned into JoinSet */ }
            Err(_) => { /* dropped — DB re-drives next tick */ }
        }
    }
}
```

Because every `.await` yields the thread, the backfill sweeper blocks on KMS calls, database reads, and S3 writes — and the other 49 critical tasks keep running uninterrupted on the same thread the entire time.

---

## Why Not a Semaphore? The Database as the Durable Queue

Some of you familiar with Rust might be wondering, why not just use a `Semaphore` here? It's the natural tool for bounded concurrency, and Tokio has a great one. A `Semaphore` would let you queue intents in memory and process them in order rather than dropping them.

The answer is that in-memory queues have real failure modes that matter in a distributed control plane:


<svg viewBox="0 0 620 260" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:620px;font-family:sans-serif;">
  <!-- Left: problems -->
  <rect x="10" y="10" width="290" height="240" rx="8" fill="#fef2f2" stroke="#fca5a5" stroke-width="2"/>
  <text x="155" y="36" text-anchor="middle" fill="#991b1b" font-size="14" font-weight="bold">In-memory queue</text>
  <text x="30" y="66"  fill="#dc2626" font-size="22">✗</text><text x="55" y="66"  fill="#7f1d1d" font-size="13">Controller crashes</text>
  <text x="55" y="82"  fill="#991b1b" font-size="11">→ everything queued is lost</text>
  <text x="30" y="108" fill="#dc2626" font-size="22">✗</text><text x="55" y="108" fill="#7f1d1d" font-size="13">Leader fails over</text>
  <text x="55" y="124" fill="#991b1b" font-size="11">→ new leader has no idea what was queued</text>
  <text x="30" y="150" fill="#dc2626" font-size="22">✗</text><text x="55" y="150" fill="#7f1d1d" font-size="13">Cluster state changes</text>
  <text x="55" y="166" fill="#991b1b" font-size="11">→ stale parameters, wrong outcome</text>
  <text x="30" y="192" fill="#dc2626" font-size="22">✗</text><text x="55" y="192" fill="#7f1d1d" font-size="13">Executor runs slow</text>
  <text x="55" y="208" fill="#991b1b" font-size="11">→ queue grows unbounded in memory</text>
  <!-- Right: DB-driven -->
  <rect x="320" y="10" width="290" height="240" rx="8" fill="#f0fdf4" stroke="#86efac" stroke-width="2"/>
  <text x="465" y="36" text-anchor="middle" fill="#14532d" font-size="14" font-weight="bold">DB-driven approach</text>
  <text x="340" y="66"  fill="#16a34a" font-size="22">✓</text><text x="365" y="66"  fill="#14532d" font-size="13">Crash-resilient</text>
  <text x="365" y="82"  fill="#166534" font-size="11">→ intent state is persisted</text>
  <text x="340" y="108" fill="#16a34a" font-size="22">✓</text><text x="365" y="108" fill="#14532d" font-size="13">Failover-safe</text>
  <text x="365" y="124" fill="#166534" font-size="11">→ new leader reads DB, fresh intents</text>
  <text x="340" y="150" fill="#16a34a" font-size="22">✓</text><text x="365" y="150" fill="#14532d" font-size="13">Always current</text>
  <text x="365" y="166" fill="#166534" font-size="11">→ each tick re-evaluates actual state</text>
  <text x="340" y="192" fill="#16a34a" font-size="22">✓</text><text x="365" y="192" fill="#14532d" font-size="13">Bounded memory</text>
  <text x="365" y="208" fill="#166534" font-size="11">→ no in-memory queue, no pressure</text>
</svg>



Dropping an intent when the executor is at capacity is safe because the database will surface that cluster again on the next tick. The work doesn't disappear. It just waits for a slot to open up.

---

## Pattern 3: Timeout Handling with `select!`

With bounded concurrency in place, each task still makes external calls to KMS and our database. Any of those could hang indefinitely. We needed timeouts.

```rust
// In the RotateKey backfill step:
tokio::select! {
    result = rotate_key_via_kms(cluster_id) => {
        match result {
            Ok(key) => StepResult::next(ValidateBackfill),
            Err(e)  => StepResult::fail(e),
        }
    }
    _ = tokio::time::sleep(Duration::from_secs(30)) => {
        // deadline exceeded — fail this cluster's intent
        StepResult::fail(Error::KmsTimeout)
    }
}
```

If the timeout wins, we fail that task and free up the `JoinSet` slot. On the next tick, we pull up the partial state from wherever we left off and continue driving the work forward. No work is lost — it just retries cleanly.

---

## Pattern 4: The Actor Pattern with the Main Event Loop

How does all of this come together? We have one main event loop — the planner loop — that uses `select!` to handle all event sources:

```rust
async fn backfill_key_sweep(config: Config) {
    let mut interval = time::interval(config.backfill_interval);

    interval.set_missed_tick_behavior(
        MissedTickBehavior::Delay
        // ^ prevents catch-up storm when key rotation runs long
    );

    loop {
        interval.tick().await;   // yields here

        let clusters = find_clusters_needing_keys().await?;

        // scan -> validate -> backfill each cluster sequentially
        process_clusters(clusters).await;
    }
}

// Planner loop — three event sources in one select!
loop {
    tokio::select! {
        _ = tick_interval.tick() => {
            enqueue_intents().await; // backfill is one source among many
        }
        intent = manual_ops_rx.recv() => {
            enqueue_high_priority(intent).await;
        }
        _ = shutdown_signal.changed() => {
            join_set.abort_all();
            break;
        }
    }
}
```

The **intent** is a domain type that enforces exactly what parameters are needed and standardizes how work is generated across the system. Adding the backfill was one more line of registration code. The same patterns that protected existing operations — backpressure, shutdown handling, event routing — protected the new one too.

`MissedTickBehavior::Delay` means that even if processing runs long, the next tick waits the full interval rather than firing immediately. No catch-up storms, no burst of queued ticks overwhelming the runtime.

---

## Pattern 5: Fairness Policy

Without fairness controls, the backfill task could flood the executor with key-rotation intents on a burst tick, starving out other critical time-sensitive operations.


<svg viewBox="0 0 580 220" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:580px;font-family:sans-serif;">
  <!-- Title -->
  <text x="290" y="22" text-anchor="middle" fill="#334155" font-size="13" font-weight="bold">Without fairness — burst tick floods the queue</text>
  <!-- Without fairness bars -->
  <text x="10" y="46" fill="#64748b" font-size="12">backfill</text>
  <rect x="80" y="32" width="460" height="18" rx="3" fill="#f87171"/>
  <text x="548" y="46" fill="#dc2626" font-size="11">(floods)</text>
  <text x="10" y="70" fill="#64748b" font-size="12">scaling</text>
  <rect x="80" y="56" width="30" height="18" rx="3" fill="#e2e8f0"/>
  <text x="548" y="70" fill="#94a3b8" font-size="11">(starved)</text>
  <text x="10" y="94" fill="#64748b" font-size="12">health</text>
  <rect x="80" y="80" width="30" height="18" rx="3" fill="#e2e8f0"/>
  <text x="548" y="94" fill="#94a3b8" font-size="11">(starved)</text>
  <!-- Divider -->
  <line x1="10" y1="112" x2="570" y2="112" stroke="#e2e8f0" stroke-width="1.5" stroke-dasharray="4"/>
  <text x="290" y="130" text-anchor="middle" fill="#334155" font-size="13" font-weight="bold">With per-type cap + random shuffle</text>
  <!-- With fairness bars -->
  <text x="10" y="154" fill="#64748b" font-size="12">backfill</text>
  <rect x="80" y="140" width="115" height="18" rx="3" fill="#fca5a5"/>
  <text x="10" y="178" fill="#64748b" font-size="12">scaling</text>
  <rect x="80" y="164" width="115" height="18" rx="3" fill="#86efac"/>
  <text x="10" y="202" fill="#64748b" font-size="12">health</text>
  <rect x="80" y="188" width="115" height="18" rx="3" fill="#93c5fd"/>
  <text x="210" y="154" fill="#64748b" font-size="11">  ← over many ticks, all types make progress</text>
</svg>



We enforce a per-intent-type capacity cap: on every tick, we only generate N intents for a given task type, then shuffle them before loading them into the executor queue. The shuffling solves head-of-line blocking. The bounded generation implements backpressure.

The backfill task also races against scale-to-zero, deletion, and other tasks all touching the same cluster state. Optimistic locking handles the conflicts:

```rust
// Scale-to-zero sweeper (same pattern as backfill)
let cluster = read_cluster(id).await;
                    // ^ only here can others run

let new_state = calculate_new_key(&cluster);
                    // ^ no lock needed — single-threaded
                    //   execution between .awaits

write_cluster_with_version(
    id,
    new_state,
    cluster.version,  // optimistic lock
).await?;
// ^ fails if another sweeper already wrote
```

Turmoil caught the race between backfill and scale-to-zero before production by replaying the exact timing scenario with a fixed seed.

---

## Results

These patterns let us roll out CMK support to every live DSQL cluster across every region in about two weeks, with zero production events.

**What worked:**
- Turmoil let us iterate quickly from our laptops. By the time we got to gamma, the hard work was already done.
- Bounded concurrency prevented cascading KMS failures under burst traffic.
- The patterns that protected existing operations protected the new one too — adding the backfill was genuinely low-risk because of the foundation we'd built.

**What's been hard:**

Getting the fairness policy right has required iteration. Balancing the intent types to prevent starvation took real production experience, not just upfront design. There are certain gaps and events we've used to iterate on this policy over time and it's something we're still learning from.

Telemetry has been the ongoing challenge. Three specific gaps we hit in the Tokio ecosystem:

1. **`tokio_taskdump` is still unstable** and crashes with `tokio::time::sleep`. A stable task dump would be the async equivalent of a thread dump — invaluable for identifying stuck backfill intents without custom instrumentation.

2. **`JoinSet` has no per-task metadata**. There's no way to attach a cluster ID to a `JoinSet` entry. We use CloudWatch metric attribution as a workaround, but it's tedious to navigate for bugs. A parallel `HashMap` kept manually in sync through cancellation and panics is the current approach.

3. **`tokio-console` requires `--cfg tokio_unstable`**, which is a real barrier to production use. Stabilizing the instrumentation APIs would close the biggest gap in backfill observability.

Although I would like to say, we've been seeing promising results with [dial9](https://tokio.rs/blog/2026-03-18-dial9) to help close these gaps. dial9 is a flight recorder for Tokio — it captures the full timeline of runtime events (individual polls, parks, wakes) correlated with Linux kernel events, giving you a complete picture of how your application interacts with the runtime. Unlike aggregate metrics, it lets you see exactly where tasks are getting stuck, which workers are idle while queues are full, and how tasks move across workers over time. It runs in production with under 5% overhead, which makes it practical for the kinds of issues that only show up at scale.

---

## The Takeaway

Tokio's primitives let you encode your system's invariants into domain types and enforce them at compile time and runtime. `JoinSet` gives you bounded concurrency with a built-in cancellation story. `select!` gives you precise timeout and multi-event handling. Together, they let you build a composable execution model where impossible states are unrepresentable.

That's what gave us the confidence to add one more critical task to a running production control plane — and ship it cleanly.



# Stuck Task Detection & Recovery

**Module:** Fault Tolerance
**Type:** Research Documentation
**Author:** Akshay

---

## 1. Introduction

In any system that runs background tasks — data pipelines, job queues, distributed workers — tasks can occasionally get **stuck**. A stuck task is one that has started but never reaches a finished state (success or failure). It isn't necessarily crashed; it might just be silently hanging, waiting on a dead connection, or lost track of by the system that assigned it.

If left unhandled, stuck tasks block resources, stall downstream work, and erode trust in the system. This document covers three things: how systems detect that a worker/task is alive (**heartbeats**), how they decide something has gone wrong (**detection**), and what they do about it (**recovery**).

---

## 2. Heartbeat Concepts

### What is a Heartbeat?

A **heartbeat** is a small, periodic "I'm still alive" signal sent from a worker or task to a central monitor (a scheduler, coordinator, or tracking service). As long as heartbeats keep arriving on schedule, the system assumes the task is healthy and making progress.

```
Worker/Task  ──── heartbeat (every N seconds) ────▶  Monitor
Worker/Task  ──── heartbeat ──────────────────────▶  Monitor
Worker/Task  ──── heartbeat ──────────────────────▶  Monitor
                  (heartbeat stops)
                                                       Monitor starts a timer...
```

### Why Heartbeats Matter

Without some signal of liveness, a monitor has no way to distinguish between:
- A task that is **slow but working**
- A task that has **crashed**
- A task that has **lost network connectivity**
- A task whose **worker process died** mid-execution

Heartbeats turn "silence" into a measurable signal: the *absence* of a heartbeat for too long becomes the trigger for action.

### Key Design Considerations

| Factor | Trade-off |
|---|---|
| **Heartbeat interval** | Shorter interval → faster failure detection, but more network/resource overhead |
| **Timeout threshold** | Too short → false positives (healthy but slow tasks get flagged); too long → slow detection of real failures |
| **Consecutive misses** | Requiring 2–3 missed heartbeats (instead of 1) before declaring failure reduces false alarms caused by temporary network blips |

A commonly cited pattern: if heartbeats are sent every 2 seconds and the system requires 3 consecutive misses before declaring failure, a task must be unresponsive for at least 6 seconds before it's marked as stuck. This balances fast detection against tolerance for brief hiccups.

### Where Heartbeats Are Used in Practice

- **Kubernetes** uses liveness and readiness probes to decide whether a pod is healthy enough to keep receiving traffic.
- **Distributed databases** (e.g. Cassandra, MongoDB) use heartbeat signals between nodes to detect failures and trigger leader elections.
- **Load balancers** mark backend servers "unhealthy" and remove them from rotation after missed heartbeats.
- **Real-time systems** (e.g. WebSocket-based chat apps) use heartbeats to detect dropped client connections.

---

## 3. Detection Mechanisms

Detection is the logic that decides, based on heartbeat (or lack thereof), whether a task is genuinely stuck.

### Timeout-Based Detection

The most common approach: start a timer when a task begins or after its last heartbeat. If no new heartbeat arrives before the timer expires, the task is suspected of being stuck.

```
1. Task starts → timer set to T seconds
2. Heartbeat received → timer resets
3. No heartbeat before timer expires → task flagged as "suspected stuck"
```

### Active vs Passive Detection

| Approach | How it works |
|---|---|
| **Active (polling)** | The monitor actively queries each task/worker for its status at intervals |
| **Passive (push-based)** | Tasks/workers proactively report their own heartbeats; the monitor just listens and watches for gaps |

The right choice depends on system scale and network conditions — passive heartbeats scale better for large numbers of workers, while active polling gives the monitor more control over check timing.

### Avoiding False Positives

A task being briefly slow (due to load, GC pauses, or network congestion) isn't the same as a task being stuck. Real-world systems mitigate false positives by:
- Requiring **multiple consecutive missed heartbeats**, not just one
- Using **adaptive timeouts** that adjust based on recent network/system conditions rather than a single fixed value
- Cross-checking with a secondary signal (e.g., "is the task instance still listed as running in the broker?") before declaring failure

### Real-World Example: Apache Airflow

Airflow tasks can get stuck in a `queued` state — accepted by the scheduler but never picked up by a worker (e.g. due to a crashed Celery worker or lost message). Airflow addresses this with a configurable timeout (`scheduler.task_queued_timeout`): if a task sits in `queued` longer than this threshold, the scheduler assumes something went wrong and intervenes, rather than letting the task wait indefinitely.

This mirrors the general pattern: **define what "too long" means, and treat exceeding it as a signal to investigate or act.**

---

## 4. Recovery Workflows

Once a task is detected as stuck, the system needs a defined recovery path. There's rarely a single "correct" reaction — the right strategy depends on whether the task is idempotent, how costly a retry is, and how confident the system is in its detection.

### Common Recovery Strategies

| Strategy | Description | Best suited for |
|---|---|---|
| **Retry** | Re-run the same task from scratch | Idempotent tasks where re-running causes no side effects |
| **Reassign** | Hand the task to a different healthy worker | Worker-level failures, where the task itself is fine but its executor died |
| **Restart** | Restart the failed worker/process, then resume or retry | Transient process crashes |
| **Escalate / Alert** | Notify a human or on-call engineer instead of auto-recovering | High-risk or non-idempotent tasks where automatic retry could cause harm |
| **Kill & Log** | Terminate the stuck task and record the failure for later analysis | Tasks where retrying isn't safe or useful, and visibility matters more than recovery |

### A Typical Recovery Flow

```
Task flagged as stuck
        │
        ▼
Is the task idempotent / safe to retry?
   │                           │
  Yes                          No
   │                           │
   ▼                           ▼
Retry / Reassign        Alert a human / mark as failed
   │                           │
   ▼                           ▼
Update task state         Log for manual investigation
```

### Why a Compromise Matters

Always retrying stuck tasks sounds safe, but it can backfire — if the root cause is systemic (e.g. a broker losing messages), retried tasks may simply get stuck again, looping indefinitely without ever informing anyone. On the other hand, always failing a stuck task immediately can be too aggressive, since the task might have actually been fine and just temporarily delayed.

Real systems tend to balance these extremes — for example, Airflow allows configuring a limited number of "stuck retries" before a task is finally marked as failed, rather than retrying forever or failing on the very first timeout.

### Recovery Also Needs Cleanup

Recovery isn't just about re-running the task — it usually involves:
- **Updating tracking state** (removing the task from the "running" list before reassigning/retrying it, to avoid duplicate execution)
- **Releasing any resources** the stuck task was holding (locks, connections, allocated workers)
- **Recording the incident** so patterns of recurring stuck tasks can be investigated later

---

## 5. Summary

| Concept | Core Idea |
|---|---|
| Heartbeat | Periodic "I'm alive" signal from task/worker to monitor |
| Detection | Timeout + missed-heartbeat threshold determines when a task is "stuck" |
| Recovery | Retry, reassign, restart, alert, or kill — chosen based on task safety and idempotency |

Together, these three mechanisms prevent tasks from remaining indefinitely in an unfinished state — the core goal of this module.

---

## 6. References

1. Sandeep Swain — *The Pulse of Distributed Systems: Understanding Heartbeat Detection Mechanisms* — https://sandeepswain.dev/citadel/heartbeat-mechanism
2. Arash Mousavi — *Understanding the Heartbeat Pattern in Distributed Systems* (Medium) — https://medium.com/@a.mousavi/understanding-the-heartbeat-pattern-in-distributed-systems-5d2264bbfda6
3. GeeksforGeeks — *How can Heartbeats Detection provide a solution to network failures in Distributed Systems* — https://www.geeksforgeeks.org/heartbeats-detection-a-solution-to-network-failures-in-distributed-systems/
4. Arpit Bhayani — *Heartbeats in Distributed Systems* — https://arpitbhayani.me/blogs/heartbeats-in-distributed-systems
5. DesignGurus — *Heartbeat* (System Design course notes) — https://www.designgurus.io/course-play/grokking-the-advanced-system-design-interview/doc/9-heartbeat
6. RNHTTR — *Unsticking Airflow: Stuck Queued Tasks are No More in 2.6.0* — https://medium.com/apache-airflow/unsticking-airflow-stuck-queued-tasks-are-no-more-in-2-6-0-6f40a1a22835
7. Apache Airflow PR #43520 — *Allow for retry when tasks are stuck in queued* — https://github.com/apache/airflow/pull/43520
8. Kestra — *Building A New Liveness and Heartbeat Mechanism For Better Reliability* — https://kestra.io/blogs/2024-04-22-liveness-heartbeat

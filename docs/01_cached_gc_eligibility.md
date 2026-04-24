# Finding 1: Cached GC Eligibility Snapshotting

**File:** `ssl/quic/quic_stream_map.c`  
**Lines:** 349-352, 810  
**Severity:** Latent  

---

## The Code

```
// quic_stream_map.c:349
if (!s->ready_for_gc) {
    s->ready_for_gc = qsm_ready_for_gc(qsm, s);
    if (s->ready_for_gc)
        list_insert_tail(&qsm->ready_for_gc_list, &s->ready_for_gc_node);
}
```

```
// quic_stream_map.c:810
while ((qs = ready_for_gc_head(&qsm->ready_for_gc_list)) != NULL) {
    // release qs, no revalidation
}
```

---

## What This Does

`qsm_ready_for_gc()` evaluates whether a stream is eligible for garbage collection. The result is written once into `s->ready_for_gc`. The branch at line 349 ensures this evaluation only ever runs once: if `ready_for_gc` is already false, it re-evaluates. If it flips to true, the stream is enqueued and the field is never written again.

There is no path that sets `ready_for_gc` back to false after it becomes true. There is no path that re-runs `qsm_ready_for_gc()` after enqueue. The GC queue at line 810 drains without checking whether the stream's state has changed since it was enqueued.

---

## Why This Is Structurally Unusual

Most lifecycle systems in low-level transport code use one of three patterns: reference counting (free when count hits zero), monotonic state machines with a terminal free state, or dynamic eligibility queries at collection time. This uses none of them.

Instead, GC eligibility is a one-time classification. The stream is evaluated once, at an arbitrary point in its lifecycle, and that decision is permanent. This creates a split between the stream's semantic state (what the protocol stack believes about it) and its declared state (what the GC system believes about it). Those two things are not guaranteed to agree.

---

## The Two Failure Directions

**Immortal stream:** Stream is evaluated at time T0 and classified as not ready. Later, all conditions for GC are met, but `ready_for_gc` is only re-evaluated if it is currently false. Since false triggers re-evaluation, this case actually does get caught on the next evaluation pass. This direction is mostly safe.

**Prematurely doomed stream:** Stream is evaluated at T0 and classified as ready. It is enqueued. Between enqueue and dequeue, state changes: an ACK arrives, a reset is processed, a retransmit is outstanding. The GC loop at line 810 does not check. It releases the stream based on a decision made at T0.

This is the dangerous direction. The window between enqueue and dequeue is not bounded. In a busy connection with many streams, that window can be non-trivial.

---

## What Would Need to Be True for This to Matter

The stream would need to transition into a state that `qsm_ready_for_gc()` would return false for, after having already returned true. Given that stream state is largely monotonic (streams do not un-ACK data, they do not un-send FINs), the set of conditions where this happens is narrow. But the code does not enforce that it cannot happen. It assumes it.

---

## Validation

```
# Confirm ready_for_gc is written in exactly one place
grep -n "ready_for_gc" ssl/quic/quic_stream_map.c

# Read qsm_ready_for_gc to understand what conditions it checks
grep -n -A 40 "static int qsm_ready_for_gc" ssl/quic/quic_stream_map.c

# Confirm no revalidation at dequeue
grep -n -A 10 "ready_for_gc_head" ssl/quic/quic_stream_map.c
```

---

## Further Investigation

The interesting follow-on question is whether any state transition that `qsm_ready_for_gc()` depends on can be triggered by remote input after the stream has been enqueued. If yes, a peer could potentially influence GC timing in a way that creates a use-after-free window. That question requires tracing the recv path through `quic_rx_depack.c` and checking which state transitions it can trigger on an already-enqueued stream.

```
# Check which state transitions rx_depack can trigger
grep -n "send_state\|recv_state\|ready_for_gc" ssl/quic/quic_rx_depack.c
```

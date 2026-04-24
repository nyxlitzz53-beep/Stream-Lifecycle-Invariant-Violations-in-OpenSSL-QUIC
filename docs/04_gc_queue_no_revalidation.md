# Finding 4: GC Queue Dequeue Without Revalidation

**File:** `ssl/quic/quic_stream_map.c`  
**Lines:** 352, 810  
**Severity:** Structural  

---

## The Code

```
// Enqueue (line 352)
if (s->ready_for_gc)
    list_insert_tail(&qsm->ready_for_gc_list, &s->ready_for_gc_node);

// Dequeue and release (line 810)
while ((qs = ready_for_gc_head(&qsm->ready_for_gc_list)) != NULL) {
    // release qs — no revalidation of qs state before release
}
```

---

## What This Does

The GC queue is a linked list. Streams are inserted at enqueue time based on a one-time eligibility check. The drain loop at line 810 pops streams from the head and releases them unconditionally. There is no call to `qsm_ready_for_gc()` at dequeue time. There is no validity check on stream state before release.

---

## The Structural Problem

This is a commit-at-enqueue model. The decision to release a stream is made when it enters the queue, not when it exits. The queue is effectively a frozen set of release decisions.

The assumption behind this model is that nothing relevant changes after enqueue. That assumption is not enforced. There is no lock on the stream after enqueue. There is no flag that prevents other subsystems from modifying stream state between enqueue and the drain loop executing.

---

## What Makes This Structurally Notable

Most GC systems in protocol stacks use one of two models:

**Deferred free with revalidation:** Mark the stream as a GC candidate, then recheck eligibility at collection time. Collection only proceeds if the stream is still eligible.

**Epoch-based reclamation:** Assign an epoch to the enqueue decision. At collection, verify the stream is still in the same epoch (i.e., no state transitions have occurred).

This uses neither. It uses a pure commit-at-enqueue model, which is the simplest possible design but also the one that requires the strongest external invariant: that stream state is effectively frozen from the moment it is enqueued.

That invariant is plausible in a single-threaded event loop where the GC drain happens in the same tick as the enqueue. It is less plausible if the queue can grow and drain across multiple event loop iterations, because state can change between iterations.

---

## The Interaction With Finding 1

Finding 1 shows that `ready_for_gc` is a cached snapshot. Finding 4 shows that the GC queue is a frozen decision set. Together, these mean:

1. Stream state is evaluated once and frozen into `ready_for_gc`
2. Stream is enqueued based on that frozen evaluation
3. Queue drains without re-evaluating

The chain from state evaluation to release has no revalidation point. If the initial evaluation was wrong (due to a state change that `qsm_ready_for_gc` did not yet account for), that error propagates all the way to the release call.

---

## Validation

```
# Confirm the drain loop structure
grep -n -A 15 "while.*ready_for_gc_head" ssl/quic/quic_stream_map.c

# Confirm no revalidation call inside the loop
grep -n "qsm_ready_for_gc" ssl/quic/quic_stream_map.c

# Check list_remove to understand what cleanup happens on explicit stream deletion
grep -n -B 2 -A 5 "list_remove.*ready_for_gc" ssl/quic/quic_stream_map.c

# Confirm whether the event loop drains GC in the same tick as enqueue
grep -rn "ossl_quic_stream_map_gc\|quic_stream_map_gc" ssl/quic/
```

---

## Further Investigation

The key question is the temporal relationship between enqueue and the GC drain. If they always happen in the same event loop tick, the window for state change is effectively zero. If they can be separated by multiple ticks, the window grows.

```
# Find where the GC drain function is called from
grep -rn "ossl_quic_stream_map_gc" ssl/quic/

# Check the event loop tick structure in quic_channel.c
grep -n "stream_map_gc\|ready_for_gc" ssl/quic/quic_channel.c
```

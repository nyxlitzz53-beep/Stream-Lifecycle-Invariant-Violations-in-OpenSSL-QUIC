# Finding 2: Multi-Owner sstream Lifetime

**Files:** `ssl/quic/quic_stream_map.c`, `ssl/quic/quic_channel.c`  
**Lines:** map.c:187, 450, 502 / channel.c:390, 3075, 3847  
**Severity:** Latent  

---

## The Code

Six independent call sites free `sstream`:

```
// quic_stream_map.c:187  — stream teardown
ossl_quic_sstream_free(stream->sstream);

// quic_stream_map.c:450  — DATA_RECVD transition
ossl_quic_sstream_free(qs->sstream);
qs->sstream = NULL;

// quic_stream_map.c:502  — RESET_SENT transition
ossl_quic_sstream_free(qs->sstream);
qs->sstream = NULL;

// quic_channel.c:390     — crypto_send[], channel init failure
ossl_quic_sstream_free(ch->crypto_send[pn_space]);

// quic_channel.c:3075    — crypto_send[], channel teardown
ossl_quic_sstream_free(ch->crypto_send[pn_space]);

// quic_channel.c:3847    — qs->sstream, channel-level GC
ossl_quic_sstream_free(qs->sstream);
```

---

## What This Does

`sstream` is the send-side stream buffer. It holds data queued for transmission. The above sites all free it independently, from different layers (stream map layer vs. channel layer), triggered by different conditions (state transitions, teardown paths, GC paths).

Notably, the DATA_RECVD and RESET_SENT paths (lines 450, 502) null the pointer after free. The teardown path at line 187 and the channel-level path at line 3847 do not show the same null assignment in the grep output, which raises the question of whether those paths are safe to call in sequence.

---

## The Structural Problem

This is not a single owner with a clear free path. It is multiple subsystems that each believe they have authority over `sstream` lifetime in their domain. The stream map layer frees it on state transition. The channel layer frees it on teardown. The GC path frees it as part of stream collection.

The question is whether these paths are mutually exclusive in practice. The answer depends on ordering guarantees between the channel teardown sequence and the stream map GC sequence, and on whether a state transition free (line 450/502) can race with a channel-level free (line 3847) on the same stream.

In a single-threaded event loop this is probably fine. But the assumption of single-threaded execution is not encoded in the structure of the code. It is an external invariant that the code relies on without stating.

---

## What Makes This Interesting

The bug class here is not double-free in the traditional sense. It is lifetime authority ambiguity. No single subsystem owns `sstream`. Each subsystem frees it when its own conditions are met, and correctness depends on those conditions being non-overlapping. That is a design property, not a code property, and design properties do not show up in static analysis.

This is also where the multi-owner pattern interacts with Finding 1. If a stream is enqueued for GC (Finding 1), and separately a state transition triggers the DATA_RECVD free path, the stream may have its `sstream` freed before the GC loop reaches it. The GC loop does not check whether `sstream` is already NULL before operating on the stream.

---

## Validation

```
# All sstream free sites
grep -rn "ossl_quic_sstream_free" ssl/quic/

# Check whether teardown path nulls the pointer
grep -n -B 2 -A 5 "sstream_free" ssl/quic/quic_stream_map.c

# Check whether channel GC path nulls the pointer
grep -n -B 2 -A 5 "sstream_free" ssl/quic/quic_channel.c

# Check what the GC loop does with sstream after dequeue
grep -n -A 20 "ready_for_gc_head" ssl/quic/quic_stream_map.c

# Check whether ossl_quic_sstream_free handles NULL input safely
grep -n -A 10 "^void ossl_quic_sstream_free" ssl/quic/quic_sstream.c
```

The last command is important. If `ossl_quic_sstream_free` does not guard against NULL input, then any path that calls it without first checking `qs->sstream != NULL` is a latent null dereference.

---

## Further Investigation

The critical question is whether the DATA_RECVD free path and the channel teardown free path can be reached for the same stream without an intervening NULL check. Tracing the full teardown sequence in `quic_channel.c` around line 3847 against the stream map state at that point would answer this.

```
# Read full channel teardown context
grep -n -B 5 -A 15 "sstream_free" ssl/quic/quic_channel.c | head -80

# Check if stream is validated before channel frees it
grep -n -B 20 "3847" ssl/quic/quic_channel.c
```

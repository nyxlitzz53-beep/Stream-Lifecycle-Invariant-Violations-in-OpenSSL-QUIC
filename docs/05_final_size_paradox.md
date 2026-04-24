# Finding 5: send_final_size Multi-Source Derivation

**Files:** `ssl/quic/quic_stream_map.c`, `ssl/quic/quic_txp.c`, `ssl/quic/quic_impl.c`  
**Lines:** map.c:169, 411, 494 / txp.c:2558 / impl.c:793, 2841, 3274, 4516, 5754  
**Severity:** Structural  

---

## The Code

```
// Initialization — sentinel for unknown (quic_stream_map.c:169)
s->send_final_size = UINT64_MAX;

// Set from sstream layer (quic_stream_map.c:411)
if (!ossl_quic_sstream_get_final_size(qs->sstream, &qs->send_final_size))
    // ...

// Set from flow control watermark (quic_stream_map.c:494)
qs->send_final_size = ossl_quic_txfc_get_swm(&qs->txfc);

// Read at TX packetizer for FIN frame (quic_txp.c:2558)
if (!ossl_quic_stream_send_get_final_size(stream, &f.final_size))
    // ...

// Checked at impl layer for write eligibility (quic_impl.c:793, 2841, 3274, 4516, 5754)
ossl_quic_sstream_get_final_size(ctx.xso->stream->sstream, NULL)
```

---

## What This Does

`send_final_size` represents the total number of bytes the send side will transmit. It is used to construct the FIN frame, to determine write eligibility, and to validate stream termination. It is initialized to `UINT64_MAX` as a sentinel meaning unknown.

It has two write paths: line 411 derives it from the sstream layer (the logical stream buffer), and line 494 derives it from the TX flow control watermark. These are not the same thing. The sstream layer knows about application data. The flow control watermark knows about bytes committed to the network. They converge at FIN time, but there is a window where they can disagree.

Additionally, `quic_impl.c` checks `ossl_quic_sstream_get_final_size` directly on `qs->sstream` rather than reading `qs->send_final_size`. This means there are two authoritative sources for final size: the struct field and the sstream object itself.

---

## The Structural Problem

Final size is not a single value with a single writer. It is a derived property with multiple independent derivation paths and multiple read sites that use different derivation methods.

The sstream-level query (`ossl_quic_sstream_get_final_size`) reflects whether a FIN has been committed to the stream buffer. The struct field `send_final_size` reflects whether a final size has been recorded at the stream map level, via either the sstream path (line 411) or the flow control path (line 494).

A reader at `quic_txp.c:2558` uses `ossl_quic_stream_send_get_final_size`, which presumably reads the struct field. A reader at `quic_impl.c:793` queries the sstream object directly. These two sources are not guaranteed to agree at all points in time.

---

## Why This Is Interesting

This is not a buffer overflow or a use-after-free. It is an epistemic inconsistency. Different parts of the stack have different beliefs about when the stream is finished, because they derive that belief from different sources.

The consequence in practice is non-deterministic behavior at stream termination boundaries. Whether a write is allowed, whether a FIN is included in a packet, whether a stream is considered closed: all of these depend on which derivation path was consulted. In edge cases involving retransmits, partial ACKs, or RESET interactions, these paths can give different answers.

---

## The Flow Control Path Is Particularly Interesting

Line 494 sets `send_final_size` from `ossl_quic_txfc_get_swm`, the send watermark. This is the total number of bytes that have been admitted by flow control. At the moment a RESET is processed, the stream may not have sent all admitted bytes. Setting final size to the flow control watermark in this case gives a final size that reflects what was permitted, not what was actually sent.

That is a meaningful distinction for stream accounting. Whether the peer's stream state machine agrees with this derivation depends on whether the RESET frame carries the same value. If there is a discrepancy, the peer and local side may have different views of the final byte offset, which has implications for flow control credit accounting.

---

## Validation

```
# All writes to send_final_size
grep -n "send_final_size\s*=" ssl/quic/quic_stream_map.c

# All reads of send_final_size vs direct sstream queries
grep -rn "send_final_size\|get_final_size" ssl/quic/ | grep -v "^Binary"

# Read the flow control watermark path in context
grep -n -B 5 -A 10 "txfc_get_swm" ssl/quic/quic_stream_map.c

# Check what ossl_quic_stream_send_get_final_size actually reads
grep -n -A 10 "stream_send_get_final_size" ssl/quic/quic_stream_map.c

# Check whether RESET path uses watermark value consistently with what is sent in the RESET frame
grep -n "RESET\|send_final_size\|txfc_get_swm" ssl/quic/quic_txp.c | head -30
```

---

## Further Investigation

The most interesting thing to verify is whether the value written at line 494 (flow control watermark) matches the `Final Size` field in the RESET_STREAM frame that gets constructed in `quic_txp.c`. If they differ, that is a protocol-level inconsistency.

```
# Find RESET_STREAM frame construction in txp
grep -n -B 5 -A 20 "RESET_STREAM\|reset_stream" ssl/quic/quic_txp.c | head -60

# Cross-reference with what final_size value is used in the frame
grep -n "final_size" ssl/quic/quic_txp.c
```

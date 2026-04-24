# Stream-Lifecycle-Invariant-Violations-in-OpenSSL-QUIC
A couple low-severity bugs that i just think are cool.


# Stream Lifecycle Invariant Violations in OpenSSL QUIC

**License:** CC BY 4.0  
**Scope:** OpenSSL QUIC stream management (`ssl/quic/`)  
**Severity:** Low / Latent / Structural  
**Author:** Independent research

---

## Overview

This repository documents structural anomalies found in OpenSSL's QUIC stream lifecycle implementation. These are not memory corruption bugs. They are not CVEs. They are findings about how the system reasons about stream state across subsystems, and where that reasoning is inconsistent, stale, or split across independent authorities.

The distinction matters. A crash is easy to classify. What is harder to classify, and harder to find, is a system that behaves correctly under normal conditions but whose internal state model is incoherent at the seams. That is what this documents.

---

## Why OpenSSL Does Not Have High Severity Bugs Here(or at least none that i could find)

This is worth addressing directly, because the answer is not obvious.

OpenSSL is a library. It does not bind sockets. It does not schedule work. It does not make autonomous decisions about network state. Everything it does is in response to data and function calls handed to it by a caller. That architectural fact is the single biggest reason high severity bugs are rare in its QUIC implementation.

For a bug to be high severity in a library like this, an attacker needs to control the inputs that trigger the bad state, and those inputs need to produce a consequence that crosses a security boundary. In most of these findings, neither condition is cleanly satisfied.

The GC snapshotting anomaly (Finding 1) can produce a stream that lives too long or dies too early. But the consequence of that, in practice, is a logic inconsistency inside the library's own bookkeeping, not a memory corruption primitive or an information leak the attacker can read. The sstream multi-owner problem (Finding 2) creates a fragile ownership boundary, but the actual free paths are guarded well enough that triggering a double free requires a very specific sequence of state transitions that normal protocol operation does not produce.

The deeper reason is that QUIC stream state is fundamentally monotonic. Streams move forward. They do not cycle. A stream that reaches DATA_RECVD does not go back to SEND. That monotonicity is a natural constraint that limits how much damage a stale snapshot can do, because the snapshot will usually be wrong in a direction that produces conservatism rather than aggression. A stream classified as not ready for GC when it actually is ready wastes memory. A stream classified as ready when it is not is the dangerous direction, and the conditions for that are narrow.

That is why these findings are latent rather than exploitable. The design has enough incidental safety properties that the structural flaws do not reach consequence. But they are real flaws, and in a different protocol with less monotonic state, they would be a different conversation.

---

## Findings

| # | Title | File | Severity |
|---|-------|------|----------|
| 1 | [Cached GC Eligibility Snapshotting](docs/01_cached_gc_eligibility.md) | `quic_stream_map.c:349` | Latent |
| 2 | [Multi-Owner sstream Lifetime](docs/02_multi_owner_sstream.md) | `quic_stream_map.c`, `quic_channel.c` | Latent |
| 3 | [Send/Recv State Independence](docs/03_send_recv_independence.md) | `quic_stream_map.c:160` | Structural |
| 4 | [GC Queue Dequeue Without Revalidation](docs/04_gc_queue_no_revalidation.md) | `quic_stream_map.c:810` | Structural |
| 5 | [send_final_size Multi-Source Derivation](docs/05_final_size_paradox.md) | `quic_stream_map.c`, `quic_txp.c`, `quic_impl.c` | Structural |

---

## Validation Commands

To reproduce the observations in this repository against the OpenSSL source tree:

```
# Finding 1: all sites that touch ready_for_gc
grep -n "ready_for_gc" ssl/quic/quic_stream_map.c

# Finding 2: all sstream_free call sites across quic/
grep -rn "ossl_quic_sstream_free" ssl/quic/

# Finding 3: all send_state and recv_state transition sites
grep -n "send_state\|recv_state" ssl/quic/quic_stream_map.c

# Finding 4: GC queue insert and dequeue sites
grep -n "ready_for_gc_list\|ready_for_gc_head\|list_insert_tail\|list_remove" ssl/quic/quic_stream_map.c

# Finding 5: all send_final_size derivation and consumption sites
grep -rn "send_final_size\|get_final_size" ssl/quic/
```

---

*All findings are based on static analysis of the OpenSSL source. No runtime exploitation was performed or attempted.*

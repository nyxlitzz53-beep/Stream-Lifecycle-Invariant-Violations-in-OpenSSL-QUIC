# Finding 3: Send/Recv State Independence

**File:** `ssl/quic/quic_stream_map.c`  
**Lines:** 160-164, 248-530 (send transitions), 548-667 (recv transitions)  
**Severity:** Structural  

---

## The Code

```
// quic_stream_map.c:160
s->send_state = (ossl_quic_stream_is_local_init(s)
    ? QUIC_SSTREAM_STATE_READY
    : QUIC_SSTREAM_STATE_RECV);

s->recv_state = (!ossl_quic_stream_is_local_init(s)
    ? QUIC_RSTREAM_STATE_RECV
    : QUIC_RSTREAM_STATE_DATA_RECVD);
```

Send and recv state machines are initialized independently and transition independently through their respective switch blocks (lines 248 and 548).

---

## What This Does

A QUIC stream has two independent state machines. `send_state` tracks the outbound side: READY, SEND, DATA_SENT, DATA_RECVD, RESET_SENT, RESET_RECVD. `recv_state` tracks the inbound side: RECV, SIZE_KNOWN, DATA_RECVD, DATA_READ, RESET_RECVD, RESET_READ.

These evolve independently. There is no joint terminal state. There is no synchronization point where both sides confirm completion before the stream is considered finished.

---

## The Structural Problem

Stream completion, from the perspective of the GC system (`qsm_ready_for_gc`), is a predicate over both state machines. A stream is ready for GC only when both sides have reached terminal states. But neither state machine knows about the other. Each transitions based solely on its own inputs.

This creates partial death states. The send side can reach DATA_RECVD (all data acknowledged) while the recv side is still in RECV (still consuming incoming data). The GC predicate handles this correctly by requiring both, but the stream itself has no concept of this. It is two automata sharing a struct.

---

## Why This Is Interesting

In most protocol implementations, stream termination is a joint event. TCP has FIN/FIN-ACK. HTTP/2 has END_STREAM on both directions. The joint event is the natural synchronization point for resource cleanup.

QUIC separates these by design (RFC 9000 explicitly models send and recv as independent). But the consequence of that design choice in the implementation is that the stream's lifecycle is the Cartesian product of two state spaces. The terminal region of that product space is the intersection of both machines' terminal states, and reaching that intersection requires both machines to independently arrive there.

What is notable is that no code enforces this explicitly. The GC eligibility check computes it as a conjunction, but the stream itself has no "fully terminated" flag, no joint callback, no synchronization primitive. The two machines just run, and correctness depends on the GC check being the only place that cares about both simultaneously.

---

## The Interesting Edge Cases

**Send terminal, recv active:** Send side reaches DATA_RECVD. Stream is sending no more data. Recv side is in SIZE_KNOWN, still waiting on in-flight packets. GC correctly defers. But if Finding 1 applies and GC eligibility was snapshotted at a moment where recv happened to look terminal, this stream could be collected while recv is still active.

**Recv terminal, send in retransmit:** Recv side drains to DATA_READ. Send side is in DATA_SENT but waiting on ACKs for retransmitted frames. Same situation in reverse.

These are not bugs on their own. They become bugs in combination with the snapshot issue in Finding 1.

---

## Validation

```
# All send_state transitions
grep -n "send_state\s*=" ssl/quic/quic_stream_map.c

# All recv_state transitions
grep -n "recv_state\s*=" ssl/quic/quic_stream_map.c

# Read qsm_ready_for_gc to see how it conjoins both
grep -n -A 50 "static int qsm_ready_for_gc" ssl/quic/quic_stream_map.c

# Check whether any joint terminal condition is enforced at the stream level
grep -rn "send_state.*recv_state\|recv_state.*send_state" ssl/quic/
```

---

## Further Investigation

The interesting follow-on is mapping which external inputs (remote frames) can advance each state machine, and whether a sequence of inputs exists that leaves one machine in a non-terminal state after the GC snapshot has been taken. That traces through `quic_rx_depack.c` for recv transitions and the TX acknowledgment path for send transitions.

```
# recv state transitions triggered by rx_depack
grep -n "recv_state\|ossl_quic_stream_map_notify" ssl/quic/quic_rx_depack.c

# send state transitions triggered by ACK processing
grep -rn "DATA_RECVD\|RESET_RECVD\|send_state" ssl/quic/quic_ackm.c
```

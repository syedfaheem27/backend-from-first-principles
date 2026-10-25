# TCP: handshake and sequence numbers

**Why the 3-way handshake exists?**
- `SYN (client -> server):` "I want to talk, my starting sequence number is X."
- `SYN-ACK (server -> client):` "Got it (X+1). My starting sequence number is Y."
- `ACK (client -> server):` "Got yours (Y+1)."

- Result: both sides know the other is reachable (can send and receive) and have agreed
  on initial sequence numbers (ISNs). No HTTP data is sent before this.
- Without it, the sender wouldn't know if anyone is listening, nor have shared state
  to detect loss, reordering or duplicates. IP packets can be lost, delayed, reordered or duplicated.
- ISNs are random, not 0, so stale packets from old connections aren't mistaken for
  new ones (and it makes spoofing harder).

**How sequence numbers work?**
- Every byte in the stream has a number. A segment carries the sequence number of its first byte.
  E.g. a segment with seq=1000 and 500 bytes of data covers bytes 1000-1499.
- The receiver replies with an ACK number = the next byte it expects (here, 1500).
  ACKs are cumulative: "I have everything before this."

## Ordering
- Segments can arrive out of order (e.g. seq 1000, 2000, 1500).
- The receiver buffers them and uses sequence numbers to put them back in order.
- It only hands contiguous data to the application, so the app always sees an ordered stream.

## Detecting lost packets
- If the segment at 1500 never arrives, the receiver keeps ACKing 1500 (duplicate ACKs), even
  though later segments (2000, ...) have arrived. The gap in numbers shows what's missing.
- The sender also detects loss when no ACK arrives within the retransmission timeout (RTO).

## Retransmission
- `Triggers:` RTO expires, or 3 duplicate ACKs (fast retransmit).
- The sender resends the segment starting at the missing sequence number.
- The receiver uses sequence numbers to discard duplicates (a segment it already has).

## Quick recap
- Problem => How sequence numbers solve it
- Reordering => Place segments by seq number, deliver in order
- Loss => Gap in numbers / ACK stuck at missing byte / no ACK before timeout
- Retransmit => Resend from the unacked seq number
- Duplicates => Same seq number already received, so drop


## Questions

1. Packets 1, 2 and 3 arrive in the order 1, 3, 2. What does the receiver do with
   packet 3 when it arrives, and what ACK does it send back at that moment?
- It ACKs “2”, meaning “I expect 2 next”. That is a duplicate ACK, since it already
  sent that ACK after packet 1. The receiver buffers 3 and doesn’t give it to the app yet.
  When 2 arrives, it delivers 2 and 3 together and ACKs “4”.
  Also, real TCP numbers bytes, not packets.

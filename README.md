# rtp-nv

RTP, the Real-time Transport Protocol, carries audio and video over UDP. A
sender cuts a stream into packets, each with a sequence number and a
timestamp; a receiver puts them back in order, measures how many were lost
and how unevenly they arrived, and tells the sender. RTCP is the control
protocol that runs beside it: sender and receiver reports, the names of the
sources, and a goodbye. This package is the wire format and the bookkeeping of
both for novo-lang: [RFC 3550](https://www.rfc-editor.org/rfc/rfc3550), with
the header extensions of [RFC 8285](https://www.rfc-editor.org/rfc/rfc8285)
and the static payload types of [RFC 3551](https://www.rfc-editor.org/rfc/rfc3551).

## What it is

A microphone that streams G.711 to a phone needs four things from an RTP
library: to build a packet with the next sequence number and timestamp, to
read a packet back, to count what arrived and what did not, and to say how
often to send a report. This package is those four, and then the rest of the
control protocol: the report blocks, the source descriptions, the check that
tells a real RTCP datagram from noise, and the round trip a report block
carries.

The package has no socket, no clock and no source of random numbers. A
program that uses it owns the UDP socket and the timer. The package decides
what a packet looks like and what the numbers in it mean.

**Nothing is allocated.** A packet is a `Vec[u8; 1500]` in the caller's
frame. The state of a source, of a sender and of the report timer is a
`@value` the caller holds. The whole package builds for a Cortex-M with no
heap.

**Time is the caller's.** Arrival times are numbers in the clock of the
payload. The report timer is told what time it is and answers whether to
send. A fixed draw for the random factor makes the schedule repeatable in a
test.

## Install

```
novo pkg add rtp-nv
```

## Example

A sender builds one packet of G.711 audio, a receiver reads it and counts it,
and the receiver makes the report block it would send back. On a device the
packet buffer is the datagram buffer and the call is made when the UDP stack
hands over a datagram.

```novo
use rtpstat
use rtpwire

fn main() [io]
    // PCMU (payload type 0): sequence 1000, timestamp 160, SSRC 0xDEADBEEF,
    // four bytes of audio.
    var p: rtpwire.RtpPkt = Vec.new()
    var b = rtpwire.builder()
    let _ = rtpwire.begin(var p, var b, false, 0, 1000, 160, 0xDEADBEEF)
    let _ = rtpwire.push_payload_str(var p, var b, "ABCD")

    let h = rtpwire.parse(var p)
    var src = rtpstat.source_new(h.ssrc, h.seq - 1, 8000)
    let _ = rtpstat.source_update(var src, h.seq - 1)
    let counted = rtpstat.source_update(var src, h.seq)
    let r = rtpstat.source_report(var src, 0)
    println("${h.status} ${h.pt} ${h.seq} ${h.payload_len} ${counted} ${r.ext_seq} ${r.lost}")
    // 0 0 1000 4 true 1000 0
```

Build and test with `novo pkg build` and `novo test tests`.

## What the package contains

| Module | Contents |
| --- | --- |
| `rtpwire` | The RTP packet of RFC 3550 section 5: the fixed header, the CSRC list, padding, the header extension and its two forms in RFC 8285 (one-byte and two-byte), a builder that writes them in order into a caller's buffer, a parser that checks them from end to end, an iterator over the extension's elements, and the 16-bit sequence arithmetic. |
| `rtcpwire` | The RTCP packets of section 6: the iterator over a compound packet, the validity check of Appendix A.2, and readers for SR, RR and their report blocks, SDES with its items, BYE and APP. |
| `rtcpbuild` | A builder for a compound packet: SR, RR, report blocks, SDES chunks and items, BYE, APP and padding. |
| `rtpstat` | The receiver's bookkeeping of Appendix A: the extended sequence number and the probation of a new source, the packets expected and lost, the fraction lost, the interarrival jitter, the last-SR time and the delay since it, the round trip of section 6.4.1, NTP time conversions, and the sender's counts. |
| `rtcptimer` | The report interval of section 6.3: the deterministic interval from the member count and the bandwidth share, the random factor, reconsideration, the reverse reconsideration when members leave, and the BYE backoff. |
| `rtpstatic` | The static payload types of RFC 3551: kind, encoding name, clock rate and channels of each, and a lookup from a name. |
| `rtpbytes` | The big-endian reads and writes the rest are made of. |

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/rtp-nv) is that reference.

## The rules a user needs

1. **A packet is built in order.** The header comes first, then the CSRC
   list, then the extension, then the payload, then padding. The builder
   refuses a step out of order and anything that does not fit. A refusal
   answers `false`, empties the packet, and marks the `RtpBuilder` failed, so
   a run of calls can be checked once at the end with `b.ok`. `begin` starts
   again.
2. **A parse is checked to the end.** `parse` checks the version, the CSRC
   list, the extension and the padding count, so a packet that parses has a
   payload that starts where `payload_off` says. A fault is a code in
   `status`.
3. **The extension is a walk.** `ext_iter` and `ext_next` step over the
   elements of an extension in the one-byte or two-byte form, skip padding
   and stop at an element that runs past the extension, which sets `fault`.
   `ext_find` returns the first element with an ID. An extension with any
   other profile value is carried and not read.
4. **A 32-bit field is an `Int`.** SSRCs, timestamps, NTP halves and counts
   are 0 to 4294967295 in an `Int`. Arithmetic that wraps at 2^32 says so
   (`ts_add`, `wrap32`); sequence numbers wrap at 2^16 and are compared on
   the circle (`seq_diff`).
5. **A new source is not believed at once.** `source_update` answers `false`
   until two packets in sequence have arrived, and for a jump of more than
   3000 until the packet after it agrees. The packet that ends the probation
   is the first one counted, as in the RFC's C code.
6. **The package does not read a clock.** `source_arrival` is told when a
   packet arrived, in the units of the payload's clock (`ms_to_units`
   converts). `source_report` and `source_sr` are told the time in
   milliseconds so the delay since the last SR can be computed.
7. **A report timer is a state the caller drives.** `timer_check` takes the
   time and a draw from 0 to 1000 and answers `RTCP_SEND` or `RTCP_WAIT`;
   `timer_sent` records the send. When the interval recomputed at the due time
   is later, `timer_check` moves the due time and answers `RTCP_WAIT`.
8. **A datagram is checked before it is believed.** `rtcp_check` is the
   validity check of Appendix A.2: version 2, a first packet that is an SR or
   an RR with no padding, lengths that add up to the datagram. It also walks
   every packet and reports one whose body is shorter than its type and count
   need.
9. **A packet type the package does not know is returned and skipped.** The
   iterator yields every packet; a caller reads the five it knows.
10. **A compound packet is closed by `rtcp_finish`.** The length and count of
    each packet are written when the next opens or when it is finished.
    Starting with `rtcp_sr` or `rtcp_rr` empties the buffer.

## Running on a microcontroller

All seven modules are `@tier(embedded)` throughout and allocate nothing, and
the package declares no effects. `tests/embedded_probe.nv` is a firmware
program for a networked microphone. It builds an RTP packet with a CSRC and a
header extension and reads it back, builds a compound RTCP packet (SR with a
report block, SDES, BYE) and runs it through the validity check, takes
packets across a sequence wrap with one lost and reports the loss and the
jitter, works out a round trip from a report block, runs the report timer and
looks up a static payload type. It is compiled for a Cortex-M as part of this
package's checks, so the claim is built rather than asserted, and under QEMU
it prints `PASS: rtp-embedded`.

The image is 25,912 bytes of flash and 520 bytes of static RAM, including the
runtime and the board's startup code. The program's own code is 12,574 bytes
of that flash. The probe's frame holds an RTP packet buffer and an RTCP
buffer of 1,512 bytes each, a small data buffer of 264 bytes and the state of
a source of 120 bytes. A caller that needs less holds fewer packets, not a
smaller type: the capacity of `RtpBuf` is `RTP_BUF_MAX`.

The device effects are the caller's. The package reads no register and no
clock, so a call costs what its caller's datagram transport costs.

## What is not included

- **SRTP and SRTCP, RFC 3711.** A datagram arrives as bytes the caller has
  already decrypted, or does not.
- **Payload formats.** The payload is bytes the package does not read. The
  packetisation of H.264, Opus and the others (RFC 6184, RFC 7587) is another
  package's.
- **Reports of the extended kinds,** XR (RFC 3611), and the feedback messages
  of RFC 4585 (NACK, PLI) and RFC 5104. The iterator returns such a packet by
  its type and the caller reads its body.
- **A jitter buffer.** The package reorders nothing. It says how late a
  packet was and how many were lost.
- **The member table of section 6.2.** The caller keeps its list of sources,
  one `RtpSource` for each, and tells the timer how many members and senders
  there are. Timing a member out is the caller's.
- **Sockets, timers and random numbers.** The package never reads one.
- **Collision resolution of section 8.2 and SSRC allocation.** The caller
  chooses its SSRC.

## Related packages

- [sdp-nv](https://novo-lang.org/packages/sdp-nv) reads and writes the session
  description that says which payload types a stream uses and on which ports.

## Tests

The suite is four files under `tests/`. What they assert comes from the RFCs
rather than from an implementation.

`tests/rtpwire_tests.nv` holds the packet to byte strings worked out by hand
from sections 5.1 of RFC 3550 and from RFC 8285, including both extension
forms, padding between elements, the reserved ID 15, and every parse fault and
every way a builder refuses. It also holds the byte arrays of pion/rtp's
`packet_test.go` (MIT) and the values that file expects them to parse to, an
independent implementation's reading of the same RFCs.

`tests/rtcp_tests.nv` holds a compound packet (SR with a report block, SDES,
BYE) to the byte, the APP packet, padding, every check of Appendix A.2 and
every fault of the SDES walk.

`tests/rtpstat_tests.nv` follows Appendix A step by step: the probation, the
sequence wrap, duplicates, late packets and restarts, the fraction lost, the
24-bit clamp, the jitter on a hand-computed arrival sequence including a
timestamp that wraps, the round trip, the report interval for each branch of
section 6.3.1, the reconsideration, and RFC 3551's tables.

`tests/rtpbytes_tests.nv` holds the byte-order helpers.

`tests/embedded_probe.nv` is the microcontroller program described above.

## Licence

Apache-2.0. See `LICENSE`.

# Changelog

All notable changes to rtp-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`rtpwire`**, the RTP packet of RFC 3550 section 5 with no allocation: the
  fixed header, the CSRC list, padding and the header extension of RFC 8285 in
  its one-byte and two-byte forms, and a raw form for any other profile. A
  builder (`begin`, `add_csrc`, `ext_begin`, `ext_add`, `ext_word`, `ext_end`,
  `push_payload`, `push_payload_str`, `push_payload_from`, `add_padding`)
  writes in order and refuses what is out of order or does not fit. `parse`
  checks the version, the CSRC list, the extension and the padding count and
  reports a fault as a code. An iterator (`ext_iter`, `ext_next`, `ext_find`)
  reads the extension's elements in place. `seq_diff`, `seq_newer`, `next_seq`
  and `ts_add` do the 16-bit and 32-bit arithmetic.
- **`rtcpwire`** and **`rtcpbuild`**, the RTCP packets of section 6: SR, RR
  with report blocks, SDES with the eight item types, BYE with a reason and
  APP; the iterator over a compound packet and the validity check of
  Appendix A.2 (`rtcp_check`); and a builder for a compound packet with
  padding.
- **`rtpstat`**, the receiver bookkeeping of Appendix A: `source_update` is
  `update_seq` with the probation of a new source, the sequence wrap and the
  restart on a jump; `source_report` makes a report block with the loss
  fraction, the cumulative loss held to 24 bits, the jitter and the LSR and
  DLSR; `source_arrival` is the jitter estimate of A.8; `rtt_ms` is the round
  trip of section 6.4.1; `ntp_middle`, `ntp_hi_of_ms` and `ntp_lo_of_ms`
  convert NTP times; a sender counts packets and octets.
- **`rtcptimer`**, the report interval of section 6.3 as a timer the caller
  drives (`timer_new`, `timer_check`, `timer_sent`, `timer_heard`,
  `timer_members_changed`, `timer_bye`), with the sender and receiver shares,
  the 5 s and 2.5 s minimums, the random factor from a caller's draw, the
  compensation of 1.21828, reconsideration, reverse reconsideration and the
  BYE backoff of section 6.3.7.
- **`rtpstatic`**, the static payload types of RFC 3551 tables 4 and 5: kind,
  name, clock rate and channels, a case-insensitive name comparison and a
  lookup from name, clock rate and channels.
- **`rtpbytes`**, the big-endian reads and writes over a buffer the caller
  owns.
- **`tests/embedded_probe.nv`**, a firmware program for a networked
  microphone, built for `--target=nrf52-qemu`. It prints `PASS: rtp-embedded`
  under QEMU. The image is 25,912 bytes of flash and 520 bytes of static RAM.
- Four test suites with vectors worked out by hand, among them a compound
  packet (SR with a report block, SDES, BYE) to the byte, and the RTP byte
  arrays of pion/rtp's `packet_test.go`.

### Not included

- SRTP and SRTCP (RFC 3711).
- Payload formats and their packetisation.
- XR (RFC 3611) and the feedback messages of RFC 4585 and RFC 5104.
- A jitter buffer.
- The member table of section 6.2 and the collision resolution of
  section 8.2.
- Sockets, timers and random numbers.

### Known

- Two `cov: skip` markers excuse the line after a bounded loop in `ext_next`
  and `sdes_next`: the loop returns on every pass that advances, so the line
  that follows it cannot run.

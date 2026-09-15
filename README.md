# ieee80211

IEEE 802.11 management-frame build/parse and the WPA2-PSK/CCMP RSN
Information Element, pure vāṇी, hardware-agnostic and heap-free.

Extracted from [Dhruva OS](https://github.com/enthusiasticgeek/dhruvaos)'s
Pi 1 kernel, where the 4-way handshake state machine's own header
comment already called out the gap this fills: a real access point
won't accept EAPOL-Key frames from a station that hasn't first
associated with it (Probe Request/Response, Open System Authentication,
Association Request/Response, in that order). Every function takes
fixed-size array parameters and writes through a `mut ref` array or
returns a small all-scalar struct -- no heap allocator, no
`extern "C"` scratch accessors, no RF/USB transmit-receive plumbing of
any kind (that's the consuming kernel's own, separately-scoped job --
this package only builds/parses the BYTES of a frame).

## What's here

- **Generic MAC header parse** (24 bytes, no Address4 -- management
  frames never set both ToDS and FromDS).
- **Generic IE scanner** (`ie_find`) for the Element-ID/Length/Value
  wrapper every IE uses, bounded against a caller-supplied length so a
  malformed length can't read past the caller's own intended region.
- **The WPA2-PSK/CCMP RSN Information Element**: build (a fixed,
  minimal, spec-compliant 22-byte IE) and a support-check parse
  (does a found RSN IE advertise CCMP + PSK, the combination this
  package's own build produces).
- **Probe Request build / Probe Response parse** -- the standard way to
  actively scan for a specific SSID and learn whether it's WPA2-PSK/
  CCMP-capable.
- **Open System Authentication request build / response parse** -- the
  only algorithm WPA2-PSK ever uses (the real "authentication" happens
  later, in the 4-way handshake itself).
- **Association Request build (with an embedded RSN IE) / response
  parse** -- the final step before EAPOL-Key can start.

Deliberately does NOT include: Beacon parsing (Probe Response covers
the same fields this package needs), WEP/Shared-Key authentication,
Deauthentication/Disassociation, or any encoding beyond what's needed
for the one real join sequence WPA2-PSK actually uses.

## Correctness

- RSN IE field order and the WPA2-PSK/CCMP cipher-suite/AKM-suite byte
  values (00-0F-AC OUI, CCMP=4, PSK=2) verified against
  wpa_supplicant's own `src/common/wpa_common.h`/`wpa_ie.c`,
  cross-checked against Wireshark's own independent
  `packet-ieee80211.c` dissector tables. The worked hex example in
  `rsn_ie_build_wpa2_psk_ccmp`'s own header comment is derived from
  these same confirmed constants.
- Frame Control bit layout, frame type/subtype values, and the
  Authentication/Association fixed-field layouts verified directly
  against Linux's own `include/linux/ieee80211.h` -- the exact structs
  `mac80211`/`cfg80211` build and parse for every real device Linux
  drives, not a derivation from prose alone.
- Every build function's own output is checked byte-exact against
  hand-computed expected bytes (not just "build then parse agrees with
  itself"); every parse function is checked against independently-
  placed bytes, plus real negative cases (wrong cipher, wrong AKM, a
  non-zero status code, an out-of-sequence transaction number, an IE
  claiming a length that would run past its own scan bound).

All self-tests pass under both `vanic run`'s default backend and
`--backend=llvm` (what Dhruva OS's own kernel build actually uses).

## Layout

- `src/core.vani` -- the primitives alone (no self-tests), for
  consumers (like Dhruva OS's own kernel) that want the functions
  without vendoring test code.
- `src/lib.vani` -- `core.vani`'s own content plus
  `ieee80211_self_test`, this package's actual `entry` point.
- `test/host_test.vani` / `test/host_core_test.vani` -- host-runnable
  validation (`vanic run test/host_test.vani`).

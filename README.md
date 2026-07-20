# DRAGNET

**Field Instrument FI-123**
Offline packet capture reader. Version 1.5.0.

Drop a `.pcap` or `.pcapng` on the page and get conversations, a protocol
breakdown, a timing ladder, and a plain-language account of what the traffic
was doing. Nothing is uploaded. Nothing is installed. One HTML file.

DRAGNET is the wired counterpart to PICKET. PICKET watches the air live and
renders verdicts on what it hears. DRAGNET reads a file after the fact and
reconstructs the story. Between them the RF side and the packet side of an
audit are covered.

---

## Running it

Double-click `dragnet.html`. That is the whole procedure.

It runs from `file://` with no server, no build step, no dev server and no
network access. The only outbound link in the file is the feedback link in
the About panel, and it only fires if you click it.

## The stations

| Station | What it holds |
| --- | --- |
| **Catch** | Slice totals, busiest addresses, and the sortable conversation table. Click a row to open it on the ladder. |
| **Breakdown** | Protocol hierarchy with per-layer packet and byte shares, traffic by service port, and the packet size profile. For a capture taken off the air, an access point table naming every radio heard, its channel, band, security suite and signal. |
| **Ladder** | One conversation drawn as a timing ladder: offset, direction, reading, byte count, plus a written summary of how the exchange went. Below it, **Follow stream**: the reassembled bytes of either direction as text or hex, with the application messages listed above them. |
| **Findings** | The written pass. An opening brief, then itemised findings sorted worst first. |
| **Packets** | The flat list, for when you want to see the raw order of events. |
| **Evidence** | What the capture is worth after the reading: engagement details, named assets, carved files with hashes, certificates, voice streams, the baseline comparison, and the session file. |

Keys `1` through `6` jump between stations. `0` casts the whole net.

## The drag bar

The signature control. The strip across the top is the entire capture drawn as
a log-scaled density band, coloured by transport. The bracketed window on it is
the net.

- Drag inside the net to slide it along the capture.
- Drag either edge to widen or narrow it.
- Drag on empty strip to cast a new net there.
- Press **Cast the whole net** or `0` to go back to everything.

Every station recomputes from whatever is inside the net. This is how you take
a five hour capture and ask what was happening in one particular ninety seconds.

## Scale

The reader does not dissect the whole file. It walks the packet headers once,
recording only where each packet sits, how long it is, when it arrived and a
shallow classification for the drag bar. Full dissection happens for the net
and nowhere else.

On a 600,000 packet, 42 MB capture that comes out as:

| Step | Time |
| --- | --- |
| Read and index the file | about 0.3 s |
| Draw the drag bar | 20 ms |
| Dissect, analyse and write findings for a 30,000 packet net | about 0.75 s |

The whole-file cost stays flat because the packets are never built until
something asks for one. The index tops out at five million packets; a single
net dissects at most three hundred thousand, and says so plainly when it is
holding back rather than quietly truncating.

Drop several files at once and they are read as one timeline, which is how a
ring buffer capture actually arrives. Mixed link types are allowed and each
packet is dissected by its own file's link type, with a warning saying the set
is mixed.

## What it reads

**Container formats**

- Classic libpcap, both endiannesses, microsecond and nanosecond timestamps
- pcapng: section headers, interface descriptions with `if_tsresol`, enhanced
  packet blocks, simple packet blocks
- gzip of either, if the browser exposes `DecompressionStream`

**Link layers**

Ethernet (with 802.1Q and QinQ tags), raw IPv4, raw IPv6, Linux cooked
capture v1 and v2, BSD loopback, and, since v1.2, 802.11 in all four of the
forms a monitor mode capture usually arrives in: radiotap, bare 802.11, Prism
and AVS. Anything else is counted and named but not dissected.

**Protocols**

ARP, IPv4, IPv6 (with extension header walking), TCP, UDP, ICMP, ICMPv6,
IGMP. Above that: DNS and mDNS with name decompression and answer parsing,
TLS records with handshake typing and SNI extraction, HTTP request and
response lines with headers, and line-level naming for FTP, SMTP, POP3, IMAP,
Telnet and IRC.

**802.11**

Radiotap fields (signal, noise, channel, frequency, band, rate, FCS status),
then the frame itself: management, control, data and extension types with
their subtypes named. Beacons and probe responses are opened for SSID,
channel, capability and the RSN or WPA information element, which is read down
to the key management suite and cipher, so a network is reported as WPA2,
WPA3, OWE, transitional, WPA, or WEP rather than just "encrypted". Probe
requests, deauthentication and disassociation with reason codes in words,
authentication algorithm and status. Unencrypted data frames are unwrapped
through LLC and SNAP and handed back to the ordinary dissectors, so ARP and IP
over the air read exactly as they do over a cable. EAPOL key frames are
recognised and numbered one through four across the handshake.

## Reading deeper

**TLS.** Handshake messages are reassembled across records before parsing,
which is the only way a certificate chain is readable at all. Client hellos
give up the requested name, the offered versions, the cipher list and a JA3
fingerprint computed exactly as the specification defines it, so it can be
compared with anyone else's. Server hellos give the negotiated version and
cipher, with weak suites named as weak. Certificates are walked in DER: subject,
issuer, validity, key type and size, curve, signature algorithm, subject
alternative names, and a SHA-256 fingerprint of the certificate itself.
Verified against real openssl output rather than only against test fixtures.

A TLS 1.3 session encrypts its own certificate. Dragnet says so where a naive
reader would leave a suspicious blank.

**Carving.** Files are pulled whole out of cleartext streams, de-chunked where
the transfer was chunked, typed by magic number rather than by extension or by
what the server claimed, and hashed with SHA-256 and MD5. A file that does not
match its declared type is flagged, and so is an executable fetched over plain
HTTP, because that is the one anyone on the path can swap.

Both hashes are written out longhand in the page. A file opened from disk has
no access to the browser crypto API, and this file must keep working from disk.

**Everything else.** DHCP with hostnames, vendor class and leases; SNMP with
the community string that is doing duty as a password; SMB1 and SMB2 with
share names off the tree connect; LDAP simple binds, which put a password in
the request as plain bytes; Kerberos principals, realms and the encryption
types still on offer; SIP call setup with the numbers and the call id; RTP
recognised by shape rather than by port, with codec, sequence loss and, for
G.711, a WAV file you can listen to; QUIC read as far as it lets anyone read,
which is the version and the connection identifiers and no further.

## Reassembly

Since v1.1 DRAGNET rebuilds before it reads.

**TCP.** One state machine per direction of every conversation, tracking the
initial sequence number from the handshake (or noting that the capture started
mid stream and the offsets are relative), the set of byte ranges already
accepted, and the far end's acknowledgements. Every segment carrying data
comes out classified as in order, out of order, retransmission, fast
retransmission, spurious retransmission, keep alive or overlap. That
distinction matters: a retransmission covers ground already accepted, while an
out of order arrival lands past a hole that has not been filled yet. Counting
the second as the first is the usual reason a capture reads worse than the
link actually is.

**IP.** IPv4 and IPv6 fragments are grouped by source, destination, identifier
and protocol, ordered by offset, and reassembled only when the run is
contiguous from zero and the last piece says so. Anything short of that is
reported as incomplete rather than quietly stitched together. Rebuilt
datagrams are re-dissected, so a DNS query spread across three fragments shows
its name.

**Streams.** The rebuilt bytes are decoded as a whole, which is the entire
point. HTTP messages are read across segment boundaries with Content-Length
and chunked bodies walked, so a request split three ways is one message and a
header sitting past the first packet is finally visible. TLS is walked record
by record with handshake messages typed and application data collapsed. Holes
are marked in the byte view rather than closed up.

Streams stop at four megabytes per direction. Past that the counts stay
correct and the byte view is short, and the findings say so.

## What the findings look for

- Encrypted destinations named in the clear via TLS SNI
- Plain HTTP use, with methods and hosts
- DNS lookups, including lookups that came back with no such name
- Credentials crossing the wire without transport encryption
- Host sweeps and port sweeps
- Connection attempts that got no answer, and attempts refused with RST
- Retransmission rate, with the three percent rule of thumb called out, broken
  down into timeout, fast and spurious retransmissions
- Segments that arrived out of order, reported as reordering rather than loss
- Holes left in rebuilt streams, which are almost always a capture problem
  rather than a network problem
- Datagrams rebuilt from IP fragments, and fragment groups left incomplete
- One IP address claimed by more than one hardware address
- Discovery chatter (mDNS, SSDP, NetBIOS) and what it advertises

**On the air**

- Every access point heard, with channel, band, security suite and signal
- One network name advertised by more than one radio, which is what an
  impersonating access point looks like and also what an ordinary multi radio
  deployment looks like. The finding says both and tells you to check the
  hardware list.
- Networks with no encryption at all
- Networks still on WEP, and networks still offering TKIP
- Access points beaconing without a name, with a note that hiding a name is
  not a security control
- Deauthentication and disassociation bursts, rated against the capture
  duration, with the reminder that these frames are unauthenticated and the
  sender named in them proves nothing
- Four way handshakes, complete and partial, with a plain account of what a
  complete one does and does not hand an attacker
- Devices naming networks in probe requests, what that leaks about where they
  have been, and why a modern phone usually does not do it
- Encrypted data frames, counted and described. No decryption is attempted.

Findings are ranked, not scored. DRAGNET will tell you what it saw and why it
matters, and leaves the judgement to you.

## Evidence

Reading a capture is half the job. The Evidence station is the other half.

- **Engagement details.** Client, reference, operator, capture point. They
  head the written report and travel in the session file.
- **Named assets.** Give an address a name and every finding, every table and
  the whole report start using it. A capture reads very differently when it
  says the badge reader instead of 10.0.7.14.
- **Carved files and certificates.** With hashes, so an artifact can be
  discussed, compared and cited without the bytes leaving the machine. Saving
  one writes it to disk deliberately, and the panel says so above the button.
- **Voice streams.** G.711 saved as WAV, which is the plainest possible
  demonstration that a call was not encrypted.
- **Accepted findings.** Mark a finding accepted with a note about why, or
  what was done about it. Accepted findings dim, sink to the bottom, and are
  recorded as accepted in the report and every export. An audit that cannot
  record a decision just produces the same list again next quarter.
- **Baseline.** Load a session file from an earlier capture and the station
  says what is new, what is gone and what is unchanged, across addresses,
  listening services, ports and access points. That is the question most
  people actually have: is this different from last time.
- **Session file.** Engagement, names, acceptances and an inventory. No
  payload, so it is safe to keep alongside a report and safe to hand to
  someone else as a baseline.
- **Test Bench export.** The same material shaped as evidence for a TEST BENCH
  (FI-081) run, so a capture can be attached to a test as observation rather
  than retyped.

## Exports

All three cover the packets currently inside the net.

- **Session JSON** for machine handoff, now carrying the engagement, asset
  names, artifacts with hashes, certificates, TLS sessions, voice streams,
  management protocol observations and which findings were accepted
- **Written report** as plain text, suitable for pasting into an engagement
  writeup, with named assets, carved files, certificates, voice and accepted
  findings as their own sections
- **Conversations CSV** for a spreadsheet
- **Session file** and **Test Bench export** from the Evidence station

Payload bytes are never written into these three exports. Only headers,
summaries and counts leave the page. The JSON gained a `reassembly` section and
an `air` section in v1.2, both metadata only.

The one exception is deliberate and lives elsewhere: **Save stream** on the
Follow stream panel writes the rebuilt payload of one direction of one
conversation to a text file. That is what the feature is for, the panel says so
above the button, and it is never reachable from the export dialog.

## Self test

The **Self test** button runs 274 assertions on demand: formatters, both
container readers across both endiannesses, a pcapng round trip, every
dissector, conversation keying, the analysis pass, and the findings engine.

Four synthetic captures are built in memory to drive them. The original mixed
wired sample; a reassembly fixture with a request cut across three segments
(with a header deliberately placed past the first one), a chunked response
whose second chunk arrives before its first, a segment that was never captured
so a hole must survive, and a DNS query in three IP fragments plus a fragment
group that must be left alone; and an air fixture with two radios beaconing the
same name, an open network, a hidden one, a device calling out its history in
probe requests, a complete four way handshake, ARP over the air, an encrypted
frame and a deauthentication burst.

And a deep fixture with an image split across segments, an executable served
as text/plain, a TLS session whose certificate message is deliberately split
across two records, an expired certificate signed with SHA-1 on a 1024 bit key,
SNMP with a default community string, an LDAP simple bind, a Kerberos AS-REQ
offering RC4, both SMB versions, DHCP, a SIP call with an RTP stream missing
one packet, and a QUIC initial.

The fixtures are part of the file, so the button works with no capture loaded
and no network.

Both hash implementations are checked against published test vectors, and the
JA3 fingerprint is checked against the published reference pair, so a
fingerprint from this tool can be compared with one from anywhere else.

A separate headless run drives the page itself through 95 checks: every
station, the drag bar, follow stream, the air panel, multi-file loading, every
export, carved file and WAV saving, asset naming changing what the findings
say, finding acceptance, session save and reload, and the baseline diff.

## Known Limitations

- Encrypted payloads are described, not decrypted. TLS reporting stops at the
  handshake. 802.11 encrypted data frames are counted, never opened. There is
  no WEP or WPA decryption and no key log support.
- Stream level decoding covers HTTP and the shape of TLS. Every other
  application protocol is still read one packet at a time.
- Rebuilt streams are held whole in memory and stop at four megabytes per
  direction.
- Reading 802.11 needs a capture taken in **monitor mode**. A capture from an
  ordinary associated interface has already been turned into Ethernet frames
  by the driver, and there is nothing left for the 802.11 dissector to read.
- 802.11 control frames carry fewer addresses than data frames, so some of
  them name only the receiver and cannot be attributed to a conversation.
- Frames failing their FCS are counted and flagged but still dissected, since
  a bad checksum in a monitor capture is usually a reception error rather than
  a malformed frame.
- Link types beyond Ethernet, raw IP, Linux cooked, BSD loopback, radiotap,
  Prism and AVS are counted and named but not dissected.
- The reader indexes up to five million packets. A single net dissects at most
  three hundred thousand, and says so rather than quietly truncating. Narrow
  the net to read the rest.
- Reassembly and defragmentation work on what is inside the net. A fragment or
  a segment outside it is not pulled in.
- The file bytes are held in memory, though the packets are not built until
  something asks for one.
- Carving works on cleartext only. Anything inside TLS is counted and
  measured, never reconstructed.
- A JA3 fingerprint identifies software, not a person and not an intent.
- RTP is recognised by shape rather than by port, so an occasional UDP stream
  may be misread as audio.
- There is no QUIC decryption and no key log support, so a QUIC connection is
  endpoints, timing and volume only.
- Timestamps are taken at face value. Captures from a machine with a bad clock
  will produce a bad timeline, and pcapng simple packet blocks carry no time
  at all.
- The scan heuristics count SYNs inside the net only. Narrowing the net can
  hide a slow sweep; widening it can make a busy load balancer look like one.
- No PWA layer. That is deliberate, and keeps the `file://` rule intact.

## Privacy

The capture is read with the browser File API and stays in page memory for the
life of the tab. Theme choice is the only thing written to local storage, and
that write is wrapped so a browser with storage blocked degrades quietly.

## License

GPL-3.0

## Credits

No vendored libraries. Every parser, every table and every pixel in this file
is hand written.

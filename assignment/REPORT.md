# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **0050**

## 1. Analysis: the loop

I built the loop by re-pointing exit 3 on s2 back at s1. On the command line, inside
the running network:

    sh ./p4cli s2 table_dump MyIngress.myTunnel_exact
    sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"

s2's entry for dst_id=3 already went to port 3, which is where h3 hangs off it. Changing it
to port 1 sends the packet back out the trunk towards s1, and s1's own entry for dst_id=3
sends it back to s2, so the packet ping-pongs forever.

`assignment/evidence/loop-before.txt` shows what that looks like over time: the table dump
before the change, then two pass counts from `logs/s1.txt` and `logs/s2.txt` immediately
after the send and five seconds later. The counts keep rising — nothing in the program
stops the packet.

The IPv4 path cannot rescue this loop. `ipv4_forward` decrements `hdr.ipv4.ttl`, but the
tunnel path (`myTunnel_forward`) never looks at the IPv4 header at all: the ingress only
runs `myTunnel_exact`, and `myTunnel_forward` only sets `egress_spec`. The TTL therefore
stays at 64 for the whole loop, no router ever sees TTL=0, and no `mark_to_drop` fires on
it. In a normal headend that decapsulates into IPv4, the TTL would matter; inside the
tunnel, it does not, because nothing on this path decrements it. The only bound on the
loop is the switch's forwarding rate — which is exactly why a hop limit was needed.

## 2. Design: the hop limit

I added one field to `myTunnel_t`:

    bit<8> hop_limit;   // switches this packet may still cross; 0 = drop

Placed **after `dst_id`**, so the first four bytes of the tunnel header are unchanged and
`proto_id` and `dst_id` still sit at the same offsets any tool already parses. A `bit<8>`
matches IPv4's TTL range (0–255) and costs one byte per packet; more than 8 bits would be
waste for a limit that the sender should set small anyway.

Starting value is **8**, set by `send.py` as `--hops` (default 8). Eight is enough headroom
for a real path (three hops in the lab) with plenty of margin, and small enough that when
the loop test hits it, the loop dies within seconds instead of flooding the logs. The
decrement happens in `myTunnel_forward`, one per switch the packet traverses, mirroring
where `ipv4_forward` decrements TTL. When the field reaches 0 in the action, `mark_to_drop`
runs, which sets `standard_metadata.mcast_grp` and the drop bit so the pipeline drops it
with no egress port selected.

Hosts had to change too, because Step 6 of the lab showed that a mismatch between the
switch's idea of the header and the sender's is silent. `send.py` and `receive.py` both
declare the field on their Scapy `MyTunnel` layer (`ByteField("hop_limit", 8)`), and
`receive.py` prints the arriving value. No parser change was needed — the parser already
extracts the whole header — but the deparser emits it because it emits `hdr.myTunnel` as a
whole, so no deparser change either.

## 3. Implementation

Three files changed:

- `tunnel.p4` — added `hop_limit` to the header, and changed `myTunnel_forward` to check it
  and decrement it, calling `mark_to_drop` at 0.
- `send.py` — added a `--hops` argument (default 8) and passed it into `MyTunnel(hop_limit=
  args.hops)`.
- `receive.py` — added `hop_limit` to its `MyTunnel` layer and printed it in the delivery log.

One detail not visible from the diff: `mark_to_drop` sets `egress_spec` to the drop port
internally, so the pipeline handles it exactly as the default-drop path in the table does —
there is no extra exit path or exception.

## 4. Evidence: it works

`assignment/evidence/delivery.txt`:

- Two tunnelled sends from h1, one to h2 (--tunnel 2) and one to h3 (--tunnel 3). The
  Scapy `show2()` output shows the sender's hop_limit at 8, and the `receive.py` logs on
  h2 and h3 both show the arriving value at **6** — the two switches each decremented once,
  which matches the pipeline.
- A plain `h1 ping -c 2 10.50.0.3` alongside, with 0% packet loss and ttl=62 in the reply
  — so `ipv4_lpm` and `ipv4_forward` are untouched and IPv4 still works.

`assignment/evidence/loop-after.txt` runs the same loop as section 1 with the new program.
Byte counts of `logs/` are shown before the send, two seconds after, and seven seconds after:

    baseline:    31,700,838
    +2s:         31,730,859   (+30 KB — the 8 hops)
    +7s:         31,730,859   (frozen — no growth)

The +30 KB step is the eight tunnel hops before the drop; the freeze at +7s is the loop
having ended. That matches the design: hop_limit starts at 8, is decremented once per
switch, and the ninth arrival at s1 finds 0 and calls `mark_to_drop`. Compare with
`loop-before.txt`, where the same loop grew to megabytes within seconds.

## 5. Evaluation

### a) The mixed rollout

Prediction (recorded in `assignment/evidence/prediction.txt` before running): s2 reading
only four bytes of the tunnel header should shift its parsing, most likely mangling the
delivery to h2, while the loop should still end because s1 decrements.

First run (`assignment/evidence/mixed.txt`):

- **Delivery**: tcpdump on h2 captured **0 packets**, and h2's `receive.py` closed its
  socket with "Network is down". Nothing arrived in a form Scapy could decode as tunnel +
  IP + UDP.
- **s2's parse of the same packet**: s2 logged `parse_myTunnel` followed by **"Bytes
  parsed: 18"**, not 19. It read the tunnel as four bytes, so the missing byte is
  `hop_limit`, and every subsequent field s2 read was shifted one byte earlier: the last
  byte of the tunnel header (`hop_limit = 8`) became the high byte of s2's IPv4
  `version/ihl`, so its IPv4 header is one byte earlier than it should be. `dst_id` is
  still correct because it is inside the first four bytes — that is why s2 forwarded it
  at all — but the IPv4 payload that follows is misaligned, and h2 gets garbage where its
  stack expects an IP header.
- **Loop**: `31810583 → 31866610 → 31866610`. It grew by 56 KB then froze. Because
  s1's program does have `myTunnel_forward` with the check, s1 decrements on every pass;
  after eight passes through s1 the field hits 0 and s1 drops it. s2 never matters to
  the loop bound because s2 never decrements — the packet bounces s1 → s2 → s1 and only
  s1 shortens the counter. My prediction matched for the loop; for delivery, I expected
  "mangled but maybe delivered" and it was "not delivered at all", which is the same
  mechanism with harsher wording — the checksum recompute alone was enough to make
  Scapy's socket give up before decoding.

This is exactly the failure the assignment's Step 6 hinted at: adding a byte to the
header changes the byte offsets after it, and a switch that hasn't been upgraded reads
the tail of the (new) header as the start of the (old) payload.

### b) A safe rollout for a mixed network

Two options, both used in practice:

1. **A new EtherType for the new header.** Run old switches on 0x1212 and new switches on
   a fresh EtherType, say 0x1213. Both programs are loaded at once but the two
   EtherTypes are matched by different parsers, so no switch ever sees a header it
   doesn't understand. Traffic is only sent with the new EtherType along paths where the
   whole path is upgraded; anything else uses the old header. This costs nothing extra
   on the wire per packet — the EtherType is already there — but it requires running two
   parallel programs on the upgraded switches, and traffic-engineering tells the
   headend which EtherType to use.

2. **Versioning and ordering: a `version` field plus a **trailing** hop-limit.** Put an
   explicit version byte at the *front* of the tunnel header, and append the hop-limit
   after all fields the old program already parses. An old switch parses the leading
   fields correctly, subtracts nothing, and — crucially — the field it does not know sits
   after the last field it does, so the offsets it uses are unchanged. New switches
   decrement it as before. The tunnel header grows on the wire (bytes are cheap), and
   nothing on an old switch breaks; the loader only upgrades switches after all paths
   that might carry that version are capable of it, or the sender can signal the version
   for a specific path.

Option 2 is the better one for a gradual rollout with **no** path-capability check needed
at the sender — the change is backward-compatible by construction. Option 1 is simpler to
reason about but requires path-tagging in the control plane.

### c) Data plane vs control plane loop protection

The **data plane** check (this hop limit) catches loops that exist in **state that is
already flowing** — a bad static route that gets traffic, a misconfigured table entry, a
transient loop while the control plane converges. It runs on every packet, so it takes
effect on the *next packet*, and the budget is enforced locally at the switch. It cannot
see that a loop **will** happen; it can only stop a loop that already carries traffic.

`check_paths()` in Week 3 is the **control plane** check — it follows a route table
before any packet exists, so it catches **misconfiguration in the data model** (a
route that points at a nonexistent next-hop, a loop that would have formed). It cannot
catch a loop introduced at runtime (a table entry changed live), and it can't bound
beyond whatever topology it was given — it computes the route chain, not an in-flight
packet. It also can't see forwarding that happens in a data-plane program (like this
one) because it has no model of P4 pipeline logic.

Real networks use both because each is silent where the other is strong: static analysis
catches the bad design before it ships, the data-plane limit saves you when the design
changes under traffic.

### d) Cost on a hardware switch

At line rate, three things cost:

- **Header size**: one byte per packet. On a 100 GbE port that's 0.008 Tb/s at line rate,
  trivially small, but it is a real cost per packet.
- **Pipeline stage**: the check `hop_limit == 0` and the decrement `hop_limit - 1` are
  cheap ALU operations in the match-action stage — a few gates on the modify stage that
  already exists for `ipv4_forward`. No new stage needed for this field alone, so no
  extra latency in that respect.
- **The bigger cost is the pipeline resources for the header field itself** — storing it
  in the PHV (packet header vector) and tracking it through the pipeline. Commercial chips
  have a fixed number of PHV bits per stage; a one-byte field that the pipeline otherwise
  would not need consumes one of them, and that PHV bit cannot be used for another field
  on the same stage. In a P4 program with lots of headers this can tip a pipeline from
  fitting to not fitting.

So: essentially free in bandwidth, cheap in logic, non-trivial in PHV budget on a real
chip. That is what "silicon constrains features" means — the language is expressive, the
chip's finite field vector is not.

## AI use

Level 3 (Open)


## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with the lab's finished program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with hop_limit at 6 on arrival, plus a plain ping |
| `evidence/loop-after.txt` | The same loop with the hop limit: byte counts stop growing |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old — delivery fails, loop still ends |
| `evidence/prediction.txt` | The prediction, written before running the mixed experiment |

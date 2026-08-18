*This project has been created as part of the 42 curriculum by ka-tan.*

# 🔌 NetPractice

> _One packet, one hop, one subnet at a time_ ✨

NetPractice is a hands-on introduction to TCP/IP networking, worked entirely through a browser-based training interface rather than code.
The goal of the project is to understand **how addressing actually decides whether a network works**: assigning valid IP addresses, choosing the right subnet mask, wiring up routers, and setting the correct default gateway so that every host in a small simulated network can actually reach the others.

It is basically a deep dive into **binary, address boundaries, and routing logic** — all the arithmetic a working network hides behind a green checkmark 🟢

---

## 📖 Description

Each level in the training interface presents a broken network diagram: a handful of hosts, switches, and routers, with a stated goal such as "host A must communicate with host C." Some fields are pre-filled and locked; the rest — IP addresses, masks, gateways, routing table entries — are editable and, as given, wrong.

The task is to work out, purely from the diagram and the mask arithmetic, what those fields need to be so that a packet from any required source can actually reach its destination and get a reply back. There are 10 levels total, each exported as a separate configuration file once solved.

---

## 🛠️ Instructions

Launch the training interface
```bash
./run.sh
```
This starts a local web server and opens the interface in your browser. If it doesn't open automatically:
```bash
python3 -m http.server 49242
```
then navigate to `http://localhost:49242` (or whatever port you chose).

### Exporting a configuration

Once every field on a level is filled in and the interface confirms all hosts can reach each other, use the **Export** button in the training interface to download that level's configuration as a JSON file. Rename it to match the level number (`levelN.json`) and move it to the repository root.

### Submission requirements

- All 10 levels must be solved and exported.
- The 10 exported configuration files — `level1.json` through `level10.json`, one per level — must be placed at the root of this repository.
- Each file must reflect a configuration that the training interface has validated as correct (all required hosts able to communicate).

## 🧠 Concepts I worked through

### 🔢 What an IP address actually is

A 32-bit number that identifies a device on a network, written as four decimal numbers 0–255 separated by dots — e.g. `1.1.1.10`. Each of those numbers is one byte (8 bits), so the whole thing is really `00000001.00000001.00000001.00001010` underneath. The dotted-decimal form is just for humans; the network only ever deals in bits.

### 🎭 What a subnet mask does

A second 32-bit number, same dotted format, that tells you which bits of the IP address are the "network" part and which are the "host" part. Wherever the mask has a `1` bit, that bit of the address is fixed and identifies the network; wherever it has a `0`, that bit is free and identifies the specific device within it.

`255.255.255.192` in binary is `11111111.11111111.11111111.11000000` — 26 ones, hence `/26`. The first 26 bits of any address using this mask are locked as the network; the remaining 6 bits vary across the devices in it, giving 2⁶ = 64 possible values.

### ➗ The bitwise AND — how the mask is actually applied

"Applying the mask" is a single bitwise operation. Compare the address and the mask bit by bit; the output bit is `1` only when **both** inputs are `1`.

| a | b | a AND b |
|---|---|---|
| 1 | 1 | 1 |
| 1 | 0 | 0 |
| 0 | 1 | 0 |
| 0 | 0 | 0 |

Because a mask is always a run of `1`s followed by a run of `0`s, ANDing copies every bit sitting under a `1` and flattens every bit under a `0` to zero. The result is the **network address**.

Worked through with `139.249.183.201 /28`:

```
IP         10001011.11111001.10110111.11001001    139.249.183.201
mask       11111111.11111111.11111111.11110000    255.255.255.240
           --------------------------------------- AND
network    10001011.11111001.10110111.11000000    139.249.183.192
                                        \__/
                                     host bits, forced to 0
```

Three things follow from this that caught me out at least once each:

- **The host bits are discarded, so many addresses collapse to the same network.** `.193`, `.201` and `.206` all AND down to `139.249.183.192` under a `/28`. That is precisely why they're neighbours.
- **The same address gives a different network under a different mask.** `.201` under `/26` gives `139.249.183.192`; under `/25` it gives `139.249.183.128`. An IP with no mask beside it is meaningless.
- **Route destinations get ANDed too.** Typing `139.249.183.1/27` into a routing table does not create a route about `.1` — it ANDs to `139.249.183.0/27` and silently becomes a route covering `.0`–`.31`.

### 🧩 What a subnet is

The group of addresses that share the same network bits — the set of devices the mask says are "local" to each other. Take an address, apply the mask, and everything left of the boundary is the subnet's identity; everything to the right is up for grabs by individual hosts. Two interfaces only see each other as neighbours if they land in the same subnet *and* agree on the mask — a mismatched mask on a shared wire is the single most common way a level breaks.

### 📏 The mask table

CIDR (`/N`) is just a count of how many of the 32 bits are locked as network bits — `255.255.255.192` and `/26` say the identical thing, one in dotted-decimal, one in shorthand.

| CIDR | Mask | Addresses per subnet | Usable hosts | Subnets per /24 |
|---|---|---|---|---|
| /24 | 255.255.255.0 | 256 | 254 | 1 |
| /25 | 255.255.255.128 | 128 | 126 | 2 |
| /26 | 255.255.255.192 | 64 | 62 | 4 |
| /27 | 255.255.255.224 | 32 | 30 | 8 |
| /28 | 255.255.255.240 | 16 | 14 | 16 |
| /29 | 255.255.255.248 | 8 | 6 | 32 |
| /30 | 255.255.255.252 | 4 | 2 | 64 |
| /31 | 255.255.255.254 | 2 | 0 | 128 |
| /32 | 255.255.255.255 | 1 | 0 | 256 |

The "addresses" and "subnets per /24" columns always multiply to 256 — a quick self-check. `/30` shows up constantly as a router-to-router link, since it gives exactly 2 usable addresses, one per end, and it is the practical floor: `/31` and `/32` have nothing left after the two reserved addresses, so NetPractice won't accept them for a link.

### 🏠 Network address, broadcast address, and counting usable hosts

Every block has two addresses that can never be assigned to an interface, and both are defined by what the **host bits** are doing:

- **Network address** — all host bits `0`. This is the identity of the subnet itself, the thing that appears in a routing table.
- **Broadcast address** — all host bits `1`. Anything sent here goes to every device in the subnet.

Same example as above, `139.249.183.201 /28`:

```
mask        11111111.11111111.11111111.11110000    255.255.255.240
network     10001011.11111001.10110111.11000000    139.249.183.192   ← host bits all 0
first host  10001011.11111001.10110111.11000001    139.249.183.193
last host   10001011.11111001.10110111.11001110    139.249.183.206
broadcast   10001011.11111001.10110111.11001111    139.249.183.207   ← host bits all 1
```

So the counting rule is:

```
total addresses  = 2^h          where h = 32 - prefix (the number of host bits)
usable hosts     = 2^h - 2      one lost to the network address, one to broadcast
```

For a `/28`, h = 4, so 2⁴ = 16 addresses and 14 usable. For a `/30`, h = 2, so 4 addresses and 2 usable — exactly enough for the two ends of a point-to-point router link and nothing more.

The fast decimal shortcut, without touching binary: **block size = 256 − (last non-255 octet of the mask)**. For `/28` that's 256 − 240 = 16, so blocks start at `.0`, `.16`, `.32`, `.48` … and `.201` falls into the one beginning at `.192`. Its broadcast is the address just before the next block starts, i.e. `.207`.

Assigning `.192` or `.207` to a host is one of the quieter ways to fail a level, because the address *looks* fine in the diagram.

### 🏷️ MAC addresses and LANs

A **LAN** (local area network) is the set of devices that can hand each other a frame **without a router in between**. Switches and cables extend a LAN; routers are what separate one LAN from another. One LAN = one subnet, and it shows up on a router as one directly-connected route.

A **MAC address** is a 48-bit identifier belonging to a single network interface, usually burned into the hardware and written in hex like `a4:83:e7:1f:2c:0b`. It is flat — no hierarchy, no mask, no arithmetic you can do on it — which is exactly why it can't be used to cross a LAN boundary. There is no way to summarise a group of MAC addresses the way you can summarise a group of IPs.

The two work together like this: **the destination IP is the final target, the destination MAC is only the next hop.** Watching a packet cross three LANs from H4 to H1 in level 10:

| Hop | Layer 2 (rewritten each hop) | Layer 3 (never rewritten) |
|---|---|---|
| H4 → R2 | src H41, dst R23 | src `139.249.183.131`, dst `139.249.183.2` |
| R2 → R1 | src R21, dst R13 | src `139.249.183.131`, dst `139.249.183.2` |
| R1 → H1 | src R11, dst H11 | src `139.249.183.131`, dst `139.249.183.2` |

**ARP** is the bridge between the two layers. When a host knows an IP on its own LAN but not the corresponding MAC, it broadcasts "who has this address?" and the owner answers. That mapping is what turns a layer-3 routing decision into an actual layer-2 frame.

This is also the real reason a **default gateway must sit inside the sender's own subnet**: the host can only physically hand a frame to something on its own LAN, and it addresses that handoff by MAC. If the gateway IP isn't local, there is nobody to ARP for, no MAC to write into the frame, and nothing to hand the packet to. The arithmetic check (`gateway AND mask == host AND mask`) is the layer-3 shorthand for a layer-2 physical constraint.

### 🔀 Switches vs. routers, and where "layer 2" comes from

The OSI model splits networking into layers; two of them explain the whole project.

A **switch** operates at layer 2 (data link). It forwards by MAC address, has no IP of its own, and makes no decisions about subnets — so it can only ever connect devices that already agree on one. It extends a LAN; it never joins two.

A **router** operates at layer 3 (network). It reads IP addresses, consults a routing table, and decides which of its own interfaces to forward a packet out of. It has one interface per LAN it joins, each with its own address valid inside that LAN, which is exactly what lets it bridge two different subnets.

| | Switch | Router |
|---|---|---|
| OSI layer | 2 (data link) | 3 (network) |
| Forwards by | MAC address | IP address |
| Has its own IP? | No | Yes, one per interface |
| Effect on a LAN | extends it | separates it |
| Unknown destination | floods every port | consults the routing table |

Hence, everything hanging off a switch must share one subnet and one mask, while every interface on a router belongs to a *different* subnet.

### 🔌 What an interface is

One network attachment point on a device — a port plus the IP and mask configured on it. A host normally has one. A router has several, one per subnet it joins, each carrying a different address that must be valid inside its own subnet.

### 🧭 Default gateway

Where a host sends a packet when the destination isn't inside its own subnet. It must be an address that is itself inside the sender's subnet — usually the router interface facing that host. Get this wrong and the packet never leaves the host's own segment.

### 🗺️ Routing table

The rulebook a router uses to decide where a packet goes next: each entry pairs a destination network (address + mask) with a next hop.

The order of operations matters, and the logs spell it out:

1. **Directly connected subnets are checked first.** Every interface gives the router an automatic route to its own subnet, and these are consulted before the table. The line `destination does not match any interface. pass through routing table` marks the moment step 1 fails and step 2 begins.
2. **Then the table, by longest prefix match.** If several entries match, the most specific one wins — `/27` beats `/25`, regardless of the order they're listed in.
3. **Then the default route.** `0.0.0.0/0` matches everything, so it catches whatever is left.
4. **Otherwise the packet is dropped** — `destination does not match any route`.

Anything reachable only through another router needs an explicit static entry: destination network, next hop = the neighbouring router's facing interface. Without it, packets for that subnet fall through the default route in the wrong direction or get dropped.

**Route summarisation.** Two adjacent, correctly aligned blocks merge into their parent by shortening the prefix one bit. `107.135.0.0/18` and `107.135.64.0/18` both AND to `107.135.0.0` under a `/17`, so one `/17` entry covers both — useful when a router only has one free slot. Alignment is the condition: `107.135.64.0/18` and `107.135.128.0/18` would *not* merge this way, since they only meet at a `/16` boundary.

### 🌍 Why 4 octets isn't enough — private addresses, NAT, and IPv6

32 bits gives 2³² ≈ 4.29 billion addresses, which is fewer than there are devices. IPv4 genuinely ran out: IANA handed the last unallocated blocks to the regional registries in February 2011, and the registries drained over the following years. Three things keep it working.

**Private addressing.** The RFC 1918 ranges — `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` — are reused by every network on earth simultaneously. Millions of machines answer to `192.168.1.10` right now. They never collide because those addresses are never routed on the public internet: uniqueness is *scoped*, not absolute.

**NAT** is what makes that survivable. A router rewrites the private source address to its own public one on the way out, and remembers enough state to reverse it on the way back, so an entire household or office shares a single globally unique address. Carrier-grade NAT pushes this further, putting thousands of subscribers behind one public IP.

**CIDR** stopped the waste at the allocation level. The original scheme was classful — fixed `/8`, `/16` and `/24` boundaries — so an organisation needing 300 addresses received a class B with 65,534 and burned the rest. Variable-length prefixes let allocations match actual need, which is the same mechanism used all through this project: `/25`, `/26`, `/28`, `/30`, each sized to its link.

**IPv6** is the actual fix: 128 bits instead of 32, so 2¹²⁸ ≈ 3.4 × 10³⁸ addresses. Adoption has been slow but is no longer marginal — Google's measurements showed native IPv6 access crossing 50% of its users for the first time in March 2026, with mobile carriers well ahead of fixed broadband.

**Most importantly, an address only has to be unambiguous within the region where it's routed.** A duplicate address isn't a problem because the number is rare, it's a problem when both copies live inside the same routing domain and the router has no way to tell them apart.

---

## 🔍 How I debugged a broken level

A fixed order of checks, cheapest first:

1. **Masks agree** on every device sharing a wire or a switch.
2. **Network addresses match** — AND each interface's IP with its own mask and confirm the neighbours land on the same result.
3. **Gateways are reachable** — `gateway AND mask == host AND mask`.
4. **Addresses are usable** — nobody sitting on the network or broadcast address, no duplicates anywhere in the same routing domain.
5. **Routing tables are valid** — a route exists for every subnet not directly connected, each destination ANDs to what you intended, and no route overlaps a subnet the router already owns.

---

## 📚 Resources

This project is a study of core TCP/IP networking concepts: **TCP/IP addressing**, **subnet masks** and CIDR notation, **default gateways**, **routers and switches**, and the **OSI model layers** (particularly layer 2 data link and layer 3 network) — alongside MAC addresses, ARP, routing tables, and IPv4/IPv6 addressing, all detailed in [🧠 Concepts I worked through](#-concepts-i-worked-through) above.

References used to understand the project topic:
- From Zero to Network Hero: A Practical Guide to NetPractice 1337_Rabat:
https://medium.com/@mohamedamintarza/from-zero-to-network-hero-a-practical-guide-to-netpractice-1337-rabat-a2ffb614a928
- TCP/IP addressing and the structure of an IPv4 address
- Binary representation and the bitwise AND operation used to derive a network address
- Subnet masks and CIDR notation
- Network and broadcast addresses, and calculating usable hosts per subnet
- Default gateways and how a host decides local vs. remote delivery
- Routing tables, longest prefix match, and route summarisation
- Routers, switches, and the layer 2 / layer 3 distinction
- MAC addresses, ARP, and the scope of a LAN
- OSI model layers, in particular the data link and network layers
- RFC 1918 private address ranges, NAT, and IPv4 exhaustion
- IPv6 and the 128-bit address space

AI was used for concept clarification (subnetting arithmetic, mask/CIDR conversion, bitwise AND, routing table behaviour, the MAC/IP layer split) and for building visual explanations of address/mask splitting, prefix coverage, and topology examples while working through the levels. All configuration values submitted were worked out and verified manually against the training interface's logs.

---

## 🌱 Final thought

It's pretty cool to learn that an IP address on its own means nothing and it's only ever meaningful next to its mask. The same two numbers can be neighbours or strangers depending entirely on where that boundary sits, and every failing level eventually traced back to that one relationship being wrong somewhere in the diagram.

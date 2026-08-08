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

## 🧠 Concepts I worked through

### 🔢 What an IP address actually is

A 32-bit number that identifies a device on a network, written as four decimal numbers 0–255 separated by dots — e.g. `1.1.1.10`. Each of those numbers is one byte (8 bits), so the whole thing is really `00000001.00000001.00000001.00001010` underneath. The dotted-decimal form is just for humans; the network only ever deals in bits.

### 🎭 What a subnet mask does

A second 32-bit number, same dotted format, that tells you which bits of the IP address are the "network" part and which are the "host" part. Wherever the mask has a `1` bit, that bit of the address is fixed and identifies the network; wherever it has a `0`, that bit is free and identifies the specific device within it.

`255.255.255.192` in binary is `11111111.11111111.11111111.11000000` — 26 ones, hence `/26`. The first 26 bits of any address using this mask are locked as the network; the remaining 6 bits vary across the devices in it, giving 2⁶ = 64 possible values.

### 🧩 What a subnet is

The group of addresses that share the same network bits — the set of devices the mask says are "local" to each other. Take an address, apply the mask, and everything left of the boundary is the subnet's identity; everything to the right is up for grabs by individual hosts. Two interfaces only see each other as neighbours if they land in the same subnet *and* agree on the mask — a mismatched mask on a shared wire is the single most common way a level breaks.

### 📏 The mask table

CIDR (`/N`) is just a count of how many of the 32 bits are locked as network bits — `255.255.255.192` and `/26` say the identical thing, one in dotted-decimal, one in shorthand.

| CIDR | Mask | Addresses per subnet | Subnets per /24 |
|---|---|---|---|
| /24 | 255.255.255.0 | 256 | 1 |
| /25 | 255.255.255.128 | 128 | 2 |
| /26 | 255.255.255.192 | 64 | 4 |
| /27 | 255.255.255.224 | 32 | 8 |
| /28 | 255.255.255.240 | 16 | 16 |
| /29 | 255.255.255.248 | 8 | 32 |
| /30 | 255.255.255.252 | 4 | 64 |
| /31 | 255.255.255.254 | 2 | 128 |
| /32 | 255.255.255.255 | 1 | 256 |

The two right columns always multiply to 256 — a quick self-check. `/30` shows up constantly as a router-to-router link, since it gives exactly 2 usable addresses, one per end.

Of every block, the **first address is the network identifier and the last is the broadcast address** — neither can ever be assigned to an interface. A /26 block of 64 gives only 62 usable addresses.

### 🔀 Switches vs. routers, and where "layer 2" comes from

The OSI model splits networking into layers; two of them explain the whole project. A **switch** operates at layer 2 (data link) — it forwards by MAC address and has no IP of its own, so it can only ever connect devices that already agree on one subnet. A **router** operates at layer 3 (network) — it reads IP addresses, consults a routing table, and decides which of its own interfaces to forward a packet out of, which is exactly what lets it bridge two different subnets.

### 🔌 What an interface is

One network attachment point on a device — a port plus the IP and mask configured on it. A host normally has one. A router has several, one per subnet it joins, each carrying a different address that must be valid inside its own subnet.

### 🧭 Default gateway

Where a host sends a packet when the destination isn't inside its own subnet. It must be an address that is itself inside the sender's subnet — usually the router interface facing that host. Get this wrong and the packet never leaves the host's own segment.

### 🗺️ Routing table

The rulebook a router uses to decide where a packet goes next: each entry pairs a destination network (address + mask) with a next hop. On forwarding, the router checks the destination against every entry and uses the **longest prefix match** — the most specific entry wins if more than one matches. If nothing specific matches, the catch-all `0.0.0.0/0` default route catches it; if nothing matches at all, the packet is dropped, which is exactly what the log means by *"destination does not match any route."*

A router's table always has one automatic entry per subnet it's directly plugged into. Anything reachable only through another router needs an explicit static entry — destination network, next hop = the neighbouring router's facing interface — or packets for that subnet fall through the default route in the wrong direction, or get dropped entirely.

---

## 📚 Resources

References used to understand the project topic:
- From Zero to Network Hero: A Practical Guide to NetPractice 1337_Rabat:
https://medium.com/@mohamedamintarza/from-zero-to-network-hero-a-practical-guide-to-netpractice-1337-rabat-a2ffb614a928
- TCP/IP addressing and the structure of an IPv4 address
- Subnet masks and CIDR notation
- Default gateways and how a host decides local vs. remote delivery
- Routers, switches, and the layer 2 / layer 3 distinction
- OSI model layers, in particular the data link and network layers

AI was used for concept clarification (subnetting arithmetic, mask/CIDR conversion, routing table behaviour) and for building visual explanations of address/mask splitting and topology examples while working through the levels. All configuration values submitted were worked out and verified manually against the training interface's logs.

---

## 🌱 Final thought

Its pretty cool to learn that an IP address on its own means nothing and it's only ever meaningful next to its mask. The same two numbers can be neighbours or strangers depending entirely on where that boundary sits, and every failing level eventually traced back to that one relationship being wrong somewhere in the diagram.

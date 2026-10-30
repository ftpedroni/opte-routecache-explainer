# OPTE RouteCache — Interactive Architecture Walkthrough

An interactive, step-by-step visualization of a proposed optimization to
[Oxide Computer's OPTE](https://github.com/oxidecomputer/opte) XDE underlay
route cache.

**[Launch the interactive walkthrough](https://ftpedroni.github.io/opte-routecache-explainer/)**

Related to [OPTE issue #539](https://github.com/oxidecomputer/opte/issues/539):
*Recast route caching as an LPM, make multipath decisions locally.*

## The Problem

The original XDE RouteCache is keyed by:

    (underlay IPv6 destination, L4 flow hash)

Each cached entry holds a single resolved forwarding decision.

Consequently, different flows to the same destination require separate
cache entries, even when they share the same underlying routing information.

The original cache has a limit of 8,192 entries. Once full, new flow keys
cannot be retained, increasing repeated route lookups and neighbor resolution.

## The Proposed Design

Instead of caching one selected next hop per flow, the new design shares
resolved next-hop sets across destinations belonging to an underlay prefix.

    Original:
      (destination, flow hash) -> resolved next hop

    Proposed:
      destination -> verified prefix
      prefix      -> resolved next-hop set
      flow hash   -> local next-hop selection

The forwarding decision is still made for each packet, but the resolved
routing information can be reused across many flows.

### Key Properties

- Shared next-hop sets, indexed by underlay prefix.
- Local ECMP selection following illumos bucket-selection semantics.
- Next-hop identity includes both gateway IPv6 address and interface.
- Fallback to illumos when local selection is not safe or permitted.
- Independent 100 ms validity checks for destination-to-prefix mappings
  and cached next-hop sets.
- Generation-based protection against stale concurrent updates.
- Bounded memory: up to 1,024 prefix sets and 65,536 exact destinations.
- No per-CPU TLB or CPU-count-dependent cache allocation.

The implementation is not a general-purpose longest-prefix-match table.
The illumos routing table remains authoritative.

## Explore the Walkthrough

The interactive visualization demonstrates:

1. How the original per-flow cache works.
2. Why flow churn creates repeated route-cache misses.
3. How resolved next hops are shared by prefix.
4. How ECMP decisions are made locally.
5. When forwarding falls back to illumos.
6. How cache expiration and revalidation work.
7. How a newly installed more-specific route is discovered.
8. The measured performance differences between the implementations.

Use **Next**, **Previous**, **Play**, and **Restart** to navigate the
explanation.

The page runs entirely in the browser without a backend.

## Experimental Results

Controlled Helios laboratory benchmarks compared the original cache,
the prefix cache with a per-CPU TLB, and the prefix cache without the TLB.

Representative throughput results for the version without TLB:

| Workload | Original | Prefix cache | Change |
|---|---:|---:|---:|
| 256 flows | 148,871 pkt/s | 143,856 pkt/s | -3.4% |
| 8,000 flows | 128,229 pkt/s | 161,039 pkt/s | +25.6% |
| 25,000 one-shot flows | 77,615 pkt/s | 108,326 pkt/s | +39.6% |

These are laboratory send-path measurements, not physical NIC throughput.
The final destination-validity correction was validated separately; the
benchmark results should not be represented as measurements of that final
binary without confirming artifact identity.

The design also eliminates the fixed per-CPU TLB allocation, which would
otherwise require approximately 1.5 MiB per port on a 128-thread system.

## Validation

The implementation was evaluated with:

- Route-selection and cache-policy unit tests.
- Comparison against illumos ECMP forwarding decisions.
- XDE release builds on Helios.
- Live forwarding tests.
- A regression scenario involving a newly installed more-specific route,
  confirming that refreshing one destination does not indefinitely extend
  the validity of another destination's prefix association.

## Scope

This repository contains an independent educational visualization.

It is not the OPTE implementation, a simulator of the complete illumos
network stack, or an official Oxide Computer project.

The source-code contribution belongs in the OPTE repository.

## Author

Felipe Thomaz Pedroni

[GitHub](https://github.com/ftpedroni)

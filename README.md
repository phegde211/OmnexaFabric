# OmnexaFabric — Zero Trust Fabric for Agentic Mesh

**OmnexaFabric lets AI agents find the tools, data, and other agents they need — and
enforces per-agent policy *before a single request leaves the source*.**

As agent meshes grow, two problems dominate. **Discovery:** an agent must locate the right
tool, dataset, or peer agent — by an application-defined identity, in a space far too large
and too dynamic to preload everywhere. **Trust:** every agent-to-agent and agent-to-tool call
should be authorized by identity-aware policy, not implicitly trusted just because it reached
the network. OmnexaFabric answers both with one mechanism.

## The idea, in brief

OmnexaFabric is a **pull-based secure service mesh**. Rather than push a global map and policy
table to every node, each source resolves only what it needs, on demand:

1. **Resolve by identifier.** An agent asks the fabric to resolve an application-defined
   identifier — a UID: a tool name, a dataset key, an agent ID — to *where* it lives, plus the
   destination's **identity metadata**. (Not merely IP→IP; **UID→destination**.)
2. **Decide at the source.** Knowing both identities, the source downloads the policy for that
   **(source, destination) pair** and **permits, denies, or governs the call before any
   application traffic is sent** — Zero Trust, enforced at the origin. A denied call never
   leaves; it is not shipped to a remote firewall merely to be dropped there.
3. **Govern the path.** Policy can insert services (inspection, firewalling), steer the path,
   and attach monitoring, audit and tracing — per agent, per call.

Valid results are cached under expiry/revocation rules, so the first call pays a small
resolution cost and the rest are fast — with far less per-node state than a push-the-world
design.

### Why this fits an agentic mesh

| Agent-mesh need | OmnexaFabric mechanism |
|---|---|
| Find a tool / dataset / peer agent by name | on-demand UID → destination + identity resolution |
| Zero Trust per-agent authorization | source downloads the (src, dst) policy and enforces it before sending |
| Least privilege; deny early | permit/deny at the origin — denied traffic never leaves |
| Inspection / audit of agent calls | policy-guided service insertion, logging, tracing |
| Scale to a large, churning agent population | pull + cache, not push-everything |

The agentic mesh is the headline application; the **same fabric** serves any identity-addressed,
policy-governed communication.

## Read more

- **Core networking underlay & architecture** — the resolution model (pull vs push), candidate
  protocols (**NHRP / LISP**), the map server, metadata carriage, and source-side policy
  enforcement:
  **[OmnexaFabric_Proposal.txt](OmnexaFabric_Proposal.txt)**
- **Worked use cases** — a simulated distributed Redis data store and a marketplace user journey
  that exercise on-demand resolution plus source-side policy:
  **[OmnexaFabric_Use_Cases.txt](OmnexaFabric_Use_Cases.txt)**

---

*This is an architectural proposal; finer policy, transport, and failure-handling remain design
work. © 2026 Prabhat Hegde. All rights reserved.*

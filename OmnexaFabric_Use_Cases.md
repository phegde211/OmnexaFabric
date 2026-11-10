# OmnexaFabric - Supporting Use Cases

## Copyright and Permissions

Copyright (c) 2026 Prabhat Hegde. All rights reserved.

OmnexaFabric: Architectural Proposal and Supporting Use Cases

Copyright is claimed in the original text, diagrams, and presentation
of this proposal.

Except as permitted by applicable law, no protected portion of this
document may be reproduced, republished, distributed, translated, or
adapted without the prior written permission of the copyright owner.

Third-party materials, product names, trademarks, and referenced
standards remain subject to their respective owners' rights.
Their inclusion does not imply affiliation or endorsement.

This notice protects the original expression of the proposal. It does
not claim exclusive ownership of underlying ideas, technical concepts,
methods, or existing protocols and standards.

Companion to [OmnexaFabric_architecture.md](OmnexaFabric_architecture.md).

These examples support the main resolution and policy-download proposal.

# 4. Use Cases

## A) Simulated Distributed Redis Data Store

Other data stores may handle distribution internally; parts of this problem may
already be solved. The intent here is to show how a service mesh using
resolution protocols can also support information retrieval through a simple
extension around existing local redis data stores.

### 4.1 Problem

a) **Memory and scalability.**
   Consider multiple POPs, each supporting up to 500,000 users, with
   approximately 10 KB of session state per user.

   ```
   500,000 users x 10 KB ~= 5 GB of data per POP
   ```

   This excludes Redis overhead. Replicating complete values across
   interacting POPs increases memory requirements as the system grows.

b) **User experience and roaming.**
   Sessions reside at the original POP. A roaming user should not need
   re-authentication solely because the serving POP changed (for example, in
   ZTNA). Roaming activities can also modify the session/profile, requiring
   updates to its owner.

c) **Marketplace example.**
   A user registered through a US-east POP can later connect through a
   US-west POP. Dashboard activity may require information that remains at
   the original location.

### 4.2 Solution

Each POP keeps the complete records it owns in its local Redis.

For a record owned elsewhere, it can keep:

a) A reduced mapping:

   ```
   UID -> owner POP interceptor IP
   ```

b) No local entry until resolution is needed.

Reduced mappings save the value portion even if key counts remain similar.
Alternatively, resolve remotely without individual non-owner entries.
Distribution/storage of reduced mappings is out of scope; MP-BGP or another
mechanism could distribute them.

### 4.3 Design

The solution has two parts:

1. Identify the location from which information can be fetched.
2. Establish the permitted path and fetch the information.

Existing local access:

```
Microservice --------------------------> Local Redis
```

Proposed local or remote access:

```
+---------------------- Current POP -----------------------+
|                                                          |
| Microservice ---> Interceptor / omnexaAgent ---> Redis   |
|                            |                    (local)  |
+----------------------------+-----------------------------+
                             |
                             | WAN / reachable underlay
                             |
           +-----------------+------------------+
           |                                    |
           | Register / resolve                 | Approved connection
           v                                    v
+----------------------+       +------------------------------+
|      Map Server      |       | Owner POP                    |
|                      |       |                              |
|                      |       | Remote interceptor           |
| PopID -> owner IP    |       |          |                   |
+----------------------+       |          v                   |
                               | Owner's local Redis          |
                               | Complete UID -> value        |
                               +------------------------------+
```

The query and response use the owner connection. They do not need to pass
through the central resolution server.

#### 4.3.1 Keyspace

Usernames are assumed to be unique.

A proposed 64-bit UID is:

```
+--------------------+-------------------------------------+
| Owner PopID        | User ID                             |
| Upper k bits       | Remaining 64-k bits                 |
+--------------------+-------------------------------------+

UID = { PopID | user-id }
```

Notes:

a) Unique-key generation is implementation-specific and out of scope. All
   keys belonging to a given owner POP share its PopID in the upper k bits.

b) Each POP registers its PopID with the map server using the selected
   NHRP/LISP registration mechanism or extension.
   Mappings on map server:

   ```
   PopID -> owner POP's reachable interceptor IP
   ```

   One registration can therefore represent many user keys.

#### 4.3.2 Interceptor module

A transparent network proxy, deployed as a separate container or sidecar with
resolution and proxy functions.
Note: Working demo for a proxy/interceptor available in GitHub.

Rationale:

a) Consumers keep their existing Redis requests and need not know whether
   access is local or remote.

b) The change remains outside existing application modules, avoiding code
   changes and their associated regression scope. Isolate or bypass the
   interceptor if necessary (-wherein keys with local Redis onwership returns
   full values while those with remote ownership fails. But this would cover
   most requests).

#### 4.3.3 Why BGP or other service mesh protocols may not fit here

Pushing compact keys/source mappings may be acceptable; pushing complete values
can become expensive. One approach pushes mappings, then pulls values through an
application protocol. This proposal prefers resolving the source on demand and
forwarding the existing Redis request.

The concern is mapping/payload distribution; it does not require BGP or an
existing service mesh to carry every Redis value.

#### 4.3.2.1 Work of the interceptor

a) Capture the Redis request inline.

   - a.1) Identify the owner PopID from the key's upper k bits.
     If it is the local PopID, forward the original RESP command to local
     Redis.

b) Resolve the Remote owner Pop for the key.

   - b.1) If it is a remote PopID, hold the request and resolve the remote
     owner's interceptor through the Map server. The owner has
     registered its PopID-to-IP mapping.
   - b.2) Obtain and enforce the applicable source-destination policy before
     admitting the connection. Once permitted, forward the original Redis
     request to the remote interceptor.
   - b.3) The remote interceptor forwards the same to its own local Redis,
     which holds the authoritative session or metadata. Note, this
     preserves the RESP requests and the redis server would treat it
     as it it originated locally.
   - b.4) The reply returns over the reverse path. The serving POP may cache
     the result under the chosen freshness/expiry rules.

c) Handling key updates:
   Updates to non-owner key cannot be done directly. Non-owner Pop needs to
   send a redis SET to owner Pop/Owner interceptor which would perform the
   authoritative update.

d) Intended update distribution:
   The proposal expects the owner's Redis/keyspace subscription mechanism to
   support updates to the other copies, provided those recipients have
   subscribed for the relevant keyspace. This is a more involved process
   with couple of solutions - out of scope for this doc).

## B) Marketplace User Journey

This is an illustrative marketplace example. It does not describe or claim
knowledge of any one internal implementation.

Topology:

```
+----------------------+       +--------------------------+
| POP1                 |<----->| Central resolution       |
| Original user owner  |       | server                   |
|                      |       |                          |
| Application          |       | UID/source mappings, or  |
|     |                |       | delegated PopID mappings |
| Interceptor          |       +-------------+------------+
|     |                |                     |
| Local Redis          |                     |
| UID1 -> metadata     |                     v
+----------+-----------+       +--------------------------+
           ^                   | POP-n                    |
           |                   | Current serving location |
           |                   |                          |
           +------------------>| Interceptor              |
               Approved path   |     ^                    |
               Query / reply   | Application <--- User-1  |
                               +--------------------------+
```

1. **User-1 opens the marketplace.**
   DNS, geolocation/proximity, or load balancing directs the user to POP1.

2. **User-1 completes identity verification and creates an account.**
   POP1 becomes the initial owner of this user's session/metadata.

3. **POP1 creates the user's UID-to-metadata record.**
   Business logic determines the UID and record contents. The record is stored
   in cache/local Redis as:

   ```
   UID1 -> session / metadata
   ```

4. **POP1 makes the source mapping discoverable.**
   The direct form in this example is:

   ```
   UID1 -> IP1
   ```

   With use case A's structured keyspace, aggregate this as:

   ```
   PopID1 -> IP1
   ```

   Derive PopID1 from UID1 instead of registering each user centrally.

   Notes:
   - 4.1) NHRP/LISP extensions are proposed to carry the relevant application
     identifiers and metadata in registration and resolution exchanges.
     The exact extension needs design.
   - 4.2) A direct user lookup uses the UID. An aggregated lookup uses its
     owner PopID prefix to reach the same information source.
   - 4.3) The directory's data structure depends on the chosen lookup
     semantics: exact, longest-prefix, partial, or other supported
     matching. Hash tables and trees are implementation choices.

5. **Other users register through their respective POPs.**
   Those POPs publish their source mappings to the central service. The topology
   resembles a hub/spine with participating POPs as leaves.

   - 5.1) Summarising user mappings reduces central entries where the key
     allocation and ownership permit aggregation. The PopID prefix in use
     case A supplies one concrete approach.

6. **Resolve on demand and communicate with the source.**
   The serving interceptor resolves the record's source through the appropriate
   authority. Central services can be chained/hierarchical.

   Obtain the source-destination policy and establish the approved
   direct/shortcut path or required service path. Forward the query, return the
   result, and retain the connection while hot and valid. Remove it when idle.

   The same mechanism supports ordinary connectivity and payment, cart, bidding,
   or other information queries. Business modules choose the identifiers and
   requests; OmnexaFabric supplies the vessel.

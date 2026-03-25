# Glamsterdam Fork: Overview & Tracking

> **Fork**: Glamsterdam = Gloas (CL) + Amsterdam (EL)
> **Target**: H1 2026 (~June 2026)
> **Headliners**: [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732) (ePBS), [EIP-7928](https://eips.ethereum.org/EIPS/eip-7928) (BALs)
> **Fork Version**: [`GLOAS_FORK_VERSION = 0x07000000`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/fork.md#configuration)
> **Last Updated**: 2026-03-25
> **Detailed Plans**: [BALs](glamsterdam_bals.md) | [ePBS](glamsterdam_epbs.md)
> **Repos**: [xatu](https://github.com/ethpandaops/xatu) | [tysm](https://github.com/ethpandaops/tysm) (Prysm overlay)
> **Specs**: [consensus-specs/gloas](https://github.com/ethereum/consensus-specs/tree/master/specs/gloas) | [execution-specs/amsterdam](https://github.com/ethereum/execution-specs/tree/forks/amsterdam)

---

## How BALs and ePBS Interact

Under ePBS, the execution payload is delivered separately from the beacon block via `SignedExecutionPayloadEnvelope`. BALs live inside the execution payload. This means:

1. **BAL data arrives with the envelope, not the block.** Cannon derivers that extract BALs must source from the envelope, not the block body.
2. **If a builder withholds the payload, there is no BAL for that slot.** The block has a bid but no execution data.
3. **Execution requests** (deposits, withdrawals, consolidations) also move to the envelope — they are processed during [`process_execution_payload`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md#new-process_execution_payload), not `process_block`.
4. **Transactions and withdrawals** move to the envelope too — all cannon derivers that extract from the block body must be updated.

---

## TYSM: Our Fastest Path to ePBS Events

[TYSM](https://github.com/ethpandaops/tysm) is our patched overlay of Prysm that hooks into the CL client and forwards events to Xatu. It currently captures blocks, attestations, blob sidecars, data column sidecars, RPC events, and engine API telemetry.

**Key insight:** Upstream Prysm already has preliminary Glamsterdam/ePBS support (payload attestation events, pool, processing, Gloas metrics). TYSM's hook architecture is ready to extend — we just need to add new event types and injection points. **This means we can capture ePBS events directly from TYSM without waiting for beacon API SSE proposals to be accepted.**

### What TYSM already captures (relevant to Glamsterdam)

| Current Event | Hook | Notes |
|--------------|------|-------|
| `OnBlockReceive` | Gossip validation | Will see Gloas blocks (with bid, no payload) |
| `OnAttestationReceive` | Gossip validation | Will see index=0/1 payload votes |
| `OnDataColumnReceive` | Gossip validation | Will see structurally-changed Gloas sidecars |
| `OnEngineNewPayload` | Execution telemetry | Will see V5 payloads with BALs |

### What TYSM needs for Glamsterdam

| New Event | Hook Into | Priority |
|-----------|----------|----------|
| `OnPayloadAttestationReceive` | Prysm's PTC gossip validation | **P0** — 512 individual PTC attestations/slot |
| `OnPayloadAttestationProcessed` | Prysm's PTC pool insertion | P1 |
| `OnExecutionPayloadEnvelopeReceive` | Prysm's envelope gossip validation | **P0** — timing between block and payload is critical |
| `OnExecutionPayloadBidReceive` | Prysm's bid gossip validation | **P0** — all builder bids |
| `OnProposerPreferencesReceive` | Prysm's preferences gossip validation | P1 |

### TYSM implementation for each new event

Each requires 5 steps (following existing patterns like `OnBlockReceive`):

1. **Event type** in `/overlay/tysm/events/` — e.g., `payload_attestation.go`
2. **Interface** in `/overlay/tysm/interfaces/` — `PayloadAttestationObserveHooks`
3. **Dispatcher** in `/overlay/tysm/dispatcher/p2p_gossip.go` — `DispatchPayloadAttestationReceive()`
4. **Emit function** in `/overlay/tysm/inject.go` — `EmitPayloadAttestationReceive()`
5. **Xatu forwarder handler** in `/overlay/tysm/xatu/forwarder/handlers.go` — convert to Xatu `TraceEvent`
6. **Prysm patch** — add `tysm.EmitPayloadAttestationReceive(...)` call at the gossip validation point

### TYSM vs Beacon API SSE: Two complementary paths

| | TYSM (direct from CL) | Beacon API SSE |
|---|---|---|
| **Availability** | We control it — can ship now | Need client dev buy-in |
| **Coverage** | Only our Prysm nodes | Any CL client |
| **Data richness** | Full gossip metadata (peer ID, message ID, receive time, topic) | Summarized event data |
| **Latency** | Immediate — at gossip validation time | After node processing |
| **Best for** | P2P propagation analysis, timing, network health | Broader fleet monitoring via sentry |

**Recommendation:** Implement TYSM hooks first (we control the timeline), continue driving SSE proposals in parallel for broader coverage.

---

## Beacon API SSE Events — [ethereum/beacon-APIs](https://github.com/ethereum/beacon-APIs)

### Already In Spec (merged to master)

These Gloas SSE events are **already in the beacon-APIs spec** ([eventstream/index.yaml](https://github.com/ethereum/beacon-APIs/blob/master/apis/eventstream/index.yaml)). We just need to implement handlers for them.

| Event | Description | Xatu Event |
|-------|-------------|-----------|
| `execution_payload_available` | Node verified execution payload and blobs availability for payload attestation | `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD` |
| `execution_payload_bid` | Node received a `SignedExecutionPayloadBid` passing gossip validation | `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD_BID` |
| `payload_attestation_message` | Node received a `PayloadAttestationMessage` passing validation | `BEACON_API_ETH_V1_EVENTS_PAYLOAD_ATTESTATION` |

PR [#587](https://github.com/ethereum/beacon-APIs/pull/587) (merged) adds `version` wrapping to all Gloas events.

### In-Flight PRs (open)

| Event | PR | Author | Notes |
|-------|------|--------|-------|
| `execution_payload_gossip` | [#588](https://github.com/ethereum/beacon-APIs/pull/588) | nflaig | **Separate** from `execution_payload_available` — this is gossip receipt timing (for contributoor). |
| `proposer_preferences` SSE + pool API | [#593](https://github.com/ethereum/beacon-APIs/pull/593) | bharath-123 | |
| `head_v2` (deprecates `head`) | [#590](https://github.com/ethereum/beacon-APIs/pull/590) | chong-he | We need to handle this. |
| `chain_reorg` + execution block hashes | [#585](https://github.com/ethereum/beacon-APIs/pull/585) | bharath-123 | Payload reorgs possible under ePBS. |
| PTC duty endpoint (next epoch lookahead) | [#592](https://github.com/ethereum/beacon-APIs/pull/592) | nflaig | |

### Other Gloas beacon-APIs Issues to Track

| Issue/PR | Description |
|---------|-------------|
| [#576](https://github.com/ethereum/beacon-APIs/issues/576) | Update forkchoice endpoint for Gloas (PayloadStatus, PTC votes) |
| [#572](https://github.com/ethereum/beacon-APIs/issues/572) | Update state v2 API for Gloas |
| [#580](https://github.com/ethereum/beacon-APIs/pull/580) | Produce block v4 with payload + envelope endpoints |
| [#584](https://github.com/ethereum/beacon-APIs/pull/584) | Builder API to construct payload envelope |

---

## Decisions Log

| # | Question | Decision | Date |
|---|---------|----------|------|
| D1 | Capture all PTC messages or only aggregated? | **All individual PTC messages** — volume (~3.6M/day) is fine | 2026-03-25 |
| D2 | Capture all builder bids or only winning? | **All bids** | 2026-03-25 |
| D3 | Store raw RLP BAL bytes? | **No** — decoded/persisted form is sufficient | 2026-03-25 |
| D4 | Event proposal priority? | **Focus on execution_payload, payload_attestation, execution_payload_bid SSE events** | 2026-03-25 |
| D5 | Upstream branch merge blocking? | **Not a blocker for planning** — merge expected ~1 month | 2026-03-25 |
| D6 | DataColumnSidecar migration? | **No migration. Additive only.** New columns for Gloas, NULL for Fulu rows. | 2026-03-25 |
| D7 | Capture `SignedProposerPreferences`? | **Yes** | 2026-03-25 |
| D8 | BAL summary table? | **No** — not needed | 2026-03-25 |

---

## Open Questions

### Architecture

1. **Block-Payload Correlation**: How do we correlate beacon blocks with their separately-arriving execution payloads in ClickHouse? Same slot + block_root? Denormalize bid fields onto payload table? Dedicated slot timeline table?

2. **Empty Block Handling**: When a builder withholds the payload — NULL execution payload fields? Explicit `payload_status` column? Both?

3. **BAL Source under ePBS**: Does the beacon node reassemble a "complete" block once both pieces arrive (so cannon block-fetch still works), or must the cannon deriver separately fetch the envelope?

4. **Timing Data Model**: Should we have a dedicated "slot timeline" table tracking block arrival, payload arrival, PTC result?

### Upstream

5. ~~**Beacon API spec formalization**~~ **ANSWERED**: The new SSE events are **NOT** being added by others. **We need to drive this.**

6. **Client support timeline**: Which consensus clients are implementing ePBS first?

7. **Builder registry bootstrapping**: Do we need a one-time state snapshot at fork activation to capture the initial builder set?

### Data Model

8. **Builder vs Validator withdrawals**: Split into separate tables or add a `withdrawal_type` column?

---

## Implementation Roadmap

### Phase 1: Complete BALs (EIP-7928) — IN PROGRESS

See [glamsterdam_bals.md](glamsterdam_bals.md). Remaining: storage reads, sentry handlers, consumoor routes, engine API tracking, `block_access_list_hash` capture, ethcore eth/71.

### Phase 2: ePBS Proto & Types

See [glamsterdam_epbs.md](glamsterdam_epbs.md). Define all new protobuf types.

### Phase 3: ePBS ClickHouse & Pipeline

Design schemas, Vector pipeline routing, migration scripts for all new event types.

### Phase 4: TYSM ePBS Hooks (no external deps — we control this)

Add new event types and injection points to TYSM for all ePBS gossip events:
1. `OnPayloadAttestationReceive` — PTC attestations from gossip
2. `OnExecutionPayloadEnvelopeReceive` — payload envelope from gossip
3. `OnExecutionPayloadBidReceive` — builder bids from gossip
4. `OnProposerPreferencesReceive` — proposer preferences from gossip
5. Xatu forwarder handlers for each new event
6. Prysm patches at gossip validation points

### Phase 5: ePBS Sentry

Requires combined go-eth2-client branch. `OnExecutionPayload` subscription, updated `OnBlock` for Gloas, timing measurement.

### Phase 6: ePBS Cannon

Requires combined go-eth2-client branch. New derivers (PayloadAttestation, ExecutionPayloadBid) + modified derivers (transactions, withdrawals, BALs sourced from envelope).

### Phase 7: ePBS P2P / Mimicry

New gossipsub topic handlers for all four new topics (via CL mimicry, separate from TYSM).

### Phase 8: Drive Beacon API Event Proposals

Write formal proposals for `execution_payload`, `payload_attestation`, `execution_payload_bid` SSE events. Get buy-in from client teams. (TYSM gives us data in the meantime.)

---

## Upstream Dependencies

| Dependency | BALs | ePBS | Notes |
|-----------|------|------|-------|
| `go-eth2-client` | [PR#7](https://github.com/pk910/go-eth2-client/pull/7) (Open) | [PR#4](https://github.com/pk910/go-eth2-client/pull/4) (Open) | Separate branches — need combined |
| `go-ethereum` | v1.17.1 (Done) | TBD | |
| `ethcore` RLP/decoding | Done | TBD | |
| `ethcore` eth/71 P2P | **TODO** | N/A | `GetBlockAccessLists`/`BlockAccessLists` handlers |
| `tysm` (Prysm overlay) | N/A | **TODO** — hooks ready to extend | Upstream Prysm has preliminary Gloas support; need new emit points |

---

## Full Event Inventory

### New Events (10 total)

| # | Event | Source | Description |
|---|-------|--------|-------------|
| 1 | `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD` | Sentry SSE | Payload envelope arrival |
| 2 | `BEACON_API_ETH_V1_EVENTS_PAYLOAD_ATTESTATION` | Sentry SSE | Individual PTC attestation |
| 3 | `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD_BID` | Sentry SSE | Builder bid |
| 4 | `BEACON_API_ETH_V1_EVENTS_PROPOSER_PREFERENCES` | Sentry SSE | Proposer preferences |
| 5 | `BEACON_API_ETH_V2_BEACON_BLOCK_PAYLOAD_ATTESTATION` | Cannon | Aggregated PTC attestations from block |
| 6 | `BEACON_API_ETH_V2_BEACON_BLOCK_EXECUTION_PAYLOAD_BID` | Cannon | Winning bid from block |
| 7 | `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_ENVELOPE` | P2P | Payload envelope gossip |
| 8 | `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_BID` | P2P | Bid gossip |
| 9 | `LIBP2P_TRACE_GOSSIPSUB_PAYLOAD_ATTESTATION_MESSAGE` | P2P | PTC attestation gossip |
| 10 | `LIBP2P_TRACE_GOSSIPSUB_PROPOSER_PREFERENCES` | P2P | Proposer preferences gossip |

### Modified Events (11 total)

| # | Event | What Changes |
|---|-------|-------------|
| 1 | `BEACON_API_ETH_V2_BEACON_BLOCK` | Block body has bid instead of execution_payload |
| 2 | `BEACON_API_ETH_V2_BEACON_BLOCK_EXECUTION_TRANSACTION` | Source from envelope |
| 3 | `BEACON_API_ETH_V2_BEACON_BLOCK_WITHDRAWAL` | Source from envelope + builder withdrawals |
| 4 | `BEACON_API_ETH_V1_EVENTS_BLOCK` | Block arrives before payload |
| 5 | `BEACON_API_ETH_V1_EVENTS_ATTESTATION` | Index field: 0=no payload, 1=payload present |
| 6 | `BEACON_API_ETH_V2_BEACON_BLOCK_ACCESS_LIST` | BAL in envelope, not block body |
| 7 | `CONSENSUS_ENGINE_API_NEW_PAYLOAD` | V5 format |
| 8 | `BEACON_API_ETH_V1_DEBUG_FORK_CHOICE` | PayloadStatus (EMPTY/FULL/PENDING) + PTC votes |
| 9 | `BEACON_API_ETH_V1_EVENTS_DATA_COLUMN_SIDECAR` | Structural change (additive columns) |
| 10 | `LIBP2P_TRACE_GOSSIPSUB_DATA_COLUMN_SIDECAR` | Structural change (additive columns) |
| 11 | `BEACON_API_ETH_V2_BEACON_BLOCK_WITHDRAWAL` | Builder withdrawals via `BUILDER_INDEX_FLAG` |

---

## Spec References

- [EIP-7732: ePBS](https://eips.ethereum.org/EIPS/eip-7732)
- [EIP-7928: BALs](https://eips.ethereum.org/EIPS/eip-7928)
- [EIP-8159: eth/71](https://eips.ethereum.org/EIPS/eip-8159)
- [Consensus specs: gloas/](https://github.com/ethereum/consensus-specs/tree/master/specs/gloas)
- [Execution specs: amsterdam/](https://github.com/ethereum/execution-specs/tree/forks/amsterdam/src/ethereum/forks/amsterdam)
- [go-eth2-client BALs PR](https://github.com/pk910/go-eth2-client/pull/7)
- [go-eth2-client ePBS PR](https://github.com/pk910/go-eth2-client/pull/4)
- [EF Checkpoint #8](https://blog.ethereum.org/2026/01/20/checkpoint-8)

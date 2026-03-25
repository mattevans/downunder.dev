# ePBS: EIP-7732 — Xatu Implementation

> **EIP**: [7732](https://eips.ethereum.org/EIPS/eip-7732) (Enshrined Proposer-Builder Separation)
> **Fork**: Glamsterdam (Gloas CL + Amsterdam EL)
> **Fork Version**: [`GLOAS_FORK_VERSION = 0x07000000`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/fork.md#configuration)
> **Last Updated**: 2026-03-25
> **Status**: Not started — blocked on upstream go-eth2-client combined branch
> **Specs**: [beacon-chain](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md) | [fork-choice](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/fork-choice.md) | [p2p-interface](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md) | [validator](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/validator.md) | [builder](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/builder.md) | [fork](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/fork.md)

---

## How ePBS Changes Block Production

ePBS splits the Ethereum block into two separately-delivered objects and adds a new committee:

```
Slot N (12 seconds)                                           Slot N+1
|                                                             |
|  0% (0s)     25% (3s)     50% (6s)     75% (9s)  100%     |
|  |           |            |            |           |        |
|  Block       Attest       Aggregate    PTC                  |
|  (bid)       deadline     deadline     attestation          |
|              (was 33%)    (was 67%)    deadline (NEW)       |
|                                                             |
|  Builder reveals payload (~2s after block)                  |
|  |-------|                                                  |
|  t=0    t=~2s                                               |
```

1. **t=0s** — Proposer publishes beacon block containing a builder **bid** (not the execution payload)
2. **t=~2s** — Builder reveals `SignedExecutionPayloadEnvelope` on P2P
3. **t=3s** — Regular attesters attest (index=0 same-slot, index=1 payload-present)
4. **t=9s** — **PTC** (512 validators) attests to payload timeliness

---

## New Types We Must Define (Proto)

### [`ExecutionPayloadBid`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md#executionpayloadbid)

```python
class ExecutionPayloadBid(Container):
    parent_block_hash: Hash32
    parent_block_root: Root
    block_hash: Hash32
    prev_randao: Bytes32
    fee_recipient: ExecutionAddress
    gas_limit: uint64
    builder_index: BuilderIndex
    slot: Slot
    value: Gwei
    execution_payment: Gwei
    blob_kzg_commitments: List[KZGCommitment, MAX_BLOB_COMMITMENTS_PER_BLOCK]

class SignedExecutionPayloadBid(Container):
    message: ExecutionPayloadBid
    signature: BLSSignature
```

### [`ExecutionPayloadEnvelope`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md#executionpayloadenvelope)

```python
class ExecutionPayloadEnvelope(Container):
    payload: ExecutionPayload
    execution_requests: ExecutionRequests
    builder_index: BuilderIndex
    beacon_block_root: Root
    slot: Slot
    state_root: Root

class SignedExecutionPayloadEnvelope(Container):
    message: ExecutionPayloadEnvelope
    signature: BLSSignature
```

### [`PayloadAttestation`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md#payloadattestationdata) (PTC)

```python
class PayloadAttestationData(Container):
    beacon_block_root: Root
    slot: Slot
    payload_present: boolean
    blob_data_available: boolean

class PayloadAttestationMessage(Container):
    validator_index: ValidatorIndex
    data: PayloadAttestationData
    signature: BLSSignature

class PayloadAttestation(Container):
    aggregation_bits: Bitvector[PTC_SIZE]  # 512 bits
    data: PayloadAttestationData
    signature: BLSSignature

class IndexedPayloadAttestation(Container):
    attesting_indices: List[ValidatorIndex, PTC_SIZE]
    data: PayloadAttestationData
    signature: BLSSignature
```

### [`Builder`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md#builder)

```python
class Builder(Container):
    pubkey: BLSPubkey
    version: uint8
    execution_address: ExecutionAddress
    balance: Gwei
    deposit_epoch: Epoch
    withdrawable_epoch: Epoch
```

### [`ProposerPreferences`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md#new-proposerpreferences)

```python
class ProposerPreferences(Container):
    proposal_slot: Slot
    validator_index: ValidatorIndex
    fee_recipient: ExecutionAddress
    gas_limit: uint64

class SignedProposerPreferences(Container):
    message: ProposerPreferences
    signature: BLSSignature
```

---

## Modified BeaconBlockBody ([spec](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md#beaconblockbody))

**Removed:**
- `execution_payload` — now in `ExecutionPayloadEnvelope`, delivered separately
- `blob_kzg_commitments` — moved into `ExecutionPayloadBid`
- `execution_requests` — moved into `ExecutionPayloadEnvelope`

**Added:**
- `signed_execution_payload_bid: SignedExecutionPayloadBid`
- `payload_attestations: List[PayloadAttestation, MAX_PAYLOAD_ATTESTATIONS]` (max 4)

**Our `BeaconBlockBodyGloas` proto must reflect this.**

---

## New Events to Implement

### Sentry SSE Events (real-time)

| Event Name | Source | Description |
|-----------|--------|-------------|
| `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD` | Beacon API SSE | Verified payload + blob availability. **Already in [beacon-APIs spec](https://github.com/ethereum/beacon-APIs/blob/master/apis/eventstream/index.yaml)** as `execution_payload_available`. |
| `BEACON_API_ETH_V1_EVENTS_PAYLOAD_ATTESTATION` | Beacon API SSE | Individual PTC attestation (~512/slot). **Already in spec** as `payload_attestation_message`. |
| `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD_BID` | Beacon API SSE | All builder bids. **Already in spec** as `execution_payload_bid`. |
| `BEACON_API_ETH_V1_EVENTS_PROPOSER_PREFERENCES` | Beacon API SSE | Proposer preference signal. [beacon-APIs PR#593](https://github.com/ethereum/beacon-APIs/pull/593) — in-flight. |

### Cannon Derived Events (from finalized blocks)

| Event Name | Source | Description |
|-----------|--------|-------------|
| `BEACON_API_ETH_V2_BEACON_BLOCK_PAYLOAD_ATTESTATION` | Cannon | Aggregated PTC attestations from the block body (max 4/block). Canonical record of what PTC votes made it on-chain. |
| `BEACON_API_ETH_V2_BEACON_BLOCK_EXECUTION_PAYLOAD_BID` | Cannon | The winning bid included by the proposer (1/block). Canonical record of bid value, builder, payment. |

### P2P Gossip Events (CL mimicry)

| Event Name | Gossip Topic | Spec |
|-----------|-------------|------|
| `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_ENVELOPE` | [`execution_payload`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md#execution_payload) | Payload envelope propagation |
| `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_BID` | [`execution_payload_bid`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md#execution_payload_bid) | Builder bid propagation |
| `LIBP2P_TRACE_GOSSIPSUB_PAYLOAD_ATTESTATION_MESSAGE` | [`payload_attestation_message`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md#payload_attestation_message) | PTC attestation propagation |
| `LIBP2P_TRACE_GOSSIPSUB_PROPOSER_PREFERENCES` | [`proposer_preferences`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md#proposer_preferences) | Proposer preference propagation |

---

## Existing Events to Modify

### BeaconBlock Events

`BEACON_API_ETH_V2_BEACON_BLOCK`, `BEACON_API_ETH_V1_EVENTS_BLOCK`, `BEACON_API_ETH_V3_VALIDATOR_BLOCK`

The Gloas block body no longer contains `execution_payload`, `blob_kzg_commitments`, or `execution_requests`. Instead it has `signed_execution_payload_bid` and `payload_attestations`.

**Changes:**
- Proto: `BeaconBlockBodyGloas` must reflect the new structure
- Block conversion functions must handle Gloas variant
- ClickHouse `beacon_api_eth_v2_beacon_block` / `canonical_beacon_block` need new columns: `builder_index`, `bid_value`, `execution_payment`, `payload_present`

### Execution Transaction / Withdrawal / BAL Derivers

`BEACON_API_ETH_V2_BEACON_BLOCK_EXECUTION_TRANSACTION`, `BEACON_API_ETH_V2_BEACON_BLOCK_WITHDRAWAL`, `BEACON_API_ETH_V2_BEACON_BLOCK_ACCESS_LIST`

Transactions, withdrawals, and BALs are now in the `ExecutionPayloadEnvelope`, not the block body. Cannon derivers must source data from the envelope.

### Attestation Events

`BEACON_API_ETH_V1_EVENTS_ATTESTATION`, `BEACON_API_ETH_V2_BEACON_BLOCK_ELABORATED_ATTESTATION`, `BEACON_P2P_ATTESTATION`

The `index` field in `AttestationData` is [repurposed](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md#beacon_aggregate_and_proof):
- `0` = same-slot attestation or payload not present (EMPTY)
- `1` = payload is present (FULL)

Consider adding a derived `payload_vote` column for Gloas+ attestations.

### DataColumnSidecar Events ([spec](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md#modified-datacolumnsidecar))

`BEACON_API_ETH_V1_EVENTS_DATA_COLUMN_SIDECAR`, `LIBP2P_TRACE_GOSSIPSUB_DATA_COLUMN_SIDECAR`

The `DataColumnSidecar` container changes:
- **Removed**: `signed_block_header`, `kzg_commitments`, `kzg_commitments_inclusion_proof`
- **Added**: `slot`, `beacon_block_root`

**Approach (decided):** Additive only. Add new columns; Fulu rows keep them NULL, Gloas populates them. No migration.

### Fork Choice Dumps ([spec](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/fork-choice.md))

`BEACON_API_ETH_V1_DEBUG_FORK_CHOICE`, `BEACON_API_ETH_V1_DEBUG_FORK_CHOICE_REORG`

Fork choice now tracks [`PayloadStatus`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/fork-choice.md#types) per node (EMPTY=0, FULL=1, PENDING=2) and new Store fields:
- `payload_states`, `payload_timeliness_vote`, `payload_data_availability_vote`

Add `payload_status` to fork choice node tables.

### Withdrawal Events ([spec](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md#modified-get_expected_withdrawals))

`BEACON_API_ETH_V2_BEACON_BLOCK_WITHDRAWAL`

`get_expected_withdrawals` now includes builder withdrawals. The `Withdrawal` type uses builder indices (with `BUILDER_INDEX_FLAG = 2^40` bit set).

### Head Event ([beacon-APIs PR#590](https://github.com/ethereum/beacon-APIs/pull/590))

`BEACON_API_ETH_V1_EVENTS_HEAD`

`head_v2` event replaces `head`. We'll need a new/updated sentry handler.

### Chain Reorg Event ([beacon-APIs PR#585](https://github.com/ethereum/beacon-APIs/pull/585))

`BEACON_API_ETH_V1_EVENTS_CHAIN_REORG`

Adds old/new execution block hashes to the reorg event (payload reorgs are possible under ePBS where the beacon block stays the same but the payload changes). New fields to capture.

---

## New ClickHouse Tables

| Table | Key Columns |
|-------|-------------|
| `canonical_beacon_block_payload_attestation` | slot, beacon_block_root, aggregation_bits, payload_present, blob_data_available, attesting_validator_count |
| `canonical_beacon_block_execution_payload_bid` | slot, builder_index, block_hash, parent_block_hash, parent_block_root, value, execution_payment, fee_recipient, gas_limit, blob_commitment_count |
| `beacon_api_eth_v1_events_execution_payload` | slot, block_root, builder_index, execution_payload_hash, arrival_time |
| `libp2p_trace_gossipsub_execution_payload_envelope` | slot, builder_index, block_root, propagation_time |
| `libp2p_trace_gossipsub_payload_attestation_message` | slot, validator_index, beacon_block_root, payload_present, blob_data_available |
| `libp2p_trace_gossipsub_execution_payload_bid` | slot, builder_index, block_hash, value, execution_payment |
| `libp2p_trace_gossipsub_proposer_preferences` | slot, validator_index, fee_recipient, gas_limit |

### Modified Existing Tables

| Table | New Columns |
|-------|-------------|
| `beacon_api_eth_v2_beacon_block` / `canonical_beacon_block` | `builder_index`, `bid_value`, `execution_payment`, `payload_present` |
| DataColumnSidecar tables | `slot` (Nullable), `beacon_block_root` (Nullable) — additive, NULL for Fulu |

---

## Sentry Changes

| Change | Description |
|--------|-------------|
| New: `OnExecutionPayload` subscription | Subscribe to `execution_payload` SSE event. Track arrival time for timing analysis. |
| Modify: `OnBlock` handler | Handle Gloas blocks — no inline `execution_payload` in body. |
| Modify: Block version detection | Route `DataVersionGloas` to correct proto conversion. |

---

## Cannon Deriver Changes

| Change | Description |
|--------|-------------|
| Modify: `BeaconBlock` deriver | Handle Gloas blocks without inline execution payload. |
| Modify: `ExecutionTransaction` deriver | Source transactions from payload envelope, not block body. |
| Modify: `Withdrawal` deriver | Source withdrawals from payload envelope. Handle builder withdrawals (`BUILDER_INDEX_FLAG`). |
| Modify: `BlockAccessList` deriver | Source BAL from payload envelope. |
| New: `PayloadAttestation` deriver | Extract aggregated PTC attestations from block body. |
| New: `ExecutionPayloadBid` deriver | Extract winning bid from block body. |

---

## Server / Event Ingester Changes

- Register all new event types in `event.go` router
- New event handlers for each new event type
- Persistence/location support for new cannon types

## Vector Pipeline Changes

- New routing rules for each new event type
- VRL transforms for denormalization
- ClickHouse sink configurations

---

## Key Spec Details for Implementation

### Constants ([spec](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md#preset))

| Name | Value |
|------|-------|
| `PTC_SIZE` | `512` |
| `MAX_PAYLOAD_ATTESTATIONS` | `4` |
| `BUILDER_REGISTRY_LIMIT` | `2^40` |
| `BUILDER_INDEX_FLAG` | `2^40` |
| `BUILDER_INDEX_SELF_BUILD` | `UINT64_MAX` |
| `DOMAIN_BEACON_BUILDER` | `0x0B000000` |
| `DOMAIN_PTC_ATTESTER` | `0x0C000000` |
| `DOMAIN_PROPOSER_PREFERENCES` | `0x0D000000` |
| `BUILDER_WITHDRAWAL_PREFIX` | `0x03` |

### Timing ([spec](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/validator.md#time-parameters))

| Parameter | Gloas Value | % of Slot |
|-----------|-------------|-----------|
| `ATTESTATION_DUE_BPS_GLOAS` | 2500 | 25% (was ~33%) |
| `AGGREGATE_DUE_BPS_GLOAS` | 5000 | 50% (was ~67%) |
| `PAYLOAD_ATTESTATION_DUE_BPS` | 7500 | 75% (NEW) |

### Fork Choice ([spec](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/fork-choice.md))

| Status | Value | Meaning |
|--------|-------|---------|
| `PAYLOAD_STATUS_EMPTY` | 0 | Block present, payload withheld |
| `PAYLOAD_STATUS_FULL` | 1 | Block + payload both received |
| `PAYLOAD_STATUS_PENDING` | 2 | Block seen, waiting for payload |
| `PAYLOAD_TIMELY_THRESHOLD` | 256 | Min PTC votes for "timely" |

### New P2P REQ/RESP ([spec](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md))

| Protocol | Request | Response |
|----------|---------|----------|
| [`ExecutionPayloadEnvelopesByRange v1`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md#executionpayloadenvelopesbyrange-v1) | `(start_slot, count)` | `List[SignedExecutionPayloadEnvelope]` |
| [`ExecutionPayloadEnvelopesByRoot v1`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md#executionpayloadenvelopesbyroot-v1) | `List[Root, 128]` | `List[SignedExecutionPayloadEnvelope]` |

### Execution Request Processing ([spec](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md#new-process_execution_payload))

Deposit, withdrawal, and consolidation requests are now processed during `process_execution_payload` (envelope processing), **not** during `process_block`. If a payload is withheld, no execution requests are processed for that slot.

### Builder Lifecycle ([builder.md](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/builder.md), [fork.md](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/fork.md))

- Onboarded via deposit mechanism with `BUILDER_WITHDRAWAL_PREFIX = 0x03`
- Active once deposit epoch is finalized (~2 epochs)
- Exit via voluntary exit (checked via `is_builder_index` flag)
- At fork activation: [`onboard_builders_from_pending_deposits`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/fork.md#new-onboard_builders_from_pending_deposits) processes all pending builder-credentialed deposits

---

## TYSM: Direct Event Capture from Prysm

[TYSM](https://github.com/ethpandaops/tysm) is our patched Prysm overlay that hooks into the CL client and forwards events to Xatu. Upstream Prysm already has preliminary Glamsterdam/ePBS support. **TYSM lets us capture ePBS events directly without waiting for beacon API SSE proposals.**

### New TYSM Events to Add

| Event | Hook Into | Xatu TraceEvent | Priority |
|-------|----------|-----------------|----------|
| `OnPayloadAttestationReceive` | Prysm PTC gossip validation | `LIBP2P_TRACE_GOSSIPSUB_PAYLOAD_ATTESTATION_MESSAGE` | **P0** |
| `OnExecutionPayloadEnvelopeReceive` | Prysm envelope gossip validation | `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_ENVELOPE` | **P0** |
| `OnExecutionPayloadBidReceive` | Prysm bid gossip validation | `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_BID` | **P0** |
| `OnProposerPreferencesReceive` | Prysm preferences gossip validation | `LIBP2P_TRACE_GOSSIPSUB_PROPOSER_PREFERENCES` | P1 |
| `OnPayloadAttestationProcessed` | Prysm PTC pool insertion | (new enriched event) | P1 |

### Implementation per event (follows existing patterns)

1. **Event type** — `/overlay/tysm/events/payload_attestation.go` (etc.)
2. **Interface** — `/overlay/tysm/interfaces/` — `PayloadAttestationObserveHooks`
3. **Dispatcher** — `/overlay/tysm/dispatcher/p2p_gossip.go`
4. **Emit function** — `/overlay/tysm/inject.go` — `EmitPayloadAttestationReceive()`
5. **Xatu forwarder** — `/overlay/tysm/xatu/forwarder/handlers.go`
6. **Prysm patch** — add emit call at gossip validation point

### Existing TYSM events that change under Gloas

| Event | Change |
|-------|--------|
| `OnBlockReceive` | Gloas blocks have bid instead of execution_payload — verify forwarding handles this |
| `OnAttestationReceive` | Index field now 0/1 payload vote — verify Xatu mapping |
| `OnDataColumnReceive` | Sidecar structure changed (slot/beacon_block_root instead of header) — verify forwarding |
| `OnEngineNewPayload` | V5 payloads with BALs + slotNumber — verify method version capture |

---

## Upstream Dependencies

| Dependency | Status | Blocker? |
|-----------|--------|----------|
| `go-eth2-client` ePBS | [pk910/go-eth2-client#4](https://github.com/pk910/go-eth2-client/pull/4) — Open | **YES** — blocks sentry + cannon work |
| `go-eth2-client` combined BALs+ePBS | Not started | **YES** — expected ~1 month |
| `go-ethereum` ePBS EL changes | TBD | Unknown |
| `ethcore` ePBS | TBD | Unknown |
| `tysm` Gloas hooks | **TODO** — architecture ready | **No** — we control this |

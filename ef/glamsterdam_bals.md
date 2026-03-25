# BALs: EIP-7928 + EIP-8159 — Xatu Implementation

> **EIPs**: [7928](https://eips.ethereum.org/EIPS/eip-7928) (Block-Level Access Lists), [8159](https://eips.ethereum.org/EIPS/eip-8159) (eth/71 P2P Exchange)
> **Fork**: Glamsterdam (Gloas CL + Amsterdam EL)
> **Branch**: `release/gloas`
> **Last Updated**: 2026-03-25
> **Specs**: [execution-specs](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/amsterdam/block_access_lists.py)

---

## Status: Mostly Complete

The core BALs pipeline is implemented on `release/gloas`. This document tracks **remaining work only**.

---

## Remaining Work

### 1. Storage Reads — Proto + Conversion

The BAL spec includes `storage_reads: List[StorageKey]` per account — slots that were accessed but not written. These are **not currently captured**.

**What's missing:**
- `BlockAccessListEntry` proto needs a `repeated google.protobuf.StringValue storage_reads` field
- `NewBlockAccessListFromGloas()` in `pkg/proto/eth/v1/conversion.go` needs to iterate `access.StorageReads`
- Denormalized `BlockAccessListChange` needs a `"storage_read"` change_type (note: reads have no `block_access_index` or value — just the key)
- ClickHouse `canonical_beacon_block_access_list` table needs to handle `storage_read` rows

**Why it matters:** Storage reads are essential for parallel execution analysis — identifying which transactions have disjoint access sets.

**Spec reference:** [block_access_lists.py — `storage_reads`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/amsterdam/block_access_lists.py)

### 2. Sentry — Gloas Block Handling

Update beacon block subscription handlers (`OnBlock`, `OnHead`) to handle `DataVersionGloas` blocks. Route to correct proto conversion (`NewEventBlockFromGloas`).

### 3. Consumoor Routes

Add BAL route handlers for consumoor (`canonical_beacon_block_access_list`).

### 4. Engine API Tracking

Update `consensus_engine_api_new_payload` event to handle V5 payload format:
- `ExecutionPayloadV4` adds [`blockAccessList`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/amsterdam/blocks.py) (RLP) and `slotNumber` (uint64)

### 5. Block Access List Hash

Capture the new [`block_access_list_hash`](https://github.com/ethereum/execution-specs/blob/forks/amsterdam/src/ethereum/forks/amsterdam/blocks.py) header field in:
- `beacon_api_eth_v2_beacon_block` table
- `canonical_beacon_block` table

### 6. ethcore — eth/71 P2P Support

ethcore's EL mimicry client (`pkg/execution/mimicry`) needs [EIP-8159](https://eips.ethereum.org/EIPS/eip-8159) message handlers:

| Message | ID | Description |
|---------|----|-------------|
| `GetBlockAccessLists` | `0x12` | Request BALs by block hashes |
| `BlockAccessLists` | `0x13` | Response with BAL data |

This enables monitoring BAL propagation across the EL P2P network. Soft response limit: 2 MiB. Validation: `keccak256(rlp(bal))` must match `block-access-list-hash` in header.

### 7. xatu-cbt

Deferred to follow-up.

---

## What's Already Done

For reference, these are complete on `release/gloas`:

| Component | Details |
|-----------|---------|
| Proto: `BlockAccessList`, `BlockAccessListEntry`, `BlockAccessListChange` | `eth/v1/block_access_list.proto` |
| Proto: `BeaconBlockGloas`, `ExecutionPayloadGloas` | `eth/v2/beacon_block.proto`, `eth/v1/execution_engine.proto` |
| Event type: `BEACON_API_ETH_V2_BEACON_BLOCK_ACCESS_LIST = 89` | `xatu/event_ingester.proto` |
| Cannon deriver: `BlockAccessListDeriver` | `pkg/cannon/deriver/beacon/eth/v2/block_access_list.go` |
| RLP decoding: `NewBlockAccessListFromGloas()` | `pkg/proto/eth/v1/conversion.go` (with tests) |
| Server event handler | Routes and validates BAL events |
| ClickHouse: `canonical_beacon_block_access_list` table | Migration 106 |
| Vector pipeline | Kafka-to-ClickHouse routing |
| Block conversion: `NewEventBlockFromGloas()` | `pkg/proto/eth/block.go` |
| Checkpoint/backfill support | Cannon location serialization |

## Upstream Dependencies

| Dependency | Status |
|-----------|--------|
| `go-eth2-client` BALs | [pk910/go-eth2-client#7](https://github.com/pk910/go-eth2-client/pull/7) — Open, using `go mod replace` |
| `go-ethereum` BALs | v1.17.1 — Done |
| `ethcore` BALs RLP | Done |
| `ethcore` eth/71 | **TODO** |

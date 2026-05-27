# xatu — glamsterdam fork readiness

> Repo: [ethpandaops/xatu](https://github.com/ethpandaops/xatu) · branch: `release/gloas` · last devnet verification: **2026-05-20** (kurtosis local)

## TL;DR

Glamsterdam (Gloas CL + Amsterdam EL) on xatu is **structurally complete and spec-aligned**. All new ePBS (EIP-7732) and BAL (EIP-7928 + EIP-8159) events are plumbed end-to-end through sentry, cannon, server, and CL mimicry; routing migrated to **consumoor** (the old vector path is not used).

**Devnet tests have been run** — local kurtosis (2026-05-20) confirmed events publishing through TYSM all the way to ClickHouse, with all three `BEACON_SYNTHETIC_*` lifecycle events round-tripping. The remaining work is devnet integration testing for the events we couldn't exercise yet — not new feature work.

**The biggest blocker is upstream**: the CL clients don't yet have their builder APIs ready, so nothing is actually publishing bids or submitting executions on devnet. That means `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_BID`, `SETTLED` outcomes on `BEACON_SYNTHETIC_BUILDER_PENDING_PAYMENT_SETTLEMENT`, and `INVALID` transitions on `BEACON_SYNTHETIC_PAYLOAD_STATUS_RESOLVED` are all dormant — not because our code is broken, but because there's nothing for the wire to carry. Once a builder-active devnet exists (or once we wire `assertoor`'s `builder-lifecycle.yaml` playbook on top of kurtosis), these light up without further xatu changes.

| area | state |
|---|---|
| ePBS (EIP-7732) wired | <span class="pill pill-done">done</span> |
| BALs (EIP-7928) wired | <span class="pill pill-done">done</span> |
| ethcore eth/71 (EIP-8159) P2P | <span class="pill pill-blocked">deferred</span> (go-ethereum lacks types) |
| TYSM observe hooks verified on devnet | <span class="pill pill-done">6/7 ✅</span> · `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_BID` dormant (no external builder) |
| Sentry / cannon verified on devnet | <span class="pill pill-todo">todo</span> |
| Snapshot test fixtures | <span class="pill pill-todo">todo</span> · 10 placeholders awaiting devnet payload dumps |

## what was implemented

### ePBS — EIP-7732 enshrined proposer-builder separation

The core infrastructure is wired across every layer:

- **Proto types** — 6 new files defining `ExecutionPayloadBid` / `SignedExecutionPayloadBid` (with `execution_requests_root`), `ExecutionPayloadEnvelope` / `SignedExecutionPayloadEnvelope` (with `parent_beacon_block_root`), `PayloadAttestationData` / `PayloadAttestationMessage` / `PayloadAttestation` / `IndexedPayloadAttestation`, `Builder`, `ProposerPreferences` / `SignedProposerPreferences`. Spec refs: [`specs/gloas/beacon-chain.md`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/beacon-chain.md) and [`p2p-interface.md`](https://github.com/ethereum/consensus-specs/blob/master/specs/gloas/p2p-interface.md).

- **New events allocated** — 12 sentry/cannon events + 3 TYSM synthetic. See the master plan for the full table.

- **Sentry SSE integration** — 6 callbacks wired in `pkg/sentry/sentry.go` with handlers in `pkg/sentry/event/beacon/eth/v1/`:
  | callback | event | description |
  |---|---|---|
  | `OnExecutionPayload` | `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD` | envelope arrival |
  | `OnExecutionPayloadGossip` | `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD_GOSSIP` | gossip-receipt timing |
  | `OnExecutionPayloadAvailable` | `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD_AVAILABLE` | verified availability |
  | `OnExecutionPayloadBid` | `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD_BID` | bid via SSE |
  | `OnPayloadAttestationMessage` | `BEACON_API_ETH_V1_EVENTS_PAYLOAD_ATTESTATION` | PTC individual messages |
  | `OnProposerPreferences` | `BEACON_API_ETH_V1_EVENTS_PROPOSER_PREFERENCES` | proposer signals |

- **Cannon derivers** — 3 new derivers wired in `pkg/cannon/cannon.go` producing canonical records from finalized blocks:
  - `NewPayloadAttestationDeriver` (`CannonType 17`) — aggregated PTC attestations per block
  - `NewExecutionPayloadBidDeriver` (`CannonType 18`) — winning bid per block
  - `NewBlockAccessListDeriver` (`CannonType 16`) — BAL deriver also used here, sources from envelopes

- **Envelope-arrival side** — derivers for transactions, withdrawals, and BALs now source from `ExecutionPayloadEnvelope` via `beacon.GetExecutionPayloadEnvelope(...)`. Nil envelope (builder withheld) → empty output. All prior `TODO(epbs)` markers resolved.

- **Fork-choice integration** — `ForkChoiceNodeV2.payload_status` (proto field 10, `UInt32Value`) round-trips via `extra_data["payload_status"]` with a `payloadStatusFromExtraData` helper that tolerates both string (`"EMPTY"`/`"FULL"`/`"PENDING"`) and numeric (`0`/`1`/`2`) shapes.

- **CL mimicry gossipsub producers** — all 4 ePBS topics wired in `pkg/clmimicry/`:
  | gossip topic | event |
  |---|---|
  | `execution_payload` | `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_ENVELOPE` |
  | `execution_payload_bid` | `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_BID` |
  | `payload_attestation_message` | `LIBP2P_TRACE_GOSSIPSUB_PAYLOAD_ATTESTATION_MESSAGE` |
  | `proposer_preferences` | `LIBP2P_TRACE_GOSSIPSUB_PROPOSER_PREFERENCES` |

- **TYSM observe hooks** — ([`ethpandaops/tysm` PR #23](https://github.com/ethpandaops/tysm/pull/23) merged to `release/runtime-cfg-hooks`) all 7 hooks wired and verified on kurtosis 2026-05-20:
  - **4 gossip topics**: `OnPayloadAttestationReceive`, `OnExecutionPayloadEnvelopeReceive`, `OnExecutionPayloadBidReceive`, `OnProposerPreferencesReceive`
  - **3 synthetic lifecycle events** (no beacon-API SSE equivalent — these are pure xatu/TYSM constructs):
    | event | fires |
    |---|---|
    | `BEACON_SYNTHETIC_PAYLOAD_STATUS_RESOLVED` | per slot when fork-choice resolves PENDING → FULL/EMPTY/INVALID |
    | `BEACON_SYNTHETIC_BUILDER_PENDING_PAYMENT_SETTLEMENT` | per epoch at settlement or drop |
    | `BEACON_SYNTHETIC_PAYLOAD_ATTESTATION_PROCESSED` | after a PTC vote clears full gossip validation |

### BALs — EIP-7928 block-level access lists

- **Proto definitions** — `BlockAccessList`, `BlockAccessListEntry` with all **6** change types: `storage_changes`, `storage_reads`, `balance_changes`, `nonce_changes`, `code_changes`, plus `touched` accounts (the 6th one is easy to miss — the spec includes it). All in `eth/v1/block_access_list.proto`.

- **Execution payload integration** — `ExecutionPayloadGloas` carries `block_access_list: BlockAccessList` (field 18) and `slot_number: uint64` (field 19, EIP-7843). The latter was never written into the plans but it's there.

- **Cannon deriver** — `NewBlockAccessListDeriver` sources from envelope (Gloas path: `envelope.Payload.BlockAccessList`; pre-Gloas: no-op).

- **RLP decoding** — `NewBlockAccessListFromGloas()` in `pkg/proto/eth/v1/conversion.go` decodes RLP into all 6 change types. Fixed 2026-05-08: `block_access_index` column widened from `UInt16` to `UInt32` per spec.

- **ClickHouse table** — `canonical_beacon_block_access_list`, migration `003_gloas_bals_support`. Denormalized: one row per change with `change_type`, `address`, `storage_key`, `block_access_index`, `new_value`.

### other gloas changes

- **EIP-7843 (SLOTNUM)** — `slot_number` field on `ExecutionPayloadGloas`. Present in `execution_engine.proto:210`. Snapshot test added 2026-05-20.
- **EIP-8159 (eth/71 P2P)** — New devp2p messages `GetBlockAccessLists` (0x12) and `BlockAccessLists` (0x13) for BAL peer-to-peer exchange. **Deferred to ethcore** (separate repo). Blocker: go-ethereum lacks `ETH71` protocol constant and message types.
- **Gloas data column sidecars** — structural change drops `signed_block_header`, `kzg_commitments`, `kzg_commitments_inclusion_proof` and adds `slot`, `beacon_block_root`. SSE side requires no work (Prysm's `DataColumnGossipEvent` SSE payload is fork-agnostic — projects `slot`/`block_root` directly). Gossip side decoder fix lives in `pkg/clmimicry/gossipsub_data_column_sidecar.go` and requires the `DataColumnSidecarGloas` SSZ variant through TYSM.

## verified vs unverified

### verified on kurtosis

Stack: 6 CL clients (2× Prysm+TYSM against Ethrex/Nethermind, Lodestar, Lighthouse pair) with `gloas_fork_epoch: 1`. Xatu server + consumoor + Kafka + ClickHouse + Zookeeper. **No sentry or cannon agents attached** on this run.

TYSM-produced events landed in ClickHouse:

| event | rows | notes |
|---|---|---|
| `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_ENVELOPE` | 131 | <span class="pill pill-done">ok</span> |
| `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_BID` | 0 | <span class="pill pill-info">environmental</span> — 6 in-protocol builders registered (visible in dora) but none submitting; all proposers self-build (`builder_index = UINT64_MAX`) |
| `LIBP2P_TRACE_GOSSIPSUB_PAYLOAD_ATTESTATION_MESSAGE` | 2,588 | <span class="pill pill-done">ok</span> |
| `LIBP2P_TRACE_GOSSIPSUB_PROPOSER_PREFERENCES` | 50 | <span class="pill pill-done">ok</span> — periodic publishes confirmed |
| `BEACON_SYNTHETIC_PAYLOAD_STATUS_RESOLVED` | 52 | <span class="pill pill-done">ok</span> — mostly PENDING→FULL, some PENDING→EMPTY. **INVALID not yet observed** |
| `BEACON_SYNTHETIC_BUILDER_PENDING_PAYMENT_SETTLEMENT` | 6 | <span class="pill pill-done">ok</span> — all `DROPPED` (no external bidding) |
| `BEACON_SYNTHETIC_PAYLOAD_ATTESTATION_PROCESSED` | 1,375 | <span class="pill pill-done">ok</span> |

### not yet verified

Sentry and cannon agents were not attached on the 2026-05-20 run. These need a follow-up devnet run:

- **Sentry SSE events** — `BEACON_API_ETH_V1_EVENTS_FAST_CONFIRMATION`, `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD`, `BEACON_API_ETH_V1_EVENTS_PAYLOAD_ATTESTATION`, `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD_BID`, `BEACON_API_ETH_V1_EVENTS_PROPOSER_PREFERENCES`, `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD_GOSSIP`, `BEACON_API_ETH_V1_EVENTS_EXECUTION_PAYLOAD_AVAILABLE`. Code is wired; needs live beacon-api endpoint with Gloas SSE subscription.
- **Cannon-derived events** — `BEACON_API_ETH_V2_BEACON_BLOCK_ACCESS_LIST`, `BEACON_API_ETH_V2_BEACON_BLOCK_PAYLOAD_ATTESTATION`, `BEACON_API_ETH_V2_BEACON_BLOCK_EXECUTION_PAYLOAD_BID`. Code is wired; needs a finalized Gloas epoch for cannon to backfill.
- **10 snapshot test fixtures** — placeholders under `pkg/clickhouse/route/{beacon,canonical,libp2p}/` with `TODO(epbs)` markers. Currently fail with `nil ... payload: invalid event`. Need real devnet payload dumps to seed.

### spec compliance audit (2026-05-13)

- Proto types match Gloas consensus-specs `master` (including evolved shapes like `parent_beacon_block_root` on `ExecutionPayloadEnvelope`)
- All 6 BAL `AccountChanges` fields captured (including `touched`)
- ClickHouse migrations align with table requirements
- No spec-breaking drift in the proto/migration/route/deriver layer

## what's left before fork

### P0 — devnet round-trip completion

1. **Snapshot fixtures** — capture real Gloas block/payload/attestation payloads from a future devnet run and seed the 10 placeholder tests in `pkg/clickhouse/route/{beacon,canonical,libp2p}/`.
2. **Sentry devnet verification** — attach `xatu sentry` to a TYSM CL beacon-api endpoint with Gloas SSE subscriptions enabled. Verify `DataVersionGloas` routing for `OnBlock` / `OnHead` end-to-end.
3. **Cannon devnet verification** — run `xatu cannon` against finalized Gloas epochs to verify `BEACON_API_ETH_V2_BEACON_BLOCK_ACCESS_LIST`, `BEACON_API_ETH_V2_BEACON_BLOCK_PAYLOAD_ATTESTATION`, `BEACON_API_ETH_V2_BEACON_BLOCK_EXECUTION_PAYLOAD_BID` and additive column populations on `canonical_beacon_block` / `_withdrawal`.

### P1 — production realism

4. **Builder-active devnet** — re-run on a devnet where the 6 registered builders actively submit bids (assertoor `builder-lifecycle.yaml` playbook may need wiring). Unblocks `LIBP2P_TRACE_GOSSIPSUB_EXECUTION_PAYLOAD_BID` and `SETTLED` outcomes on `BEACON_SYNTHETIC_BUILDER_PENDING_PAYMENT_SETTLEMENT`.
5. **Adversarial mutator** — exercise `INVALID` payload-status transitions on `BEACON_SYNTHETIC_PAYLOAD_STATUS_RESOLVED` via the data-column-mutator hook (lives in tysm). Currently only `FULL` and `EMPTY` observed.

### ongoing — upstream tracking

6–9 — see [upstream PRs to watch](#upstream-prs-to-watch) below.

## upstream dependencies

## upstream PRs to watch

These are the schematic blockers.

| PR | repo | title | impact | status |
|---|---|---|---|---|
| [#5241](https://github.com/ethereum/consensus-specs/pull/5241) | consensus-specs | Per-builder configs in `ProposerPreferences` | <span class="pill pill-blocked">high</span> — additive proto + ClickHouse columns | open |
| [#593](https://github.com/ethereum/beacon-APIs/pull/593) | beacon-APIs | `proposer_preferences` SSE + pool API | <span class="pill pill-blocked">high</span> — our handler shipped early; if topic name/schema diverges, we follow | open |
| [#5221](https://github.com/ethereum/consensus-specs/pull/5221) | consensus-specs | Separate pending builder deposits queue | <span class="pill pill-todo">med</span> — affects builder registry snapshot shape (open Q #6) | draft |
| [#590](https://github.com/ethereum/beacon-APIs/pull/590) | beacon-APIs | `head_v2` event, deprecates `head` | <span class="pill pill-todo">med</span> — add Gloas+ subscription, keep `head` for pre-Gloas | open |
| [#585](https://github.com/ethereum/beacon-APIs/pull/585) | beacon-APIs | Execution block hashes in `chain_reorg` SSE | <span class="pill pill-todo">med</span> — additive nullable columns `old/new_execution_block_hash` | open |

## recommended successor actions

1. Attach `xatu sentry` + `xatu cannon` to the next devnet run (Gloas-enabled).
2. Seed the 10 snapshot fixtures from real devnet payloads on that run.
3. Monitor consensus-specs #5241 and beacon-APIs #593 for breaking changes — both are HIGH-impact and would force schema deltas.
4. Keep an eye on go-ethereum to see if it's picked up eth/71 types (currently blocking EIP-8159 P2P).

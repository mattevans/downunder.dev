# kurtosis devnet

> The full devnet config I've been using locally to exercise the Gloas pipeline end-to-end.

## what runs on the host

The TYSM `xatu` hook publishes to `host.docker.internal:8080`, so you need:

- **xatu server** on the host, listening `:8080` (gRPC) — receives events from the TYSM `xatu` hook
- **consumoor + kafka + clickhouse + zookeeper** downstream of xatu to land events
- **bad-tysm** on the host (port `:8666`) — auto-discovers TYSM containers via docker label scan and lets you drive activations from the UI

## kurtosis YAML

This config inlines both `xatu-config.yaml` (the TYSM xatu-hook output config) and `hooks.yaml` (the TYSM runtime hook config) via the `extra_files` block. Those files get mounted into every TYSM container at `/etc/tysm-xatu/` and `/etc/tysm-hooks/` respectively, and consumed via `--xatu-config-file` and `--tysm-hook-config-file` flags on the beacon-chain.

```yaml
extra_files:
  xatu-config.yaml: |
    name: "tysm-xatu-integration"
    labels: {}

    # Events configuration
    events:
      recvRpcEnabled: false
      sendRpcEnabled: false
      dropRpcEnabled: false
      rpcMetaControlIHaveEnabled: false
      rpcMetaControlIWantEnabled: false
      rpcMetaControlIDontWantEnabled: false
      rpcMetaControlGraftEnabled: false
      rpcMetaControlPruneEnabled: false
      rpcMetaSubscriptionEnabled: false
      rpcMetaMessageEnabled: false
      leaveEnabled: false
      pruneEnabled: false
      graftEnabled: false
      rejectMessageEnabled: false
      publishMessageEnabled: false
      deliverMessageEnabled: false
      duplicateMessageEnabled: false
      gossipSubBeaconBlockEnabled: true
      gossipSubAttestationEnabled: true
      gossipSubBlobSidecarEnabled: true
      syntheticHeartbeatEnabled: true
      identifyEnabled: true

    # Node configuration (simplified for Tysm)
    node:
      name: "tysm-node"

    ethereum:
      network: "kurtosis"

    sharding:
      # Configuration for events without sharding keys (Group D events)
      noShardingKeyEvents:
        enabled: true # Process all events without sharding keys
      # Topic-based sharding configuration
      topics:
        # Catch all event topics at 100% sampling.
        ".*":
          totalShards: 512
          activeShards: [ "0-511" ]
    outputs:
      - name: xatu
        type: xatu
        config:
          address: host.docker.internal:8080
          tls: false
          maxQueueSize: 500000
          batchTimeout: 1s
          exportTimeout: 15s
          maxExportBatchSize: 1000
          workers: 5
    coordinator:
      #xatu-grpc.analytics.staging.platform.ethpandaops.io:443
      address: host.docker.internal:8080

  hooks.yaml: |
    hook_logging:
      enabled: true
      classes: ["validate", "mutate"]
      observe_sample_rate: 0.0

    api:
      enabled: true
      listen: "0.0.0.0:8675"
      auth_token: "dev"
      max_activation_duration: "12h"

    hooks:
      - name: xatu
        enabled: true

      - name: metrics
        enabled: true
        config:
          peer_metrics:
            enabled: true
            connection_tracker_cleanup_interval: 5m
            max_tracked_connections: 10000

      - name: data-column-mutator
        enabled: false
        config:
          mutationProbability: 0.0
          enabledStrategies:
            - "byte-flip"
            - "kzg-corruption"
            - "column-index-swap"
          maxMutationsPerEvent: 1
          logMutationDetails: true
          enablePublishing: false
          publishOriginalAndMutant: false
          publisher:
            publishTimeout: 30
            maxRetries: 3
            retryDelay: 1
          strategies:
            byte-flip:
              numBitsToFlip: 16
              seed: 0
            kzg-corruption:
              corruptCommitments: true
              corruptProofs: true
              corruptBoth: false
            column-index-swap:
              seed: 0

participants_matrix:
  el:
    - el_type: nethermind
      el_image: ethpandaops/nethermind:glamsterdam-devnet-4
    - el_type: ethrex
      el_image: ethpandaops/ethrex:glamsterdam-devnet-4
  cl:
    - cl_type: prysm
      cl_image: ethpandaops/tysm:glamsterdam-devnet-4-tmp-002
      vc_image: ethpandaops/tysm-validator:glamsterdam-devnet-4-tmp-002
      cl_extra_params:
        - --xatu-config-file=/etc/tysm-xatu/xatu-config.yaml
        - --tysm-hook-config-file=/etc/tysm-hooks/hooks.yaml
        - --secret-access-key=i-love-pandas
        - --subscribe-all-subnets
        - --subscribe-all-data-subnets
        - --disable-resource-manager
        - --disable-connection-manager
        - --prepare-all-payloads
      cl_extra_mounts:
        "/etc/tysm-xatu": "xatu-config.yaml"
        "/etc/tysm-hooks": "hooks.yaml"
    - cl_type: lodestar
      cl_image: ethpandaops/lodestar:glamsterdam-devnet-4
    - cl_type: lighthouse
      cl_image: ethpandaops/lighthouse:glamsterdam-devnet-4

network_params:
  gloas_fork_epoch: 1
  withdrawal_type: "0x01"
  validator_balance: 40000
  genesis_gaslimit: 150000000
  gas_limit: 150000000

additional_services:
  - dora
  - assertoor
  - spamoor
  - checkpointz
spamoor_params:
  image: ethpandaops/spamoor:master

mev_type: buildoor

# NOTE: buildoor does not produce bids in this setup. Production EPBS devnets
# (glamsterdam-devnets/ansible/inventories/devnet-{2,3}) point buildoor at a
# custom Prysm build (bharath-123/prysm:buildoor-apis-glam-devnet-4) that
# keeps emitting payload_attributes SSE events post-Gloas — TYSM's Prysm base
# doesn't have those patches yet. Once they're merged upstream and TYSM
# rebases, the bidding path here should light up without further config.
# Until then, this section just keeps buildoor running so it's in the
# network (registered builder visible in Dora).
buildoor_params:
  image: ethpandaops/buildoor:main
  builder_api: true
  epbs_builder: true
  extra_args:
    # Buildoor self-deposits as an in-protocol builder using the prefunded wallet.
    - "--lifecycle"

assertoor_params:
  run_stability_check: false
  run_block_proposal_check: false
  tests:
    - { file: "https://raw.githubusercontent.com/ethpandaops/assertoor/refs/heads/master/playbooks/gloas-dev/builder-lifecycle.yaml" }

snooper_enabled: false
global_log_level: debug

checkpointz_params:
  image: ethpandaops/checkpointz:gloas-latest

port_publisher:
  additional_services:
    enabled: true
```

## run steps — end-to-end stack

Four terminals, in this order:

```bash
# 1. in your local clone of ethpandaops/xatu — start the data pipeline
#    (consumoor + kafka + clickhouse + zookeeper + xatu server on :8080).
#    TYSM containers publish events to host.docker.internal:8080.
cd <path-to>/xatu
docker compose up -d

# 2. in your local clone of ethpandaops/ethereum-package — spin up the enclave
cd <path-to>/ethereum-package
kurtosis run \
  --enclave tysm \
  --image-download missing \
  --args-file gloas2.yaml \
  .

# 3. in your local clone of ethpandaops/bad-tysm — start with docker discovery
#    on; it'll auto-pick-up every TYSM container kurtosis spawned in step 2
cd <path-to>/bad-tysm
BADTYSM_CONFIG_FILE=./config.yaml \
BAD_TYSM_AUTH_REF_TOKEN=dev \
go run ./cmd/badtysm

# 4. new terminal — bad-tysm frontend (Vite on :5173, proxies /api/* to :8666)
cd <path-to>/bad-tysm/frontend
pnpm dev
```

Then open:

- <http://localhost:5173> — **bad-tysm UI** (Vite dev server). Fleet view should show every TYSM container kurtosis spawned, auto-discovered via docker label scan. See the [bad-tysm page](#/bad-tysm) for the config.yaml format.

## related

- [xatu page](#/xatu) — what events should land + verified rows from 2026-05-20
- [tysm page](#/tysm) — how TYSM registers itself with bad-tysm on boot; hook API for activating the data-column-mutator
- [bad-tysm page](#/bad-tysm) — docker discovery configuration (`BADTYSM_DISCOVERY_DOCKER_*` env vars), socat host-mode mechanics

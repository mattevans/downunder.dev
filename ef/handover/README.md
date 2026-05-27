# overview

Three projects are in-flight right now. Each gets a dedicated page below with everything I'd want to know if I were starting tomorrow: what's done, what's left, what's blocked on whom, and where to look in the code.

<div class="cards">

<div class="card">

#### [xatu · glamsterdam](#/xatu) <span class="pill pill-flight">in-flight</span>

*consensus + execution observability*

All new gloas events (ePBS + BALs) implemented end-to-end through sentry, cannon, server and CL mimicry. Routing migrated to consumoor (not vector). Most events verified on a kurtosis devnet; the rest waiting on upstream deps.

</div>

<div class="card">

#### [tysm · hooks + discovery](#/tysm) <span class="pill pill-flight">in-flight</span>

*prysm fork with runtime hooks*

All new gloas event publishing wired into xatu. New runtime HTTP API for controlling hooks. New push-discovery so `bad-tysm` can find and drive instances. Branch: `release/runtime-cfg-hooks`.

</div>

<div class="card">

#### [bad-tysm · control plane](#/bad-tysm) <span class="pill pill-new">new</span>

*tysm control plane*

Complete overhaul. Postgres-backed registration/dereg, heartbeats, run hooks on specific nodes, visualise hook events from ClickHouse, build and execute multi-node scenarios. Embedded React UI.

</div>

</div>

## where to start

If you're inheriting this work cold, I'd read in this order:

1. **[xatu](#/xatu)** first — it's the data plane everything else feeds. The status is the most material thing: what's verified vs unverified, what's blocked on devnet availability.
2. **[tysm](#/tysm)** next — explains the events, the new hook API, and how the discovery contract with bad-tysm works. The mermaid diagram on that page is the single best 30-second overview of how things wire together.
3. **[bad-tysm](#/bad-tysm)** last — the control plane only makes sense once you've seen the things it controls.
4. **[kurtosis](#/kurtosis)** for the end-to-end devnet config I've been using locally — drop-in YAML + run steps.

---
name: dtnpm-rego-routing
description: >-
  Guides operators through writing and customizing neighbor-driven OPA/Rego routing policies (`routing.rego`, `package dtn.routing`) in `dtnpm` (`dtnpm-mod-rego-routing`) according to Hardy BPA's RIB model, including `VIA`/`FORWARD_REPLICATE`/`DROP` routes, APRS and Spray-and-Wait templates, and WebUI/YAML configuration. Use when configuring dynamic peer-driven routing rules.
---

# Customizing Rego Routing Policies (`routing.rego`) in `dtnpm`

Use this skill when a user wants to configure dynamic neighbor-driven routing in `dtnpm` (`dtnpm-mod-rego-routing`) using OPA/Rego (`package dtn.routing`).

## 1. Architecture & Hardy RIB Philosophy

`dtnpm` wraps every registered CLA with `ObservedCla` (`dtnpm-core/src/cla_observer.rs`). When a neighbor peer connects or disconnects (`add_peer` / `remove_peer`), `RegoRoutingEngine` updates `input.peers` and triggers `RegoRoutingAgent` to reconcile `data.dtn.routing.routes` against Hardy BPA's RIB (`add_route` / `remove_route`).

Follow Hardy's RIB rules (see `../hardy/docs/user-docs/configuration/bpa-server.md` and `../hardy/bpa/docs/routing_subsystem_design.md`):
- **Priority order (lower = checked first)**:
  - `0`: Admin endpoints & direct CLA peers
  - `1`: Local services (`service-priority`)
  - `100`: Default static routes & routing agents
- **Pattern specificity**: Exact EIDs beat narrow wildcards, which beat broad wildcards.
- **Default-to-Wait**: Unmatched bundles stay in `Waiting` until a valid route appears — they are not dropped unless an explicit `DROP` rule matches.
- **Least-privilege**: Never install blanket catch-all wildcards (`dtn:**`, `ipn:**`) toward arbitrary peers. Only install routes for explicitly authorized destination patterns and allowed neighbor peers.
- **Default behavior**: The embedded `dtnpm-mod-rego-routing/policies/routing.rego` defines `default routes := []` and a unified per-bundle `decide` rule (local unregistered → `WAIT`, direct destination → `FORWARD`, APRS strict hop → `FORWARD`, APRS CLA/band alias → `FORWARD` constrained to matching CLA, APRS ANSRVR group alias → `FORWARD` constrained to subscribed peers, Spray-and-Wait `L > 1` → `FORWARD` with `retain: true` / `L <= 1` → `WAIT`, plain bundle → `FALLBACK` to `DefaultRib` static routes). See [ADR-0001](../../../dtnpm/docs/adr/0001-aprs-cla-and-group-aliases.md).

## 2. Policy Contract (`package dtn.routing`)

### Input Schema

- **`data.dtn.routing.routes`** receives `(input.node, input.peers, input.current_dtn_time_ms)`.
- **`data.dtn.routing.decide`** (evaluated per bundle by `RegoRib::decide`) additionally receives `input.bundle` (`id`, `source`, `destination`, `aprs_strict_next_hop`, `aprs_active_alias`, `treb_visited_nodes`, `delivered_peers`, `spray_quota`, `payload_len`, `previous_node`) and per-peer `cla_aliases` and `groups`:

```json
{
  "node": {
    "local_eid": "dtn://node1/",
    "node_eids": ["dtn://node1/"],
    "blacklist_sources": ["dtn://blocked/"],
    "supported_aliases": ["TCPCL", "VHF", "ARES"],
    "group_members": { "ARES": ["dtn://node2/"] }
  },
  "peers": [
    {
      "cla_name": "tcpcl-1",
      "cla_type": "tcpclv4",
      "peer_id": 1,
      "peer_eid": "dtn://node2/",
      "cla_addr": "127.0.0.1:4556",
      "cla_aliases": ["TCPCL"],
      "groups": ["ARES"]
    }
  ]
}
```

### Output Schema (`data.dtn.routing.decide` & `routes`)

`decide` returns an object `{ "action": "FORWARD" | "WAIT" | "DROP" | "FALLBACK", ... }`:
- `FORWARD`: requires `next_hop` (string), optional `peer_id` (number, pins multi-CLA peer to the matching CLA), optional `retain` (bool).
- `WAIT`: holds bundle in Hardy's waiting queue until topology/policy/subscriptions change.
- `DROP`: deletes bundle with optional `reason_code` (integer).
- `FALLBACK`: delegates to `DefaultRib` static `routes:` / Default-to-Wait.

`routes` is a set (or array) of route objects:

| Field | Type | Required | Notes |
|---|---|---|---|
| `pattern` | string | Yes | Valid Hardy `EidPattern` (e.g., `"dtn://dest/**"`, `"ipn:0.99.*"`) |
| `action` | string | Yes | `"FORWARD_REPLICATE"`, `"VIA"`, `"DROP"`, or `"REFLECT"` |
| `next_hop` | string | For `VIA` / `FORWARD_REPLICATE` | Next-hop EID (usually `peer.peer_eid`) |
| `priority` | integer (`u32`) | Optional (default `100`) | Lower number takes precedence |
| `reason_code` | integer (`u64`) | Optional (for `DROP`) | BPv7 deletion reason code |

### Built-in Helpers (`data.dtn.helpers`)

- `helpers.peer_allowed(peer, input.node)` — `true` when `peer.peer_eid` is not in `blacklist_sources`.
- `helpers.is_local_destination(eid, input.node)` — `true` when `eid` belongs to the local node (prevents self-loops).
- `helpers.peer_in_group(peer, group_name)` — `true` when `peer` is currently subscribed to ANSRVR group `group_name`.

## 3. Ready-to-Use Policy Templates

Reference templates are provided in `dtnpm-mod-rego-routing/policies/`:
- `routing.rego` — Neutral default (`default routes := []`).
- `spray-and-wait-routing.rego` — Spray-and-Wait-only (`FORWARD_REPLICATE` for explicit `allowed_destinations` / `allowed_relays`).
- `aprs-routing.rego` — APRS-like `VIA` routes (priority `90`) for explicit `aprs_destinations` / `aprs_relays`.
- `aprs-spray-and-wait-routing.rego` — Combined APRS (`VIA` priority `90`) + Spray-and-Wait (`FORWARD_REPLICATE` priority `100`).

## 4. APRS Aliases (CLA/Band & ANSRVR Groups)

Legacy AX.25 `WIDE1`/`WIDE2` aliases are removed from default policies. Block 200 trajectory aliases support two DTN-native scopes:

1. **Kind 1 — CLA / Band / Transport Aliases (New n-N Paradigm only):**
   - Configure `aliases: ["VHF", "2M"]` (or `["UHF"]`, `["10m"]`, `["2G4"]`, `["QO100"]`, `["44NET"]`) on each `clas[]` entry in `HamConfig`.
   - `dtnpm` automatically appends the transport tag (`TCPCL`, `LTP`, `HQFBP`, `MERCURY`, `FILE`).
   - Bundles use New n-N syntax (`1 <= N <= n <= 7`): `--aprs-path VHF2-2` (or `--aprs-path QO100-1-1` for digit-ending aliases). Callsigns with SSIDs like `F4JXQ-9` parse unambiguously as strict hops. Forwarded strictly over peers connected on a CLA advertising that alias (with `peer_id` pinning), or parked in `WAIT` if none are connected.
2. **Kind 2 — ANSRVR Group Aliases (`dtn://<node>/ansrvr`):**
   - Built-in service accepting plain-text commands: `J <GROUP>`, `U <GROUP>`, `K <GROUP>`, `L [GROUP]`, `CQ <GROUP> <MSG>`, `H`/`HELP`.
   - Subscriptions persist in SQLite (`ansrvr_subscriptions`) with TTL pruning (default 24h), enforce source-only self-management, honour `DirectionalKeyStore` `VerifyPolicy`, reject reserved transport/CLA names, and rate-limit `CQ` fan-out (`cq-rate-limit-secs`).
   - Active group memberships feed directly into `RegoRoutingEngine` so Block 200 New n-N group hops (e.g. `--aprs-path ARES2-2`) route only to subscribed connected peers.
   - Manage via `rego-routing.ansrvr` in YAML/TOML or REST API (`GET /api/rego_routing/v1/groups`, `DELETE /api/rego_routing/v1/groups/{group}?member=<node_eid>`).

## 5. How to Apply & Verify

1. **YAML config** (`HamConfig.rego_routing`):
   ```yaml
   rego-routing:
     policy-dir: /path/to/policies        # loads routing.rego if present
     # OR inline override:
     custom-routing-rego: |
       package dtn.routing
       ...
   ```
2. **REST API / WebUI**: `PUT /api/rego_routing/v1/policy` with `custom_routing_rego` (persisted in SQLite `rego_policy_config`), or edit directly in the **Rego Routing** WebUI tab.
3. **Verify**:
   ```bash
   cd dtnpm-mod-rego-routing && cargo test --offline --test rego_policies_test
   ```

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
- **Least-privilege**: Never install blanket catch-all wildcards (`dtn://**`, `ipn:**`) toward arbitrary peers. Only install routes for explicitly authorized destination patterns and allowed neighbor peers.
- **Default behavior**: The embedded `dtnpm-mod-rego-routing/policies/routing.rego` defines `default routes := []` so `dtnpm` uses static declarative `routes` + blacklist by default.

## 2. Policy Contract (`package dtn.routing`)

### Input Schema

```json
{
  "node": {
    "local_eid": "dtn://node1/",
    "node_eids": ["dtn://node1/"],
    "blacklist_sources": ["dtn://blocked/"]
  },
  "peers": [
    {
      "cla_name": "tcpcl-1",
      "cla_type": "tcpclv4",
      "peer_id": 1,
      "peer_eid": "dtn://node2/",
      "cla_addr": "127.0.0.1:4556"
    }
  ]
}
```

### Output Schema (`data.dtn.routing.routes`)

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

## 3. Ready-to-Use Policy Templates

Reference templates are provided in `dtnpm-mod-rego-routing/policies/`:
- `routing.rego` — Neutral default (`default routes := []`).
- `spray-and-wait-routing.rego` — Spray-and-Wait-only (`FORWARD_REPLICATE` for explicit `allowed_destinations` / `allowed_relays`).
- `aprs-routing.rego` — APRS-like `VIA` routes (priority `90`) for explicit `aprs_destinations` / `aprs_relays`.
- `aprs-spray-and-wait-routing.rego` — Combined APRS (`VIA` priority `90`) + Spray-and-Wait (`FORWARD_REPLICATE` priority `100`).

## 4. Current Hardy Limitations & Workarounds

- **Per-bundle routing / size-based routing**: Hardy's RIB currently resolves routes by `bundle.primary.destination` (not per-bundle payload size or `route_key`). `routing.rego` only sees `(input.node, input.peers)`, not individual bundles.
- **Different paths for the same destination**: Install candidate routes in `routing.rego` (e.g. `FORWARD_REPLICATE` or distinct priorities/hops) and filter per `(bundle, contact / peer_eid)` in `egress.rego` using `SKIP`, or use distinct service EIDs at the destination (e.g. `dtn://dest/telemetry` vs `dtn://dest/files`).
- **APRS per-bundle trajectory**: Strict hops and `WIDE1`/`WIDE2` aliases in Block 200 continue to be evaluated per bundle in `ingress.rego` (`route_key`).

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

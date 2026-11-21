---
name: dtnpm-rego-filtering
description: >-
  Guides operators through customizing per-bundle ingress and egress OPA/Rego filtering policies (`ingress.rego`, `egress.rego`, `helpers.rego`, `package dtn.ingress`, `package dtn.egress`) in `dtnpm` (`dtnpm-mod-rego-routing`), including blacklist enforcement, loop prevention, Spray-and-Wait / APRS mutations, and `SKIP` filtering. Use when editing admission or per-contact forwarding rules via YAML config or the Rego Routing WebUI/API.
---

# Customizing Rego Filtering Policies (`ingress.rego` / `egress.rego`) in `dtnpm`

Use this skill when a user wants to customize per-bundle admission, loop prevention, extension block mutations, or per-contact forwarding decisions in `dtnpm` (`dtnpm-mod-rego-routing`).

## 1. Pipeline Overview

- **`package dtn.ingress`** (`dtnpm-mod-rego-routing/policies/ingress.rego`):
  Runs when a bundle enters the BPA (from a CLA or local origination). Responsible for:
  - Source/destination blacklist enforcement (`DROP`).
  - Lifetime / Hop Count limit checks (`DROP`).
  - Storm & loop prevention: duplicate `seen_cache`, TREB visited-node check, APRS trap/loop check (`DROP`).
  - APRS trajectory evaluation (Block 200 `trajectory_control`): alias decrement (`MUTATE_APRS_ALIAS`), strict hop advancement (`MUTATE_APRS_STRICT`), and `route_key` next-hop hint.
  - Spray-and-Wait quota seeding (Block 194 / Block 200) and TREB/HopCount updates.

- **`package dtn.egress`** (`dtnpm-mod-rego-routing/policies/egress.rego`):
  Runs for each `(bundle, next_hop_peer / contact)` forwarding attempt. Responsible for:
  - Direct destination delivery (`FORWARD_DIRECT`).
  - Split-horizon / visited-node loop guards (`SKIP` / `DROP`).
  - Spray-and-Wait Wait phase (`quota <= 1` -> `SKIP` for relays) and Spray phase (`quota >= 2` -> `FORWARD_REPLICATE` with `MUTATE_SPRAY_QUOTA`).
  - APRS broadcast (`FORWARD_BROADCAST`) vs unicast (`FORWARD_UNICAST`) and TREB departure mutations.

## 2. Decision Output Format

### Ingress (`data.dtn.ingress.decision`)
```json
{
  "action": "ACCEPT",
  "reason_code": null,
  "reason": "Bundle accepted for forwarding / storage",
  "generate_status_report": false,
  "route_key": null,
  "mutations": []
}
```
Valid `action` values: `"ACCEPT"`, `"DROP"`.

### Egress (`data.dtn.egress.decision`)
```json
{
  "action": "FORWARD_REPLICATE",
  "reason": "Binary Spray-and-Wait: splitting L=8 -> tx=4, keep=4",
  "mutations": [
    {
      "block_type": 194,
      "operation": "MUTATE_SPRAY_QUOTA",
      "transmitted_quota": 4,
      "retained_quota": 4
    }
  ]
}
```
Valid `action` values: `"FORWARD"`, `"FORWARD_DIRECT"`, `"FORWARD_REPLICATE"`, `"FORWARD_UNICAST"`, `"FORWARD_BROADCAST"`, `"SKIP"`, `"DROP"`.

## 3. Filtering Multiple Candidate Routes in `egress.rego`

Because Hardy's RIB currently resolves routes by `bundle.primary.destination` (not per-bundle payload attributes), if two bundles share the same destination EID but should take different links:
1. Configure the candidate routes in `routing.rego` (or static routes).
2. In `egress.rego`, return `"action": "SKIP"` when a bundle should not use a given `input.peer_eid` or `input.contact` (for example, skipping a low-bandwidth radio CLA for specific services or blocks while allowing a TCPCL peer).

## 4. Safety Rules When Overriding

- Always import `rego.v1` and reuse `data.dtn.constants` and `data.dtn.helpers` instead of duplicating block-type numbers or EID matching logic.
- Keep `default decision` safe and explicit; test both positive and negative cases (blacklisted EIDs, expired bundles, `HopLimit` reached, loop detection).
- Remember that `custom_ingress_rego` / `custom_egress_rego` overrides replace the entire `ingress.rego` / `egress.rego` module, so keep the required rules (blacklist, hop limit, storm/loop guards, Spray-and-Wait/APRS rules you rely on).

## 5. How to Apply & Verify

1. **YAML config** (`HamConfig.rego_routing`):
   ```yaml
   rego-routing:
     policy-dir: /path/to/policies        # loads ingress.rego / egress.rego / helpers.rego / dtn_constants.rego
     # OR inline overrides:
     custom-ingress-rego: |
       ...
     custom-egress-rego: |
       ...
   ```
2. **REST API / WebUI**: `PUT /api/rego_routing/v1/policy` (`custom_ingress_rego`, `custom_egress_rego`) or the **Rego Routing** tab.
3. **Verify**:
   ```bash
   cd dtnpm-mod-rego-routing && cargo test --offline --all-targets
   ```

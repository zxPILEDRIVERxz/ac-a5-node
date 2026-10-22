# ac-a5-node

ESPHome node that listens to and (when armed) transmits on the A5 IR-bus of a
Ducted/Mini-split AC, running on an XIAO C6. The AC's Tuya dongle stays
unplugged during normal operation; this node is the only thing on the bus.

## Files

| File | Purpose |
|---|---|
| `ac-a5-control-c6.yaml` | Full ESPHome config (entities, decoder, climate) |
| `aciq_tx.h` | A5 frame builder + CRC16 (byte-identical to ACiQ's), `!include`d by the yaml |
| `docs/` | Wiring + bus-takeover state docs |

Secrets are **not** in this repo: they are managed and shared from the ESPHome
dashboard (device-builder), which provisions `secrets.yaml` on pod `/config`.

## Where these files must live to be buildable

Both push paths build on the ESPHome build-server pod (ns `esphome`, dir `/config`):

1. **Dashboard (primary):** LAN-only, no auth — `http://192.168.0.222` (cluster LB
   into the esphome service; device-builder dashboard). It reads pod `/config` directly.
2. **HA ESPHome app:** uploads a tarball of its *stored project* from
   `/config/esphome/` on the HA host (192.168.0.117) into the pod. Its stored
   copy must be kept in sync with this repo (flat files only — no subdirs).

Sync this repo to the pod (run from a box with the cluster kubeconfig):

    P=$(kubectl -n esphome get pods -l app.kubernetes.io/name=esphome -o jsonpath='{.items[0].metadata.name}')
    kubectl -n esphome cp ac-a5-control-c6.yaml $P:/config/
    kubectl -n esphome cp aciq_tx.h            $P:/config/

(`secrets.yaml` is already on the pod, dashboard-managed.)

The pod's `/config` has its own git; commit there after syncing.

## Constraints (do not break)

- `esphome.name` stays `ac-a5-listen` (OTA identity shared with the proto build).
- Only one of proto / permanent is powered at a time (same OTA name).
- TX must be disarmed while anything else (e.g. Tuya) could drive the line;
  plugging Tuya back in while disarmed/listening is safe.
- ESPHome >= 2026.9/10 required (built-in Template Climate platform).

## Notes

- `aciq_tx.h` cannot be delivered by ESPHome `packages:` (YAML fragments only);
  it must sit next to the yaml on disk at build time — hence this repo + sync step.
- The ported `aciq_k18w` custom component is being replaced by the built-in
  Template Climate; the component tree does not belong in this repo.

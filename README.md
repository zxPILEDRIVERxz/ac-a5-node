# ac-a5-node

ESPHome node that listens to and (when armed) transmits on the A5 IR-bus of a
Ducted/Mini-split AC, running on an XIAO C6. The AC's Tuya dongle stays
unplugged during normal operation; this node is the only thing on the bus.

## Files

| File | Purpose |
|---|---|
| `ac-a5-control-c6.yaml` | Full ESPHome config (entities, decoder, climate) |
| `aciq_tx.h` | A5 frame builder + CRC16 (byte-identical to ACiQ's). Fetched from this repo at build time — never copied into project dirs |
| `library.json` | Minimal PlatformIO manifest; required so the repo can be a git source under `esphome: libraries:` |
| `wrapper.yaml` | The file stored in HA; pulls the config from this repo on every build |
| `docs/` | Wiring + bus-takeover state docs |

Secrets are **not** in this repo: they are managed and shared from the ESPHome
dashboard (device-builder), which provisions `secrets.yaml` on pod `/config`.

## How aciq_tx.h reaches the build (no flat copies)

The header is a git-fetched ESPHome library. The config declares:

    esphome:
      includes:
        - <aciq_tx.h>          # angle-bracket form: resolves via the library's include path
      libraries:
        - aciq_tx=https://github.com/zxPILEDRIVERxz/ac-a5-node#<commit-sha>

On the native ESP-IDF toolchain, ESPHome shallow-clones this repo at the pinned
commit into its component cache and registers the clone as a managed component
whose include directory is the repo root, so `<aciq_tx.h>` resolves from
main.cpp. **No project directory — pod or HA — keeps a copy of `aciq_tx.h`.**

### Keeping the header fresh (SHA-bump discipline)

The library cache is keyed by (URL, fragment) and is never refreshed, so the
pinned fragment is the *only* freshness lever:

1. Change `aciq_tx.h` (or `library.json`) and push that commit.
2. In a **follow-up commit**, bump the fragment in `ac-a5-control-c6.yaml` to
   the SHA of the header commit and push again:

       git log -1 --format=%H -- aciq_tx.h library.json

Each build then compiles exactly the pinned header — deterministic on both
push paths, no manual syncing.

## Where these files must live to be buildable

Both push paths build on the ESPHome build-server pod (ns `esphome`, dir
`/config`):

1. **Dashboard (primary):** LAN-only, no auth — the device-builder dashboard
   reads pod `/config` directly.
2. **HA ESPHome app:** bundles its *stored project* (just `wrapper.yaml` +
   secrets) into a tarball and ships it to the pod for compilation. The config
   yaml is re-pulled from this repo on every build, and the header comes from
   the library fetch above — nothing else needs manual syncing. A stray flat
   `aciq_tx.h` left in the HA project dir is inert: only referenced files are
   bundled, and no include resolves to a project-dir copy anymore.

Sync the config to the pod (run from a box with the cluster kubeconfig):

    P=$(kubectl -n esphome get pods -l app.kubernetes.io/name=esphome -o jsonpath='{.items[0].metadata.name}')
    kubectl -n esphome cp ac-a5-control-c6.yaml $P:/config/

(`secrets.yaml` is already on the pod, dashboard-managed.) The pod's `/config`
has its own git; commit there after syncing. One exception to git-freshness:
the wrapper *file itself* is stored in HA — if you ever edit `wrapper.yaml`,
update the stored copy too.

## Constraints (do not break)

- `esphome.name` stays `ac-a5-listen` (OTA identity shared with the proto build).
- Only one of proto / permanent is powered at a time (same OTA name).
- TX must be disarmed while anything else (e.g. Tuya) could drive the line;
  plugging Tuya back in while disarmed/listening is safe.
- ESPHome >= 2026.9/10 required (built-in Template Climate platform, native
  ESP-IDF toolchain library handling).

## Notes

- `library.json` exists only because this ESPHome version rejects git-sourced
  libraries without a manifest; keep it boring (its content never changes).
- The ported `aciq_k18w` custom component is being replaced by the built-in
  Template Climate; the component tree does not belong in this repo.

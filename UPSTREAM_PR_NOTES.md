# Upstream PR Notes

## Local deploy shape in this worktree

This local deploy keeps the current gate-safe default intact:

- Default Docker sandbox creation still forces `--network=none`.
- Local operators can opt out with a two-part recipe:
  set `HERMES_ALLOW_NETWORK=1` and change `~/.hermes/config.yaml`
  `terminal.docker_extra_args` so it no longer pins `--network=none`
  (for example, set `--network=bridge` or remove the flag entirely so Docker
  falls through to the default bridge mode).
- Config wins over env for `terminal.docker_extra_args` in the real
  `hermes -z --cli` path because the CLI/gateway bridge makes config
  authoritative. `HERMES_ALLOW_NETWORK=1` only lifts the forced air-gap; it
  does not override a config-pinned `--network=none`.
- When that opt-out is active, Hermes stops stripping `--network` / `--net`,
  so the config-selected network mode reaches `docker run`.
- Reuse now rejects a labeled container whose actual Docker network mode does
  not match the newly requested mode.

This is intentionally **not** the upstream-correct default. It preserves the
already-verified air-gap gate in this environment.

## Upstream-correct design

For `NousResearch/hermes-agent`, the cleaner design is the inverse default:

- Docker networking should be **config-driven**, not hardcoded in
  `DockerEnvironment.__init__`.
- Default behavior should be **normal Docker networking** (no unconditional
  `--network=none`).
- Air-gap should be an **explicit opt-in** config setting, e.g.
  `terminal.docker_network_mode: none` or `terminal.air_gap: true`.
- `terminal.docker_extra_args` should continue to bridge through CLI and
  gateway startup paths so config-requested `--network=...` and other raw
  `docker run` flags reliably reach the constructor.
- Reuse should stay **network-aware** so changing between `none`, default
  bridge, `host`, or a custom network cannot silently re-adopt a stale
  container created under a different mode.

## Concrete upstream changes

1. Add a first-class terminal config knob for Docker network mode.
2. Remove the unconditional `--network=none` force from
   `tools/environments/docker.py`.
3. Keep the CLI and gateway `docker_extra_args` config→env bridge fixed.
4. Keep reuse invalidation on network mismatch.
5. Cover both the explicit config knob and raw `docker_extra_args` network
   passthrough with tests.

## Draft PR title

`fix(docker): make network mode config-driven and prevent stale-network container reuse`

## Draft PR body

```md
## Summary

This fixes two Docker backend issues:

1. `terminal.docker_extra_args` was not consistently bridged through the CLI
   and gateway config→env startup paths, so config-supplied Docker network
   flags could be silently dropped before container creation.
2. Cross-process container reuse matched only on labels, so Hermes could
   re-adopt a container created under a different Docker network mode than the
   current request.

## What changed

- Wire `terminal.docker_extra_args` through both startup bridge maps
  (`cli.py` and `gateway/run.py`).
- Make Docker network selection config-driven instead of hardcoding
  `--network=none` in the constructor.
- Preserve user-requested `--network` / `--net` flags when networking is
  allowed.
- Probe a candidate container's actual `HostConfig.NetworkMode` before reuse
  and skip reuse when it differs from the newly requested mode.

## Why

The bridge bug meant `config.yaml` could request Docker extra args but real
agentic tasks still launched containers without them. The reuse bug meant a
network-mode change could silently inherit an older container's network
surface, which breaks both air-gap expectations and general Docker usability.

## Tests

- default network behavior
- config bridge coverage for `docker_extra_args`
- network-aware reuse rejection on mismatch
```

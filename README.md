# RuView — honest fork

> **Fork of [`ruvnet/RuView`](https://github.com/ruvnet/RuView), rewritten from verifiable facts only.**
> Every claim below points at a file in this tree. Numbers without a reproducer are labeled as such.
> Upstream's [`PROOF.md`](PROOF.md) (reproduce-or-retract grading) is kept as-is.
> Status: **research prototype. Not a product, not a medical device, not safety-certified.**

RuView is WiFi Channel State Information (CSI) sensing: ESP32 nodes capture CSI,
a Rust workspace (`v2/`, 67 crates) processes it into presence, vital-sign, and
count estimates, and a sensing server exposes them over HTTP/MQTT. Camera-free by
construction. Accuracy beyond the items in "What reproduces" is unvalidated.

## What reproduces today

```bash
bash scripts/prove.sh            # workspace gate + assertion tests (147 lines, read it first)
python archive/v1/data/proof/verify.py   # must print VERDICT: PASS (deterministic SHA-256 replay)
```

### Execution evidence (fork author, Windows + Rust 1.89 — ran, not read)

| Check | Result |
|-------|--------|
| `archive/v1` deterministic proof | **VERDICT: PASS**, bit-exact SHA-256 (243,200 feature bytes) |
| `cargo test -p wifi-densepose-vitals` | 102 passed + 1 doc-test, incl. synthetic-sine heartbeat detection and noise-is-never-valid |
| `cargo test -p wifi-densepose-signal` | 519 passed |
| `cargo test -p wifi-densepose-bfld` (+ `--test soul_match`) | 25 + 13 passed, incl. `cardiac_alone_cannot_separate_identity_matches_audit` |
| `cargo test -p cog-person-count` | 19 passed, incl. `untrained_class_argmax_is_flagged_low_confidence` |
| `cargo test -p cog-pose-estimation default_config` | `default_config_emits_frames_with_real_model` passed |
| Python bindings (wheel built locally with maturin, installed, pytest) | 20 passed (`python/tests/test_vitals.py`) |
| NOT run here | Firmware build (no ESP-IDF), full-workspace suite, hosted weight downloads |

`PROOF.md` grades each claim MEASURED (re-ran here), CLAIMED (cited, not reproduced),
or GATED (needs hardware/data/checkpoint not shipped). Trust the grade, not headlines.

## Implemented (with paths)

| Area | State | Evidence |
|------|-------|----------|
| Breathing DSP, 0.1–0.5 Hz bandpass | Implemented, on-body accuracy unvalidated | `v2/crates/wifi-densepose-vitals/src/breathing.rs` |
| Heart-rate DSP, 0.8–2.0 Hz bandpass | Implemented, experimental, not medical | `v2/crates/wifi-densepose-vitals/src/heartrate.rs` |
| Presence: trained head + model-free phase-variance path | Implemented; head accuracy per project docs only | `v2/crates/wifi-densepose-sensing-server/src/`, HF repo `ruvnet/wifi-densepose-pretrained` (external) |
| Person counting (dedup divisor default 3.0, runtime `/api/v1/config/dedup-factor`), fall pipeline (raw events + rolling-window risk score) | Wired end-to-end; cog falls back to single-person zero-confidence stub without trained weights; latencies unmeasured | `src/main.rs` (dedup endpoints), `src/semantic/fall_risk.rs`, `v2/crates/cog-person-count/src/inference.rs`, `fusion.rs` |
| Through-wall model (Fresnel-zone geometry) | Math model only; range unmeasured | `v2/crates/wifi-densepose-signal/src/fresnel.rs` |
| ESP32-S3/C6 firmware (edge DSP, CSI collection) | Real ESP-IDF; NDP frame still a TODO placeholder, mocks QEMU-only | `firmware/esp32-csi-node/main/edge_processing.c`, `csi_collector.c`, `mock_csi.c` (`CONFIG_CSI_MOCK_ENABLED`-gated) |
| Sensing server (`sensing-server` binary, v0.3.5) with `--mqtt` HA-DISCO publisher | Shipped | `v2/crates/wifi-densepose-sensing-server/src/cli.rs`, `src/mqtt/discovery.rs` |
| 8 Home Assistant blueprint files | On disk | `examples/ha-blueprints/*.yaml` (8 files) |
| Matter/HAP bridge (`cog-ha-matter`: mDNS, manifest, runtime sources) | Scaffolding; no documented working commissioned bridge | `v2/crates/cog-ha-matter/src/` (crate describes MQTT integration, ADR-116) |
| Python wheels `wifi-densepose` + `ruview` meta-package (PyO3/maturin) | Source present; build from `python/` | `python/pyproject.toml`, `python/ruview-meta/pyproject.toml`, `python/src/lib.rs` |
| Web UIs (Observatory, pose-fusion, pointcloud) | Work, but **default to labeled synthetic demo data**, not live RF | `ui/pose-fusion/js/csi-simulator.js` (`mode='demo'` default), `ui/observatory/js/demo-data.js` (`source:'demo'`) |
| CLI, MCP server, Claude/Codex plugin, MetaHarness package | Present; see their own READMEs for usage | `tools/ruview-cli/`, `tools/ruview-mcp/README.md`, `plugins/ruview/README.md`, `harness/ruview/README.md` |
| "105-cog catalog" | Remote registry, not vendored; 3 cog crates ship here | `docs/adr/ADR-102-edge-module-registry.md`, `v2/crates/cog-{pose-estimation,person-count,ha-matter}/` |

## Model weights: three tiers

| Tier | Checkpoint | Measured state |
|------|-----------|----------------|
| Published, external | `ruvnet/wifi-densepose-pretrained` (encoder + presence head) | Project-reported 82.3% held-out temporal-triplet accuracy; loader caveats documented (published `model.safetensors` header is NUL-padded — see `CHANGELOG.md`; `csi-embed-v2.safetensors` loads normally) |
| Committed, weak | `v2/crates/cog-pose-estimation/cog/artifacts/pose_v1.safetensors` | **PCK@20 3.0% / PCK@50 18.5%** on a 217-sample holdout (below the ≥35% target); runtime is a centred-skeleton stub returning `confidence: 0.0` (`src/inference.rs`, `cog/README.md`) |
| Architecture only | `archive/v1` `DensePoseHead` | Random init, zero checkpoint files, deprecated (`archive/v1/DEPRECATED.md`) |

Context: the repo's own MM-Fi study reports near-chance **cross-subject zero-shot**
transfer (~10%) vs high in-domain `random_split` scores
(`docs/benchmarks/mmfi-wifi-sensing-study.md`). Quote benchmark numbers with the protocol.

## Explicitly not established

On-body breathing/heart-rate accuracy · live single-ESP32 17-keypoint accuracy ·
through-wall range · multi-AP scaling math · Matter-commissioned bridge ·
person re-identification (project-measured: not separable on WiFi-only cardiac/
respiratory channels, gap ≈ 0.0005) · any medical, emergency, or safety use ·
exemption from GDPR-style rules for sensing identifiable persons (camera-free
removes video obligations only — assess locally).

## Build and run

Requires Rust 1.89 (`v2/rust-toolchain.toml`) and, for firmware, ESP-IDF.

```bash
git submodule update --init --recursive   # 10 submodules (ruvector, rvcsi, rufield, …);
                                          # without this, vendor/ and 4 v2/crates dirs are EMPTY
                                          # (verified: cargo fails workspace resolution on a fresh clone)
cd v2 && cargo test --workspace --no-default-features
```

| Goal | Pointer |
|------|---------|
| Firmware build/flash | `firmware/esp32-csi-node/README.md` (S3 + C6 targets) |
| Python wheel build | `python/README.md` (maturin; extras: `aether`, `mat`) |
| Docker images | `docker/Dockerfile.rust`, `docker/Dockerfile.python`, `docker/docker-compose.yml` |
| Calibration/room training | `docs/calibration-guide.md` |
| Home Assistant setup | `docs/integrations/home-assistant.md`, ADR-115 |
| Benchmarks/methods | `docs/benchmarks/`, `docs/adr/` (258 ADRs), `docs/WITNESS-LOG-028.md` |

## Repo map

`v2/crates/` (Rust workspace) · `firmware/esp32-csi-node/` · `python/` ·
`ui/` (demos, synthetic default) · `tools/` · `scripts/` (validators, `prove.sh`) ·
`harness/` · `plugins/` · `docs/` · `examples/` · `archive/v1/` (deprecated) ·
`aether-arena/` (benchmark harness; board empty pending real runs).

33 CI workflows in `.github/workflows/`; firmware matrix is strict, legacy
archive jobs are non-gating by design (`ci.yml`).

## License and support

MIT — see [`LICENSE`](LICENSE). Bugs and questions: GitHub Issues on your fork
(upstream: `ruvnet/RuView`).

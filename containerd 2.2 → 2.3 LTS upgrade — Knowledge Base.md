# containerd 2.2 → 2.3 LTS upgrade — Knowledge Base

Oct 27, 2026 · @Boass

## TL;DR

Every node in the fleet moves from **containerd 2.2.1 to 2.3.x LTS**, in place, with Ansible, one node at a time per cluster.

- **Why:** containerd 2.2 is end of life on **6 November 2026**; 2.3 is the first annual LTS, supported until **30 April 2028**.
- **Scope:** all 600+ clusters, every node, including the management clusters hosting the Kamaji tenant control planes.
- **Baseline:** Kubernetes 1.35, Ubuntu 22.04, kernel 5.15 GA, upstream containerd tarballs.
- **Order:** lab → dev → staging → prod, each environment gated by a soak.
- **Gap:** production finishes after 6 Nov; that period is covered by a documented SecNumCloud risk acceptance.

The full plan lives in `prd.md`; the lab test kit in `containerd-deepdive.tar.gz`.

## Context

2.3 is the only target that is both a supported upgrade from 2.2 and covers the next two Kubernetes minors. Upstream supports upgrades to the next minor, or LTS to LTS, so 2.2 → 2.4 is not a supported path.

| containerd | Status | End of life | Kubernetes versions recommended |
| --- | --- | --- | --- |
| 2.4 | Active, non-LTS | 16 May 2027 | 1.37 |
| **2.3** | **LTS (target)** | **30 Apr 2028** | **1.36, 1.37** |
| 2.2 (current, 2.2.1) | Active | 6 Nov 2026 | 1.35, 1.36 |
| 1.7 | End of life | 30 Sep 2026 | 1.35 is the last minor to support 1.x |

Source: [containerd RELEASES.md](https://github.com/containerd/containerd/blob/main/RELEASES.md) (Kubernetes support matrix).

- The fleet runs 2.2.1 while the 2.2 branch is at 2.2.9: eight patch releases, security fixes included, are already missing.
- Ubuntu 22.04 standard support ends in April 2027; 2.3 runs on both 22.04 and 24.04.
- The upstream `containerd-<ver>-linux-amd64.tar.gz` is dynamically linked against glibc 2.35, which is what Ubuntu 22.04 ships.

## Key decisions

| Topic | Decision | Why |
| --- | --- | --- |
| Target | 2.3.x LTS, latest patch at freeze, runc version pinned by the 2.3 tag | Supported path from 2.2, support to Apr 2028 |
| Method | In place via Ansible, upstream binaries from the internal artifact store | No extra OpenStack quota needed |
| Parallelism | 1 node at a time per cluster, many clusters in parallel | Smallest blast radius per cluster |
| Order | Lab → dev → staging → prod | Existing promotion model |
| Config file | Stays `version = 3` on disk during rollout | 2.2 cannot read a v4 file, so rollback would break |
| Registry config | Keep certs.d + legacy `mirrors`/`configs` for now; migrate later | Legacy keys are only removed in containerd 2.7 |
| Rollback | Reinstall 2.2.x binaries and config; wipe state by default | Storage formats are not guaranteed across a minor downgrade |
| Windows | Agreed maintenance windows per tenant | Tenant commitments |
| EOL gap | SecNumCloud risk acceptance, full test depth kept | Safety over the date at fleet scale |

## What changes from 2.2 to 2.3

The delta is small; NRI is the only item flagged as breaking, and whether the fleet uses NRI is still being confirmed.

| Area | Change in 2.3 | What we do |
| --- | --- | --- |
| NRI | Plugin names may no longer contain commas; NRI v0.1.0 plugins are no longer supported | Inventory NRI clients on every node; test each agent version in the lab |
| Config | New config version 4; older files are migrated in memory at startup | Keep the v3 file; check startup logs; switch to v4 only after 100% rollout |
| OOM | OOMKilled event handling reworked | OOM tests: plain container, restart loop, init container |
| Stats | Background collector computes `usageNanoCores` for containers and sandboxes | Compare `kubectl top`, HPA input and eviction with the 2.2 baseline |
| CRI | Container status carries `ImageId`; sandbox annotations passed | Check security and ops tools that parse CRI output |
| Shim | New shim bootstrap protocol; `shim.Command` deprecated | Test that running containers survive the daemon swap |
| Deprecated, removed in 2.4 | `enable_cdi`, `cni.bin_dir`, OTLP/tracing keys | Remove them from the template now |

Source: [containerd 2.3.0 release notes](https://github.com/containerd/containerd/releases/tag/v2.3.0).

## Per-node upgrade procedure

The Ansible role runs these steps on one node at a time; a failed step stops that node and leaves it cordoned.

1. **Pre-checks:** node Ready, PDBs allow eviction, free space on `/var/lib/containerd`, internal registries reachable.
2. **Back up** containerd, shim, ctr and runc binaries, the systemd unit and `/etc/containerd` to `/var/backups/containerd-2.2/`.
3. **Cordon and drain:** `kubectl drain --ignore-daemonsets --delete-emptydir-data --timeout=<t>`. Never force; skip the node on timeout.
4. **Stop** kubelet, then containerd.
5. **Install** the pinned containerd 2.3.x tarball and runc; refresh the systemd unit from the release.
6. **Config:** keep `version = 3`, apply only the template clean-ups; `containerd config dump` must succeed.
7. **Start** containerd, run the node checks, then start kubelet.
8. **Verify:** node Ready, DaemonSets (Calico, security agents) Running, smoke pod pulls from each registry family.
9. **Uncordon** and record versions, timings and check results in the wave report.

Inside a cluster: workers pool by pool, control-plane nodes last, etcd health checked between each control-plane node. On management clusters, check that Kamaji tenant control-plane pods reschedule and tenant API servers stay reachable during each drain; their etcd is slow, so use generous drain timeouts.

## Validation: the deep-dive kit

Every test runs twice on the same lab node, on 2.2.1 and then on 2.3.x, and `compare.sh` puts the two runs side by side. A difference is not automatically a failure, but each one must be explained before the lab gate. Lab clusters only: the tests pin pods to nodes, use a privileged namespace, trigger OOMs and restart daemons.

| Script | Runs on | What it proves |
| --- | --- | --- |
| `node/00-collect-snapshot.sh <label>` | Node, root | Versions, checksums, redacted config, deprecations, `runc features`, cgroup/kernel/AppArmor state, NRI use, socket consumers |
| `node/01-post-upgrade-checks.sh` | Node, root | Expected versions, config still v3, CRI ready, systemd cgroup driver, pinned pause image, pulls per registry, no leaked shims or mounts |
| `node/02-live-restore.sh snapshot\|verify` | Node, root | Containers started by 2.2.1 keep the same PID across the swap and stay manageable by 2.3 |
| `node/03-lab-upgrade.sh upgrade\|rollback [--wipe]` | Node, root, `LAB=1` | Checksum-verified swap and the rollback, timed |
| `node/04-critest.sh` | Drained node, root | CRI conformance (cri-tools 1.35) |
| `node/05-sampler.sh` | Node, root | CSV of containerd and shim memory, threads, FDs, mounts for leak detection |
| `node/06-kernel-log-scan.sh` | Node, root | Kernel bugs, hung tasks, overlayfs errors, AppArmor and seccomp denials |
| `cluster/10-integration-tests.sh` | Workstation | cgroups, OOM, in-place resize, security context, userns, exec/logs/port-forward/cp, debug, pulls, stats, sidecars and probes, restart survival |
| `cluster/20-conformance.sh node\|full` | Workstation | Sonobuoy node conformance or full conformance |
| `cluster/30-churn.sh` | Workstation | Pod churn with start-latency percentiles |
| `compare.sh <baseline> <after>` | Anywhere | Recorded values and PASS/FAIL changes side by side |

### Run it on one lab node

```bash
export NODE=lab-w1 IMAGE=registry.internal/lib/busybox:1.37
export PULL_TEST_IMAGES="certsd-mirror.internal/lib/busybox:1.37 legacy-mirror.internal/lib/busybox:1.37"

# 1. Baseline on 2.2.1
sudo node/00-collect-snapshot.sh baseline-2.2.1
sudo RESULTS_DIR=results/lab-w1-baseline node/01-post-upgrade-checks.sh
RESULTS_DIR=results/lab-w1-baseline cluster/10-integration-tests.sh all

# 2. Live restore across the swap (no drain, lab only)
cluster/10-integration-tests.sh setup-live-restore
sudo node/02-live-restore.sh snapshot
sudo LAB=1 node/03-lab-upgrade.sh upgrade /srv/artifacts/containerd-2.3.6
sudo node/02-live-restore.sh verify

# 3. Same suite on 2.3.x
sudo node/00-collect-snapshot.sh after-2.3.6
sudo RESULTS_DIR=results/lab-w1-after EXPECT_CONTAINERD=2.3.6 node/01-post-upgrade-checks.sh
RESULTS_DIR=results/lab-w1-after cluster/10-integration-tests.sh all restart-survival
sudo node/06-kernel-log-scan.sh

# 4. Compare
./compare.sh results/lab-w1-baseline results/lab-w1-after
```

### Values to watch in the comparison

- `cg_cpu_weight_250m`: the requests-to-`cpu.weight` conversion depends on the runc version; a change shifts CPU sharing between pods with no error.
- `oom_events`, `oom_loop_restarts_45s`: OOM reasons and counts must match the baseline.
- `stats_burner_usageNanoCores`, `top_burner`: feed `kubectl top`, HPA and eviction.
- `userns_result`: expected `rejected` on kernel 5.15, with no half-started container.
- `deprecation_warnings`, `journal_migration_lines`: any new deprecation is fixed before production.
- `nri_clients`: every NRI agent must still register on 2.3.
- Churn p95 start latency and sampler trends: flat over 24 h on both versions.

**Lab exit:** all suites green, every delta explained, rollback rehearsed twice (with and without `--wipe`).

## Rollout and wave gates

&#91;embedded content: rollout · 5 phases, 4 gates\]

Each environment opens only when the previous gate is passed; production cannot start before 2.2 reaches end of life, hence the risk acceptance. Dates are the draft plan from `prd.md`.

A wave stops automatically when:

- a node fails its post-checks twice (that cluster stops);
- `ErrImagePull`/`ImagePullBackOff`, `CrashLoopBackOff` or `OOMKilled` counts rise above the pre-wave baseline (the wave stops);
- a tenant incident is attributed to the runtime (all waves stop, post-mortem).

## Rollback

A node goes back to 2.2.x by restoring the backup and, by default, wiping containerd state; this only works because the config file stayed at `version = 3`.

1. Drain the node.
2. Stop kubelet and containerd.
3. Restore the 2.2.x binaries, runc, systemd unit and `/etc/containerd` from `/var/backups/containerd-2.2/`.
4. Wipe `/var/lib/containerd` and `/run/containerd`. Upstream does not guarantee storage formats across a minor downgrade; drop this step only if the lab rehearsal proves a downgrade without wiping is clean.
5. Start containerd, run the node checks, start kubelet, uncordon.

In the lab, `sudo LAB=1 node/03-lab-upgrade.sh rollback --wipe` performs these steps.

- 2.2 is end of life from 6 Nov, so a rolled-back node is temporary and gets a ticket to re-upgrade.
- A wiped node re-pulls all its images; throttle concurrent rollbacks to protect the internal registries.

## Troubleshooting and FAQ

| Symptom or question | Likely cause | What to do |
| --- | --- | --- |
| Security or resource agent stops working after the swap | NRI plugin uses the v0.1.0 API or has a comma in its name | `journalctl -u containerd \| grep -i nri`; update the agent or roll the node back |
| Pods stuck in `ErrImagePull` | Registry entry not honoured (certs.d or legacy mirror) | `crictl pull <image>` on the node; compare `/etc/containerd/certs.d` and the legacy keys with the snapshot |
| Startup logs mention config migration | Expected: 2.3 migrates the v3 file to v4 in memory | Nothing; do not rewrite the file to v4 before rollout ends |
| `ctr deprecations list` shows new entries | Keys deprecated in 2.2/2.3 still in the template | Fix the template before the next environment |
| Pod with `hostUsers: false` will not start | Kernel 5.15 lacks idmapped mounts on overlayfs (needs 6.3+) | Expected on our fleet; tell the tenant user namespaces are not available |
| `kubectl top` or HPA values differ from before | New stats collector in 2.3 | Compare with the lab baseline; report if outside the explained delta |
| Node stuck draining | PDB blocks eviction, or slow Kamaji tenant etcd | Never force; skip the node and contact the workload owner |
| Why not go straight to 2.4? | 2.2 → 2.4 is not a supported path, and 2.4 is non-LTS (EOL May 2027) | Stay on 2.3 until the next LTS |
| Can I use `ctr` in scripts? | `ctr` is an unsupported debug tool upstream; it can change between minors | Prefer `crictl` in runbooks and automation |

## Compliance evidence (SecNumCloud)

Each wave leaves an evidence trail that goes with its change record.

- Artifacts come only from the internal store, with SHA256 checksums and signatures verified at ingestion.
- Before/after snapshots (`snapshots/`) and test results (`results/`); registry credentials are redacted by the kit.
- One change record per wave, plus the weekly burndown until 100%.
- The risk acceptance for the post-6 Nov gap: scope, committed end date, daily containerd/runc advisory watch, fast-track path for a critical CVE on 2.2.
- A vulnerability scan of the containerd and runc binaries at freeze and at every patch bump.

## Open questions

- Is NRI used anywhere in the fleet, and by which agents?
- Who owns the risk acceptance, and what is the committed end date?
- Follow-ups: migrate legacy registry keys to certs.d; plan Ubuntu 22.04 → 24.04 before April 2027.

## Sources

- [containerd RELEASES.md](https://github.com/containerd/containerd/blob/main/RELEASES.md): support dates, Kubernetes matrix, upgrade path, config versions
- [containerd 2.3.0 release notes](https://github.com/containerd/containerd/releases/tag/v2.3.0): changes, breaking items, release assets

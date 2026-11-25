# AGENTS.md: openshift/google-cadvisor

OpenShift's fork of [google/cadvisor](https://github.com/google/cadvisor). Kubelet in
openshift/kubernetes links cadvisor in as a library. This fork exists only to carry
fixes into an OCP release before the upstream version that contains them is picked up.

## Read this first

- **`master` is a stale upstream mirror** (cadvisor v0.47.0+31, May 2023). Do not branch from it
  or send code PRs to it. A "rebase" PR against `master` (#31) was closed and replaced by a
  release branch.
- **All real work happens on `openshift-<ocp>-cadvisor-<upstream-version>` branches.** Each one
  is cut from the upstream tag and holds only a few carry commits on top. Snapshot as of
  Nov 2026; the `go.mod` check in step 2 below is the source of truth:

  | Branch | Base | Consumed by openshift/kubernetes |
  |---|---|---|
  | `openshift-5.1-cadvisor-0.60.5` | `v0.60.5` / `lib/v0.60.5` | `master`, `release-5.1` (tag `lib/v0.60.5-openshift-5.1-fix`) |
  | `openshift-4.21-cadvisor-0.52.1` | `v0.52.1` | `release-4.21` (tag `v0.52.1-openshift-4.21-1`) |
  | `openshift-4.17-cadvisor-v0.49.0` | `v0.49.0` | `release-4.17` (`-4.17-3`), `release-4.18` (`-4.17-4`) |
  | `openshift-4.16-cadvisor-v0.48.1` | `v0.48.1` | `release-4.16` (tag `v0.48.1-openshift-4.16-2`) |

  Branch names are inconsistent: older ones use a `v` before the version and newer ones do not,
  and there is a typo in `openshift-4.8-cdavisor-v0.39.2`. Run `git branch -r` instead of
  guessing a name.
- **Many OCP releases have no fork branch.** `release-4.19`, `4.20`, `4.22` and `5.0` of
  openshift/kubernetes use upstream cadvisor directly. A fork branch is created only when a
  carry is needed. One branch can serve several OCP releases (4.18 uses the 4.17 branch).
- To see what OpenShift carries on a branch, run
  `git fetch --tags https://github.com/google/cadvisor`, then
  `git log --oneline v0.52.1..origin/openshift-4.21-cadvisor-0.52.1`.

## Fix workflow (upstream first)

1. **Land the fix in google/cadvisor first.** Every recent carry is a cherry-pick of an upstream
   PR: #3541 (hung fs), #3522 (hidden devices), #3756/#3762 (lock contention), #3944 (StatsEq).
   Fork-only changes have not been merged: #28 was held for upstream, and #33 was closed.
2. **Find the branch** for the target OCP release. Check the `replace` line in
   openshift/kubernetes `go.mod` on that release branch:
   `gh api "repos/openshift/kubernetes/contents/go.mod?ref=release-4.21" --jq .content | base64 -d | grep cadvisor`.
   If there is no `replace` line, the release uses upstream and a maintainer has to create a
   fork branch from the matching upstream tag first. Ask; do not create it yourself.
3. **Cherry-pick** onto that branch with `git cherry-pick -x <upstream-sha>` and open a PR against
   it. PR title: `OCPBUGS-NNNNN: <summary>`, which the Jira bot validates. Put links to the
   upstream PRs and the Jira bug in the body. Include before/after numbers for performance
   changes (#32 is a good example). Older PRs used `UPSTREAM: google/cadvisor: <PR#>: …`, which
   is still acceptable.
4. **Validate locally.** No CI runs on this fork: PRs have no status checks, and GitHub Actions
   has never run here. Real validation happens in openshift/kubernetes CI after the bump.
5. **A maintainer merges and pushes a tag**, named `v<upstream>-openshift-<ocp>-<n>` and
   incremented for each new carry. Modules in subdirectories need the directory prefix, e.g.
   `lib/v0.60.5-openshift-5.1-fix`. Agents must not push tags or branches.
6. **Bump openshift/kubernetes.** Open a PR that edits the `replace` directive,
   regenerates vendor (o/k#2530 used `make update`), and commits `go.mod`, `go.sum`,
   `vendor/modules.txt` and `vendor/github.com/google/cadvisor/...`. Title it
   `OCPBUGS-NNNNN: UPSTREAM: <carry>: bump cadvisor …` (o/k#2819, #2325). Use `<drop>` instead
   of `<carry>` if the next k8s rebase should remove the pin (o/k#2530).
   - The fork tag is a semver *prerelease* of the upstream version, so it sorts *below* it.
     That is why o/k keeps `require github.com/google/cadvisor[/lib] vX.Y.Z` and redirects with
     `replace`. Never put a fork tag on the `require` line alone.

Prow commands work on PRs (`/hold`, `/close`, `/lifecycle`), but this repo has no OWNERS file.
Maintainers merge with the GitHub merge button.

## Code map (verify on the branch you are touching)

Starting with **v0.60 (the 5.1 branch), the kubelet-facing code lives in a separate Go module
`github.com/google/cadvisor/lib`** under `lib/`. Kubelet on o/k `master` vendors only `lib/...`.
The root module (`cmd/`, `info/`, docker handler, `client/`, `perf/`, `resctrl/` …) still exists:
`info/v1` is now a set of type aliases to `lib/model`. On branches before v0.60 the same
packages sit at the repo root without the `lib/` prefix.

| Area | v0.60+ (`lib/`) | ≤ v0.52 (root) |
|---|---|---|
| Public stats/spec types kubelet consumes (`ContainerStats`, `MemoryStats`, `MachineInfo`, `StatsEq`) | `lib/model/` | `info/v1/`, `info/v2/` |
| Manager: container tree, housekeeping loop, subcontainer lookup | `lib/manager/` (`manager.go`, `container.go`) | `manager/` |
| In-memory stats cache | `lib/cache/memory/` | `cache/memory/` |
| cgroup v1/v2 stats reading (via `opencontainers/cgroups`) | `lib/container/libcontainer/` | `container/libcontainer/` |
| Raw cgroup handler (pod/QoS/system slices) and watcher | `lib/container/raw/` | `container/raw/` |
| CRI-O / containerd / systemd factories | `lib/container/{crio,containerd,systemd}/` | `container/...` |
| Shared handler helpers, fs usage handler, cgroup test fixtures | `lib/container/common/` | `container/common/` |
| Filesystem info, `GetDirUsage`, device maps | `lib/fs/fs.go`; statfs in `lib/fs/vfs/stats.go` | `fs/fs.go` |
| Machine/topology/sysfs parsing | `lib/machine/`, `lib/utils/sysfs/` | `machine/`, `utils/sysfs/` |
| Prometheus metrics | `lib/metrics/` | `metrics/` |

Changing an exported type or signature in `lib/model` / `info/v1` breaks the kubelet build.
Grep openshift/kubernetes `pkg/kubelet/cadvisor` and `pkg/kubelet/stats` before you change it.

## Build, test, lint

The Makefile is upstream's. Targets differ by branch, so check the branch's `Makefile`.

- `make test`: runs `go test -race` on all packages except `integration/`, then in `cmd/`, and on
  v0.60+ also in `lib/`. Each Go module has to be tested from its own directory.
- `make presubmit`: runs golangci-lint (v2.6.2 on 5.1, v1.64.5 on 4.21; the Makefile
  auto-installs it), `build/check_gotidy.sh` (go mod tidy across `lib`, `.`, `cmd`) and
  `build/check_boilerplate.sh` (Apache header with "Google Inc." copyright is required on new
  files).
- `make tidy` / `make format` (goimports with `-local github.com/google/cadvisor`).
- **Linux-only:** about 49 files under `lib/` have `//go:build linux`. On macOS,
  `go test ./manager/` fails to compile. Use `GOOS=linux go vet ./...` for a quick check, or
  `make container-test` (docker or podman, golang image) to actually run the tests.
- Optional build tags: `libpfm` (`make test-with-libpfm`), `libipmctl`; see `build/config/*.sh`.
- `make docker-test-integration` / `docker-test-integration-crio` start a privileged container
  that runs a real cadvisor against docker or CRI-O. `make test-integration` on the host may
  need sudo. These are rarely needed for carry PRs.
- Benchmarks for the manager lock changes are in `manager_bench_test.go`. Run them with
  `cd lib && go test -bench . ./manager/` (or `./manager/` from the repo root on 4.21).

## Gotchas from real OpenShift bugs

- **Never block the kubelet on a filesystem syscall.** A hung NFS or other mount used to make
  `statfs` hang and kubelet got stuck on restart (OCPBUGS-48834, upstream #3541). statfs now
  runs in a goroutine with a 2s timeout (`getVfsStats` in `fs/fs.go`; `GetVfsStats` in
  `lib/fs/vfs/stats.go` on v0.60+). Keep any new statfs, stat or walk path similarly
  bounded. `GetDirUsage` walks the whole tree with no timeout and is limited only by a pool of
  20 tokens (`maxConcurrentOps`). A timeout for it was proposed (#33) and closed unmerged.
- **Lock contention in `manager` shows up as kubelet API latency** (OCPBUGS-57665). `containersLock`
  became a `sync.Map`, timestamps became `atomic.Int64`, and the cache lock was removed (#32,
  P99 10s to 1.7s). Do not reintroduce wide locks in the `GetContainerInfoV2` / `getSubcontainers`
  paths.
- **`StatsEq` controls housekeeping backoff.** With cgroup v2 `memory.high` (MemoryQoS), monotonic
  counters such as `Pgscan`, `Pgsteal`, `WorkingsetRefault*` and `Events.High` change on every
  read. A `reflect.DeepEqual` comparison then kept housekeeping pinned at 1s, which caused about
  40% extra allocations in kubelet (OCPBUGS-129642, upstream #3944, commit 37731167). If you add
  a counter field to `MemoryStats`, decide whether `memoryStatsUsageEq` should compare it.
- **CRI-O pod cgroups are not systemd scopes** (CRI-O cannot make a scope without a process in
  it). Filtering raw/systemd cgroups by scope dropped pod network metrics, so the filter was
  reverted on 4.17 (OCPBUGS-38515, #27).
- **Multipath/NVMe hidden devices** (`/sys/block/nvme0c0n1`) produced `device=""` fs metrics
  until the sysfs hidden check trimmed whitespace (#30, upstream #3522).
- Keep carries cherry-picked unchanged from upstream so the next rebase onto a newer upstream
  tag can drop them cleanly.

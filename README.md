# rulego-fleet

The RuleGo ecosystem ships as a **composition**: a core runtime image, an ABI
contract it was built against, and a set of plugins loaded into it at
runtime. No single repository releases that composition - each member ships
its own bytes - so the composition itself had no identity. This repository
is that identity: it **names the set** (a manifest) and **proves the
composition proposition** at the named coordinates.

## The proposition

> For the member set declared in a manifest, loaded together into the pinned
> runtime image, the process starts and stays up cleanly, every declared node
> type and response processor registers, and no shared Go package appears at
> two different versions in one process.

That is the only kind of statement this repository makes. A sentence that
goes green on the bytes of exactly one member repository is a member's
statement and does not belong here; a sentence that needs bytes from two or
more coordinates belongs here.

## Boundary

| in scope | out of scope |
| --- | --- |
| assertions over the set: clean startup, co-load, co-register, shared-package identity | any single-member assertion (member gates: own bytes, own sidecar, own registration, own unit tests) |
| the manifest: which coordinates form the current set, per arch | the ABI contract itself (`plugin-abi.lock`, owned by `rulego-server-docker`) |
| readback of the running process (`/api/v1/components`, `go version -m`, startup logs) | building or signing shipping bytes; publishing any release |
| re-verification of a batch after migration; onboarding a new member; re-running the set on a runtime upgrade | member repository changes of any kind; core repository changes |
|  | per-plugin functional and flow behavior (member e2e covers it; for the v1.0.0 batch, `rulego-engineering` task 10-20 step 4.3) |

## Position

This repository depends on every member and on the contract and runtime
images; **nothing in a release path depends on it.** No member `release.yml`,
no member CI job, no core workflow reads a run of this repository. The
consumers of its evidence are the batch dispatcher (a human approval
checklist item: the latest fleet run over this batch's set is green) and the
next manifest's author (who diffs against it).

Consequence, stated plainly: a red fleet run never blocks a member's
release. It blocks the *next* dispatch decision - the cost of placing the
set's evidence where the set, not a member, is the subject.

## Manifest

`manifest.json` at the repository root, schema `fleet-manifest/v1`. There is
exactly one manifest: it describes the **current composition**. The
composition changes (a member joins, a member moves versions, the runtime
image moves) by editing that one file; its history is the git history of this
repository, and re-verifying an earlier composition is a checkout of an
earlier commit of this repository - same model as a lock file, and the same
shape as the ecosystem's own per-repo `plugin-abi-release.json`.

- `abi`: `abi_id` + `lock_digest`. Every member's published sidecar must
  carry exactly these values; this is the contract boundary of the set.
- `runtime`: `image` (digest-pinned `-plugin` runtime) + `rulegoRelease`
  (the `io.rulego.server.release` label the image must carry).
- `sdk`: digest-pinned build image, consumed only by `source: "build"`
  members.
- `members[]`: one entry per member with `repo`, `artifact` (with an
  `{arch}` placeholder), and a `source`:
  - `release`: `tag` + per-arch `sha256` of the published `.so`. The run
    downloads and verifies against those values; a mismatch is a run failure.
  - `build`: `commit` (must be reachable on the member's origin) + `path`
    (where the Go module root sits in that repository, `.` for root). The run
    checks out the commit and builds with the pinned SDK. This is a
    *transient* state: it exists only while the pinned member's release does
    not, and the run marks every built member as such in its summary. The
    migration to `release` is the manifest edit that retires the last one.
- `shared_packages[]`: module paths embedded by two or more members; for
  each, the `dep` line of `go version -m` must be byte-identical across all
  members that carry it.
- `expected`: `nodes[]` + `outProcessors[]` - the registration readback.

## Verification

One job, one architecture at a time (matrix: amd64, arm64):

1. **Acquire** - per member, per source kind; `release` verifies the
   published `.sha256` and sidecar; `build` verifies the freshly built
   sidecar's own fields (platform, its own bytes).
2. **Pin the runtime** - pull by digest; assert the `rulegoRelease` label.
3. **Load** - install all members' `.so` into the runtime's data directory
   and start it; wait for `/api/v1/components`.
4. **Read back** - `expected` assertion on the component registry; startup
   log free of package-identity / plugin-load errors.
5. **Identity** - `go version -m` on every loaded member; shared-package
   lines equal across members.

Steps 3-5 run the proposition. Steps 1-2 make its inputs what the manifest
says they are.

## Provenance

Created 2026-10-24, out of the RuleGo v1.0.0 multi-repo upgrade
(`rulego-engineering` task `10-20-rulego-v1-upgrade`): the co-existence
assertion had lived inside two members' release gates, each downloading
already-published siblings, and a synchronized ABI migration deadlocked on
exactly that cycle. The decision record - options, the member/composition
split, the rejected candidates, and the boundary ruling for this
repository - is in that task's
`research/coexistence-gate-verdict.md`.

## License

MIT - see [LICENSE](LICENSE).

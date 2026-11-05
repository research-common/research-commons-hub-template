# What this template needs from the `commons` tool

> **Contract.** This file states what `.github/workflows/hub-check.yml` and `README.md` assume from the
> `commons` tool, first met by its `v0.5.0-alpha.1` release. Where the tool's own documentation and this file
> disagree after a release, the template is wrong and gets fixed. Section references are to the tool's design
> notes (`docs/DESIGN-notes-*.md`) and conformance cases to `docs/CONFORMANCE.md` in the tool repository.

Each requirement has an id (`H-nn`) so a test in the tool can name the one it checks.

## 1. Build

| id | Requirement |
|---|---|
| H-01 | The repository builds a single static binary with `CGO_ENABLED=0 go build -trimpath -o <out> ./cmd/commons`, using only modules pinned by its `go.sum`. |
| H-02 | `commons --version` exits 0 and prints the tool version and the commit it was built from (one `name: value` line each; with `--json`, one object with `version` and `commit`). An unstamped build reports the release's version, not a placeholder. |
| H-03 | `commons hub check` uses the `git` executable present on the machine to read revisions (`git cat-file`, `git diff --raw -z`, `git merge-base`). It reads objects as raw bytes: no text conversion, no `textconv`, no filters, no attributes or configuration taken from the checked tree. No hook runs. |

## 2. `commons hub check`

Run from inside a hub checkout (the working directory or any directory below the `rc-store` marker).

```text
commons hub check [--base <rev>] [--json]
```

### 2.1 What it reads

| id | Requirement |
|---|---|
| H-10 | **Hub root.** The nearest ancestor directory holding `rc-store`. A missing marker, or a marker naming a layout this tool does not implement, is exit 2 before any work. |
| H-11 | **With `--base <rev>`:** `<rev>` must resolve to a commit in this repository, else exit 2. The change under check is every path that differs between `merge-base(<rev>, HEAD)` and `HEAD`. The statement set the check evaluates is the tree at `<rev>` plus the statements the change adds: what merging the change would hold. |
| H-12 | **Policy from the base.** With `--base`, the hub policy is `hub-policy.json` **as it is at `<rev>`**, read from the git object database, never from the working tree or from `HEAD` (claims-model §11; C-core-115). Without `--base`, it is the file at `HEAD`, so the template's workflow never runs a pull request's check without `--base`: when the base commit is unavailable it fails the job with exit 2 rather than judge the change under the policy it carries (a push to `main` may run without `--base`, since `HEAD` is then `main`'s own policy). An absent file means the documented defaults. A file that is present and unparseable, or that carries an undefined field, is exit 2 before any work (§13; C-core-133a). The output names the revision the policy came from and its digest. |
| H-13 | **No other input.** Nothing in the checked tree configures the check: no environment file, no tool configuration, no trust list. Signer trust is not an input (§2.4 below). |
| H-14 | **No side effects.** The check writes nothing in the hub, installs nothing, keeps no cache that a later run reads, and makes no network request. Locations in `holds` statements are never dereferenced. The output says so in one line (`no registry change; no fetch; no cache`; §13 "dry runs state their side effects"; C-core-137). |

### 2.2 What it checks

Every object is checked by the tool's one validation core, the same function `pull`, `push` and the
pre-pin check call (claims-model §11; C-core-109). The check evaluates **every** object in the change and
reports each one; it never stops at the first refusal.

The **data surface** of a hub is exactly:

- `blobs/sha256/<64 lowercase hex>`, the blob whose sha256 is that name;
- `statements/<signer-dir>/<64 lowercase hex>`, one statement whose id is that name, where
  `<signer-dir>` is the statement's `signer` with `:` replaced by `-` (`ed25519-<64 hex>` or `0x<40 hex>`).
  The file holds exactly the statement's canonical bytes (wire note §1, §2, `sig` included), with no
  trailing newline. These are the paths the tool's local store uses, so a hub clone is also a working store.

The **control plane** is every other path the template ships: `hub-policy.json`, `rc-store`, `README.md`,
`LICENSE`, `CONTRIBUTING.md`, `docs/`, `.github/`, `.gitignore`, `.gitattributes`, plus the paths the base
revision's `hub-policy.json` lists in `control_plane` (a maintainer's own root files, such as `CITATION.cff`).

| id | Situation in the change | Verdict | Class · reason · scope | Cases |
|---|---|---|---|---|
| H-20 | added statement that fails to parse, is not canonical, fails its signature, or whose id differs from its file name | refused | intrinsic · `malformed statement` / `bad signature` / `hash mismatch` · that file | C-core-09, C-core-12, C-core-110 |
| H-21 | added statement under another key's directory | refused | intrinsic · `line in another key's log` · that file | C-core-14, C-core-110 |
| H-22 | added statement over the single-statement size bound, or any object over a `limits` entry | refused | operational · `size limit <n>` (name and value) · that object | C-core-112 |
| H-23 | added statement of an `rc` major, `alg` or `crit` extension the tool does not implement | admitted | labelled `unsupported`; placed in its signer's log by the linkage contract. Its file name must still equal its id (computable in every major, wire §6); its directory cannot be checked against a signature the tool cannot verify, so it is attributed to nobody until a verified line of that key names it | wire §6 |
| H-24 | added statement whose predecessor (or a `merges` entry) is not in the evaluated set | admitted | labelled `detached: waiting for …`, the gap named; nothing refused for being detached | C-core-143 |
| H-25 | added statement forking its signer's log, or with an incoherent `ts` | admitted | labelled `FORK` / `incoherent`; a fork is never a refusal | C-core-106, C-core-127 |
| H-26 | added blob whose bytes do not hash to its name | refused | intrinsic · `hash mismatch: expected <h1>, got <h2>` · that file | C-core-110, C-core-117 |
| H-27 | added blob while the policy says `blobs: external-only`, or whose digest is in `not_served` | refused | reader policy · `hub rules: <field>` · that file | C-core-111 |
| H-28 | added blob no statement in the evaluated set references | admitted | labelled `unreferenced` | storage §1.1 |
| H-29 | statement referencing a blob the hub does not hold | admitted | labelled `unavailable (not held locally)`; never invalid | storage §6.2, §9; C-core-119, C-core-120 |
| H-30 | any data-surface file deleted, modified or renamed, except deleting a blob whose digest is in the base policy's `not_served` | refused | intrinsic · `held object changed` · that path | wire §7 (append-only) |
| H-31 | a path that is neither data surface nor control plane (including `..`, absolute or symlinked paths, files at the wrong depth, upper-case hex) | refused | intrinsic · `outside the data surface` · that path | C-core-110, C-core-114 |
| H-32 | a control-plane path changed **together with** any data-surface change | the control-plane path is refused | intrinsic · `outside the data surface` · that path; the data objects are still checked, and checked under the base policy | C-core-114, C-core-115 |
| H-33 | control-plane paths changed and **no** data-surface path | nothing to admit | reported as `control-plane change: needs maintainer review`, every path listed; never applied by the check | C-core-115 |
| H-34 | `series.refuse_premature_freeze` is on and an added series edition declares `frozen_before` later than its members' computed boundary | refused | reader policy · `hub rules: series.refuse_premature_freeze` · that edition | C-core-104 |
| H-34a | `selection.refuse_incomplete` is on (tool newer than v0.5.0-alpha.1) and an added edition declaring `selection: completeness` lists fewer readings than the hub holds in the claim's scope | refused | reader policy · `hub rules: selection.refuse_incomplete` · that edition; only readings the hub holds count | C-core-59, C-core-60 |
| H-34b | as H-34a, but the claim cannot be decided (no scope; a reading with no readable position or source) | could not check | the edition named with the token; never a pass | I4 |
| H-35 | as H-34, but the boundary is `unchecked` or `not comparable` | could not check | the edition named with the token; never a pass | C-core-102, I4 |
| H-36 | an added statement carries `intended_for` | admitted | the named edition re-applied **as held at the base revision**, and the current edition, shown side by side as the hub's computation; the publisher's recorded outcome shown as the publisher's word | C-core-98 |

Rule evaluations print exactly one of `ok`, `fail`, `unchecked`. Every refusal names its class (intrinsic,
reader policy, operational), its reason and its scope, and never suggests a blanket override (§11, §13;
C-core-113, C-core-121). Refusing one object leaves every other object's verdict unchanged.

**Without `--base`** (a full check of the tree at `HEAD`): H-20 to H-29 apply to every object held, H-31
to every path. H-30, H-32, H-33 do not apply (there is no change). A blob present while its digest is in
`not_served` is reported (`not served: still present`) and does not change the exit code.

### 2.3 Exit codes

The values and their meanings are the tool's (claims-model §13). For `hub check` they mean:

| Exit | Meaning for `hub check` | Cases |
|---|---|---|
| 0 | Every object in the change would be admitted, possibly with labels (`contested`, `detached`, `FORK`, `unsupported`, `unreferenced`, `unavailable`). Also: a control-plane-only change (H-33). Exit 0 says the check succeeded, not that everything referenced is valid. | C-core-135 |
| 1 | At least one object would be refused (H-20 to H-22, H-26, H-27, H-30 to H-32, H-34): each one is a check that ran to completion and failed. | C-core-110 – C-core-112 |
| 2 | The check did not start: unknown option, malformed argument, `--base` not a commit here, no hub root, unknown layout marker, unreadable policy file at the policy revision. Nothing is evaluated and no partial result is printed. | C-core-133a |
| 3 | Nothing would be refused, and admission of at least one object could not be decided (H-35). The objects are named with their token. | C-core-133, I4 |

`hub check` runs no code from the hub, so exit 5 (environment mismatch) and exit 4 (nothing to re-run),
which belong to `verify`, never occur. No other value is emitted except by a crash of the tool itself.

### 2.4 What it deliberately does not check

| id | Not checked | Why |
|---|---|---|
| H-40 | Whether a signer is trusted, is a maintainer, or may publish here | Trust is each reader's policy and depends on the identity design, which is open. Maintainers apply their own trust at `commons pull`. |
| H-41 | Future timestamps | Refused at write time by the writer (§10; C-core-105), not at a gate, where the reader's clock decides nothing about the signer's. |
| H-42 | Time evidence quality (`anchors` proofs) | Verified by each reader against block evidence it trusts, stored locally, never transported (Bitcoin-time note §9). CI checks the `anchors` statement's validity only. |
| H-43 | Execution | Admission is not execution (§11). CI never runs a workflow, comparator or anything else from the hub. |
| H-44 | Completeness | A change is a package; packages make no completeness claims (§11; C-core-01). |

### 2.5 Output

| id | Requirement |
|---|---|
| H-50 | Human output: one line per refused or labelled object (path, id or digest in full, verdict, class, reason), then the policy revision and digest, the tool version, and a summary count per verdict. Peer-supplied text (descriptions, reasons, notes) is rendered inert (C-core-122). |
| H-51 | `--json`: one JSON object on stdout with the same content, each object's verdict typed; on exit 2, an RFC 9457 problem-details object with `class`, `scope`, `committed: false` and `next` (§13). |
| H-52 | Every state token and exit code is from the tool's documented closed vocabulary (C-core-132). |

## 3. Other commands the README mentions

The README's "Read it" and "Contribute" sections use these. Their contracts are the tool's own; this
template only needs them to exist under these names.

| Command | Used for |
|---|---|
| `commons status --limits` | listing the limit names `hub-policy.json` may set, with the tool's defaults (storage §8) |
| `commons publish …`, `commons push` | contributing statements and blobs into a clone of the hub |
| `commons pull <remote>` | a maintainer's local ingest, applying their own trust |
| `commons verify <id>`, `commons explain <id>` | reading |

# The hub policy file

`hub-policy.json` at the root of this repository is the hub's **admission policy**: the rules this hub
applies, beyond the format's own validity checks, to what it accepts. It is the hub's half of what the
design notes call reader policy (claims-model note §11, I9): a hub is a reader that also serves what it
holds.

> **Draft.** This file format is a proposal for the next format of Research Commons. No released tool
> reads it yet. The schema is `docs/hub-policy.schema.json`; the tool that implements it is authoritative
> where the two differ, and the difference is a bug in one of them.

## Three rules about the file itself

1. **It is control plane, never data.** A pull request, a package or any other incoming tree cannot change
   it. When a change touches `hub-policy.json`, `commons hub check` reports the file as a control-plane
   change and does not apply it (claims-model note §11, "data never carries control"; CONFORMANCE
   C-core-114, C-core-115). `commons pull` refuses it with class `intrinsic: outside the data surface`. The
   maintainers change the policy through the repository's own review and merge process, never through
   `pull`.
2. **CI reads it from the base revision.** The check evaluates every object in a pull request under the
   policy the target branch already has, so a pull request that loosens the policy and adds an object only
   the looser policy admits fails (C-core-115). A policy change takes effect for the changes checked after
   it is merged.
3. **It is strict JSON.** The tool reads it with the same parser it uses for statements: duplicate keys are
   refused, numbers are integers. A file that is present and unreadable, or that carries a field this
   version does not define, stops the check before any work with exit 2 (claims-model note §13). An absent
   file means every default below.

A policy change is authorized by whoever may merge to the default branch: the policy a change replaces is
what admits the change (I9). The file names no keys and grants no key anything.

## Fields

Every field is optional except `policy`. Each one exists because a merged design note says a hub may decide
it; the citation is beside it.

| Field | Default | What it decides | Source |
|---|---|---|---|
| `policy` | (required) | Format tag, `rc-hub-policy/1`. | — |
| `blobs` | `"stored"` | `"external-only"` makes this a hub that carries statements and external `holds`, and no blob bytes. A change that adds a file under `blobs/` is then refused under reader policy, scope that file. | claims-model §11, "bytes may live elsewhere"; storage §9 |
| `limits` | `{}` | Operational limits by name, as integers. An object over a limit is refused with class `operational`, the limit name and value printed. Names are the ones the tool documents (`commons status --limits`); an unknown name is a configuration error (exit 2). Empty means the tool's defaults. | claims-model §11 (operational class); storage §8 ("limits are read from the reader's policy"); C-core-112 |
| `not_served` | `[]` | Content digests this hub does not serve. Adding a listed blob is refused under reader policy. Removing a listed blob is the one deletion CI admits: every other removal is refused, because a delivery never removes what a hub holds. So a takedown is two merges: the policy entry, then the removal. Statements about the content are kept and shown; the content displays as not held here (policy). | claims-model §13 ("a hub that must stop serving something does so under its own admission policy"); storage §6.2; C-core-140 |
| `control_plane` | `[]` | Paths this hub adds to the control plane, beside the ones the template ships: exact paths (`CITATION.cff`, `NOTICE`) or directory prefixes ending in `/` (`scripts/`). A change to one is reported and not applied, like a change to `README.md`; without the entry the same file is refused as outside the data surface. Entries are added, never subtracted, and are read at the base revision, so a change cannot add a path for itself. An entry that names or covers `blobs`, `statements` or `rc-store`, an unsafe path, a repeat, or more than 256 entries is exit 2, naming the entry. | claims-model §11 ("data never carries control"); CLI-CONTRACT §2.2 |
| `series.refuse_premature_freeze` | `false` | Refuse a growing-series edition whose declared `frozen_before` is later than the boundary its members' finality rules compute. The edition stays a valid statement everywhere else. A boundary the check cannot compute is reported as could-not-check (exit 3), never as a pass. **The first next-format tool ships no finality re-check for series segments**, so with this set every added series edition is could-not-check (exit 3) until one ships; leave it `false` unless you want series editions held for review. | claims-model §8 (growing series); C-core-104 |
| `selection.refuse_incomplete` | `false` | **Needs tool v0.5.0-alpha.2 or later** (the pinned release; the alpha.1 tool treats this field as undefined: exit 2). Refuse an edition whose `completeness` claim ("every reading of class C from source S in window W") fails against the readings this hub holds: a reading in scope that the edition does not list. The edition stays a valid statement everywhere else. A claim the check cannot decide (a reading with no readable position or source, or a claim with no scope) is reported as could-not-check (exit 3), never as a pass. The check uses only statements the hub holds; it says nothing about readings it has not seen. Editions with no completeness claim are unaffected. | claims-model §6 (3), §8; C-core-59, C-core-60 |

`docs/hub-policy.example.json` shows every field set. The schema gives the `selection` object no default of its own,
so a tool that fills in schema defaults does not write it for a hub still pinned to v0.5.0-alpha.1 (that tool exits 2
on it). The example's `limits` entry uses
`blob_bytes` as a stand-in: the tool has not published its limit names yet, and project-wide size policy is still open.

## What is deliberately not in this file

- **Which keys may publish here, who the maintainers are, which attesters to trust.** Trust is reader
  policy that depends on the identity design, which is still open. The format's position today is that a
  key is an identity and trust is chosen by each reader; this template keeps signer trust out of CI and
  out of this file, as the alpha did. Each maintainer applies their own trust when they `commons pull`.
- **Collection and task rules.** Those are part of a resource's editions, which are signed statements, so
  they travel with the data and anyone can re-apply them (claims-model §8). CI re-applies the edition a
  statement names, as held at the base revision, and the current edition, and shows both (C-core-98). The
  hub does not restate them here.
- **Execution.** CI never runs code from the hub. What a reader may execute is that reader's own execution
  policy, kept on their machine (claims-model §11, "admission is not execution").
- **Selection disclosure** (curated, rule-based, completeness claim). A selection is shown only when it is
  signed; an unsigned file cannot carry it. A collection declares it in its editions. How a hub as a whole
  declares one is not decided yet; until then a hub's selection displays as `undeclared`.
- **Reference data, subscriptions, remotes.** Each reader's own choice (I9).

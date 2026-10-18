# Research Commons hub (template, next format)

> **Draft for the next format.** This branch is a template for hubs in the next format of Research
> Commons, which is specified in the tool repository's design notes and not yet released. The tool it calls
> does not exist as a command yet: `.github/workflows/hub-check.yml` pins a placeholder and fails until a
> release is pinned, and `docs/CLI-CONTRACT.md` says what that release must provide. For a hub you can run
> today, use the template's `main` branch, which targets the alpha.

> **Template.** Click **Use this template** to create a hub for your topic, then rename it in this heading.

A **Research Commons hub** is a git repository that holds signed statements and the bytes they talk about.
It contains no code. The `commons` tool lives in its own repository,
[research-common/research-commons](https://github.com/research-common/research-commons), and you update it
there, never through a hub.

## What is in a hub

```text
rc-store                         layout marker: {"layout":"rc-store/1"}
blobs/sha256/<64 hex>            content: the bytes whose sha256 is the file name
statements/<signer>/<id>         one signed statement per file; the file name is the statement id,
                                 the directory is the signer (":" written as "-")
hub-policy.json                  this hub's admission policy (control plane, see below)
```

That is the whole data surface. There is no shared file that anyone edits:

- A **blob** has no owner and no metadata. Two people who publish the same bytes share one blob.
- A **statement** is one key's signed word about something: that it authored some bytes, observed them,
  reproduced them, holds a copy, retracts them; an edition of a collection; a time anchor. Each key's
  statements form its own log, chained by the predecessor each one names inside its signature. Nobody can
  add to, edit or remove another key's log.
- **Anchors** are statements too (`anchors`), with the timestamp proof stored as a blob. They need no
  directory of their own.
- **Collections, tasks and growing series** are resources: a genesis statement and signed editions, in their
  controllers' logs. Their membership rules travel inside the editions.
- **Indexes, caches, grades and views** are not stored. Every reader computes them from the blobs and
  statements it holds, under its own policy. Deleting them changes no answer.

A hub may also hold **no blobs at all**: a `holds` statement can point at bytes stored elsewhere (an object
store, a content-addressed network, a plain URL), and a reader verifies the full hash whatever the source.
Set `"blobs": "external-only"` in `hub-policy.json` for that configuration.

## Start a hub

1. Create a repository from this template.
2. Edit `hub-policy.json` if the defaults do not suit you (`docs/hub-policy.md` explains each field). Commit
   it on the default branch.
3. Pin the tool in `.github/workflows/hub-check.yml` (see *Upgrading the tool pin*).
4. Protect the default branch: require the `hub-check` job and a review before merging.
5. Say in this README what the hub is for and how it selects what it admits: curated by its maintainers,
   by a published rule anyone can re-apply, or a completeness claim others can check. This is prose for
   people. The tool shows only a selection someone signed (a collection declares one in its editions), so a
   hub as a whole displays as `undeclared`, which readers treat as curated.

## Read it

```bash
git clone <this hub> && cd <this hub>
commons verify <id>          # re-check an artifact against the statements you hold
commons explain <id>         # the statements, policy and checks behind a status
```

Every status a reader sees is scoped: "corroborated by 2 *within the statements you hold*". A hub is one
source; a reader who follows several sees what each holds, including contradictions a single hub left out.

## Contribute

1. Generate a signing key (the tool does this on first run).
2. In a fork or branch of this hub, publish with the tool. It writes your statements under
   `statements/<your key>/` and the bytes under `blobs/sha256/`. Commit only those.
3. Open a pull request. CI runs `commons hub check --base <target>` and reports a verdict for every object.
4. A maintainer merges, or ingests your branch on their own machine with `commons pull`, which also applies
   their own trust.

You never edit an existing file in a hub. To correct something, publish a new statement: a later
`authored` that supersedes your earlier bytes, a `retracts`, a new edition.

## What CI checks

Each object in a pull request goes through the tool's one validation core, the same function `pull` and
`push` use. In outline (the full table is `docs/CLI-CONTRACT.md`):

- **Only data paths change.** A pull request may add files under `blobs/` and `statements/` and nothing
  else. It may not modify, rename or delete anything already held. Changes to `hub-policy.json`, this
  README, the workflow or any other file are reported as control-plane changes for a maintainer to review,
  and are never applied as part of a data change.
- **Bytes match their names.** Every blob hashes to its file name; every statement's id is its file name.
- **Signatures and form.** Every statement is canonical, signed by the key its directory names, and well
  formed.
- **This hub's policy, from the base revision.** The check reads `hub-policy.json` as it is on the branch
  being merged into, never from the pull request. A pull request that loosens the policy and adds something
  only the looser policy admits fails.
- **Collection rules.** Where a statement says which collection edition it was checked against before
  publishing, CI re-applies that edition and the current one, and shows both.

What CI admits with a label rather than refusing: statements from a tool version newer than the checker
(`unsupported`), statements whose predecessor the hub does not hold yet (`detached`), a key whose log has
forked (`FORK`), bytes held elsewhere (`unavailable`), and every disagreement between keys (`contested`).
None of those is a reason to refuse valid signed data.

Exit codes: `0` everything would be admitted · `1` at least one object refused, each named with its reason
and scope · `2` the check could not start (bad option, unreadable policy) · `3` admission of an object
could not be decided. Every non-zero code fails the job.

## What CI deliberately does not decide

- **Who is trusted.** CI does not check whether a signer is a maintainer, is known, or may publish here. Trust
  is each reader's own policy and stays on their machine; maintainers apply theirs when they `commons pull`.
  A valid statement from an unknown key passes CI; whether anyone believes it is a separate question.
- **Whether anything is true.** Signatures prove who said something, not that it is right. Corroboration,
  grades and contradictions are computed by each reader.
- **Time.** CI checks that an anchor statement is well formed, not that its proof is genuine. Each reader
  verifies timestamp proofs against block evidence it trusts.
- **Execution.** CI never runs anything from the hub. A reader decides for itself what it executes, and
  runs foreign code sandboxed.
- **Completeness.** A pull request is a package of objects. It claims nothing about what else exists.

## Removing content

A hub never deletes statements, including retractions. A retraction is displayed wherever the artifact is
shown; it does not by itself stop anyone serving the bytes. If this hub must stop serving some bytes, add
their digest to `not_served` in `hub-policy.json` (one merge), then remove the blob (a second merge, which
CI admits because the base policy now lists it). Statements about the content remain, and readers see it as
not held here. Deleting the bytes here does not erase them anywhere else.

## Coming from an alpha hub

The next format is a clean break. There is no importer and no dual-format reading, and the tool refuses an
alpha hub tree (`registry/`, `store/`, `.commons-hub`) as malformed rather than guess at it.

- **Content carries over unchanged.** The same bytes have the same sha256, so you do not regenerate data;
  you republish it.
- **Statements are new.** Authorship, licences and time evidence are re-signed and re-anchored under the new
  format. Priority established under the alpha does not carry over.
- **Start a new repository** from this template and republish into it. Keep the alpha hub as it is, for
  reference; do not convert it in place.

## Upgrading the tool pin

`hub-check.yml` builds the tool from a **full commit SHA**, with the release tag as a trailing comment, so a
change upstream cannot alter what this hub accepts until you choose to move the pin. To upgrade:

1. Pick a release and resolve its tag to a commit:
   `git ls-remote https://github.com/research-common/research-commons.git 'refs/tags/<tag>^{}'`.
2. In a pull request, set `TOOL_REF` to that SHA and update the tag comment.
3. Before merging, run `commons hub check` with the old and the new tool on this hub's default branch. A
   check that newly refuses held data needs a decision before you merge, not after. Read the tool's
   `CHANGELOG.md` for the versions in between.

The CI log prints the tool commit and `commons --version` on every run.

## Maintainers

How a hub names its maintainers is not settled for the next format yet. Until it is, list them here in
prose. A list in this file grants nothing to anyone: CI does not read it.

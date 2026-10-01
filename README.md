# Research Commons hub (template)

> **Template.** Click **Use this template** to create a hub for your topic, then rename
> it in this heading and in `.commons-hub`. Setup: `GETTING-STARTED.md` in the tool repo.
> Or skip the template: `commons hub init <dir>` generates the same files.

A **Research Commons hub**: a data-only git repository (manifests in `registry/`,
content-addressed blobs in `store/`). It contains **no code**. The `commons` tool lives in
a separate repository — [research-common/research-commons](https://github.com/research-common/research-commons) — and you
update it there, never through this hub.

## Read it

```bash
git clone https://github.com/research-common/research-commons.git ~/research-commons   # the tool, once
export PATH="$HOME/research-commons/bin:$PATH"
git clone <this hub> && cd <this hub>                             # the data
commons list --type collection        # topics hosted here
commons collection show <cl-id>       # endorsed + claimed members, open work
commons verify <id>                   # re-derive it yourself
```

`commons` finds this hub automatically from any directory inside it (the `.commons-hub`
marker file); `commons hub where` says which data root is in use.

## Contribute

1. Mint a signing key and tell the maintainers your address (`commons peer whoami`).
2. Fork this hub (or push a branch if you have write access), publish from inside your
   clone, commit `registry/` + `store/`, and open a pull request.
3. CI runs `commons hub check`: the PR may only touch `registry/` and `store/`, every
   blob must match its hash, every signature must verify.
4. A maintainer ingests it through the gate — `commons pull <your-remote> --branch
   <branch>` — which additionally checks signer trust. Endorsement = the maintainer
   republishing the collection with your artifact as a member; until then it shows as
   CLAIMED via your `--link part-of:cl-…`.

Full guide: `docs/COLLABORATING.md` in the tool repository.

## Upgrading the tool pin

`.github/workflows/hub-check.yml` checks the tool out at a **full commit SHA**, with the
release tag as a trailing comment, and installs its dependencies with `npm ci` from the
tool's lockfile. A change upstream therefore can't alter what this hub's gate accepts until
you choose to move the pin. To upgrade:

1. Pick a release from the tool's releases page and resolve the tag to its commit:
   `git ls-remote https://github.com/research-common/research-commons.git 'refs/tags/<tag>^{}'`.
2. In a pull request, set `ref:` to that SHA and update the tag comment.
3. Before merging, run `commons hub check` locally with both the old and the new tool on
   this hub's `main`. A gate that newly refuses existing data needs a decision before you
   merge, not after. Read the tool's `CHANGELOG.md` for the versions in between.

The CI log prints the tool commit and `commons --version` on every run.

## Maintainers

| handle | signing address |
|---|---|
| _(add yourself)_ | `0x…` |

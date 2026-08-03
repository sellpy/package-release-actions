# package-release-actions

Shared release plumbing for Sellpy's npm packages. Three things live here:

- **`semver-label`** — a composite action that resolves a pull request's `major`/`minor`/`patch`
  label to an npm bump type, failing unless exactly one is present. This is the single
  definition of that rule.
- **`.github/workflows/npm-publish-master.yml`** — a reusable workflow that publishes a
  single-package repo from its default branch.
- **`.github/workflows/npm-publish-preview.yml`** — a reusable workflow that publishes a
  throwaway preview build of a single-package repo from any other branch.

This repository is public so that consuming repositories — including private ones — can
resolve the action and the reusable workflows without any org-level sharing setting to keep
enabled. That is the only reason it is public: it is an internal tool, built around Sellpy's
conventions, and it is unlikely to be useful outside them. No support or stability promises
are made to anyone outside Sellpy, and issues or pull requests from outside are not expected.

Nothing here is a secret: the action and the workflow contain no credentials. Consumers pass
their own `NPM_TOKEN` in, and it stays in the calling repository's secrets.

## Why the version is not in package.json

The released version used to live in two places that had to agree — `package.json` on the
default branch, and the registry — with nothing enforcing that they did. The old step order
made drift likely rather than exotic: `npm publish` ran before `git push`, so a failed push
left a stranded tag and the default branch sitting on an already-published version, and
every later run tried to republish it until someone fixed the branch by hand.

The registry is the natural home for it, because that is where the irreversible step already
happens. The truth and the point of no return become the same place, so they cannot disagree.

Consuming repos keep a `0.0.0-managed` placeholder in `package.json` to make it explicit
that the field is not read.

## Why no personal access token

Publishing needed a PAT only because it pushed the version bump back to a protected branch,
which the built-in `GITHUB_TOKEN` cannot do. Nothing is pushed to the default branch now —
only a `vX.Y.Z` tag, as a pointer for humans — and tags are not covered by branch
protection, so `GITHUB_TOKEN` is sufficient. A PAT is tied to one person and expires; when
it lapses the whole release path goes down with it.

## Why checkout overrides the ref

The workflow triggers on a pull request closing rather than on a push to the default branch,
because the semver label lives on the pull request — a push event carries no labels, so there
would be nothing to read `major`/`minor`/`patch` from.

That trigger is what makes the override necessary. On a `pull_request` event, checkout does
not hand you the default branch. It hands you a merge preview GitHub computed in advance —
"this PR's branch merged into master" — calculated when the PR was last updated and never
refreshed when master moves afterwards:

1. PR A is opened. GitHub computes A-merged-into-master.
2. PR B merges. Master now has B.
3. PR A merges, this workflow fires, and checkout returns the preview from step 1 — which has
   no B in it.
4. The published tarball is missing B, even though master has it.

`ref: ${{ github.event.repository.default_branch }}` fetches the branch's actual current tip
instead. None of this applies to a `push`-triggered workflow, where the default checkout is
already the commit that just landed — which is why the override looks redundant until you
notice the trigger.

## Using the publish workflow

```yaml
name: Publish master
on:
  pull_request:
    branches: [master]
    types: [closed]

# Serialise publishes. The next version is derived from the registry, so two runs reading
# the registry at once would resolve the same version and one would fail. Never cancel: a
# cancelled run could stop between publishing and tagging.
concurrency:
  group: publish-master
  cancel-in-progress: false

jobs:
  validate:
    uses: ./.github/workflows/code-validation.yml
  publish:
    if: github.event.pull_request.merged == true
    needs: validate
    permissions:
      contents: write
    uses: sellpy/package-release-actions/.github/workflows/npm-publish-master.yml@v1
    with:
      labels: ${{ toJSON(github.event.pull_request.labels.*.name) }}
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_AUTOMATION_TOKEN }}
```

`permissions: contents: write` is required in the caller and is not optional. A called
workflow can lower the permissions it is handed but never raise them, so the `contents: write`
declared inside this workflow is a ceiling, not a grant — without it in the caller, the tag
push fails with a 403 that says nothing about permissions.

The caller owns the trigger, the concurrency group and the test gate, so each repo keeps its
own validation workflow and its own publish secret name. `actions/checkout` inside a
reusable workflow checks out the *calling* repo, which is what we want.

`labels` is the only input and `NPM_TOKEN` the only secret, which keeps the contract small
enough to hold stable across `@v1`. Node version (`.nvmrc`), runner (`ubuntu-22.04`) and
registry (`https://registry.npmjs.org`) are fixed in the workflow, because they are the same
in every repo that publishes this way — parameterising them would encode drift rather than
remove it. An optional input can be added later without a breaking change if one of them
ever genuinely needs to vary.

The workflow outputs `version`, the version it published.

## Using the preview-build workflow

A preview build publishes the tip of a branch as `0.0.0-<branch>-<sha>` under a dist-tag named
after the branch, so `npm i @sellpy/commons@canary` — or `@int-2497-urbify` — resolves it. It
is throwaway: no tag, no release, nothing written back to the branch.

```yaml
name: Publish preview build
on:
  push:
    branches: [dev, canary]

# Serialise publishes per branch. Never cancel: a cancelled run could stop midway through
# publishing.
concurrency:
  group: publish-${{ github.ref_name }}
  cancel-in-progress: false

jobs:
  validate:
    uses: ./.github/workflows/code-validation.yml
  publish:
    needs: validate
    uses: sellpy/package-release-actions/.github/workflows/npm-publish-preview.yml@v1
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_AUTOMATION_TOKEN }}
```

This contract is smaller than the master one: **no inputs at all**, one secret, and the same
`version` output. The dist-tag and the version segment both derive from `github.ref_name`, so
there is nothing for a caller to pass and nothing for two repos to disagree about. As with the
master workflow, the caller owns the trigger, the concurrency group and the validation gate.

No `permissions` block is needed, unlike the master workflow — nothing is pushed back, so the
default token is already sufficient.

Add `workflow_dispatch` to the trigger where publishing an arbitrary feature branch is
useful. Branch names are sanitised into the version segment and the dist-tag
(`int-2497/urbify` → `int-2497-urbify`), so any branch is safe to dispatch against; the
sanitisation is unconditional rather than an option, because a repo that does not need it is
unaffected by it.

### Why a preview does not push the version back

Two of the three workflows this replaced ran `npm version` and then
`git push --follow-tags` back to the branch, which is the same pattern the master path was
changed to stop doing, for the same reasons — plus one specific to previews: `npm version`
trips over the tag it created on the previous run, so re-running a preview on an unchanged
commit fails. `--no-git-tag-version --allow-same-version` fixes that and needs no git
identity, no `contents: write` and no push.

One consequence to know about when porting a repo that used to push back: the version field
it last pushed is now frozen on that branch forever. Reset it to the `0.0.0-managed`
placeholder, on **every** preview branch rather than just the one you happened to look at.

### Why there is no build step, and why the run can fail before installing

Neither workflow here builds explicitly. Both rely on npm's lifecycle hooks — `prepare`,
`prepack` or `prepublishOnly` — firing during `npm ci` and `npm publish`, so the build stays
defined by the repo rather than duplicated in shared CI.

The preview workflow checks that at least one of those hooks exists and fails the run if none
does. That check is not theoretical: `automation-commons` had no build hook and would have
published an empty `dist/`, and `react-native-scroll-anchor`'s hook fired but resolved the
wrong `tsc`. A package published with no build output is the one failure in this pipeline that
produces a green run and is discovered by a consumer instead, which is worth a step that
cannot pass by accident. It runs before `npm ci` so the run fails in seconds.

`dev` and `canary` are long-lived and re-cut from the default branch by hand, so a hook added
to the default branch is not present on them until someone merges it across. That is why the
check lives in the shared workflow rather than being fixed once per repo.

## Using the label action on its own

The PR gate needs the same rule, without a publish:

```yaml
name: PR Validation
on:
  pull_request:
    types: [opened, labeled, unlabeled, synchronize]
jobs:
  require-labels:
    name: Exactly one Semantic Version label
    runs-on: ubuntu-22.04
    steps:
      - uses: sellpy/package-release-actions/semver-label@v1
        with:
          labels: ${{ toJSON(github.event.pull_request.labels.*.name) }}
```

No `actions/checkout` is needed — that is only required for an action stored in the calling
repo itself.

## Versioning

Consumers pin `@v1`. The `v1` tag moves as fixes land; cut `v2` for a breaking change to
inputs or behaviour.

`npm-publish-master.yml` references `semver-label@v1` internally, so the two are released
as a pair — bump both when cutting a new major.

## What this does not cover

- **Monorepos.** `sellpy/design-system` publishes three packages with independent versions
  and cross-package pinning. Both workflows here resolve one package from the repo root, so
  neither can serve it; it needs a workspace-aware variant of each.
- **Release-triggered packages.** `sellpy/react-native-scroll-anchor` publishes on a GitHub
  release rather than a labelled merge.

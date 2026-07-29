# package-release-actions

Shared release plumbing for Sellpy's npm packages. Two things live here:

- **`semver-label`** — a composite action that resolves a pull request's `major`/`minor`/`patch`
  label to an npm bump type, failing unless exactly one is present. This is the single
  definition of that rule.
- **`.github/workflows/npm-publish-master.yml`** — a reusable workflow that publishes a
  single-package repo from its default branch.

This repository is private, so consuming repositories can resolve the action and the workflow
only because it is shared with the organisation — Settings → Actions → General → "Accessible
from repositories in the sellpy organization". If that is ever turned off, every consumer's
publish job fails at workflow resolution, with an error that looks nothing like an npm
problem.

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
    uses: sellpy/package-release-actions/.github/workflows/npm-publish-master.yml@v1
    with:
      labels: ${{ toJSON(github.event.pull_request.labels.*.name) }}
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_AUTOMATION_TOKEN }}
```

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

- **Preview builds.** `dev`/`canary` publishes stay per-repo: some sanitise arbitrary
  branch names into a version segment and dist-tag, and that logic is not uniform enough to
  share usefully.
- **Monorepos.** `sellpy/design-system` publishes three packages with independent versions
  and cross-package pinning. It needs its own shape.
- **Release-triggered packages.** `sellpy/react-native-scroll-anchor` publishes on a GitHub
  release rather than a labelled merge.

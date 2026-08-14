# package-release-actions

Shared release plumbing for Sellpy's npm packages. Five things live here:

- **`semver-label`** — a composite action that resolves a pull request's `major`/`minor`/`patch`
  label to an npm bump type, failing unless exactly one is present. This is the single
  definition of that rule.
- **`require-build-hook`** — a composite action that fails when a package has a `build` script
  but no npm lifecycle hook to run it. Internal to the workflows below rather than something a
  caller invokes.
- **`.github/workflows/npm-publish-master.yml`** — a reusable workflow that publishes a
  single-package repo from its default branch.
- **`.github/workflows/npm-publish-preview.yml`** — a reusable workflow that publishes a
  throwaway preview build of a single-package repo from any other branch.
- **`.github/workflows/npm-publish-workspace-master.yml`** — the default-branch publish for one
  workspace of a monorepo, called once per package.

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

`labels` is the only input and `NPM_TOKEN` the only required secret, which keeps the contract
small enough to hold stable across `@v1`. Repos with private `@sellpy/*` dependencies also pass
the optional `NPM_READ_TOKEN` — see [Why `npm ci` runs before the publish token is
written](#why-npm-ci-runs-before-the-publish-token-is-written). Node version (`.nvmrc`), runner (`ubuntu-22.04`) and
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

This contract is smaller than the master one: **no inputs at all**, one required secret, and the
same `version` output. The dist-tag and the version segment both derive from `github.ref_name`, so
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
commit fails.

Both flags on `npm version` are load-bearing. `--no-git-tag-version` keeps the bump inside the
runner, so there is no commit and no tag to push and the workflow needs neither a git identity
nor `contents: write`. `--allow-same-version` is what makes a re-run on an unchanged commit
succeed, since the version it computes is derived from the SHA and therefore identical to the
one already in `package.json` from the previous run.

One consequence to know about when porting a repo that used to push back: the version field
it last pushed is now frozen on that branch forever. Reset it to the `0.0.0-managed`
placeholder, on **every** preview branch rather than just the one you happened to look at.

### Why there is no build step, and why the run can fail before installing

None of the publish workflows here build explicitly. They all rely on npm's lifecycle hooks —
`prepare`, `prepack` or `prepublishOnly` — firing during `npm ci` and `npm publish`, so the
build stays defined by the repo rather than duplicated in shared CI.

A check enforces that reliance rather than assuming it, and it lives in the
`require-build-hook` action rather than inline in each workflow. It was duplicated verbatim in
every one of them, differing only in which directory's `package.json` it read, which is the same
shape that earned `semver-label` its own action: one rule, one parameter, several callers.

The rule is deliberately narrow: **a package with a `build` script must have a hook that runs
it.** A package with no `build` script has nothing to build and passes. That distinction
matters — `@sellpy/design-system-commons` publishes source directories with no build step at
all, and a blanket "must have a hook" rule would fail it for no reason, forcing either a skip
switch on the contract or a fake `prepack` in the repo.

What the rule does catch is the genuinely dangerous state: a build that exists but is not wired
to publishing. `automation-commons` was in it and would have published an empty `dist/`.
`fetch-graphql-schema` shows how quiet the failure is — `main` and `bin` both point into `lib/`,
which is gitignored, so a publish without its `prepack` would ship a package whose entry point
and CLI do not exist. Nothing about the run would say so, which is why this is worth a step that
cannot pass by accident. It runs before `npm ci` so the run fails in seconds.

`dev` and `canary` are long-lived and re-cut from the default branch by hand, so a hook added
to the default branch is not present on them until someone merges it across. That is why the
check lives in the shared workflow rather than being fixed once per repo.

### Why `npm ci` runs before the publish token is written

All three workflows install first and only then overwrite `.npmrc` with `NPM_TOKEN`. The order
is deliberate and easy to "tidy" into a bug.

Where a repo has private `@sellpy/*` dependencies — `pdf-creator` and `design-system` are the
current cases — the install needs a token that can *read* them, and `NPM_TOKEN` generally
cannot: a publish token for one package carries no read rights on another. Writing `NPM_TOKEN`
before installing therefore breaks the install.

Those repos pass the optional `NPM_READ_TOKEN` secret, which is appended to `.npmrc` in a step
before `npm ci`. Appended rather than written over the top, so any non-secret config the repo
commits — `engine-strict` and the like — survives; the publish step then overwrites the file, so
the read token does not linger into the publish. Repos with no private dependencies pass
nothing and the step is a no-op.

Use a **read-only** token. It is the least it needs, and it is the one credential here that a
repo hands to every install rather than only to the publish step.

What makes it worth documenting rather than leaving to be rediscovered is the error you get.
npm answers **404, not 403**, for a private package the caller may not fetch, so the failure
reads as a missing tarball or a bad version rather than a permissions problem, and the obvious
next move — checking whether the dependency exists — finds nothing wrong.

## Using the workspace publish workflow

`npm-publish-workspace-master.yml` is `npm-publish-master.yml` resolved against a workspace
directory instead of the repo root. One call publishes one workspace. It exists because neither
single-package workflow can serve a monorepo: both read the root `package.json` for the package
name, the version and the build hook.

```yaml
name: Publish master
on:
  pull_request:
    branches: [master]
    types: [closed]

concurrency:
  group: publish-master
  cancel-in-progress: false

jobs:
  validate:
    uses: ./.github/workflows/code-validation.yml
  commons:
    if: github.event.pull_request.merged == true
    needs: validate
    permissions:
      contents: write
    uses: sellpy/package-release-actions/.github/workflows/npm-publish-workspace-master.yml@v1
    with:
      workspace: packages/commons
      labels: ${{ toJSON(github.event.pull_request.labels.*.name) }}
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_AUTOMATION_TOKEN }}
  react-web:
    if: contains(github.event.pull_request.labels.*.name, 'react-web')
    needs: commons
    permissions:
      contents: write
    uses: sellpy/package-release-actions/.github/workflows/npm-publish-workspace-master.yml@v1
    with:
      workspace: packages/react-web
      labels: ${{ toJSON(github.event.pull_request.labels.*.name) }}
      pin-dependency: '@sellpy/design-system-commons@${{ needs.commons.outputs.version }}'
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_AUTOMATION_TOKEN }}
```

Two inputs are required — `workspace` and `labels` — plus the same `NPM_TOKEN` secret, the same
optional `NPM_READ_TOKEN`, the same `version` output as the single-package workflow, and the same
`contents: write` requirement in the caller.

### The tag prefix is derived, not configured

`packages/react-web` tags `react-web_v18.3.1`. The workspace basename is the prefix, because a
monorepo cannot use a bare `vX.Y.Z` for three packages with independent versions. That is one
input this workflow does not need.

### Ordering is the caller's job, and `needs:` is what does it

`pin-dependency` writes an exact sibling version into this workspace's `dependencies` before
publishing, so the published tarball depends on a concrete version rather than a floating range.
The sibling must have been published first — `needs: commons` above is not stylistic, it is the
ordering.

### The pin rewrites the manifest and must not install

It uses `npm pkg set`, not `npm install --save-exact`. The difference matters and is easy to
"fix" in the wrong direction: `npm install` would do two things where only one is wanted —
rewrite the manifest *and* re-resolve the module.

A monorepo's workspaces are symlinked into the root `node_modules`, so a sibling normally
resolves to its source directory. Installing a pinned version replaces that link with a copy
downloaded from the registry, under `packages/<name>/node_modules/`. The published tarball then
looks correct, and the build silently gets worse.

**This has already happened.** `sellpy/design-system` publishes design tokens as plain
JavaScript with no type declarations, so TypeScript recovered their types by inferring over the
source — which it only does outside `node_modules`. Once the pin moved the sibling inside
`node_modules`, every token import became `any`, and `@sellpy/design-system-react-native@13.0.1`
shipped 27 `any`s in its `.d.ts` where 12.19.2 had none. The build only warned, so the release
was green.

Note the interaction that made it possible: while the placeholder version convention is in use,
a pinned version can never equal the workspace's `0.0.0-managed`, so npm always treats it as an
outside dependency. Before the placeholder, the requested version happened to match the
workspace's real one and npm linked it instead — which is why this worked by accident for a long
time.

Rewriting the manifest alone keeps the sibling resolving through the workspace, which is also
the more correct build: it compiles against the source of the commit that produced the pinned
version, rather than a round-trip through the registry.

Sequence the calls with `needs:` rather than letting them fan out. Three parallel calls would
race, and the failure is not a clean error: `react-web` would install whatever version of
`commons` the registry happened to have at that moment.

### Why nothing is reset afterwards

The pin exists only in the runner. Nothing is committed and nothing is pushed to the default
branch, so the `"*"` range stays in git — which is what makes the workspace resolve locally for
development — while the published package carries the exact version.

That is a behaviour change worth being explicit about for a repo migrating from a workflow that
committed the pin: the *reset* step such a workflow needs disappears along with the push, rather
than having to be reimplemented here.

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

The workflows reference `semver-label@v1` and `require-build-hook@v1` internally, by absolute
path with the tag hardcoded, so everything here is released together — bump all of it when
cutting a new major. A `./`-relative path is not an option: inside a reusable workflow it
resolves against the *caller's* checkout rather than this repo, and expressions are not allowed
in `uses:`.

The practical consequence is that a new internal action cannot be referenced until `v1` includes
it. Because the workflow and the action are resolved at the same tag, moving `v1` to a commit
containing both is atomic from a consumer's point of view — but cutting a release that moves
only one of them would break every caller.

## Known duplication, and when to remove it

`npm-publish-workspace-master.yml` still repeats two things from `npm-publish-master.yml`: the
registry read and bump, and the npm auth ordering. That is deliberate and temporary.

They look more alike than they are. The version resolution differs between root and workspace —
`npm pkg set --workspace`, and the version read back from `package.json` because `npm version`
prefixes the workspace name in its output — and differs again between master and preview, where
the version comes from the commit rather than the registry. Extracting them means choosing one
shape for three genuinely different behaviours, which is worth doing against a working workspace
release rather than ahead of one.

The build-hook check is the counter-example and is already extracted, into
`require-build-hook`: it was byte-identical in every workflow with a single parameter, so its
seam was a fact rather than a guess. Apply the same test to the rest — identical code with one
input, not merely similar-looking steps.

## What this does not cover

- **Monorepo preview builds.** `npm-publish-preview.yml` reads the repo root, so
  `sellpy/design-system`'s preview path still needs a workspace-aware variant. Deferred until
  the master path here has shipped a release.
- **Release-triggered packages.** `sellpy/react-native-scroll-anchor` publishes on a GitHub
  release rather than a labelled merge.

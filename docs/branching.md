# Branching and releases

Everything happens on `main`. Changes arrive through short-lived topic branches
and squash-merged pull requests, and a release is a signed tag on a `main`
commit. There are no `develop`, `release/*` or `hotfix/*` branches.

## Contents

- [Where do I open my pull request?](#where-do-i-open-my-pull-request)
- [Naming topic branches](#naming-topic-branches)
- [The flow](#the-flow)
- [Releasing](#releasing)
- [Fixing a released version](#fixing-a-released-version)
- [Tags and versions](#tags-and-versions)
- [Protection rules](#protection-rules)
- [Frequently asked questions](#frequently-asked-questions)

## Where do I open my pull request?

**Against `main`.** It is the default branch, so a fork and a fresh clone
already point at it.

`main` must always be releasable: tests pass, and anything user-visible has an
entry under `Unreleased` in [CHANGELOG.md](../CHANGELOG.md). Large features land
behind a flag or in small, independently safe pull requests rather than on a
long-lived branch.

## Naming topic branches

`<type>/<short-description>`, with the type matching the Conventional Commits
type you will use in the pull request title:

```text
feat/provider-config-credentials
fix/xrd-established-condition
docs/vscode-client-setup
chore/bump-golangci-lint
refactor/toolset-registration
test/tree-traversal-edge-cases
perf/managed-resource-pagination
ci/windows-matrix
```

Include the issue number if it helps you: `fix/1234-xrd-established-condition`.

Keep one logical change per branch. If your branch needs the word "and" to
describe it, it is two branches.

## The flow

```shell
git clone https://github.com/<you>/crossplane-mcp-server.git
cd crossplane-mcp-server
git remote add upstream https://github.com/ravibagri5/crossplane-mcp-server.git

git fetch upstream
git switch -c feat/provider-config-credentials upstream/main

# work, then
make check
git commit --signoff
git push origin feat/provider-config-credentials
```

Open the pull request against `main`. Keep your branch current by rebasing on
`upstream/main`, not by merging it in:

```shell
git fetch upstream
git rebase upstream/main
git push --force-with-lease
```

Pull requests are squash-merged, so the pull request title becomes the commit
message on `main`. Write it as a
[Conventional Commit](https://www.conventionalcommits.org/):

```text
feat(diagnostics): report ProviderConfig credential resolution
fix(resources): fall back to Established on XRDs without Ready
docs: document VS Code client configuration
feat(resources)!: rename crossplane_list_managed to crossplane_managed_resources
```

A `!` or a `BREAKING CHANGE:` footer marks a change to a tool contract. Those
force a minor bump before 1.0 and a major bump after it, and they must be
described in the pull request's breaking-changes section.

## Releasing

Maintainers only. Open a release checklist issue from the template to track it.

1. **Prepare.** In a pull request to `main`, rename the CHANGELOG
   `## [Unreleased]` heading to `## [X.Y.Z] - YYYY-MM-DD`, add a new empty
   `## [Unreleased]` above it, and update `server.json`, `manifest.json` and
   `smithery.yaml` if they carry the version. Merge it.
2. **Optionally tag a candidate** on the updated `main` to try the published
   binaries and image against real control planes before the final release:

   ```shell
   git switch main && git pull --ff-only
   git tag -s vX.Y.Z-rc.1 -m "vX.Y.Z-rc.1"
   git push origin vX.Y.Z-rc.1
   ```

   Fixes go to `main` as ordinary pull requests; tag `-rc.2` and so on as needed.
3. **Tag the release** on `main`:

   ```shell
   git switch main && git pull --ff-only
   git tag -s vX.Y.Z -m "vX.Y.Z"
   git push origin vX.Y.Z
   ```

The release workflow refuses a tag that is not on `main`, and a final tag
without a `## [X.Y.Z]` CHANGELOG section. It then publishes the GitHub release
with that section as its notes, signed archives, checksums, SBOMs, MCP bundles
and multi-arch images at `ghcr.io/ravibagri5/crossplane-mcp-server`. A release
candidate uses its own CHANGELOG section if one exists, otherwise `Unreleased`.
There is nothing to merge back afterwards.

## Fixing a released version

Fix forward. Merge the fix to `main` with a CHANGELOG entry, then release a
patch version from `main` as above. Only the latest release is supported, so
there are no maintenance branches or backports.

Security fixes follow [SECURITY.md](../SECURITY.md) and are prepared privately
before anything is pushed.

## Tags and versions

The project follows [Semantic Versioning](https://semver.org/). Before 1.0 the
tool contract is not yet frozen, so:

| Change | Pre-1.0 | Post-1.0 |
| --- | --- | --- |
| Renaming or removing a tool, or making an optional argument required | minor | major |
| Adding a tool, or an optional argument | patch or minor | minor |
| Bug fix, docs, internals | patch | patch |

Tag format, always on `main`:

- `vX.Y.Z` — a release.
- `vX.Y.Z-rc.N` — an optional release candidate.

Tags are signed (`git tag -s`). Pre-release tags are published as GitHub
pre-releases, update the `rc` image tag, and never update `latest`. Never move
or delete a tag that has been pushed; tag a new version instead.

## Protection rules

Configured on the repository; listed here so contributors know what to expect.

**`main`**

- Pull request required; direct pushes blocked.
- One approval required, plus CODEOWNERS review. Stale approvals are dismissed
  when new commits land.
- Required check: `Check sign-off` (DCO).
- Branches must be up to date before merging.
- Squash merge only; linear history and conversation resolution required;
  force pushes and deletion blocked.

### Why CI is not a required check

CI is path-filtered: `ci.yaml` carries `paths-ignore` for `**.md` and `docs/**`,
so a documentation-only pull request skips it entirely. A skipped workflow never
reports a status, and a required check that never reports blocks the merge
button forever. Requiring CI would therefore make every docs pull request
unmergeable.

The DCO workflow has no path filter, so it always reports and is safe to
require. It explicitly succeeds for pull requests opened by `dependabot[bot]`;
human-authored pull requests still need signed-off commits.

CI still runs and is still visible on every pull request that touches code, and
a red build is a blocker in review even though the button does not enforce it.
To make CI genuinely required, `ci.yaml` would need an aggregating job that runs
unconditionally and reports success when the path-filtered jobs are skipped.

### A note on admin bypass

`enforce_admins` is off, so maintainers can merge past these rules. That is a
deliberate concession to the project having one maintainer: GitHub does not let
anyone approve their own pull request, so a strict reading of "one approval"
would make the repository unmergeable.

Turn `enforce_admins` on once there is a second maintainer. Until then, treat
the bypass as something to be embarrassed about using, not a routine step.

## Frequently asked questions

**My change is a one-line typo. Do I still need a pull request?**
Yes. Every change reaches `main` through a reviewed, signed-off pull request.

**Can I base my branch on another contributor's branch?**
Yes, but say so in the description and rebase onto `main` once theirs merges.

**How do users avoid unreleased changes on `main`?**
By installing a tagged release. `main` is releasable, but only tags are
releases, and the `latest` image only ever points at a final release.

**Why not keep `develop` and release branches?**
They gave every change a soak period, at the cost of merge-backs, retargeted
pull requests and fixes that had to be ported between branches. With one
maintainer, an optional release candidate tagged on `main` gives the same
chance to test against real control planes with a fraction of the process.

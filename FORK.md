# What this fork changes, and how it is maintained

A fork of [`dart-lang/http`](https://github.com/dart-lang/http) carrying fixes I
needed in production before upstream could take them.

`master` tracks upstream and carries none of those fixes. Each one lives on a
**long-lived patch branch**, consumed as a git dependency pinned by commit SHA.

## The rule everything else follows

**Never rewrite a published patch branch.**

Downstream projects pin these branches by full commit SHA. Rebasing, squashing, or
force-pushing changes the SHA, so the pin stops resolving and the dependent build
breaks. Patch branches move only forward: one change on top of what is already there.

## Patch branches

| Package | Branch | Upstream base | Pin tag | What it adds |
|---|---|---|---|---|
| `cupertino_http` | [`patch/cupertino_http-3.0.2`](https://github.com/dangthaison91/http-dart/tree/patch/cupertino_http-3.0.2) | 3.0.2 | `cupertino_http-3.0.2+1` | `close()` cancels in-flight tasks instead of throwing; an optional per-request `URLSessionTaskTransactionMetrics` sink (the DNS / TCP / TLS / time-to-first-byte split), which upstream does not surface at all |
| `cronet_http` | [`patch/cronet_http-1.6.0`](https://github.com/dangthaison91/http-dart/tree/patch/cronet_http-1.6.0) | 1.6.0 (release commit `2cb6c12`) | `cronet_http-1.6.0+1` | releases the JNI global references leaked at every terminal callback — the leak overflowed the JNI global reference table (maximum 51,200 entries) and aborted the app; also back-ports `quicHints` from 1.8.0 |

**Branch names** are `patch/<package>-<upstream version>`. The version belongs in the
name because a patch branch serves exactly one upstream version line. Moving to a
newer upstream version opens a *new* branch and leaves the old one untouched, so
anything still pinned to it keeps resolving.

**Pin tags** are `<package>-<upstream version>+<n>`, marking each commit a consumer has
actually pinned. The `+n` counts patch releases on top of that upstream version, the
way pub reads build metadata. A tag keeps a pinned commit reachable even if its branch
is ever deleted by mistake. (Upstream's own tags use a `-v` form, such as
`cupertino_http-v3.0.2`, so these do not collide.)

**Base commits.** A new branch starts at the release commit of the version being
patched, so its diff against upstream reads as exactly the changes and nothing else.

`patch/cupertino_http-3.0.2` deviates: it starts at upstream `master` (`44496f4`)
rather than at the 3.0.2 release commit (`ab3d00d`). It stays that way. The two are
identical for this package — `git diff ab3d00d..44496f4 -- pkgs/cupertino_http` is
empty — so the deviation costs nothing, and correcting it would mean rewriting a
published branch.

## Adding a change

1. Branch off the patch branch.
2. One logical change per commit, in [Conventional Commits](https://www.conventionalcommits.org/) form, scoped to the package: `fix(cronet_http): …`.
3. Add a dated entry at the top of that package's `PATCH.md` changelog, naming the defect and the fix.
4. Open a pull request **into the patch branch** — not into `master`, and not upstream. Merge it without squashing; the per-change history is what the branch exists to keep.
5. Bump the pinned `ref` in the consumer, in its own change. Once that lands, tag the commit it pinned `…+<n+1>`. Tag what was consumed, never the branch tip — the tip usually runs ahead, and a tag on an unconsumed commit makes the pin history lie.

## Changelogs

Each package's `PATCH.md`, on its patch branch, is that package's changelog: newest
entry first, dated, one entry per generation, each naming the defect and the fix.

Upstream's own `CHANGELOG.md` files stay untouched on purpose. Adding entries there
would create a conflict on every port to a newer upstream version, and would buy
nothing — pub never reads a git dependency's changelog.

## Sending a patch upstream

An upstream pull request does **not** come from a patch branch. Cut a fresh
`upstream-pr/<topic>` branch from `upstream/master`, carry over only the change itself,
and delete it once the pull request closes.

What that costs differs sharply by package:

| Package | Cost of the upstream port |
|---|---|
| `cupertino_http` | Small. The base is already upstream `master`, so the commits apply as they stand. Leave out `PATCH.md`. |
| `cronet_http` | A rewrite, not a cherry-pick. `master` carries `cronet_http` 1.9.0 on `jni ^1.0.0`: about 4,300 lines added and 3,400 removed against the 1.6.0 base, with the generated JNI bindings replaced wholesale. The fix has to be re-authored against current `master`, and its comments translated from Vietnamese. |

## Why cronet_http is still on 1.6.0

`datadog_flutter_plugin` 3.x requires `jni ^0.14.2`, while `cronet_http` 1.7.0 and
newer require `jni ^0.15.2`. Until that conflict resolves there is no way off 1.6.0 —
which is why the leak fixes had to be carried here rather than taken from a newer
release.

## Starting a branch for a new upstream version

`master` here is only the repository's landing page. Work never branches from it; new
patch branches come from the `upstream` remote directly:

```bash
git remote add upstream https://github.com/dart-lang/http.git   # once
git fetch upstream --tags
git checkout -b patch/<package>-<version> <upstream release commit>
```

Two consequences follow, and both are fine: `master` may fall behind upstream, because
nothing depends on it being current; and the banner at the top of `README.md` makes
this repository's README differ from upstream's, which is the point of the banner.

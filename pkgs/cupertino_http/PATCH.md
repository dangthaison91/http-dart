# cupertino_http 3.0.2 — patches

Two changes on top of the stock `cupertino_http` 3.0.2 release, kept on the branch
`patch/cupertino_http-3.0.2`. Both are opt-in: a client that passes no new argument
behaves exactly as upstream does.

Every hand-written change carries a `LOCAL PATCH` comment. The one thing that marker
cannot cover is the regenerated `native_cupertino_bindings.dart` — machine-generated
output carries no comment — so identify that part by the two added `ffigen.yaml`
include lines.

## Changelog

Newest first, one entry per generation.

### Generation 2 — per-request URLSession metrics · 2026-08-05

`cupertino_http` does not surface `URLSessionTaskTransactionMetrics` at all, so there
is no way to tell whether a request opened a fresh connection or reused a warm one.
Those metrics carry the only direct evidence: the DNS / TCP / TLS / time-to-first-byte
split, per request.

Added in three commits — regenerate the bindings with a `didFinishCollectingMetrics`
dispatch, wrap the metrics objects in a Dart type, then expose an optional `onMetrics`
sink and a `metricsKeyResolver` on `CupertinoClient`.

Correlation works through `taskDescription`: the resolver names a task from the
outgoing request (an `X-Request-Id` header, say), and the sink receives that name back
with the metrics. When no resolver is supplied, or none matches, the client mints a
fallback key prefixed `cupertino_http.metrics.` — deliberately namespaced so it cannot
collide with another `taskDescription` writer, and recognisable as "not correlatable"
by whoever consumes the sink.

`taskDescription` has to be written while the task is still suspended, so arming
happens before `resume()`.

### Generation 1 — `close()` cancels instead of throwing · 2026-08-05

Upstream's `close()` throws when any task is still in flight, which leaves no clean way
to retire a shared client: the caller has to drain every request first or swallow the
error. It now cancels the in-flight tasks instead, so closing is safe at any moment.

## What is patched

The tree is stock 3.0.2 apart from the changes above; `pubspec.yaml` is untouched.
Check the delta at any time, from a checkout of this branch:

```bash
git diff ab3d00d..HEAD -- pkgs/cupertino_http     # ab3d00d = the 3.0.2 release commit
```

Expect `PATCH.md` plus one hunk per `LOCAL PATCH` site, and the regenerated bindings.
**A hunk you cannot name is a bug** — resolve it before upgrading.

The upstream `pubspec.yaml` needs **no edit** when this package is consumed as a git
or path dependency, even though it declares `path:` dev-dependencies into this
monorepo and its own `dependency_overrides`. Pub honours neither for a dependency —
only the root or workspace package's dev-dependencies and overrides are resolved.
Keeping the pubspec stock is what keeps the diff above clean, so **do not "fix" it**.

## Build shape — a code-assets package, not a Flutter plugin

This surprises people coming from `cronet_http`, and it changed between
`cupertino_http` 2.x and 3.x: 3.0.2 has no `pluginClass` and no CocoaPods podspec. The
native side is built by a Dart **build hook** (`hook/build.dart`) that emits code
assets, so there is nothing for the iOS plugin registrar to pick up and nothing in the
Podfile.

Practical consequences:

- `flutter clean` does not clear the hook's output; the hook's own cache does.
- If the native symbols go missing at runtime, look at the hook, not at the pod install.

### Verifying the native side actually built

```bash
# the built app bundle should carry the framework and name the bindings asset
find build/ios -name 'cupertino_http.framework' -maxdepth 6
grep -o 'package:cupertino_http/src/native_cupertino_bindings.dart' \
  build/ios/iphoneos/Runner.app/Frameworks/App.framework/flutter_assets/NativeAssetsManifest.json
```

## Regenerating the bindings (ffigen)

Generation 2 adds two `ffigen.yaml` include lines and regenerates
`native_cupertino_bindings.dart`. Regeneration is only needed when a change touches the
native surface.

Run it on the **same macOS SDK** the baseline was generated with. Regeneration is
otherwise idempotent, but a newer SDK adds additive, OS-version-gated API — safe to
ship, though it means the diff carries hunks you did not write. Name any such hunk
before committing it.

## Upgrading to a new upstream version

A new upstream version gets a **new branch**; this one is not rebased onto it, because
its commits are pinned by SHA elsewhere.

1. Capture the current delta with the `git diff` above. Every hunk should map to a
   generation in the changelog.
2. Cut the new branch from the new upstream release commit, so its diff against
   upstream stays readable:
   ```bash
   git fetch upstream --tags
   git checkout -b patch/cupertino_http-<new version> <release commit>
   ```
3. Carry `PATCH.md` over.
4. Re-apply each generation in order, smallest first, reading the upstream code around
   each site rather than forcing the patch — upstream may have fixed it (drop the
   generation and record that) or moved the code.
5. Regenerate the bindings only if a generation touches the native surface.
6. Re-test on a device. This is transport code; unit tests alone do not clear it.
7. Update this file: the version in the title, the base commit in the `git diff`
   command, and the changelog.

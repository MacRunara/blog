# The Xcode build-cache consistency problem, and how we solved it

<!-- canonical: https://github.com/macrunara/docs/blob/main/blog/02-xcode-build-cache-consistency/index.en.md -->
<!-- Status: DRAFT. Replace TODO[eng] markers with verified data (mostly from the benchmarks repo) before publishing. -->

*Caching is the only free lunch in CI — and Xcode's cache is the one most likely to poison your build. Here's a map of what is safe to cache across ephemeral runners, what silently corrupts, and the keying scheme we settled on.*

---

Every CI team eventually writes this line:

```yaml
- uses: actions/cache@v4
  with:
    path: ~/Library/Developer/Xcode/DerivedData
    key: deriveddata-${{ github.sha }}
```

And every CI team eventually deletes it, after a week of flaky builds that only happen on CI, never locally, and always disappear when you "clear the cache". The uncomfortable truth: **DerivedData is not a cache you can safely move between machines — or even between builds.** It is a build *artifact*, full of absolute paths, fingerprints and timestamps, and Xcode treats it as authoritative. Restore it wrong and the compiler happily consumes stale modules while your source tree says otherwise.

This post is the consistency model we use on Macrunara's ephemeral runners: a tiered answer to "what may cross a VM boundary, keyed by what, verified how".

## Tier 0: what never crosses the boundary

- **DerivedData (as a whole).** Object files and module caches inside it reference absolute paths and are keyed to the exact toolchain build. On ephemeral VMs the restore cost + corruption risk exceeds the savings. We accept the cold-build penalty and instead attack cold-build time itself (fast nodes, prewarmed image — see [part 1](https://github.com/macrunara/docs/blob/main/blog/01-m4-cluster-architecture/index.en.md)).
- **Anything touched by code signing.** Keychains, provisioning profiles, intermediate signing state. Ephemeral by design (see [part 3](https://github.com/macrunara/docs/blob/main/blog/03-ephemeral-runner-ios-signing/index.en.md)).
- **`~/Library/Caches` wholesale.** Indiscriminate cache dirs mix safe data (downloaded archives) with unsafe data ( fingerprints, locks). Cache *specific subtrees*, never the forest.

## Tier 1: source-level caches — always safe

These contain *inputs*, not *outputs*, so consistency reduces to ordinary content-addressing:

| Cache | Path | Key |
|---|---|---|
| SwiftPM checkouts | `.build` is output, but `~/Library/Caches/org.swift.swiftpm` plus the resolved manifest | `Package.resolved` hash + Swift version |
| CocoaPods | `Pods/` + `~/.cocoapods` | `Podfile.lock` hash |
| Node (RN) | `node_modules` (or yarn cache) | `yarn.lock` hash + node version |
| Gradle (Android) | `~/.gradle/caches`, `~/.gradle/wrapper` | `*.gradle*` + `gradle-wrapper.properties` hashes |

Two details that matter:

1. **Key on lockfiles, include the toolchain in the key.** `spm-${{ runner.os }}-${{ hashFiles('Package.resolved') }}-${{ env.XCODE_VERSION }}`. A cache built with Swift 6.1's compiler is not valid input for Swift 6.2's, and "sometimes it works" is exactly the failure mode you don't want.
2. **The image version is part of the key.** Our nodes run versioned golden images (`macos-mobile v1.2`). When the image rolls forward, old caches invalidate cleanly instead of degrading mysteriously. The image version is exported into every job's environment — use it.

## Tier 2: build-output caches — only with a correctness oracle

If you must cache build outputs (some of our customers' monorepos genuinely need it), you need an *oracle* that can tell a valid cache entry from a poisoned one:

- **Xcode's own fingerprinting (Xcode 15+).** `xcodebuild` emits explicit module dependencies; Swift driver fingerprints cover file content + flags. Caching is safe *iff* the key includes: source hash, toolchain build version, image version, and every build setting that leaks into fingerprints (`SWIFT_VERSION`, optimization level, sanitizer flags).
- **Remote build caching systems.** [Tuist's cache](https://tuist.io) (content-addressed, framework-level) and Bazel's remote cache are the two serious options for large iOS codebases. Both replace "trust the directory" with "trust the hash" — which is the entire point. We run Tuist-cache-friendly nodes <!-- TODO[eng]: confirm we can say this / link docs -->, and for Bazel shops the story is even better because correctness is Bazel's core promise.
- **sccache-style compilers** don't cover Swift's whole pipeline; treat them as a C/C++/Rust accelerator, not an Xcode solution.

## The benchmark that justifies the model

Theory is cheap; we measured the tiers on the cluster (full methodology in the [benchmarks repo](https://github.com/macrunara/benchmarks)):

| Strategy | Cold | Warm | Correctness incidents / 100 builds |
|---|---|---|---|
| No cache (Tier 0 only) | <!-- TODO[eng] --> 420s | 420s | 0 |
| Tier 1 caches (lockfile-keyed) | 420s → <!-- TODO[eng] --> ~240s | ~180s | 0 |
| Tier 1 + naive DerivedData tar | ~180s | ~90s | <!-- TODO[eng]: measured incident rate before we banned it --> |

The middle row is the sweet spot: most of the wall-clock win, none of the flakiness. The third row is why "just cache DerivedData" is banned in our templates.

## Practical checklist (steal this)

1. Cache **inputs** (SPM/Pods/node/gradle), never whole DerivedData.
2. Key = lockfile hash + toolchain version + image version.
3. Set `fail-on-cache-miss: false` — a cache is an optimization, never a correctness dependency.
4. Keep restore-keys *prefix-free* for anything build-output-shaped; fallbacks here are how stale modules sneak in.
5. Measure: log cache-hit rate and cold/warm build times per pipeline. A cache you don't measure is a superstition.

## What's next

**Part 3** — code signing on machines that are destroyed after your job: why ephemerality is a security *feature*, and the exact keychain dance that makes it work.

---

*Macrunara runs your GitHub Actions jobs on ephemeral Mac mini (M4/M6) VMs with this cache model built into the node image. The [flutter](https://github.com/macrunara/flutter-example) and [ios-swift](https://github.com/macrunara/ios-swift-example) example repos show the Tier-1 setup in action; the landscape of caching tools lives on [awesome-macos-ci](https://github.com/macrunara/awesome-macos-ci).*

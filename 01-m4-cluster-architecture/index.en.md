# Running GitHub Actions CI on a Mac mini cluster (M4 and M6): architecture, virtualization and scheduling

<!-- canonical: https://github.com/macrunara/docs/blob/main/blog/01-m4-cluster-architecture/index.en.md -->
<!-- Status: DRAFT. All TODO[eng] markers must be replaced with verified numbers by the infra team before publishing. -->

*How we turned a rack of Mac minis into ephemeral, one-job-one-VM runners for GitHub Actions — and the engineering decisions that made it boring (in a good way).*

---

Running iOS CI in the cloud has an annoying fundamental constraint: **macOS only runs on Apple hardware**. There is no `docker run macos`. Every "macOS runner" you have ever used — hosted or self-built — bottoms out at a physical Apple machine somewhere. The question is never *whether* there is a Mac, but *how honestly and efficiently you slice it*.

This post describes how the Macrunara cluster works: the hardware layout, why we virtualize with Virtualization.framework instead of running jobs on bare metal, and how a job travels from `runs-on: [macos, arm64, macrunara]` in your workflow file to a fresh VM on a Mac mini (M4 or M6) in our rack.

## The hardware: boring on purpose

Our compute units are the **Mac mini M4 and M6** — <!-- TODO[eng]: exact configs for both generations (chip variant / RAM / SSD) --> the best perf-per-watt-per-rack-unit we've found for CI. Both generations run the same node image and sit in one pool; the scheduler doesn't distinguish them, and neither does your workflow. Each rack shelf holds <!-- TODO[eng]: N --> minis behind a shared PSU shelf and a 10GbE fabric <!-- TODO[eng]: confirm switch model -->, with every machine on a managed PDU so we can hard-power-cycle a wedged node from the control plane.

Two physical-ops lessons we learned the hard way:

- **Cooling is a front-to-back problem, not a temperature problem.** Mac minis ingest air from the bottom rim. Rack them flat with a gap and they thermally throttle under sustained `xcodebuild` load; rack them on edge with sealed channels and they sustain full boost indefinitely. Our cold-build benchmark numbers (see the [benchmarks repo](https://github.com/macrunara/benchmarks)) only hold because of this.
- **SSDs wear.** CI is a write-heavy workload — every ephemeral VM writes and deletes tens of GB per job. We monitor TBW per node and rotate machines out before they hit the vendor threshold. <!-- TODO[eng]: current wear numbers / rotation policy -->

## Why virtualize at all?

Running jobs directly on the host OS is simpler, and it's what most self-hosted setups do. It's also a security and reproducibility dead end: build N leaves behind keychains, derived data, pods caches and environment variables for build N+1 to trip over — or to steal. A CI runner is executing *arbitrary code from other people's pull requests*.

So every job on Macrunara gets a **fresh, ephemeral VM**:

```
job claimed → clone VM from golden image → boot (~TODO[eng]: Xs)
            → register JIT runner with GitHub → run job
            → destroy VM → disk back to the pool
```

macOS on Apple Silicon has a first-class virtualization API — **Virtualization.framework** — and we manage VMs with a tart-style toolchain on top of it <!-- TODO[eng]: confirm what we can name publicly; if we built on tart, credit it here -->. Key properties we rely on:

- **Copy-on-write clones.** Booting a VM doesn't copy the golden image; it creates an APFS-backed clone that only stores deltas. Boot-to-runner-ready is seconds, not minutes.
- **Hardware-backed isolation.** Each VM gets its own kernel, keychain and secure enclave view. When the job ends, the VM — including every secret it ever touched — ceases to exist. (More on why this matters for code signing in [part 3 of this series](https://github.com/macrunara/docs/blob/main/blog/03-ephemeral-runner-ios-signing/index.en.md).)
- **One job, one VM, no exceptions.** No warm pool of "mostly clean" machines. Cleanliness is not a state we maintain; it's a state we *recreate* every time.

## The golden image

The base of every VM is our `macos-mobile` image (currently v1.2): macOS 26 + Xcode 26.x + Flutter 3.47 + Android SDK 35 + JDK 17 + Node 20 + CocoaPods, with licenses pre-accepted and the simulator runtimes pre-warmed. The full manifest is public in our [quickstart repo](https://github.com/macrunara/quickstart).

Two design decisions worth stealing:

1. **Bake, don't bootstrap.** Every minute spent installing toolchains in a job is a minute billed and a minute of flakiness (network, upstream outages). If a dependency is used by >20% of jobs, it belongs in the image. Everything else is the job's own business.
2. **Version the image like software.** Images are immutable artifacts with semver tags and a changelog. Jobs pin behavior implicitly by label; we roll the fleet forward image version by image version, and a bad image is a rollback, not an incident. The `flutter: command not found` class of failure disappears when "which environment am I on?" has a one-word answer.

## Scheduling: from `runs-on:` to a booted VM

GitHub Actions' self-hosted runner model is pull-based, which maps beautifully onto ephemeral VMs. The control loop looks like this:

```
                ┌──────────────┐
   workflow ──▶ │ GitHub       │ queued job with labels
                │ Actions API  │ [macos, arm64, macrunara]
                └──────┬───────┘
                       │ poll / webhook
                ┌──────▼───────┐
                │ Orchestrator │ capacity check, node pick
                └──────┬───────┘
                       │ clone golden image, boot VM
                ┌──────▼───────┐
                │ JIT runner   │ registration token via API,
                │ registration │ single-use, expires with the VM
                └──────┬───────┘
                       │ job runs inside the VM
                ┌──────▼───────┐
                │ VM destroyed │ deltas discarded, node reclaims disk
                └──────────────┘
```

Details that matter in practice:

- **JIT runners, not long-lived registrations.** We request a just-in-time runner registration from the GitHub API for each job. The runner binary inside the VM connects out, claims exactly one job, and the registration is useless afterwards — there is no long-lived runner token sitting on a disk anywhere.
- **Labels are a contract, not a hint.** `[macos, arm64, macrunara]` is the exact triplet our orchestrator matches. A job asking for `self-hosted` or `macos-latest` will never be picked up — it queues forever *by design*, which is why the first FAQ in our docs is about labels.
- **Capacity is sized for the burst, not the average.** CI traffic is spiky (Monday morning merges, release tags). We keep headroom for the p95 queue depth <!-- TODO[eng]: current headroom/target pickup latency -->, and the orchestrator's most important metric is *queue wait time*, not CPU utilization.
- **Egress goes through an internal proxy.** VMs have no direct internet route; package registries, the Actions API and artifact uploads are reached through a caching proxy layer. This is why your first CocoaPods CDN pull on our nodes can be slower than you expect — and why the second one isn't.

## What the numbers look like

From our public [benchmarks](https://github.com/macrunara/benchmarks) (methodology and raw CSVs in the repo): a medium-size Swift app cold-builds in ~7 minutes on an M4 node vs ~12 on GitHub's hosted M1, and VM boot adds only <!-- TODO[eng]: measured VM boot overhead --> seconds to job startup. M6 node numbers land in the next monthly benchmark run — <!-- TODO[eng]: fill in once M6 is measured -->. We'll keep publishing these monthly — the whole point of this series is that the claims are checkable.

## What's next

- **Part 2** — the build-cache consistency problem: what you may and may not cache across ephemeral runners, and how we keep incremental builds correct.
- **Part 3** — doing iOS code signing safely on infrastructure where the machine is thrown away after your job.
- **Part 4** — the TCO spreadsheet: hosted vs. managed vs. self-built macOS CI, with real numbers.

*Macrunara is a dedicated Mac mini M4/M6 runner service for GitHub Actions. If you want to poke at this architecture yourself, the [quickstart](https://github.com/macrunara/quickstart) gets a job onto our cluster in about three minutes — and the [awesome-macos-ci](https://github.com/macrunara/awesome-macos-ci) list maps the whole landscape, competitors included.*

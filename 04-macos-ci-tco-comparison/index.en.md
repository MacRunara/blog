# The real cost of macOS CI: hosted vs. managed vs. self-built, a TCO spreadsheet

<!-- canonical: https://github.com/macrunara/docs/blob/main/blog/04-macos-ci-tco-comparison/index.en.md -->
<!-- Status: DRAFT. All rates are TODO[pricing] placeholders — verify every price against
     vendor pages on publish day (they change), and keep the model honest: this post is
     only credible if the numbers are checkable. -->

*Most "macOS CI is expensive" takes compare sticker prices and stop there. The honest comparison has four lines: compute, queue time, maintenance hours, and the cost of being slow. Here's the full model — with the spreadsheet logic exposed so you can plug in your own numbers.*

---

## The workload model (change these to match your team)

Everything below derives from four inputs. We use a mid-size iOS team as the running example:

| Input | Example value |
|---|---|
| Engineers pushing code | 10 |
| CI jobs per workday | 60 (push + PR + nightly) |
| Average job duration (M1-class) | 12 min |
| Workdays per month | 21 |

That's **~756 build-hours per month** on M1-class hardware. Every price line below is a function of this number — plus the two lines everyone forgets.

## Line 1: Compute (the sticker price)

| Option | Pricing model | Effective monthly compute cost for 756 build-hours |
|---|---|---|
| GitHub-hosted `macos-14` (M1) | **Free for public repos.** Private: billed minutes, macOS multiplier ×10 vs Linux <!-- TODO[pricing]: current per-minute rate → ~$X/min --> | Public: **$0** · Private: ~$<!-- TODO[pricing] -->/mo |
| GitHub larger runner (M-series) | Per-minute, ~2× the standard macOS rate <!-- TODO[pricing] --> | ~$<!-- TODO[pricing] -->/mo |
| MacStadium / MacWeb (dedicated) | Per-machine monthly, ~$<!-- TODO[pricing] -->/Mac mini | N machines for your p95 concurrency: ~$<!-- TODO[pricing] -->/mo |
| Self-built (own Mac minis) | CapEx ~$<!-- TODO[pricing] -->/mini amortized 3y + colo/power | ~$<!-- TODO[pricing] -->/mo + your time (Line 3) |
| **Macrunara** | Flat per node per month (M4/M6) <!-- TODO[pricing]: list price --> | Nodes sized for p95 concurrency: ~$<!-- TODO[pricing] -->/mo |

The sticker price already hides a subtlety: **billing granularity vs. job shape.** Per-minute billing punishes long queues of small jobs (each pays its own setup overhead); flat per-node pricing punishes idle nights and weekends. Your job histogram decides which hurts more — plot it before choosing.

## Line 2: Queue time (the price nobody lists)

A CI minute and a *developer's* minute are not the same currency. When a PR waits 25 minutes for a runner, the author context-switches; research on task-switching puts the reload cost at 10-23 minutes of focus. <!-- TODO[eng]: cite a specific study or drop the numbers and keep the qualitative claim -->

Quantify it for the model:

```
monthly queue cost = (avg queue wait + setup overhead per job)
                   × jobs/month
                   × context-switch penalty factor
                   × loaded hourly cost of an engineer (~$<!-- TODO[pricing] -->/h)
```

For our example team at a conservative 25% penalty factor, **8 minutes of average queue+setup overhead ≈ $<!-- TODO[eng] -->/month in lost focus** — often larger than the entire compute bill. This is the line where "free" hosted runners turn out to be the most expensive option for a private-repo team, and where per-node flat pricing (no queue inside your capacity) earns its keep.

## Line 3: Maintenance hours (self-built's iceberg)

Owning the machines means owning: OS and Xcode upgrades across the fleet, runner binary updates, certificate renewals, disk wear (SSDs are consumables under CI load), network/PDU/cooling, and the 2 a.m. "runner went offline" pages. Industry rule of thumb for a small self-built macOS fleet: **0.2–0.5 FTE**. At a loaded salary, that's $<!-- TODO[pricing] -->–$<!-- TODO[pricing] --> per month — typically more than the hardware itself.

This is also where virtualization discipline pays: golden images + ephemeral VMs (see [part 1](https://github.com/macrunara/docs/blob/main/blog/01-m4-cluster-architecture/index.en.md)) turn "patch the fleet" from a snowflake-herding exercise into an image rebuild. If you self-build, budget this or you will quietly pay it in incidents.

## Line 4: The cost of being slow

Faster hardware doesn't just save billed minutes — it shrinks Line 2 and changes developer behavior. When CI is 12 minutes, people push and wait; when it's 40, they batch changes, PRs get bigger, review quality drops, and release cadence stretches. This line refuses precise measurement, but any honest TCO acknowledges it: **build duration is a compounding input to team velocity, not just an invoice line.** (Our own measured M4-vs-M1 deltas are public in the [benchmarks repo](https://github.com/macrunara/benchmarks) — use them or your own measurements.)

## The comparison, assembled

| Cost line / month (example team) | GitHub hosted | Larger runner | Dedicated (MacStadium) | Self-built | Macrunara |
|---|---|---|---|---|---|
| Compute | $<!-- TODO --> | $<!-- TODO --> | $<!-- TODO --> | $<!-- TODO --> | $<!-- TODO --> |
| Queue time (lost focus) | $<!-- TODO --> | $<!-- TODO --> | ~$0 within capacity | ~$0 within capacity | ~$0 within capacity |
| Maintenance (0.2–0.5 FTE) | $0 | $0 | $0 | $<!-- TODO --> | $0 |
| **Total** | **$<!-- TODO -->** | **$<!-- TODO -->** | **$<!-- TODO -->** | **$<!-- TODO -->** | **$<!-- TODO -->** |

We deliberately publish this with the model exposed rather than a "we're cheapest" verdict, because the honest answer is conditional:

- **Public open-source repos**: GitHub-hosted is free and fine. (That's why our OSS program competes on *speed*, not price.)
- **Small private teams, low volume**: hosted per-minute usually wins until queue pain shows up in Line 2.
- **Sustained volume or latency-sensitive teams**: flat per-node pricing (dedicated or Macrunara) wins once the queue and maintenance lines dominate — in our modeling, somewhere around <!-- TODO[eng]: break-even build-hours --> build-hours/month.
- **Self-built** wins only if you *enjoy* running hardware or have compliance constraints that demand it. Price the FTE honestly before deciding you do.

## Reproduce this for your team

1. Pull your last 90 days of Actions usage (Settings → Billing, or the usage API) to get real jobs/day and minutes/job.
2. Compute Line 2 with your actual p50 queue wait from Actions metrics.
3. Plug your salary bands and vendor quotes into the four lines.
4. If the flat-node column wins, [try us on one repo](https://github.com/macrunara/quickstart) — the fork-and-run examples take three minutes, and the [benchmarks](https://github.com/macrunara/benchmarks) show what we measured, reproducibly.

---

*Part 4 of 4. The series: [cluster architecture](https://github.com/macrunara/docs/blob/main/blog/01-m4-cluster-architecture/index.en.md) · [build-cache consistency](https://github.com/macrunara/docs/blob/main/blog/02-xcode-build-cache-consistency/index.en.md) · [signing on ephemeral runners](https://github.com/macrunara/docs/blob/main/blog/03-ephemeral-runner-ios-signing/index.en.md). Landscape map: [awesome-macos-ci](https://github.com/macrunara/awesome-macos-ci).*

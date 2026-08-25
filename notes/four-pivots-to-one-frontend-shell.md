# Four pivots to one frontend shell

Three months turning a fleet of independently deployed React apps into one composed platform. The plan changed four times.

**Status, because it changes how you should read everything below**: 8 apps of ~13 are absorbed and live in production. This is a mid-flight report, not a retrospective on a finished thing. Three of the four pivots trace to one root cause I'll name up front, since the article's honest lesson isn't "we learned by building": **we put a pre-release bundler underneath the composition layer, and paid for it three times.**

## The pain, in three layers

1. **Release coordination.** 10 to 20 library bumps a week, times N apps. Hundreds of manual actions a week: dependency PRs, release tickets, manual promotions. A meaningful fraction of an engineer's year, permanently.
2. **UX.** Full document reload between apps. Lost scroll, lost state. Users perceive N disconnected products, because that is literally what they are.
3. **Tech debt.** React, the design system and the auth library re-bundled per app. Framework drift. No cross-app E2E test, no cross-app state.

Three hard constraints: apps **cannot be merged** (separate repos, ownership, deploys and URLs stay), the migration must be **progressive and non blocking** for feature work, and each phase is **opt-in per app**.

That last one kills most of the elegant answers before you start.

## Plan 1: audit, then cheap, then maybe expensive later

The committed sequence was: normalize the fleet, spike a shared navigation Web Component, then ship shared singletons over CDN (design system, auth, translations), and defer the runtime shell to a decision a year out. Sensible. Cheapest thing first.

The audit found drift across 13 React apps distributed **5 compliant / 5 mid / 3 heavy**, with 11 of 13 diverging on TypeScript path aliases alone, and 9 systematic patterns fixable by a single codemod or template change.

I originally wrote that up as "bimodal, not much in between". The numbers say otherwise and I should have read my own table: the middle is tied for the largest group. Half the fleet being mid-drift is a more common situation than a clean split, and more useful to say.

Then the execution number that surprised me most: **12 apps normalized in five days.** The estimate had been two to three engineer-months. Codemod-driven fleet work is drastically cheaper than it looks when the drift is systematic, which is the actual argument for auditing first.

## Pivot 1: what changed the plan

Five days after the audit completed, we cancelled the Web Component spike and the CDN singletons outright and started the expensive option immediately.

**The conversation that forced it.** Transitions between apps are frequent and the journeys are deep multi-hop chains. There is no isolatable subset of apps you can do first. Which means **SPA continuity between apps is the requirement**, not a nice-to-have. CDN singletons deduplicate bytes, they do not give you that. We would have spent months buying the wrong thing.

**The talk that made the answer credible.** At React Paris, 26-27 March 2026, Taylor Quinn-Bohmann (Staff Software Engineer at Tesla) gave [Share Once, Ship Everywhere: Federated UI at Scale](https://www.youtube.com/watch?v=Rlq7S2xcI_Q): Module Federation to share live, data-rich React components at runtime, without synchronized releases or dependency headaches.

What made it useful wasn't the architecture, it was the problems they had already hit and solved. A blog post shows you the final diagram; a good talk gives you the list of things that broke on the way there. So when the journey conversation forced the question seven weeks later, Module Federation wasn't a research project, it was a thing I'd watched someone run well above our scale.

I want to be careful about the causality, because my own contemporaneous notes from the pivot day credit only the conversation. The talk isn't in them. Reconstructing it now, it's what made "start the expensive option today" feel like a two-month decision rather than a two-year one, but I can't claim it flipped anything on paper. Treat that as memory, not record.

The audit still paid for itself, just not the way it was scoped to. With the fleet aligned (same React major, canonical build config, canonical auth client), the estimate for the shell **roughly halved**. That is a re-estimate, not a measured saving. Nobody has costed what it actually took.

## Pivot 2: the tooling wasn't there

First spike, week one. Both Module Federation plugins for our bundler crashed on the version we were on. Not "had rough edges", crashed.

We replaced them with a native dynamic `import(url)` plus library mode on each remote. Single-file ESM bundle per remote, 1.6 KB, loaded into the host with Suspense and an error boundary. Three of seven acceptance criteria proven in week one.

**A spike that reveals your tooling is unusable is a successful spike.** It cost a week and saved us from finding out in month four.

## Pivot 3: the constraint I quietly relaxed

Two weeks later, with singleton sharing proven live (zero reload, shared auth, store, feature flags, design system, one React root), we abandoned runtime composition entirely and moved to build-time composition in a frontend monorepo, where the shell lazy-imports each app as an internal package.

Here is the part that would be easy to tell flatteringly, and I'm not going to.

The analysis had **already settled this a day earlier**, on paper: given a hard separate-deploy constraint, runtime composition is required regardless of repo topology. The axes are orthogonal. That was written down.

So how did build-time composition get chosen the next day ? Because I **revised the constraint**. I decided the product objective (zero reload) took precedence over strict deployment independence. Once you relax that, build-time composition is clearly cleaner, and it is: one lockfile, one version policy, no shared-singleton negotiation, no runtime version contract.

Three days later we reversed it and went back to Module Federation, on a different bundler.

**The transferable lesson is not "we built it to learn the constraint binds".** The analysis had the right answer before anything was built. The lesson is: *the requirement you quietly relax to make an option viable is the one that comes back and kills it.* If you catch yourself revising a constraint you had written down as hard, that is the moment to stop and get it re-agreed explicitly, not to keep going because the new option looks cleaner.

## Pivot 4: the bundler, finally named

The return to Module Federation came with a correction I'd rather not have needed.

We had justified pivot 3 partly as "runtime Module Federation isn't mature enough". That was a **mis-attribution**, and our own retrospective says so. MF is Webpack-native and production-mature on the webpack-family bundlers. The blocker was never MF; it was that our bundler was pre-release and had *dropped* native MF support, deferring to a still-green port.

Trace it back and three of the four pivots have one cause. Pivot 2, the plugins crashing. Pivot 3, "runtime composition isn't ready", which was the bundler not being ready. Pivot 4, the correction. And the bundler's MF status was a public fact the whole time, evaluable in an afternoon before any of it.

That's the boring-technology argument in its purest form. The novel component sat in the *foundation* of the composition layer, where it bought us nothing, and it cost three plan reversals.

## What it bought, measured

**Method first, because n matters.** One run per topology, cold cache, driven through the browser devtools protocol, on a fast low-RTT network. The standalone side had its pod uninstalled, so its `index.html` was served locally and **+52 ms was added to its timings** to compensate a TTFB artefact. Both runs stop at the SSO redirect, so **no post-auth Core Web Vitals were measured**. The two sides are re-ported, not byte-identical, and use different bundlers with different chunk splitting. Totals are comparable, "own code" figures are not.

**Cold first load, federation loses:**

| | Federated | Standalone (adjusted) | Delta |
|---|---|---|---|
| Bytes (brotli) | 1408 KiB | 1221 KiB | +187 KiB (+15%) |
| Serial waterfall depth | 7 levels | 3 levels | +4 |
| All app code available | 370 ms | ~209 ms | +161 ms |

The +161 ms is the lab number and it's the least interesting one. **The waterfall-depth gap scales with round-trip time: on 4G at ~150 ms RTT, those 4 extra serial levels are roughly +600 ms.** That's the figure that would actually change someone's decision.

**Cross-app navigation, federation wins, not close:** an in-shell hop to another app cost **10.4 KiB** and no document teardown, against **1251 KiB** and a full reload standalone. React state, store and query cache all survive. (A second measurement, ~41 KiB, counts one app's own code excluding the singletons it provides; different accounting, don't read the two as a range.)

Two findings I did not expect:

**The remote is 96% shared libraries.** A 962 KiB remote contained 920 KiB of singletons it *provides* to the share scope (737 KiB of that is the design system) and 41 KiB of its own code. Dedup works. But the *remote* is the provider, not the host, so the design system only starts downloading at hop 8.

**The standalone topology has zero cache reuse.** Each standalone app ships the design system under its own content hash. Three standalone apps means the same design system downloaded three times, ~2.3 MB.

And the conclusion that mattered most: **the first-load penalty is not intrinsic to Module Federation.** Roughly 94 KiB and 3 to 4 of the 4 extra hops come from eagerly registering every remote at boot, before the app renders. That's a bootstrap choice, and it scales linearly with absorbed apps (94 KiB at 4 apps, ~470 KiB projected at 15) while being paid on every page load regardless of route.

**The cost this table doesn't show, and should.** We collapsed a dozen independent failure domains into one. Before, a broken app was a broken app. After, one bad remote registration is the entire platform behind a single error page. That is the central architectural trade of this migration, and measuring it in bytes and milliseconds while leaving availability out of the ledger is exactly the mistake I'd call out in someone else's write-up.

**And we never re-measured layer 1.** The migration was justified on release-coordination pain quantified in engineer-time. Everything above is measured in KiB and ms. With 8 of 13 apps absorbed we have also *added* plumbing: a monorepo, a second bundler in the fleet, a per-environment federation manifest pipeline, a bespoke deploy path, and a new class of production incident. Whether that nets out on the original metric, I don't know. We haven't measured it. That's a gap, not a rhetorical concession.

## The outage

Registering one new remote took the whole platform down: production behind a single error page, and test and staging before it.

Mechanism: the shared-singleton config treated `eager` as a **per-entry preference**. Only React and React-DOM were host-provided. Every other singleton in the host's synchronous boot tree was lazy, therefore **order dependent**. Registering one more remote flipped the order, and a remote ended up providing the shell's router.

**We had identified this exact risk and then accepted it back.** The build-time decision, nine weeks earlier, listed as its number-one rationale that it *structurally eliminates the shared-singleton version contract*: one lockfile, single-version policy, no negotiation. We chose that architecture partly to kill this risk, reverted three days later for deploy independence, and carried no mitigation across the reversal. The mitigation was invented after the outage.

Seven things worth stealing:

**A version tie is the dangerous case, not a mismatch.** A mismatch logs a warning and carries on with one instance. A *tie* silently defers to load order. Every remote was on the identical router version, which is exactly why it broke. Note this depends on running `strictVersion: false`, and under `strictVersion: true` a mismatch throws instead. `false` is still the right call, it stops a patch skew taking a route down, but the price is that your only signal is a console warning in a browser nobody instruments.

**Non-prod broke 58 minutes before production.** Test and staging were already dark with the identical defect when the change was promoted. That's the process finding, and it's worse than the technical one: there was no gate that loads the composed shell against the target environment's **pinned** manifest before promoting. The defect isn't in any artifact, it's in the *combination*, so only the combination can be tested.

**We don't know the detection time.** It was never quantified, and that's structurally hard here: every asset returned 200, the origin was healthy, the failure was entirely in the browser. A client-side composition failure produces no server-side signal at all. The missing controls are error reporting on the shell's root boundary and a synthetic that loads the composed shell per environment.

**The throw location decides the blast radius.** The failing call sat above the app boundary, so the root error boundary ate everything. But be careful generalising this: what broke was the *router*, and if the router can't resolve there are no segments to route to, so a per-segment boundary would have caught nothing. Boundary placement would have contained the three providers *after* it in the chain. It structurally cannot contain the router.

**Fixing the reported symptom cost a second cycle.** Making only the router eager shipped, then immediately surfaced the next provider in the chain. Four providers, same mechanism. Fix the rule, not the instance.

**Dev could not reproduce it, and never will.** Dev follows a moving channel, so every remote comes from one commit. Pinned environments mix builds from weeks apart. That asymmetry *is* the bug class, and it's a permanent property of the topology, not a gap you close by testing harder on dev.

**The rollback lever needed no deploy.** In a per-environment pinned-manifest topology, the manifest *is* the rollback surface, and it is far faster than shipping anything: drop the offending remote keys and the platform comes back. Worth knowing. Also worth saying that ours was a hand-edit on a CDN, done live by three people, with no review and no audit trail, immediately diverging from the gitops source of truth. That's a second incident queued behind the first. The manifests are already versioned per environment, so a reviewed one-command revert is small work and a large reduction in 3am risk.

**On the fix.** The repair was a code change: make every host singleton eager, shipped across two cycles. What prevents recurrence is a CI assertion that every host entry is eager, no remote entry is, and both maps carry the same names. Don't let me imply a lint rule fixed a production outage.

And that assertion is necessary but not sufficient. It enforces one rule learned from one failure, on a config file. The failure *class* is runtime composition of independently versioned artifacts, and its next instance won't be an eagerness mistake, it'll be a peer-dep skew or a remote built against a design-system major the host doesn't have. No config lint sees those. The control that matches the class is a smoke test that boots the composed shell against each environment's real pinned manifest and asserts it renders, gating promotion. That's also the control that would have caught the 58-minute window.

## What I'd tell someone starting this

- Write down which constraints are hard **before** you evaluate options. Then notice when you start revising one to make an option work, because that's the failure mode, not a clever insight.
- Keep the novel technology out of the foundation. A pre-release bundler under the composition layer cost us three plan reversals and bought nothing.
- Normalize the fleet first even if you throw the plan away. 12 apps in five days, and it roughly halved the estimate for what came next.
- Build the thing you're unsure about and deploy it for real, but don't mistake that for having no analysis. Ours had the right answer a day before we built the wrong thing.
- Measure both topologies on the journey users actually take, publish your n, and put availability in the ledger next to the bytes.
- Go to conferences and listen for the failure list rather than the architecture.
- Your composition layer will fail in ways your dev environment structurally cannot reproduce. Gate promotion on the composed shell against each environment's pinned manifest.

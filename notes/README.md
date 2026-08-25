# notes

Things that cost me a day, written down so they cost you less.

Frontend architecture, design systems, micro-frontends, and where AI assisted development actually breaks down. No schedule, no newsletter. I write one when something was expensive enough to be worth it.

- [Four pivots to one frontend shell](four-pivots-to-one-frontend-shell.md)
  Three months turning a fleet of independently deployed React apps into one composed platform. The plan changed four times, three of them tracing to the same root cause. Includes the benchmark, the outage, and the constraint I quietly relaxed.

- [Saying no is most of the design system job](saying-no-is-most-of-the-design-system-job.md)
  Eight rules I enforce in review on a design system consumed by a dozen apps, each with the case that produced it, and the two where a reviewer showed me the rule was weaker than I'd written it.

- [Subpath imports cut both ways](module-federation-subpaths-cut-both-ways.md)
  The same import saves you 900 module resolutions or silently duplicates your library across every remote. The variable isn't who owns the package, it's whether the specifier matches a share key.

- [Green CI, white page](green-ci-white-page.md)
  A CJS-only transitive dependency plus an externalised React equals a shim that throws in the browser. Nothing in the pipeline executes the built bundle, so every gate passes. Also: I diagnosed it from the manifest and was wrong twice.

- [I measured my token spend, then got the conclusion wrong](what-an-ai-coding-agent-actually-costs.md)
  33 sessions out of 682 caused 86.8% of my cost. I read that as "where the waste is". One division shows it's mostly "where the work is", and the real lever is worth ~20%, not 86%.

Found something wrong in one of these ? Open an issue, I'd rather be corrected than quoted.

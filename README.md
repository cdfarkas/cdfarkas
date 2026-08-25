## Clément Farkas

Frontend architect, France. Design systems, micro-frontends, and making AI assisted development reliable at team scale instead of impressive in a demo.

Ten years in, my center of gravity moved from *writing the feature* to *making the feature cheap to write for everyone else*: the component nobody has to fork, the migration that lands across twenty repos without a war room, the CI signal you can actually trust.

## What I work on

**Design systems as a product.** A React component library consumed by a dozen or so downstream apps. Composition over props, a deliberately small API surface, and the versioning discipline that stops one component change from breaking every consumer at once. The hard part was never the components, it's saying no to the twelfth boolean prop and shipping a layout instead.

**Micro-frontend plumbing.** Module Federation share scopes, bundler migrations, and the class of bug that only exists in the built artifact: every gate green, white page in production. jsdom won't catch it for you and neither will a passing type check. You debug those from the bundle, not from the source.

**Fleet scale migrations.** Canary one representative repo through real CI, read the signal, then fan out. Never the reverse, however mechanical the change looks. The change that "obviously can't break anything" is the one that takes down six apps at 6pm.

**AI assisted engineering, industrialised.** Custom agents, hooks and skills, multi-agent orchestration for multi-repo work, and a knowledge layer that survives across sessions instead of being re-derived every morning. Most of the value sits in the guardrails, the checks that stop a confident wrong answer from reaching a branch, not in the prompts.

## How I work

- **Reuse before create.** A new util that duplicates an existing one is a bug, not a contribution.
- **Derive from the source of truth**, never from a stale port or a plausible memory of how it used to work.
- **Verify the claim, not the vibe.** "Tests pass" means the full suite ran and I read the output.
- **Prove it rather than assume it.** If a number worries me, I'd rather measure it than argue about it.
- **Docs ship in the same commit** as the code they describe, or they're already wrong.
- **Own the mistake fast.** I correct my own reviews in public. It's what makes the rest of the directness worth anything.

Signature line, for the people I work with: *plus d'approves et moins de blabla*.

## Writing

Things that cost me a day, written down so they cost you less. All in [notes/](notes/).

- [Four pivots to one frontend shell](notes/four-pivots-to-one-frontend-shell.md) - three months of micro-frontend migration, the four pivots, the benchmark, and the outage.
- [Saying no is most of the design system job](notes/saying-no-is-most-of-the-design-system-job.md) - eight rules I enforce in review, each with the case that produced it, and where they break.
- [Subpath imports cut both ways](notes/module-federation-subpaths-cut-both-ways.md) - the same import saves 900 module resolutions or silently duplicates your library, depending on your share scope.
- [Green CI, white page](notes/green-ci-white-page.md) - a bundler bug no gate can see, and me getting the diagnosis wrong twice.
- [I measured my token spend, then got the conclusion wrong](notes/what-an-ai-coding-agent-actually-costs.md) - 86.8% of the cost in 4.8% of sessions, and why that number does not mean what I said it meant.

## Currently chewing on

Where the boundary sits between a design system and the product consuming it. How to make an agent's output verifiable instead of merely plausible. What a bundler migration really costs once you count the consumers.

## Stack

**Languages** TypeScript, JavaScript, HTML, CSS
**Frontend** React, Next.js, Module Federation, TanStack Query/Router/Table, Redux, Zustand
**Styling** Tailwind, CSS Modules, Sass, design tokens
**Build** Vite, rsbuild, esbuild, Webpack, pnpm monorepos, Node.js, Bun
**Testing** Vitest, Jest, Playwright, Cypress
**Platform** GitHub Actions, GitLab CI, Docker, PostgreSQL
**AI assisted** Claude Code (agents, hooks, skills, MCP), Cursor, Copilot

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=flat&logo=react&logoColor=000)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat&logo=nodedotjs&logoColor=FFF)
![pnpm](https://img.shields.io/badge/pnpm-F69220?style=flat&logo=pnpm&logoColor=FFF)
![Playwright](https://img.shields.io/badge/Playwright-2EAD33?style=flat&logo=playwright&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat&logo=githubactions&logoColor=white)
![Claude Code](https://img.shields.io/badge/Claude_Code-D97757?style=flat&logo=anthropic&logoColor=white)

## Elsewhere

Ask me about frontend architecture, design system API design, Module Federation, or where AI assisted development breaks down. Tell me I'm wrong about any of it, I'd rather be corrected than quoted.

[Bluesky](https://bsky.app/profile/cfarkas.bsky.social)

<sub>I do what I must, because I can.</sub>

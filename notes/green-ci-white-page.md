# Green CI, white page

A component library that externalises React and bundles everything else will, sooner or later, pull in a CJS-only transitive module that calls `require('react')`. The bundler has nothing to resolve that to, so it emits a shim that throws. Every gate stays green. The consuming app shows a white page.

And when it happens, don't reason about it from `package.json`. Read the bundle. I did the opposite and got it wrong twice in a row.

## The failure

Release goes out. Types check, lint passes, full unit suite green, the lib publishes. The consuming app boots to a blank page with one console error:

```
Calling `require` for "react" in an environment that doesn't expose the `require` function
```

The lib externalises `react`. That's correct and non negotiable for a shared component library, otherwise consumers get a second React and an invalid hook call. But it *bundles* its accessibility primitives and their whole tree.

Somewhere in that tree sits a module with no ESM build. It does `require('react')`. `react` is external, so the bundler has nothing to resolve that call to and wires it to a runtime shim that throws the moment it runs.

## Why nothing caught it

The shim only throws where `require` is absent **and** the ESM artifact actually executes. Neither condition was ever met before production:

- **Node consumers load the CJS entry.** `main` and `exports.require` point at `dist/cjs/index.js`, where `require` is a real function and the shim is never reached.
- **Nothing in CI executes the ESM artifact at all.** The gates were a type check, a linter, and a jsdom unit suite running against *source*. Not one of them loads `dist/es/index.js`.

That second point is the one that matters, and I want to be precise because I got it wrong myself the first time: it isn't that Node is somehow immune. It's that no step in the pipeline ever ran the built ESM bundle in a runtime without `require`. The only place that happens is a browser, and no gate opened one.

## The part I got wrong

I diagnosed it from the manifest. `package.json` declared `"type": "module"` while `main` and `exports.require` pointed at `dist/cjs/index.js`, a `.js` file whose content is CommonJS. Under `"type": "module"` that's a real contradiction, and I was confident about it.

It wasn't the bug. It's latent, it fires for nobody today, and I delivered that answer confidently. Twice.

Another agent read the **built bundle** and got it in one step: a newly added component was the first thing in the lib to import a new corner of the accessibility library, which reach a CJS-only state-management module, the one doing `require('react')`.

The distinction that cost me the day: **a manifest describes intent, only the artifact shows what actually shipped.** I had already built `dist/`. One grep would have settled it.

Worth saying out loud since it's the more useful half: when a peer or a tool comes back with a competing analysis, verify theirs before defending yours. Mine was a real finding that happened to be irrelevant, which is the most seductive kind of wrong.

## A dependency diff proves nothing here

This is the counter-intuitive part, and it's why the bug feels like it came from nowhere.

`package.json` was **byte-identical** between the working release and the broken one. Nothing added, nothing removed, nothing bumped.

The trigger isn't a dependency change, it's a change in which modules become *reachable*. Adding one component that imports one new corner of an existing dependency pulled a previously unreached CJS module into the graph. Bisecting dependencies will never find that. Bisecting imports will.

## The invariant this actually teaches

I first wrote this up as "a library must never carry its own copy of a singleton dependency". That's a real rule, and it is **not** the rule this incident supports.

The module that broke us was a `useSyncExternalStore` shim. It has no instance identity and no module-level state. Two copies of it are harmless. Nobody would have classified it as a singleton, so the singleton rule would not have prevented this.

The invariant the incident actually supports is narrower and more useful:

> **Don't bundle a CJS-only module that references a dependency you have marked external.** The bundler has nothing to resolve the reference to, so it emits a shim that throws at runtime.

Keep the singleton rule as a separate, weaker point. It's true for React, for i18next in a shared lib, for a PDF worker. It just isn't what happened here.

## The playbook

For any "works in Node, breaks in the browser" or otherwise bundler-shaped error:

1. **Grep the built output for the error string itself.** Not the source, `dist/`.
2. **Find which chunk defines the throwing helper**, then trace the minified alias back through that chunk's `export { x as m }` map to its definition.
3. *Then* form a theory.

Two practical notes. If your package ships a docs site inside the tarball (a built Storybook for instance), that directory legitimately contains the same shim, so exclude it when scanning or you'll chase your own docs. And before releasing a component that reaches a new corner of a big dependency, grep the bundle for the shim's error string and for the `require("react")` assignment pattern.

That grep catches *this shape, with your current bundler*. Different bundlers word the message differently, the string moves between versions, and minification restructures the assignment. Re-derive it when you upgrade rather than trusting a copied regex.

## The gate that would have caught it, which we still don't have

The immediate fix was to add the offending module and its subpaths to the bundler `external` list and declare it in `dependencies`, so consumers resolve it from the library's own `node_modules`.

The structural fix, which I recommend and have **not** shipped, is to externalise the whole accessibility library. Being honest about why it hasn't happened: at least one consumer already declares its own copy, so the lib ships a duplicate today, but I don't know that *every* consumer does. Externalising it forces the ones that don't to add a dependency, and turns styling primitives into a version-compatibility surface. That's a migration, not a config line, and I haven't priced it yet.

What I'd do first instead, and what would actually have caught this, is about ten lines:

```ts
// smoke.spec.ts, run in a real browser against the built artifact
import { Button } from '../dist/es/index.js'

test('the built ESM bundle mounts in a browser', async () => {
  render(<Button>ok</Button>)
  expect(screen.getByRole('button')).toBeVisible()
})
```

Point it at `dist/`, run it in a real browser engine (Playwright, or Vitest's browser mode), make it a release gate. It's the only check in the whole pipeline that satisfies both conditions the shim needs to throw.

## What transfers

- A green unit suite says nothing about the artifact. If you publish a bundle, load the bundle in a browser as a release gate.
- Manifest describes intent, artifact describes reality. When they disagree, the artifact wins.
- Externalising is a contract, not a free win: every external becomes a version your consumer must satisfy. Externalise what has instance identity, and don't pretend the rest is free.

# Subpath imports cut both ways

Importing `some-lib/narrow-thing` instead of `some-lib` is usually a win. But if `some-lib` is in your Module Federation share scope, the exact same move takes it out of the singleton and duplicates it across every remote.

The variable is not who owns the package. It is whether the specifier matches a share key. That distinction took me an outage's worth of reading to get right, so here it is in full.

## The good half

A component needed one hook out of a big accessibility library. Obvious import:

```ts
import { useFocusRing } from 'ui-primitives'
```

That barrel resolved **918 further CJS modules**. The Jest suite went from 59s to 110s, and two chart suites started failing because a `waitFor` on a deferred chunk only had the 1s default and the extra modules pushed it over.

Narrow import:

```ts
import { useFocusRing } from 'ui-primitives/useFocusRing'
```

**19 modules.** Works because the project uses `moduleResolution: "bundler"` and the lib ships an `exports` map.

Be precise about what that number is, because I wasn't at first. **That is test-runner resolution cost, not bundle weight.** Jest resolves through the `require` condition and gets the CJS build; a bundler gets the ESM one, and that package declares `sideEffects: false`, so your production chunk is probably tree-shaken fine. I have not measured the bundle delta.

Which is not a reason to shrug. Doubling a test suite's runtime is a real cost that nobody attributes to an import statement, and it is the kind of thing that quietly makes a repo unpleasant to work in. Just don't let anyone sell you the barrel-vs-subpath change as a bundle-size win without a bundle measurement.

The trap here isn't technical, it's social: four existing components in the same repo imported from the bare barrel. Copying your neighbours is normally the right instinct, and here it costs you 900 module resolutions. Before you copy a pattern from a sibling, count how many files do it the other way.

## The bad half

Now the case where the subpath is actively harmful.

The package is a design system, published internally, consumed by a micro-frontend estate. Module Federation shares it as a singleton so every remote uses one instance.

Someone wants to ship a heavy feature (charts, a PDF viewer, anything with real weight) without making every consumer pay for it. Intuitive move, a second entry point:

```json
{
  "exports": {
    ".": "./dist/es/index.js",
    "./charts": "./dist/es/charts.js"
  }
}
```

Isolation, right ?

No. **In a federated estate, a new subpath export is not a weight isolation mechanism, it's a duplication mechanism.**

MF `shared` keys are matched against the **module request**. The host declares:

```ts
shared: {
  '@acme/design-system': { singleton: true, requiredVersion: '^19.0.0' },
}
```

That is a bare key. A request for `@acme/design-system/charts` doesn't match it, so it falls outside share scope negotiation entirely. Every remote importing it bundles its own copy of that entry, plus its own unshared copy of whatever internals it drags along (the icon component, the theme context, the stylesheet).

## What a prefix share actually does

Look at a working MF config and React is usually declared twice:

```ts
shared: {
  'react':      { singleton: true, eager: true, requiredVersion: '^19.0.0' },
  'react/':     { singleton: true, eager: true, requiredVersion: '^19.0.0' },
  'react-dom':  { singleton: true, eager: true, requiredVersion: '^19.0.0' },
  'react-dom/': { singleton: true, eager: true, requiredVersion: '^19.0.0' },
}
```

I used to explain the trailing-slash entry as "it makes `react/jsx-runtime` resolve to the same instance as `react`". That is wrong, and it's worth correcting because it's a common piece of folklore.

`react/jsx-runtime` is a **separate module**. It does not import `react` at all, and it builds elements from `Symbol.for('react.transitional.element')`, which uses the cross-realm symbol registry. Duplicate it and the copies still produce elements the shared React accepts. There is no instance to preserve.

What a prefix share does is simpler: it makes **the subpath module itself** a share-scope entry, so it gets deduplicated and version-negotiated across the estate instead of being bundled once per remote. The stakes are therefore proportional to what sits behind the subpath. Duplicating `react/jsx-runtime` costs a few hundred bytes and nothing else. Duplicating `react-dom/client` is a different conversation, since that is where the reconciler lives.

Same mechanism for your own library, and this is the point: **your package almost certainly has no `'@acme/design-system/'` sibling.** Nobody thinks to add one, because the package only ever had one entry point when the config was written.

## Why it's so easy to miss

The two halves of the decision live in **different repositories**.

The packaging choice (the `exports` map, the entry points) is made in the library repo. Whether a specifier is shared or duplicated at runtime is decided in the consuming app's MF config, which the library can't see and doesn't build against.

From inside the library repo, adding an export looks completely safe. Local dedup checks inspect the library's own built bundle; they have zero visibility into share-scope negotiation happening in someone else's build. No error, no warning, no failing test. The only symptom is that the estate gets slower, and nobody attributes that to a one-line change in an `exports` map.

## Do you have this bug right now ?

Open a federated page in the browser and read the share scope:

```js
Object.keys(__webpack_share_scopes__.default)
```

That lists every key that was actually negotiated. If the module you assumed was shared isn't in there, it isn't shared. Takes ten seconds and answers the question directly, which is more than any static check I know of will do for you.

Then, statically: grep your MF config for bare keys with no `'name/'` sibling. Every one of them is a package whose subpaths will duplicate.

And do that for **third-party packages too**, which is the part I got wrong for months. A typical estate shares a router, a data-fetching library, an i18n runtime and a state store on bare keys with no siblings. A subpath import of any of those duplicates exactly the same way a subpath of your own library does. Ownership has nothing to do with it.

## The remedies, from best to worst

**Lazy-load inside the library.** A dynamic `import()` behind the heavy component keeps a single shared instance and still keeps the weight out of the initial chunk. Transparent for consumers, no coordination with another repo. This is the one I'd reach for.

**Declare the exact subpath as its own share key.** `'@acme/design-system/charts'` alongside the bare key. Narrower than a prefix share, which silently share-scopes every future subpath you add.

**Add the prefix share.** Works, and it lands in the *consuming* repo, in a config that is often guarded by invariants precisely because per-entry sharing decisions are how shells go down. One-way door, price it as such.

One caveat on the last two that I have not verified and won't pretend to: the bare key and the subpath key become **two independent share-scope entries**. If your subpath entry reaches shared internals (a theme context, say) through relative imports, whether both entries end up with one instance or two depends on which build provides each entry. Host provides the root, a remote provides `/charts`, and you could plausibly get two theme contexts and silent default values. Test it before you rely on it.

There is a second-order trap worth knowing, since it is even quieter. If your build emits CSS to a **constant** filename (check `assetFileNames` in your config, a hand-rolled override like `css/${packageName}[extname]` is common enough), a second entry collides on that filename. And if exactly one file in your estate imports the library stylesheet, the second bundle ends up imported by nobody and the feature ships unstyled. Vite's default `assets/[name]-[hash][extname]` doesn't have this problem.

## The honest version of the two-topology argument

I originally wrote that a subpath export "fails both topologies at once". That's too strong, and the reviewer who pushed back on it was right.

A published library is usually consumed two ways: through Module Federation, and installed directly by standalone apps. For the **standalone** half, a subpath export with `sideEffects: false` is the textbook mechanism for isolating optional weight. It works. For the **federated** half, it breaks the singleton.

So the real argument isn't "it fails both". It's that **one packaging decision cannot serve both topologies**, and the only shape that does is the lazy import inside the library.

That distinction matters for another reason. It's tempting to reason "the design system is shared, so its weight is paid once by the host". Only true for the federated half. Standalone apps pay every added kB **in full, per app, whether they use the feature or not**.

## One line

**Take the subpath for anything outside your share scope. For anything shared on a bare key, yours or not, a subpath escapes the singleton.**

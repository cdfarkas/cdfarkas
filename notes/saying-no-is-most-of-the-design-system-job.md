# Saying no is most of the design system job

A design system consumed by a dozen apps is not a component folder. It is a public API you can't refactor unilaterally, owned by a team that isn't the one paying for your mistakes.

Everything below is a rule I actually enforce in review, with the case that produced it. None of it is about visual design.

## 1. Composition over props, and settle it before the first release

A small composite panel arrived in review with four booleans: `showHint`, `showOptions`, `showToggle`, `showActions`, rendering blocks in a fixed order. It merged like this instead:

```tsx
<Panel value={value} onChange={preview} onCommit={save}>
  <Panel.Body />
  <Panel.Options />   {/* move this line to reorder the blocks */}
  <Panel.Toggle>Set as default</Panel.Toggle>
  <Panel.Actions />
</Panel>
```

The argument I'd have made a year ago is "four booleans is sixteen combinations to test". That sounds good and it's mostly rhetoric: nobody supports 2⁴, there are three or four real configurations and four snapshot tests cover them. Don't lead with the number.

The argument that actually holds is **ordering**: not one of those flag combinations lets a consumer put the options above the body. With composition it's a line move. That one is unanswerable.

The second argument is consistency, and it's stronger than it looks: **30-plus components in the library already use the compound pattern, 17 of them sharing state through a context.** The flag-driven panel was the deviation. In a library, matching the established shape is worth more than any individual API being marginally nicer.

**Two things I have to concede**, because a reviewer pushed on them and was right.

Compound parts give you **no compile-time guarantee that a required part is present**. A consumer can render `<Panel>` with no `.Body` and get a silently broken component. Four booleans with sane defaults cannot produce that. There is no clean fix, only a runtime warning.

And the commit rule gets more complex, not less. With flags, the root read `showActions` to know whether an edit commits immediately. Once composition owns that, the root can't see whether a validation step exists, so `.Actions` registers itself with the context on mount. Use a **counter, not a boolean**: a remount that mounts the new part before unmounting the old must never momentarily read as "no validation step", and a counter survives StrictMode's +1 / -1 / +1.

So the honest trade is: you buy ordering and consistency, you pay a registration protocol and a lost compile-time guarantee. I'd make that trade again, but it isn't free and I shouldn't pretend it is.

**Do it before you publish.** Deleting those four props cost nothing because the component had never shipped. After a release, every flag needs a deprecated alias and a full deprecation cycle. Compound-vs-flags is a five-minute conversation before the first release and a quarter of migration afterwards.

## 2. The design brief describes the layout, not the API

Those four booleans came straight from the brief, which said "independent toggles, fixed order".

That is UX describing the **visual result**. It does not constrain the technical contract. A fixed visual order does not require a fixed structural one, and the two are trivially compatible here: consumers write the blocks in the order the design shows.

Design owns what it looks like. It does not own the shape of the API. Worth saying out loud in review, calmly, because the brief gets quoted as if it settled the question.

## 3. Follow the house pattern, not the minority

A new component family first shipped as a plain object namespace, copied from an existing component that does exactly that.

Except that component is in the minority. At the time: **31 components used `Object.assign(Component, { ...subs })` and 6 used a pure object literal** (it's 35 now, the counts drift). Copying your neighbours is normally the right instinct in a large codebase. It is also how you inherit the one deviation instead of the convention.

So it became `Widget = Object.assign(WidgetRoot, { A, B, Legend })`, with no redundant `Widget.Container`, because making the root *be* the container costs nothing: the shared highlight context resolves to null outside, and the parts already rendered standalone.

Before you copy a pattern from a sibling component, count how many components do it the other way.

## 4. Read the code before you pick the component

A card grid needed sorting, filtering and selection. The obvious answer was the library's table component in grid mode. Reading it killed that:

- **It cannot filter at all.** `enableColumnFilters: false` is hard-wired and no filtered row model is registered. Of the three capabilities that motivated the choice, it supplies two.
- **No controlled sorting prop**, only an initial value plus a change notification. So the column header row is the only sort affordance, which puts a table header above a card grid whose columns don't align with it. Hide the header and the grid becomes unsortable.
- **Grid mode requires virtualization and a fixed pixel height.** The consumers here live in panels that grow with content.
- **At one column, grid mode silently degrades to table rows.** That footgun is present in the library's own reference story.
- **The row template is discovered by duck-typing.** The parent inspects children for `'row' in child.props`, which forces every call site to write `row={undefined as unknown as Row<T>}` just to be recognised. A cast lie, visible in the library's own story file. And because discovery tests `typeof child.type === 'function'`, the template can never be `forwardRef`.

That last one is the real exhibit. The first four are capability gaps you could argue about; a public API that requires consumers to lie to the type system to be seen is a design defect you can't.

The answer was a headless table owned by the page, handing the design system rows already sorted and filtered. Not a new API, a documented pattern with a reference story and a test.

The general form: a component's reputation is not its contract. Open it.

## 5. The real constraint is rarely the one being argued about

The standing argument for routing that grid through the table component was virtualization, for scale.

At a few hundred cards the binding constraint was not the DOM. It was the image component, which fetches every source on mount and cannot be made lazy by construction. Virtualization only *masks* that by not mounting off-screen rows. The correct fix is in the image component, and until it lands, no grid substrate changes the request count.

**Where this flips**, because a rule without a boundary isn't a rule: if a screen genuinely reaches thousands of cards *and* the image component still preloads eagerly, grid-mode virtualization becomes the pragmatic containment. Request count is not the only constraint at that size, DOM node count and memory are real. The rule holds at hundreds, not at thousands.

Fixing the visible symptom in the wrong layer is how a design system accumulates components nobody can remove. But say where the layer changes.

## 6. Deleting before release costs a file, after release costs a cycle

While exploring that grid, a bridge component was built, tested, and proven to work by rendering. Then deleted before merge.

It worked. It was also a known API-shape violation shipped into the public surface for a path nobody had chosen, and its props type was already exported, so removing it *after* a release would have cost a deprecation cycle. Before release it cost one `rm`.

Being right about the direction is not enough to justify shipping the thing.

## 7. Where a file lives decides which gates it gets, and that's a bug too

Coverage collection was scoped to `src/components/**` with a 100% threshold, and the review gate for "is this new component justified" keyed off the same directory. So `src/templates/` escaped both. A template earns 100% of nothing.

Which inverts the incentives perfectly: a trivial badge variant in `components/` gets both gates, and the most branch-heavy composite in the library gets neither. Easy to trip without meaning to, because a brief saying "new Template" points straight at `templates/`.

We split by weight: the stateful, branchy part to `components/`, only the page-level layout stays in `templates/`.

I used to summarise this as "when a gate has a blind spot, move the code, not the gate". I don't think that generalises, and I'd push back on anyone who quoted it at me now. Taken as a rule it says *reshape the codebase around your CI configuration*, and over a couple of years that gives you directory layouts encoding tooling history instead of architecture.

There are two separate facts here, and they deserve separate treatment:

- **The split is right on its own merits.** Stateful logic and page layout are different weights and belong apart. That argument needs no reference to coverage.
- **The gate gap is a real second problem** with its own ticket. Roughly ten existing templates are uncovered and always were. "Widening the glob would fail the build today" is an argument about migration cost, not about where code belongs.

I did the first and filed the second. Say both, don't let the gate be the justification for the architecture.

## 8. Pin what you assume with a test

Both parts of that family have a test asserting they render with no `<Chart>` ancestor, **counting rendered marks** rather than checking the surface exists.

The reason is specific and worth getting right, because I stated it sloppily the first time. Under our Jest setup, a size-measuring component renders *almost nothing*: the ResizeObserver mock's `observe` is a bare `jest.fn()` that never invokes the callback, jsdom's `getBoundingClientRect()` returns all zeros, and the rendering library bails before emitting the SVG. Measured on one of them: **106 bytes of DOM, no `<svg>` at all, and nothing logged.**

So asserting the surface exists would have *failed loudly*, which is fine. The dangerous assertions are the weaker ones: "it rendered without throwing", a non-empty container, a snapshot of a 106-byte div. Those pass vacuously on a component that cannot draw. Count the marks, because the marks are the only thing that proves it drew.

Note that this is a property of *our* test setup, not of jsdom. With a ResizeObserver polyfill that actually fires, you won't reproduce it. Check your own mock before assuming you're safe.

The general rule: every property you *decided* to keep (an optional wrapper, standalone use, a backwards-compatible alias) needs a test that fails when someone removes it. Otherwise it isn't a decision, it's a coincidence.

---

None of this is exotic. It is mostly the discipline of treating a shared library as an API with users who can't be forced to migrate, and being willing to have the awkward conversation before merge instead of the migration ticket after release.

Disagree with any of it ? I'd genuinely like to hear the counter-case, especially on 4 and 7.

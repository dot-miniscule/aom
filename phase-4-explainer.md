### Phase 4: Full Introspection of an Accessibility Tree

The **Computed Accessibility Tree** API will allow authors to access
the full computed accessibility tree -
all computed properties for the accessibility node associated with each DOM element,
plus the ability to walk the computed tree structure including virtual nodes.

This will make it possible to:
  * write any programmatic test which asserts anything
    about the semantic properties of an element or a page.
  * build a reliable browser-based assistive technology -
    for example, a browser extension which uses the accessibility tree
    to implement a screen reader, screen magnifier, or other assistive functionality;
    or an in-page tool.
  * detect whether an accessibility property
    has been successfully applied
    (via ARIA or otherwise)
    to an element -
    for example, to detect whether a browser has implemented a particular version of ARIA.
  * do any kind of console-based debugging/checking of accessibility tree issues.
  * react to accessibility tree state,
    for example, detecting the exposed role of an element
    and modifying the accessible help text to suit.

#### Why is accessing the computed properties being addressed last?

**Consistency**
Currently, the accessibility tree is not standardized between browsers:
Each implements accessibility tree computation slightly differently.
In order for this API to be useful,
it needs to work consistently across browsers,
so that developers don't need to write special case code for each.

We want to take the appropriate time to ensure we can agree
on the details for how the tree should be computed
and represented.

**Performance**
Computing the value of many accessible properties requires layout.
Allowing web authors to query the computed value of an accessible property
synchronously via a simple property access
would introduce confusing performance bottlenecks.

We will likely want to create an asynchronous mechanism for this reason,
meaning that it will not be part of the `accessibleNode` interface.

**User experience**
Compared to the previous three phases,
accessing the computed accessibility tree will have the least direct impact on users.
In the spirit of the [Priority of Constituencies](https://www.w3.org/TR/html-design-principles/#priority-of-constituencies),
it makes sense to tackle this work last.


#### ComputedAccessibleNode
It is currently possible to check accessible attributes for a DOM element by using the `getAttribute` method:
```js
element.hasAttribute("aria-labelledby");
element.getAttribute("aria-labelledby");
```
However, determining which accessible properties apply to an element can be cumbersome, as there can be multiple
Accessing the computed accessible properties of a DOM element will be designed to mimic the behaviour of `getComputedStyle()`, but implemented as an asynchronous Promise-based API. A `ComputedAccessibleNode` can be requested for a specific element from the window:

```html
<button id="composeBtn">Compose</button>
```
```js
var composeBtn = document.getElementById('composeBtn');
window.getComputedAccessibleNode(element).then(
  function(computedAccessibleNode) {
    console.log(computedAccessibleNode.role);  // resolved!
  },
  function(error) {
    console.log(error);  // rejected, but why?
  });
```

The rejection case of the `Promise` is discussed in [Open Questions](#open-questions-rejection-case). Alternatively, we can use `await`:

```js
var computedAccessibleNode = await window.getComputedAccessibleNode(element);
console.log(computedAccessibleNode.role);  // "button".
```
We can now access all of the computed properties exposed from one object, without relying on a leaky abstraction. This is distinct from [Phase 1](#phase-1-modifying-accessible-properties) as it would return the computed role, `"button"`, rather than `null`.  Exactly what properties a `ComputedAccessibleNode` should expose is discussed in in [Open Questions](#open-questions-accessible-properties).

#### Accessibility Tree Snapshots
`ComputedAccessibleNode`s should represent a consistent snapshot of the Accessibility Tree.

#### Open Questions
There are a number of open questions about Phase 4 that need to be discussed. This list is by no means exhaustive, but should provide a good platform for beginning discussion on the specifics of the public API design.

##### 1. Accessible Properties
The first question to answer is what properties are we expecting to expose via this API, and how? A lot of the properties are fairly self explanatory, such as `role` and `name` returning a string, or `colCount` returning an integer etc. Other attributes are not so straightforward, such as `checked`. DOM elements expose a `checked` property on all input types, which can take either of the values `true` or `false`. . Other attributes are not so straightforward, such as `checked`. Although checkboxes can only be in one of two states, checked or unchecked upon a form submission, they can take on a third visual state - `indeterminate`.

```html
<input type="checkbox" id="someCheckbox"/>
```

```js
var checkbox = document.getElementByID("someCheckbox");
checkbox.indeterminate = true;
```
This is purely a visual state, meaning that it does not impact the value of the `checked` attribute:

```js
console.log(checked.checked)  // false
consoel.log(checked.indeterminate)  // true
```

ARIA reflects these states by allowing an author to set the checked state with `aria-checked`, assigning it to a string value of either `true`, `false`, or `mixed` (the latter reflecting the indeterminate state). This mixed checked state can be used to indicate that a parent checkbox has some nested child checkboxes that are a combination of `checked` and `unchecked`. What should a `ComputedAccessibleNode` expose these three visual states? However, as a checkbox cannot be submitted with a value of `mixed` or `indeterminate` should this be stored in a separate attribute? Three possible options are:
1. Mimic `aria-checked`and return a string indicating which state the element is in.
2. Expose an enum for each state.
3. Implement the `checked` attribute as a boolean, with a separate boolean attribute for `mixed`/`indeterminate`.

One thing that needs to be agreed upon is the full set of attributes that a `ComputedAccessibleNode` should provide read access to. Is this the same as the set of properties available to be set via the `AccessibleNode` or `Element` interface? The current proposed IDL is:

```
interface ComputedAccessibleNode {
    readonly attribute DOMString? autocomplete;
    readonly attribute DOMString? checked;
    readonly attribute DOMString? keyShortcuts;
    readonly attribute DOMString? name;
    readonly attribute DOMString? placeholder;
    readonly attribute DOMString? role;
    readonly attribute DOMString? roleDescription;
    readonly attribute DOMString? valueText;

    readonly attribute long? colCount;
    readonly attribute unsigned long? colIndex;
    readonly attribute unsigned long? colSpan;
    readonly attribute unsigned long? level;
    readonly attribute unsigned long? posInSet;
    readonly attribute long? rowCount;
    readonly attribute unsigned long? rowIndex;
    readonly attribute unsigned long? rowSpan;
    readonly attribute long? setSize;

    readonly attribute ComputedAccessibleNode? parent;
    readonly attribute ComputedAccessibleNode? firstChild;
    readonly attribute ComputedAccessibleNode? lastChild;
    readonly attribute ComputedAccessibleNode? previousSibling;
    readonly attribute ComputedAccessibleNode? nextSibling;

    readonly attribute boolean? atomic;
    readonly attribute boolean? busy;
    readonly attribute boolean? disabled;
    readonly attribute boolean? modal;
    readonly attribute boolean? readOnly;

  readonly attribute float? valueNow;
  readonly attribute float? valueMin;
  readonly attribute float? valueMax;

    [CallWith=ScriptState] Promise ensureUpToDate();
};
```

A full list of available attributes can be found [on this trix]() (aboxhall@: should this list be migrated somewhere it can be viewed externally? Putting it into the explainer seems wrong, should it be uploaded as a separate markdown or filed into an issue on the wicg page once the explainer is updated there?).


##### 2. Rejection Case
If we are returning a promise when requesting a `ComputedAccessibleNode`, what is the rejection case?

##### 3. Lifecycle of the Accessible Tree
`ComputedAccessibleNode`s within the same Frame all share a reference to the same tree snapshot, which means the tree is created when the first node is requested, and is destroyed with the Frame. One alternative is to queue up a number of requests for `ComputedAccessibleNode`s somewhere (returning a pending `Promise` to the callee), and when the next layout/style recalc is performed, resolve any outstanding `Promise`s. Computing the tree lazily would allow for batch retrieval of `ComputedAccessibleNode`s but care would have to be taken the ensure the idea of a consistent snapshot is maintained.

##### 4. Accessibility Setting
The Accessibility setting must be enabled for Phase 4. The current proposed implementation forces this when a `ComputedAccessibleNode` is requested, and does not ever switch it off, which may not be the best option in the long term. The lifecycle of Phase 4 from the first request of a `ComputedAccessibleNode`

##### 5. Traversing across Accessible Trees
Currently Phase 4 is proposed as containing `ComputedAccessibleNode`s within a single Frame, but in future the capability to explore from one Accessible Tree in one Frame to another.
- TODO: Use cases for this?
- Considerations? How to get from one "root" to "another"?
  - What is the relation between Frame roots? Siblings?
  - What owns each Frame? roots of tree's in different frame?
  - Is each tree still owned by the frame?

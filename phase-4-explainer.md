<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Phase 4: Full Introspection of an Accessibility Tree](#phase-4-full-introspection-of-an-accessibility-tree)
  - [Motivation](#motivation)
  - [ComputedAccessibleNode](#computedaccessiblenode)
    - [`ensureUpToDate()`](#ensureuptodate)
  - [Open Questions](#open-questions)
    - [1. Accessible Properties](#1-accessible-properties)
    - [2. Rejection Case](#2-rejection-case)
    - [3. Traversing across Accessible Trees](#3-traversing-across-accessible-trees)
    - [4. Non-dom Nodes](#4-non-dom-nodes)
    - [5. Updating ComputedAccessibleNodes](#5-updating-computedaccessiblenodes)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Phase 4: Full Introspection of an Accessibility Tree

The **Computed Accessibility Tree** API will allow authors to access
the full computed accessibility tree -
all computed properties for the accessibility node associated with each DOM element,
plus the ability to walk the computed tree structure including [virtual nodes](explainer.md#phase-3-virtual-accessibility-nodes), and [potentially non-DOM nodes, such as ::before content](#4-non-dom-nodes).

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
  * Perform console debugging of accessibility properties.

For earlier exploration of this API, see the rest of the [explainer](explainer.md)

## Motivation
It is currently possible to check accessible attributes for a DOM element by using the `getAttribute` method:
```js
element.hasAttribute("aria-labelledby");
element.getAttribute("aria-labelledby");
```
However, determining which accessible properties apply to an element can be cumbersome, as there can be multiple sources for an accessible property, for example see [WAI-ARIA name calculation](https://www.w3.org/TR/wai-aria-1.1/#namecalculation)). This means that if an author wants to write accessibility tests, they would need to be familiar with how accessible properties are calculated. This API would abstract away the implementation details from what the final computed accessible properties of an element or page are.

## ComputedAccessibleNode
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

The rejection case of the `Promise` is discussed in [Open Questions](#2-rejection-case). Alternatively, we can use `await`:

```js
var computedAccessibleNode = await window.getComputedAccessibleNode(element);
console.log(computedAccessibleNode.role);  // "button".
```
We can now access all of the computed properties exposed from one object, without relying on a leaky abstraction.

This is distinct from [Phase 1](#phase-1-modifying-accessible-properties) as it would return the computed role, `"button"`, rather than `null`.  Exactly what properties a `ComputedAccessibleNode` should expose is discussed in in [Open Questions](#1-accessible-properties).

### `ensureUpToDate()`
One decision that needs to be made is what kind of guarantees are we able to make on how accurately a `ComputedAccessibleNode` is with the DOM content. For example:

```html
<div role=checkbox class="custom-checkbox" id="toggle"></div>
```

```js
var element = document.getElementById("toggle");
var computedAccessibleNode = window.getComputedAccessibleNode(element);
element.setAttribute("checked", "true");
console.log(computedAccessibleNode.checked);  // Will be "false", but should it be?
```

One solution that provides is to expose a method that makes it the author's responsibility to update any `ComputedAccessibleNode`s they hold a reference to:

```js
computedAccessibleNode.ensureUpToDate();

// Will print the most up to date value of the checked attribute.
console.log(computedAccessibleNode.checked);  // true
```

The trade-offs for this approach and other alternatives are discussed in [Open Questions](#5-updating-computedaccessiblenodes).

## Open Questions
There are a number of open questions about Phase 4 that need to be discussed. This list is by no means exhaustive, but should provide a good platform for beginning discussion on the specifics of the public API design.

### 1. Accessible Properties
The first question to answer is what properties are we expecting to expose via this API, and how? A lot of the properties are fairly self explanatory, such as `role` and `name` returning a string, or `colCount` returning an integer etc. Other attributes are not so straightforward, such as `checked`.

DOM elements expose a `checked` property on all input types, which can take either of the values `true` or `false`. Although checkboxes can only be in one of two states, checked or unchecked upon a form submission, they can take on a third visual state - `indeterminate`.

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
console.log(checked.indeterminate)  // true
```

ARIA reflects these states by allowing an author to set the checked state with `aria-checked`, assigning it to a string value of either `true`, `false`, or `mixed` (the latter reflecting the indeterminate state). This mixed checked state can be used to indicate that a parent checkbox has some nested child checkboxes that are a combination of `checked` and `unchecked`. How should a `ComputedAccessibleNode` expose these three visual states? As a checkbox cannot be submitted with a value of `mixed` or `indeterminate` should the indeterminate state be stored in a separate attribute as it is on the DOM Element? Three possible options are:
1. Mimic `aria-checked`and return a string indicating which state the element is in.
2. Expose an enum for each state.
3. Implement the `checked` attribute as a boolean, with a separate boolean attribute for `mixed`/`indeterminate`.

One thing that needs to be agreed upon is the full set of attributes that a `ComputedAccessibleNode` should provide read access to. Is this the same as the set of properties available to be set via the `AccessibleNode` or `Element` interface? The current proposed IDL is:

```
interface ComputedAccessibleNode {
    readonly attribute boolean? atomic;
    readonly attribute boolean? busy;
    readonly attribute boolean? disabled;
    readonly attribute boolean? modal;
    readonly attribute boolean? readOnly;

    readonly attribute long? colCount;
    readonly attribute unsigned long? colIndex;
    readonly attribute unsigned long? colSpan;
    readonly attribute unsigned long? level;
    readonly attribute unsigned long? posInSet;
    readonly attribute long? rowCount;
    readonly attribute unsigned long? rowIndex;
    readonly attribute unsigned long? rowSpan;
    readonly attribute long? setSize;

    readonly attribute float? valueNow;
    readonly attribute float? valueMin;
    readonly attribute float? valueMax;

    readonly attribute DOMString? autocomplete;
    readonly attribute DOMString? checked;
    readonly attribute DOMString? keyShortcuts;
    readonly attribute DOMString? name;
    readonly attribute DOMString? placeholder;
    readonly attribute DOMString? role;
    readonly attribute DOMString? roleDescription;
    readonly attribute DOMString? valueText;

    readonly attribute ComputedAccessibleNode? parent;
    readonly attribute ComputedAccessibleNode? firstChild;
    readonly attribute ComputedAccessibleNode? lastChild;
    readonly attribute ComputedAccessibleNode? previousSibling;
    readonly attribute ComputedAccessibleNode? nextSibling;

    [CallWith=ScriptState] Promise ensureUpToDate();
};
```

TODO(meredithl): insert link to full list of potential properties in spec.


### 2. Rejection Case
If we are returning a promise when requesting a `ComputedAccessibleNode`, what is the rejection case?

### 3. Traversing across Accessible Trees
Currently Phase 4 is proposed as containing `ComputedAccessibleNode`s within a single Frame, but in future the capability to explore from one Accessible Tree in one Frame to another.
- TODO: Use cases for this?
- Considerations? How to get from one "root" to "another"?
  - What is the relation between Frame roots? Siblings?
  - What owns each Frame? roots of tree's in different frame?
  - Is each tree still owned by the frame?

### 4. Non-dom Nodes
We may want the ability to traverse to non-DOM nodes through the `ComputedAccessibleNode` API. For example, traversing to the root of a page, or to [virtual accessibility nodes](explainer.md#phase-3-virtual-accessibility-nodes).

### 5. Updating ComputedAccessibleNodes
This section will discuss teh possibilities for when a `ComputedAccessibleNode` is considered "up to date", which is when it reflects the most recent changes in the DOM. Currently, `ComputedAccessibleNode` provides a manual method for refreshing itself, which can be called by the author when they know a change has been made. For example, an author could register a `MutationObserver` in order to be notified when DOM content has been changed:

```html
<div role=checkbox class="custom-checkbox" id="toggle"></div>
```

```js
var element = document.getElementById("custom-checkbox");
var computedAccessibleNode = await window.getComputedAccessibleNode(element);
var mutationObserver = new MutationObserver(function(mutations) {
  computedAccessibleNode.ensureUpToDate();
})

// Starts listening for changes in the root HTML element of the page.
mutationObserver.observe(element, {
    attributes: true,
    characterData: true,
    childList: true,
    subtree: true,
    attributeOldValue: true,
    characterDataOldValue: true
});

mutationObserver.disconnect();
```

This means that any time the source element is updated (or any of its attributes by setting the `attributes` flag in the observer to `true`) the `ComputedAccessibleNode` will be refreshed to reflect the latest changes.

Another option is to make `ComputedAccessibleNode` a live object, so that it is (almost) always up to date with DOM content, meaning that rather than a `ComputedAccessibleNode` belonging to a snapshot of the Accessibility Tree.

TODO(meredithl)


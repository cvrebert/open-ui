---
menu: Proposals
name: Popover API (Explainer)
path: /components/popover.research.explainer
pathToResearch: /components/popup.research
layout: ../../layouts/ComponentLayout.astro
---

- [@mfreed7](https://github.com/mfreed7), [@scottaohara](https://github.com/scottaohara), [@BoCupp-Microsoft](https://github.com/BoCupp-Microsoft), [@domenic](https://github.com/domenic), [@gregwhitworth](https://github.com/gregwhitworth), [@chrishtr](https://github.com/chrishtr), [@dandclark](https://github.com/dandclark), [@una](https://github.com/una), [@smhigley](https://github.com/smhigley), [@aleventhal](https://github.com/aleventhal), [@jh3y](https://github.com/jh3y)
- January 20, 2023

Please also see the [WHATWG html spec PR for this proposal](https://github.com/whatwg/html/pull/8221).

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

## Table of Contents

- [Background](#background)
  - [Goals](#goals)
  - [See Also](#see-also)
  - [Popover vs. Dialog](#popover-vs-dialog)
- [API Shape](#api-shape)
  - [HTML Content Attribute](#html-content-attribute)
  - [Showing and Hiding a Popover](#showing-and-hiding-a-popover)
    - [Declarative Triggers](#declarative-triggers)
    - [Javascript Trigger](#javascript-trigger)
    - [CSS Pseudo Class](#css-pseudo-class)
    - [Rendering](#rendering)
    - [Shown vs Hidden Popovers](#shown-vs-hidden-popovers)
    - [Animation of Popovers](#animation-of-popovers)
  - [IDL Attribute and Feature Detection](#idl-attribute-and-feature-detection)
  - [Events](#events)
  - [Focus Management](#focus-management)
  - [Anchoring](#anchoring)
  - [Backdrop](#backdrop)
- [Behaviors](#behaviors)
  - [Automatic Dismiss Behavior](#automatic-dismiss-behavior)
    - [Light Dismiss](#light-dismiss)
    - [Nested Popovers](#nested-popovers)
    - [The Popover Stack](#the-popover-stack)
    - [Nearest Open Ancestral Popover](#nearest-open-ancestral-popover)
    - [Close signal](#close-signal)
  - [Classes of Top Layer UI](#classes-of-top-layer-ui)
  - [Accessibility / Semantics](#accessibility--semantics)
  - [Disallowed elements](#disallowed-elements)
- [Example Use Cases](#example-use-cases)
  - [Generic Popover (Date Picker)](#generic-popover-date-picker)
  - [Generic Popover (`<selectmenu>` listbox example)](#generic-popover-selectmenu-listbox-example)
  - [Manual](#manual)
- [Additional Considerations](#additional-considerations)
  - [Exceeding the Frame Bounds](#exceeding-the-frame-bounds)
  - [Shadow DOM](#shadow-dom)
  - [Eventual Single-Purpose Elements](#eventual-single-purpose-elements)
- [The Choices Made in this API](#the-choices-made-in-this-api)
  - [Alternatives Considered](#alternatives-considered)
  - [Why a content attribute?](#why-a-content-attribute)
  - [Design decisions (via OpenUI)](#design-decisions-via-openui)
  - [Articles](#articles)

# Background

A very common UI pattern on the Web, for which there is no native API, is "popover UI", also sometimes called "popovers", "pop up UI", or "popovers". Popovers are a general class of UI that have three common behaviors:

1.  Popovers always appear **on top of other page content**.
2.  Popovers are **ephemeral**. When the user "moves on" to another part of the page (e.g. by clicking elsewhere, or hitting ESC), the popover hides.
3.  Popovers (of a particular type) are generally **"one at a time"** - opening one popover closes others.

This document proposes a set of APIs to make this type of UI easy to build.

## Goals

Here are the goals for this API:

- Allow any element and its (arbitrary) descendants to be rendered on top of **all other content** in the host web application.
- Include **“light dismiss” management functionality**, to remove the element/descendants from the top-layer upon certain actions such as hitting Esc (or any [close signal](https://wicg.github.io/close-watcher/#close-signal)) or clicking outside the element bounds.
- Allow this “top layer” content to be fully styled, sized, and positioned at the author's discretion, including properties which require compositing with other layers of the host web application (e.g. the box-shadow or backdrop-filter CSS properties).
- Allow these top layer elements to reside at semantically-relevant positions in the DOM. I.e. it should not be required to re-parent a top layer element as the last child of the `document.body` simply to escape ancestor containment and transforms.
- Include an appropriate user input and focus management experience, with flexibility to modify behaviors such as initial focus.
- **Accessible by default**, with the ability to further extend semantics/behaviors as needed for the author's specific use case.
- Avoid developer footguns, such as improper stacking of dialogs and popovers, and incorrect accessibility mappings.
- Avoid the need for any Javascript for most common cases.
- Easy animation of show/hide operations.

## See Also

See the [original `<popup>` element explainer](https://open-ui.org/components/popup.research.explainer-v1), and also the comments on [Issue 410](https://github.com/openui/open-ui/issues/410) and [Issue 417](https://github.com/openui/open-ui/issues/417). See also [this CSSWG discussion](https://github.com/w3c/csswg-drafts/issues/6965) which has mostly been about a CSS alternative for top layer.

This proposal was discussed in OpenUI on [Issue 455](https://github.com/openui/open-ui/issues/455), which was closed as [resolved](https://github.com/openui/open-ui/issues/455#issuecomment-1050172067). In WHATWG/html, [Issue 7785](https://github.com/whatwg/html/issues/7785) tracks the addition of this feature.

There have been **many** discussions and resolutions at OpenUI of various aspects of this proposal, and those are listed in the [Design Decisions](#design-decisions-via-openui) section. Requests for standards positions have been requested from [Gecko](https://github.com/mozilla/standards-positions/issues/698) and [WebKit](https://github.com/WebKit/standards-positions/issues/74).

There have been three TAG reviews for this set of functionality: [#599](https://github.com/w3ctag/design-reviews/issues/599) (closed waiting for anchor positioning), [#680](https://github.com/w3ctag/design-reviews/issues/680) (closed recommending change from `<popup>` to `popup` attribute), [#743](https://github.com/w3ctag/design-reviews/issues/743) (open).

## Popover vs. Dialog

A natural question that [commonly arises](https://github.com/openui/open-ui/issues/581) is: what's the difference between a "popover" and a "dialog"? There are two ways to interpret this question:

1.  What are the difference between these general UX patterns and words?
2.  What are the technical implementation differences between `<div popover>` and `<dialog>`?

In both cases, an important distinction needs to be made between **modal** and **non-modal** (or **modeless**) dialogs. With a **modal** dialog, the rest of the page (outside the dialog) is rendered **`inert`**, so that only the contents of the dialog are interactable. Importantly, a popover is **non-modal**. Almost by definition, a popover is not permanent: it goes away (via light dismiss) when the user changes their attention to something else, by clicking or tabbing to something else. For these reasons, if the dialog in question needs to be **modal**, then the `<dialog>` element is the way to go.

Having said that, dialogs can also be **non-modal**, and popovers can be set to not light-dismiss (via **`popover=manual`**). There is a significant area of overlap between the two Web features. Some use cases that lie in this area of overlap include:

1. "Toasts" or asynchronous notifications, which stay onscreen until dismissed manually or via a timer.
2. Persistent UI, such as teaching-UI, that needs to stay on screen while the user interacts with the page.
3. Custom components that need to "manually" control popover behavior.

Given these use cases, it's important to call out the technical differences between a non-modal `<dialog>` and a `<div popover=manual>`:

- A `<div popover=manual>` is **in the top layer**, so it draws on top of other content. The same is not true for a non-modal `<dialog>`. This is likely the most impactful difference, as it tends to be difficult to ensure that a non-modal `<dialog>` is painted on top of other page content.
- A `<dialog>` element always has **`role=dialog`**, while the `popover` attribute can be applied to the **most-semantically-relevant HTML element**, including the `<dialog>` element itself: `<dialog popover>`.
- The popover API comes with some **"nice to have's"**:
  - popovers are easy to animate both the show and hide transitions, via pure CSS. In contrast, JS is required in order to animate `dialog.close()`.
  - popovers work with the invoking attributes (e.g. `popovertoggletarget`) to declaratively show/hide them with pure HTML. In contrast, JS is required to show/close a `<dialog>`.
  - popovers fire both a "popovershow" and a "popoverhide" event. In contrast, a `<dialog>` only fires `cancel` and `close`, but no event is fired when it is shown.

For the above reasons, it seems clear that for use cases that need a **non-modal** dialog which has popover behavior, a `<dialog popover>` (with the most appropriate value for the `popover` attribute) should be preferred. Importantly, if the use case is **not** meant to be exposed as a "dialog", then another (non-`<dialog>`) element should be used with the `popover` attribute, or a generic `<div popover role=something>` should be used with the appropriate role added.

# API Shape

This section lays out the full details of this proposal. If you'd prefer, you can **[skip to the examples section](#example-use-cases) to see the code**.

## HTML Content Attribute

A new content attribute, **`popover`**, controls both the top layer status and the dismiss behavior. There are several allowed values for this attribute:

- **`popover=auto`** - A top layer element following "Auto" dismiss behaviors (see below).
- **`popover=manual`** - A top layer element following “Manual” dismiss behaviors (see below).

Additional values for the `popover` attribute may become available in the future.

So this markup represents popover content:

```html
<div popover="auto">I am a popover</div>
```

As written above, the `<div>` will be rendered `display:none` by the UA stylesheet, meaning it will not be shown when the page is loaded. To show the popover, one of several methods can be used: [declarative triggering](#declarative-triggers), [Javascript triggering](#javascript-trigger), or [page load triggering](#page-load-trigger).

Additionally, the `popover` attribute can be used without a value (or with an empty string `""` value), and in that case it will behave identically to `popover=auto`:

```html
<div popover="auto">I am a popover</div>
<div popover>I am also an "auto" popover</div>
<div popover="">So am I</div>
```

For convenience and brevity, the remainder of this explainer will use this boolean-like syntax in most cases, e.g. `<div popover>`.

## Showing and Hiding a Popover

There are several ways to "show" a popover, and they are discussed in this section. When any of these methods are used to show a popover, it will be made visible and moved (by the UA) to the [top layer](https://fullscreen.spec.whatwg.org/#top-layer). The top layer is a layer that paints on top of all other page content, with the exception of other elements currently in the top layer. This allows, for example, a "stack" of popovers to exist.

### Declarative Triggers

A common design pattern is to have a button which makes a popover visible. To facilitate this pattern, and avoid the need for Javascript in this common case, three content attributes (`popovertoggletarget`, `popovershowtarget`, and `popoverhidetarget`) allow the developer to declaratively toggle, show, or hide a popover. To do so, the attribute's value should be set to the idref of another element:

```html
<button popovertoggletarget="foo">Toggle the popover</button>
<div id="foo" popover>Popover content</div>
```

When the button in this example is activated, the UA will call `.showPopover()` on the `<div id=foo popover>` element if it is currently hidden, or `hidePopover()` if it is showing. In this way, no Javascript will be necessary for this use case.

If the desire is to have a button that only shows or only hides a popover, the following markup can be used:

```html
<button popovertoggletarget="foo">Toggle the popover</button>
<button popovershowtarget="foo">This button only shows the popover</button>
<button popoverhidetarget="foo">This button only hides the popover</button>
<div id="foo" popover>Popover content</div>
```

Note that all three attributes can be used together like this, pointing to the same element. However, using more than one triggering attribute on **a single button** is not recommended.

When the `popovertoggletarget`, `popovershowtarget`, or `popoverhidetarget` attributes are applied to an activating element, the UA will automatically map this attribute with the appropriate accessibility semantics. For instance, the initial implementation will expose the appropriate `aria-expanded` state based on whether the popover is shown or hidden. As the popover API matures, there may need to be further discussion with the ARIA working group to determine if additional ARIA semantics, if any, are necessary.

These attributes are only supported on buttons (including `<button>`, `<input type=button>`, etc.) as long as the button would not otherwise submit a form. For example, this is not supported: `<form><input type=submit popovertoggletarget=foo></form>`. In that case, the form would be submitted, and the popover would **not** be toggled.

The declarative trigger attributes can also be accessed via IDL:

```javascript
// These set the IDREF for the target element, and not the element itself:
myButton.popoverToggleTarget = idref
myButton.popoverShowTarget = idref
myButton.popoverHideTarget = idref
```

### Javascript Trigger

To show and hide the popover via Javascript, there are two methods on HTMLElement:

```javascript
const popover = document.querySelector('[popover]')
popover.showPopover() // Show the popover
popover.hidePopover() // Hide a visible popover
```

Calling `showPopover()` on an element that has a valid value for the `popover` attribute will cause the UA to remove the `display:none` rule from the element and move it to the top layer. Calling `hidePopover()` on a showing popover will remove it from the top layer, and re-apply `display:none`.

There are several conditions that will cause `showPopover()` and/or `hidePopover()` to throw an exception:

1. Calling `showPopover()` or `hidePopover()` on an element that does not contain a [valid value](#html-content-attribute) of the `popover` attribute. This will throw a `NotSupportedError` `DOMException`.
2. Calling `showPopover()` on a valid popover that is already in the showing state. This will throw an `InvalidStateError` `DOMException`.
3. Calling `showPopover()` on a valid popover that is not connected to a document. This will throw an `InvalidStateError` `DOMException`.
4. Calling `hidePopover()` on a valid popover that is not currently showing. This will throw an `InvalidStateError` `DOMException`.

<!-- Removing until it's back in action -->
<!-- ### Page Load Trigger

As mentioned above, a `<div popover>` will be hidden by default. If it is desired that the popover should be shown automatically upon page load, the `defaultopen` attribute can be applied:

```html
<div popover defaultopen>
```

In this case, the UA will immediately call `showPopover()` on the element, as it is parsed. If multiple such elements exist on the page, only the first such element (in DOM order) on the page will be shown.

Note that more than one `manual` popover can use `defaultopen` and all such popovers will be shown on load, not just the first one:

```html
<div popover=manual defaultopen>Shown on page load</div>
<div popover=manual defaultopen>Also shown on page load</div>
```

The `defaultopen` content attribute can also be accessed via IDL:

```javascript
myDiv.defaultOpen = true;
``` -->

### CSS Pseudo Class

When a popover is open, it will match the `:open` pseudo class:

```javascript
const popover = document.createElement('div')
popover.popover = 'auto'
popover.matches(':open') === false
popover.showPopover()
popover.matches(':open') === true
```

### Rendering

The UA stylesheet for popovers will look like this:

```css
[popover] {
  position: fixed;
  inset: 0;
  width: fit-content;
  height: fit-content;
  margin: auto;
  border: solid;
  padding: 0.25em;
  overflow: auto;
  color: CanvasText;
  background-color: Canvas;
}

[popover]::backdrop {
    pointer-events: none !important;
}

[popover]{match this when popover is hidden} {
  display: none;
}
```

### Shown vs Hidden Popovers

As seen above, the styling rules manage "showing" vs. "hidden" via these UA stylesheet rules:

```css
[popover]{when hidden}: {
  display: none;
}
[popover] {
  position: fixed;
}
```

The above rules mean that a popover, when not "shown", has `display:none` applied, and that style is removed when one of the methods above is used to show the popover. Note that the `display:none` UA stylesheet rule is not `!important`. In other words, developer style rules can be used to override this UA style to make a not-showing popover visible in the page. In this case, the popover will **not** be displayed in the top layer, but rather at its ordinary `z-index` position within the document. This can be used, for example, to make popover content "return to the page" instead of becoming hidden. Care must be taken, if the `display` property is set by developer CSS, to ensure the popover visibility isn't adversely affected. For example, developer CSS could do this:

```css
[popover] {
  /* Make the popover a flexbox: */
  display: flex;
}
[popover]:not(:open) {
  /* But make sure not to show it unless it's open: */
  display: none;
}
```

Be mindful when using other stylesheets that you haven't authored. These may set the `display` value on your popover elements. For example, if you use a `menu` element as a `popover` and you use earlier versions of [normalize.css](https://necolas.github.io/normalize.css/). In this case, normalize.css has a rule that sets `display: flex` on `menu` elements.

### Animation of Popovers

The show and hide behavior for popovers is designed to make animation of the show/hide trivially easy:

```css
[popover] {
  opacity: 0;
  transition: opacity 0.5s;
}
[popover]:open {
  opacity: 1;
}
```

The above CSS will result in all popovers fading in and out when they show/hide. To make this work, a two-step procedure must be followed by the popover as it is shown and hidden:

**`showPopover()`:**

1. Fire the `show` event, synchronously. If the event is cancelled, stop here.
2. Move the popover to the top layer, and remove the UA `display:none` style.
3. Update style. (Transition initial style can be specified in this state.)
4. Set the `:open` pseudo class.
5. Update style. (Animations/transitions happen here.)
6. Focus the first contained element with `autofocus`, if any.

**`hidePopover()`:**

1. Capture any already-running animations on the `Element` via getAnimations(), including animations on descendent elements.
2. Stop matching the `:open` pseudo class.
3. If the hidePopover() call is the result of the popover being **"forced out"** of the top layer, e.g. by a modal dialog, fullscreen element, or the element being removed from the document:

   a. Queue the `hide` event (asynchronous).

   **Otherwise,**

   a. Fire the `hide` event, synchronously.

   b. Restore focus to the previously-focused element.

   c. Update style. (Animations/transitions start here.)

   d. Call getAnimations() again, remove any that were found in step <span>#</span>1, and then wait until all of the remaining animations finish or are cancelled.

4. Remove the popover from the top layer, and add the UA `display:none` style.
5. Update style.

In addition to the above, if a "higher priority" top layer element later appears (such as a modal dialog or fullscreen element), step <span>#</span>4/5 of the `hidePopover()` steps are immediately executed, to immediately hide any animating-hide popovers.

Note that because the `popoverhide` event is synchronous, animations can also be triggered from within the `popoverhide` event listener:

```javascript
popover.addEventListener('popoverhide', () => {
  popover.animate({ transform: 'translateY(-50px)' }, 200)
})
```

Also note that **while animating**, a popover will be **a)** in the top layer, but **b)** not matching the `:open` pseudo class. This is required in order that transitions can be triggered by matching `:open` as seen in the CSS above. It will be important, generally, for developer tools to provide helpful information about the state of top-layer elements.

## IDL Attribute and Feature Detection

The `popover` content attribute will be [reflected](https://html.spec.whatwg.org/#reflect) as a nullable IDL attribute:

```webidl
[Exposed=Window]
partial interface Element {
  attribute DOMString? popover;
```

This not only allows developer ease-of-use from Javascript, but also allows for a feature detection mechanism:

```javascript
function supportsPopover() {
  return HTMLElement.prototype.hasOwnProperty('popover')
}
```

Further, only [valid values](#html-content-attribute) of the content attribute will be reflected to the IDL property, with invalid values being reflected as the value `manual`. For example:

```javascript
const div = document.createElement('div')
div.setAttribute('popover', 'AUTO')
div.popover === 'auto' // true (note lowercase)
div.setAttribute('popover', 'invalid!')
div.popover === 'manual' // true
div.removeAttribute('popover')
div.popover === null // true
```

This allows feature detection of the values, for forward compatibility:

```javascript
function supportsPopoverType(type) {
  if (!HTMLElement.prototype.hasOwnProperty('popover')) return false // Popover API not supported
  const testElement = document.createElement('div')
  testElement.popover = type
  return testElement.popover == type.toLowerCase()
}
supportsPopoverType('manual') === true
supportsPopoverType('invalid!') === false
```

## Events

An event is fired synchronously when a popover is shown or hidden. This event can be used, for example, to populate content for the popover just in time before it is shown, or update server data when it closes. The event provides a `currentState` and `newState` for the popover. The value of these properties can either be "open" or "closed". The events looks like this:

```javascript
const popover = Object.assign(document.createElement('div'), { popover: 'auto' })
popover.addEventListener('beforetoggle', ({ currentState, newState }) => {
  if (currentState === 'closed') console.info('Popover is closed')
  if (newState === 'open') console.info('Popover is being shown')
})
```

The `beforetoggle` event is cancellable when the `newState` is equal to "open". Doing so keeps the popover from being shown. You can't cancel the hiding of a popover as that would interfere with the one-at-a-time and light-dismiss behaviors. The `beforetoggle` event is fired synchronously.

## Focus Management

Elements that move into the top layer may require focus to be moved to that element, or a descendant element. However, not all elements in the top layer will require focus. For example, a modal `<dialog>` will have focus set to its first interactive element, if not the dialog element itself, because a modal dialog is something that requires immediate attention. On the other hand, a `<div popover=manual>`, which may represent a dynamic notification message (commonly referred to as a "notification", "toast", or "alert"), or potentially a persistent chat widget, should not immediately receive focus, even if it contains focusable elements. This is because such popovers are meant for out-of-band communication of state, and are not meant to interrupt a user's current action. Additionally, if the top layer element **should** receive immediate focus, there is a question about **which** part of the element gets that initial focus. For example, the element itself could receive focus, or one of its focusable descendants could receive focus.

To provide control over these behaviors, the `autofocus` attribute can be used on or within popovers. When present on a popover or one of its descendants, it will result in focus being moved to the popover or the specified element when the popover is shown. Note that `autofocus` is [already a global attribute](https://html.spec.whatwg.org/multipage/interaction.html#the-autofocus-attribute), but the existing behavior applies to element focus on **page load**. This proposal extends that definition to be used within popovers, and the focus behavior happens **when they are shown**. Note that adding `autofocus` to a popover descendant does **not** cause the popover to be shown on page load, and therefore it does not cause focus to be moved into the popover **on page load**, unless the `defaultopen` attribute is also used.

The `autofocus` attribute allows control over the focus behavior when the popover is shown. When the popover is hidden, often the most user friendly thing to do is to return focus to the previously-focused element. The `<dialog>` element currently behaves this way. However, for popovers, there are some nuances. For example, if the popover is being hidden via light dismiss, because the user clicked on another element outside the popover, the focus should **not** be returned to another element, it should go to the clicked element (if focusable, or `<body>` if not). There are a number of other such considerations. The behavior on hiding the popover is:

- A popover element has a **previously focused element**, initially `null`, which is set equal to `document.activeElement` when the popover is shown, if a) the popover is a `auto` popover, and b) if the [popover stack](#the-popover-stack) is currently empty. The **previously focused element** is set back to `null` when a popover is hidden.

- When a popover is hidden, focus is set back to the **previously focused element**, if it is non-`null`, in the following cases:

  1. Light dismiss via [close signal](https://wicg.github.io/close-watcher/#close-signal) (e.g. Escape key pressed).
  2. Hide popover from Javascript via `hidePopover()`.
  3. Hide popover via a **popover-contained**\* triggering element with `popoverhidetarget=pop_up_id` or `popovertoggletarget=pop_up_id`. The triggering element must be popover-contained, otherwise the keyboard or mouse activation of the triggering element should have already moved focus to that element.
  4. Hide popover because its `popover` type changes (e.g. via `myPopover.popover='something else';`).

- Any other actions which hide the popover will **not** cause the focus to be changed when the popover hides. In these cases, the "normal" behavior happens for each action. Some examples include:
  1. Click outside the popover (focus the clicked thing).
  2. Click on a **non-popover-contained**\* triggering element to close popover (focus the triggering element). This is a special case of the point just above.
  3. Tab-navigate (focus the next tabbable element in the document's focus order).
  4. Hide popover because it was removed from the document (event handlers not allowed while removing elements).
  5. Hide popover when a modal dialog or fullscreen element is shown (follow dialog/fullscreen focusing steps).
  6. Hide popover via `showPopover()` on another popover that hides this one (follow new popover focusing steps).

The intention of the above set of behaviors is to return focus to the previously focused element when that makes sense, and not do so when the user's intention is to move focus elsewhere or when it would be confusing.

\* In the above, "a popover contained triggering element" means the triggering element is contained within **the popover being hidden**, not any arbitrary popover.

## Anchoring

A new attribute, `anchor`, can be used on a popover element to refer to the popover's "anchor". The value of the attribute is a string idref corresponding to the `id` of another element (the "anchor element"). This anchoring relationship is used for two things:

1. Establish the provided anchor element as an [“ancestor”](#nearest-open-ancestral-popover) of this popover, for light-dismiss behaviors. In other words, the `anchor` attribute can be used to form nested popovers.
2. The referenced anchor element could be used by the [Anchor Positioning feature](https://drafts.csswg.org/css-anchor-1/).

The anchor attribute can also be accessed via IDL:

```javascript
// This sets the IDREF for the anchor element, and not the element itself:
myPopover.anchor = idref
```

## Backdrop

Akin to modal `<dialog>` and fullscreen elements, popovers allow access to a `::backdrop` pseudo element, which is a full-screen element placed directly behind the popover in the top layer. This allows the developer to do things like blur out the background when a popover is showing:

```html
<div popover>I'm a popover</div>
<style>
  [popover]::backdrop {
    backdrop-filter: blur(3px);
  }
</style>
```

Note that in contrast to the `::backdrop` pseudo element for modal dialogs and fullscreen elements, the `::backdrop` for a popover is styled by a UA stylesheet rule `pointer-events: none !important`, which means it cannot trap clicks outside the popover. This ensures the "click outside" light dismiss behavior continues to function. For this reason, and because `popover=manual` popovers do not have light-dismiss behavior, it is not recommended that the `::backdrop` be styled in a non-transparent way for `popover=manual` popovers. Doing so would obscure the rest of the page, while still allowing the user to click or keyboard navigate to obscured elements. In this case, a better approach might be to use a modal `<dialog>` element, which will trap focus within the `<dialog>`.

# Behaviors

## Automatic Dismiss Behavior

For elements that are displayed on the top layer via this API, there are a number of things that can cause the element to be removed from the top-layer, besides the ones [described above](#showing-and-hiding-a-popover). These fall into three main categories:

- **One at a Time.** Another element being added to the top-layer causes existing top layer elements to be removed. This is typically used for “one at a time” type elements: when one popover is shown, other popovers should be hidden, so that only one is on-screen at a time. This is also used when “more important” top layer elements get added. For example, fullscreen elements should close all open popovers.
- **Light Dismiss.** User action such as clicking outside the element, hitting Escape, or causing keyboard focus to leave the element should all cause a displayed popover to be hidden. This is typically called “light dismiss”, and is discussed in more detail in [this section](#light-dismiss).
- **Other Reasons.** Because the top layer is a UA-managed resource, it may have other reasons (for example a user preference) to forcibly remove elements from the top layer.

In all such cases, the UA is allowed to forcibly remove an element from the top layer and re-apply the `display:none` popover UA rule. The rules the UA uses to manage these interactions depends on the element types, and this is described in more detail in [this section](#classes-of-top-layer-ui).

### Light Dismiss

The term "light dismiss" for a popover is used to describe the user "moving on" to another part of the page, or generally being done interacting with the popover. When this happens, the popover should be hidden. Several actions trigger light dismiss of a popover:

1. Clicking or tapping outside the bounds of the popover. Any `mousedown` event will trigger **all open popovers** to be hidden, starting from the top of [the popover stack](#the-popover-stack), and ending with the [nearest open ancestral popover](#nearest-open-ancestral-popover) of the `mousedown` event's `target` `Node`. This means clicking on a popover or its trigger or anchor elements will not hide that popover.
2. Hitting the Escape key, or generally any ["close signal"](#close-signal). This will cause the topmost popover on [the popover stack](#the-popover-stack) to be hidden.

### Nested Popovers

For `popover=auto` only, it is possible to have "nested" popovers. I.e. two popovers that are allowed to both be open at the same time, due to their relationship with each other. A simple example where this would be desired is a popover menu that contains sub-menus: it is commonly expected to support this pattern, and keep the main menu showing while the sub-menu is shown.

Popover nesting is not possible/applicable to the other popover types, such as `popover=manual`.

### The Popover Stack

The `Document` contains a "stack of open popovers", which is initially empty. When a `popover=auto` element is shown, that popover is pushed onto the top of the stack, and when a `popover=auto` element is hidden, it is popped from the top of the stack.

### Nearest Open Ancestral Popover

The "nearest open ancestral popover" `P` to a given `Node` N is defined in this way:

> `P` is the topmost popover in [the popover stack](#the-popover-stack) for which any one of the following is true:
>
> 1. `P` is a flat-tree DOM ancestor of `N`.
> 2. `N` is a descendant of another popover `P2`, which has an `anchor` attribute whose value is equal to `P`'s `id` or any of `P`'s flat-tree descendent's `id`s.
> 3. `P` contains a [triggering element](#declarative-triggers) whose target is another popover `P2`, which contains `N`.
>
> If none of the popovers in [the popover stack](#the-popover-stack) match the above conditions, then `P` is `null`.

The purpose of the algorithm above is to allow "nesting" of `popover=auto` popovers in several natural ways:

1.  **Using traditional DOM tree descendants:** when one popover is a descendant of another popover, these two are "nested".
2.  **Using the `anchor` attribute:** when a popover has an `anchor` attribute that points to another popover (or descendant of another popover), these two are "nested". This allows the `anchor` attribute to be used with the [Anchor Positioning API](https://tabatkins.github.io/specs/css-anchor-position/), transparently defining a clear "nested" relationship between the popovers. It is not necessary to use the Anchor Positioning API.
3.  **Using a [declarative triggering element](#declarative-triggers) (e.g. `popovertoggletarget`):** when a triggering element is contained in a popover, and can therefore trigger another popover, the second popover is "nested" within the first.

Note that because [the popover stack](#the-popover-stack) can only contain `popover=auto` popovers, it is only possible to "nest" popovers within `popover=auto` popovers.

### Close signal

The "close signal" [proposal](https://wicg.github.io/close-watcher/#close-signal) attempts to unify the concept of "closing" something. Most typically, the Escape key is the standard close signal, but there are others, including the Android back button, Accessibility Tools dismiss gestures, and the Playstation square button. Any of these close signals is a light dismiss trigger for the topmost popover.

## Classes of Top Layer UI

As described in this section, the popover types (`auto` and `manual`) each have slightly different interactions with each other. For example, `auto`s hide other `auto`s, but `manual`s do not close each other. Additionally, there are other (non-popover) elements that participate in the top layer. This section describes the general interactions between the various top layer element types, including the various flavors of popover:

- Popover (**`popover=auto`**)
  - When opened, force-closes other `popover=auto`s, except for [ancestor popovers](#nearest-open-ancestral-popover).
  - It would generally be expected that a popover of this type would either receive focus, or a descendant element would receive focus when invoked.
  - Dismisses on [close signal](https://wicg.github.io/close-watcher/#close-signal) or a click outside the element.
- Manual (**`popover=manual`**)
  - Does not force-close any other element type.
  - Does not light-dismiss - closes via timer or explicit close action.
- Dialog (**`<dialog>.showModal()`**)
  - When opened, force-closes `popover=auto`.
  - Dismisses on [close signal](https://wicg.github.io/close-watcher/#close-signal)
- Fullscreen (**`<div>.requestFullscreen()`**)
  - When opened, force-closes `popover=auto`, and (with spec changes) dialog
  - Dismisses on [close signal](https://wicg.github.io/close-watcher/#close-signal)

## Accessibility / Semantics

Since the `popover` content attribute can be applied to any element, and this only impacts the element's presentation (top layer vs not top layer), this addition does not have any direct semantic or accessibility impact. The element with the `popover` attribute will keep its existing semantics and AOM representation. For example, `<article popover>...</article>` will continue to be exposed as an implicit `role=article`, but will be able to be displayed on top of other content. Similarly, ARIA can be used to modify accessibility mappings in [the normal way](https://w3c.github.io/html-aria/), for example `<div popover role=note>...</div>`.

As mentioned in the [Declarative Triggers](#declarative-triggers) section, accessibility mappings will be automatically configured to associate the popover with its trigger element, as needed.

## Disallowed elements

While the popover API can be used on most elements, there are some limitations. It is legal to apply the `popover` attribute to a `<dialog>` element, for example. However, doing so causes `dialog.showModal()` (the `<dialog>` API to show it modally) to throw an exception. This is because it is confusing that an element with `popover` can be shown in a modal fashion. Similarly, calling `element.requestFullscreen()` on an element that has the `popover` attribute will return a rejected promise.

# Example Use Cases

This section contains several HTML examples, showing how various UI elements might be constructed using this API.

**Note:** these examples are for demonstrative purposes of how to use the `popovertoggletarget` and `popover` attributes. They may not represent all necessary HTML, ARIA or JavaScript features needed to fully create such components.

## Generic Popover (Date Picker)

```html
<button popovertoggletarget="datepicker">Pick a date</button>
<my-date-picker role="dialog" id="datepicker" popover>...date picker contents...</my-date-picker>

<!-- No script - the popovertoggletarget attribute takes care of activation, and
     the `popover` attribute takes care of the popover behavior. -->
```

## Generic Popover (`<selectmenu>` listbox example)

```html
<selectmenu>
  <div popover slot="listbox" behavior="listbox">
    <option>Option 1</option>
    <option>Option 2</option>
  </div>
</selectmenu>

<!-- No script - `<selectmenu>`'s listbox is provided by a `<div popover>` element,
     which takes care of popover behavior -->
```

## Manual

```html
<div role="alert">
  <my-manual-container popover="manual"></my-manual-container>
</div>

<script>
  window.addEventListener('message', (e) => {
    const container = document.querySelector('my-manual-container')
    container.appendChild(document.createTextNode('Msg: ' + e.data))
    container.showPopover()
  })
</script>
```

# Additional Considerations

## Exceeding the Frame Bounds

Allowing a popover/top-layer element to exceed the bounds of its containing frame poses a serious security risk: such an element could spoof browser UI or containing-page content. While the [original `<popup>` proposal](https://open-ui.org/components/popup.research.explainer-v1) did not discuss this issue, the [`<selectmenu>` proposal](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/ControlUICustomization/explainer.md#security) does have a specific section at least mentioning this issue. Some top-layer APIs (e.g. the fullscreen API) make it possible for an element to exceed the frame bounds in some cases, great care must be taken in these cases to ensure user safety. Given the complete flexibility offered by the popover API (any element, arbitrary content, etc.), there would be no way to ensure the safety of this feature if it were allowed to exceed frame bounds.

For completeness, several use counters were added to Chromium to measure how often this type of behavior (content exceeding the frame bounds) might be needed. These are approximations, as they merely measure the total number of times one of the built-in “popover” windows, which can exceed frame bounds because of their carefully-controlled content, is shown. The popovers included in this count include the `<select>` popover, the `<input type=color>` color picker, and the `<input type=date/etc>` date/time picker. Data can be found here:

- Total popovers shown: [0.7% of page loads](https://chromestatus.com/metrics/feature/timeline/popularity/3298)
- Popovers appeared outside frame bounds: [0.08% of page loads](https://chromestatus.com/metrics/feature/timeline/popularity/3299)

So about 11% of all popovers currently exceed their owner frame bounds. That should be considered a rough upper bound, as it is possible that some of those popovers **could** have fit within their frame if an attempt was made to do so, but they just happened to exceed the bounds anyway.

In any case, it is important to note that this API cannot be used to render content outside the containing frame.

## Shadow DOM

Note that using the API described in this explainer, it is possible for elements contained within a shadow root to be popovers. For example, it is possible to construct a custom element that wraps a popover type UI element, such as a `<my-popover>`, with this DOM structure:

```html
<my-popover>
  <template shadowroot="closed">
    <div popover="auto">
      This is a popover:
      <slot></slot>
    </div>
  </template>
  Popover text here!
</my-popover>
```

In this case, the (closed) shadow root contains a `<div>` that has `popover=auto` and that element will be shown on the top layer when the custom element calls `div.showPopover()`.

This is "normal", and the only point of this section is to point out that even shadow dom children can be promoted to the top layer, in the same way that a shadow root can contain a `<dialog>` that can be `showModal()`'d, or a `<div>` that can be `requestFullscreen()`'d.

## Eventual Single-Purpose Elements

There might come a time, sooner or later, where a new popover-type HTML element is desired which combines strong semantics and purpose-built behaviors. For example, a `<notification>` or `<listbox>` element. Those elements could be relatively easily built via the APIs proposed in this document. For example, a `<notification>` element could be defined to have `role=alert` and `popover=manual`, and therefore re-use this popover API for always-on-top rendering. In other words, these new elements could be _explained_ in terms of the lower-level primitives being proposed for this API.

# The Choices Made in this API

Many decisions and choices were made in the design of this API, and those decisions were made via numerous discussions (live and on issues) in [OpenUI](https://open-ui.org/), a WHATWG [Community Group](https://open-ui.org/working-mode).

## Alternatives Considered

To achieve the [goals](#goals) of this project, a number of approaches could have been used:

- An HTML content attribute (this proposal).
- A dedicated `<popup>` element.
- A CSS property.
- A Javascript API.

Each of these options is significantly different from the others. To properly evaluate them, each option was fleshed out in some detail. Please see this document for the details of that effort:

- [Other Alternatives Considered](/components/popup.proposal.alternatives)

That document discusses the pros and cons for each alternative. After exploring these options, the [HTML content attribute](#html-content-attribute) approach [was resolved by OpenUI](https://github.com/openui/open-ui/issues/455#issuecomment-1050172067) to be the best overall.

## Why a content attribute?

Again, refer to the [Other Alternatives Considered](/components/popup.proposal.alternatives) document for an exhaustive look at the other alternatives. That document runs through a fairly complete design of the other alternatives, to see what they would look like.

This section simply tries to summarize the primary reason that a content attribute was chosen to enable the popover behavior: a content attribute allows _behavior_ to be applied to _any element_. That is useful:

```html
<dialog popover="auto">
  This is a "light dismiss" dialog
  <button>Ok</button>
  <button>Cancel</button>
</dialog>
```

Here, the developer is building **a dialog**. Since HTML [strongly encourages](https://html.spec.whatwg.org/multipage/dom.html#semantics-2) the use of the correct element that properly represents the **semantics of the content**, the proper element in this case is a `<dialog>`.

```html
<menu popover="auto">
  <li><button>Item 1</button></li>
  <li><button>Item 2</button></li>
  <li><button>Item 3</button></li>
</menu>
```

Similarly here, the developer is building **a menu**, so they should use the `<menu>` element. This adheres to the semantic definition of a `<menu>` element, and allows the content to be programmatically exposed as a listing of buttons.

In both cases, not only does the use of the semantically-correct element make it easier to understand the content, it also enforces the appropriate content rules and behaviors that are specific to each element. Further, using the correct element allows the UA to correctly represent the content in the accessibility tree, making assistive technologies better able to assist users in navigating the content.

In an alternative proposal where the popover behavior is enabled via a special element, e.g. `<popup>`, all of the above would need to be carefully managed by the developer via ARIA roles and attributes, as there is no single role that can consistently be used to identify a `<popup>` element:

```html
<popup role="dialog">
  <popup role="list">...etc...</popup>
</popup>
```

and that violates the [first rule of ARIA](https://www.w3.org/TR/using-aria/#firstrule), which is essentially that if there's an element that properly represents the content, use that, and don't use ARIA.

By having `popover` be a content attribute that purely confers behavior upon an existing element, the above problems are nicely resolved. Semantics are provided by elements, and behaviors are confered on those elements via attributes. This situation is exactly analogous to `contenteditable` or `tabindex`, which confer specific behaviors on any element. Imagine a Web in which those two attributes were instead elements: `<contenteditable>` and `<tabindex index=0>`. In that Web, many common patterns would either be very convoluted or simply not possible.

## Design decisions (via [OpenUI](https://open-ui.org/))

Many small (and large!) behavior questions were answered via discussions at OpenUI. This section contains links to some of those:

- [Overall discussion of the content attribute based approach](https://github.com/openui/open-ui/issues/455)
- [Why `show`/`hide` and not `open`/`close`, and why not parallel to `<dialog>`](https://github.com/openui/open-ui/issues/322)
- [Rules for multiple triggering elements on a single button.](https://github.com/openui/open-ui/issues/523#issuecomment-1106686358)
- [Should there be a `::backdrop` pseudo element](https://github.com/openui/open-ui/issues/519)
- [Should `showPopUp()` and `hidePopUp()` throw](https://github.com/openui/open-ui/issues/511)
- [Should `blur()` be a light dismiss trigger](https://github.com/openui/open-ui/issues/415)
- [IDL reflects only valid values](https://github.com/openui/open-ui/issues/491#issuecomment-1118927375)
- [Why `togglepopup` (bikeshed)](https://github.com/openui/open-ui/issues/508)
- [Why `defaultopen` (bikeshed)](https://github.com/openui/open-ui/issues/500)
- [Why `display:none` for hidden popovers](https://github.com/openui/open-ui/issues/492)
- [Why Close Signals and not just ESC](https://github.com/openui/open-ui/issues/320)
- [Naming of the `:top-layer`/`:open` pseudo class](https://github.com/openui/open-ui/issues/470)
- [Support for "boolean-like" behavior for `popup` attribute](https://github.com/openui/open-ui/issues/533)
- [Returning focus to previously-focused element](https://github.com/openui/open-ui/issues/327)
- [The `show` and `hide` events should not be cancellable](https://github.com/openui/open-ui/issues/321)
- [The `show` event should be cancellable after all](https://github.com/openui/open-ui/issues/579)
- [The `popup` attribute confers behavior and not semantics](https://github.com/openui/open-ui/issues/495#issuecomment-1164827851)
- [Mouse down vs mouse up for light dismiss](https://github.com/openui/open-ui/issues/529)
- [Imperative API for content attributes](https://github.com/openui/open-ui/issues/382)
- [`.popup` vs `.popUp`](https://github.com/openui/open-ui/issues/546#issuecomment-1158190204)
- [Interactions between auto, hint, and manual](https://github.com/openui/open-ui/issues/525)
- [Show and hide animation behavior](https://github.com/openui/open-ui/issues/335)
- [`popuptoggletarget`, `popupshowtarget`, `popuphidetarget`](https://github.com/openui/open-ui/issues/382#issuecomment-1184773425)
- [Invoking attributes only supported on buttons](https://github.com/openui/open-ui/issues/420)
- [Differences between dialog and popover](https://github.com/openui/open-ui/issues/581)
- [Interactions between popover, dialog, and fullscreen](https://github.com/openui/open-ui/issues/520)

Here are all non-spec-text related OpenUI popover issues, both open and closed:

https://github.com/openui/open-ui/issues?q=is%3Aissue+label%3Apopover+-label%3Apopover-spec

Here are all current, open, non-spec-text related OpenUI popover issues:

https://github.com/openui/open-ui/issues?q=is%3Aissue+is%3Aopen+label%3Apopover+-label%3Apopover-spec+-label%3Apopover-v2

## Articles

- [Pop-ups: They're making a resurgence! – developer.chrome.com](https://developer.chrome.com/blog/pop-ups-theyre-making-a-resurgence/)
- [Dialogs, modality and popovers seem similar. How are they different? – hidde.blog](https://hidde.blog/dialog-modal-popover-differences/)
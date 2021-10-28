# Cooperatively-sized iframes

Developers have [continually asked for](https://stackoverflow.com/search?q=resize+iframe) the ability for an iframe to change size based on its contents. This proposal is trying to make that happen!

## Examples

### Declarative CSS-based sizing

An iframe that always auto-resizes to fit the content inside of it:

```html
<iframe style="contain-intrinsic-size: from-element 500px 500px"
        src="iframe.html"></iframe>

<!-- In iframe.html --->
<html style="intrinsic-size: auto">
```

This example illustrates the two-sided opt-in for this feature:

* The containing page styles the `<iframe>` element with `contain-intrinsic-size: from-element`. It can optionally provide an initial intrinsic size, in this case `500px 500px`. The `from-element` indicates that the `<iframe>` element's [intrinsic size](https://developer.mozilla.org/en-US/docs/Glossary/Intrinsic_Size) will be taken from whatever the contained page indicates.
* The iframed page styles its `<html>` element with `intrinsic-size: auto`. The `intrisic-size` property is how it communicates the desired intrinsic size up to the containing page; the value of `auto` indicates that the desired intrisic size equal to its laid-out size.

The final result is that the iframe will start out at 500x500, but quickly change as content loads into it and the browser does appropriate layout synchronization work. (This work could potentially span multiple processes, and so could take a couple of frames to go through.) If we just use `contain-intrinsic-size: from-element` without the `500px 500px` default, then iframe will start out at the usual 300x150 size until such layout work happens.

Note that [`contain-intrisic-size`](https://developer.mozilla.org/en-US/docs/Web/CSS/contain-intrinsic-size) is a preexisting CSS property; the `from-element` value is new in this proposal. The `intrinsic-size` property, on the other hand, is entirely new to this proposal.

A modification of the above example where the containing page can place some restrictions on the iframe's size, to make sure it never goes above 600x600:

```html
<iframe style="contain-intrinsic-size: from-element 500px 500px;
               max-width: 600px; max-height: 600px;"
        src="iframe.html"></iframe>
```

A modifiation of the above example where the containing page restricts the width to always be 500px, while letting the height vary freely:

```html
<iframe style="contain-intrinsic-size: from-element 500px 500px;
               width: 500px;"
        src="iframe.html"></iframe>
```

If the iframed page wants to change its size to specific numbers, the most common way to do this would be to change its `<html>` element's `width` and `height` properties. Then the `intrinsic-size: auto` will propagate that size change request to the parent. So for example, if the iframe wanted to always be 800px wide but automatically derive its height from its content, the following would be the simplest way to do that:

```html
<!-- In iframe.html -->
<html style="intrinsic-size: auto; width: 800px;">
```

Alternately, the iframed page can specifically set the `intrinsic-size` property, disconnected from its `width` and `height` values. This requests that the container resizes, separately from the `<html>` element. This is similar to how in a top-level page you can restrict the dimensions of the `<html>` element, even though the viewport will be arbitrarily sized based on the user resizing the window or similar.

```html
<!-- In iframe.html -->
<html style="intrinsic-size: 800px 900px; width: 400px; height: 900px;">
```

### Implications for JavaScript APIs

As usual, all CSS properties can be set from JavaScript using the CSS Object Model APIs. For example, the following iframe starts out at 500x500 (per the outer page's defaults), but randomly resizes every second according to the code inside the iframe:

```html
<iframe style="contain-intrinsic-size: from-element 500px 500px"
        src="iframe.html"></iframe>

<!-- In iframe.html --->
<script>
setInterval(() => {
  document.documentElement.style.intrinsicWidth = Math.random() * 1_000;
  document.documentElement.style.intrinsicHeight = Math.random() * 1_000;
}, 1_000);
</script>
```

(As a reminder, if we just use `contain-intrinsic-size: from-element`, without the `500px 500px` default, then iframe will start out at the usual 300x150 size until the first update of `intrinsic-size`.)

Other preexisting JavaScript APIs that are likely to be used in tandem with this feature are the `resize` event, and the various methods of measuring viewport size. Crucially, these give insight into whether a resize request actually went through or not:

```js
// In iframe.html

window.onresize = () => {
  // If this doesn't get logged, then the outer page probably didn't have
  // `contain-intrinsic-size: from-element`. Or maybe you were already at the max/min size
  // and so nothing changed.
  console.log("I did end up resizing!");
  console.log("Final viewport dimensions, including any restrictions due to max-width/max-height/etc.:");
  console.log(`${window.innerWidth}x${window.innerHeight}`);
};

// Try resizing! It might cause the above event!
document.documentElement.style.intrinsicSize = "900px 900px";
```

Note that these resizes will always be asynchronous, i.e., the `resize` event will not fire, and `window.innerWidth` will not change, until at least another event loop turn. TODO see if we can say anything more specific, once we get into spec details.

## The proposal

We introduce new `intrinsic-width`, `intrinsic-height`, `intrinsic-block-size`, `intrinsic-inline-size`, and `intrinsic-size` properties. These have the usual relations to each other as other CSS box sizing properties. From now on we only discuss `intrinsic-width` and `intrinsic-height`.

These properties can have either `none`, `auto`, or a `<length>` as their value, with `none` being the default. These properties only have any impact on the document element (usually a `<html>` element). When applied to that element, they set the element's "from-element intrinsic width/height" as follows:

* If the value is `none`, then the from-element intrinsic with/height is null.
* If the value is `auto`, then the from-element intrinsic with/height automatically updates to match the document element's laid-out width/height. (TODO formally, what is the "laid-out" size?)
* Otherwise, if it's a `<length>`, then the from-element intrinsic width/height is set to that value. (TODO probably need to say something about computation context in case people use relative units.)

TODO `auto` seems pretty subtle: We need to make sure that it isn't constrained by the container, but instead by the viewport (for width) and not constrained (for height).

We then introduce new values for the [existing `contain-intrisic-width` and `contain-intrinsic-height` properties](https://drafts.csswg.org/css-sizing-4/#intrinsic-size-override): `from-element` and `from-element <length>`. (These also affect `contain-intrinsic-block-size`, `contain-intrisic-inline-size`, and `contain-intrinsic-size` in the usual ways.) They work as follows:

* If the value is `from-element`, then the element's explicit intrinsic inner width/height is equal to its from-element intrinsic width/height if that is not null, or the per-element default value if it is null (e.g., 300px width/150px height for iframes).
* If the value is `from-element <length>`, then the element's explicit intrinsic inner width/height is equal to its from-element intrinsic width/height if that is not null, or to the given length value if it is null.

Note that the viewport sizing inside the iframe (exposed via, e.g., `window.innerWidth` and `window.innerHeight`, or the firing of `resize` events) already derives from the `<iframe>` element's intrinsic size. So, we automatically get the appropriate updates to those APIs via the above mechanisms.

## Security and privacy considerations

The main security and privacy issue with this problem space is that we cannot leak information about cross-origin content without explicit opt-in. That is, if the outer page were able to create an auto-resizing iframe without the embedded page's consent, then the outer page would be able to measure the width/height of the inner page, which leaks potentially sensitive information (such as logged-in state).

This proposal solves that by requiring the inner page to opt in, by using the `intrinsic-size` property on its document element. This serves as an explicit sign-off that it wants to communicate the given intrinsic size request to the outer page.

The outer page opt-in, via `contain-intrinsic-size: from-element`, can also be seen as a security measure: if an iframe were able to cause itself to expand, without consent from the embedding page, then the iframe might be able to cover up important elements of the outer page or induce [clickjacking](https://owasp.org/www-community/attacks/Clickjacking).

Combined, this bidirectional opt-in ensures that we have the same security and privacy properties of today's [ad-hoc custom protocols](#continue-to-do-this-through-ad-hoc-custom-protocols): i.e., this proposal is no more powerful than what can be done today, and is instead just more ergonomic and performant.

A forward-facing note: some upcoming proposals, such as [portals](https://github.com/WICG/portals) and [fenced frames](https://github.com/shivanigithub/fenced-frame/), attempt to restrict communication between cross-origin embedded content and its embedder, for privacy reasons. We must ensure that such features do _not_ pass along the from-element intrinsic width/height values across the embedding boundary, as doing so would provide a communications channel.

## Alternatives considered

### Continue to do this through ad-hoc custom protocols

Developers accomplish cooperatively-sized iframes today through JavaScript message-passing using `postMessage()`: for example,

```html
<iframe class="autosize" src="iframe.html"></iframe>
<script>
window.addEventListener("message", ({ data, source }) => {
  if (data.event === "resize") {
    source.frameElement.style.width = data.width;
    source.frameElement.style.height = data.height;
  }
});
</script>
```

coupled with corresponding code in `iframe.html` to watch for content resizes and send appropriate `{ event: "resize", width, height }` messages to `window.parent`.

Examples of this approach in the wild include [Pym.js](https://blog.apps.npr.org/pym.js/), answers to [this StackOverflow question](https://stackoverflow.com/q/153152/3191), and various links shared by developers in [whatwg/html#555](https://github.com/whatwg/html/issues/555) or [other StackOverflow questions](https://stackoverflow.com/search?q=resize+iframe).

We believe that this pattern deserves promotion to a built-in browser capability for the following reasons:

* Developer satisfaction: the above links show how this is clearly a common desire among web developers. Allowing them to accomplish it easily, without extra third-party script, would make them happy, and enable them to more easily create good user experiences.

* Coordination: providing a standard mechanism for cooperatively-sizing iframes would avoid both sides needing to agree on the message format or library to use. Instead, everyone could use the standard CSS-based pattern.

* Performance: especially given cross-origin iframes, detecting resizes and passing messages across the frame boundary via JavaScript, only then to perform a resize of the containing frame, can be expensive. By providing a direct declaration of intent to the layout engine, browsers can integrate the resizing process into their usual layout passes.

### Using a HTTP header

In [whatwg/html#555](https://github.com/whatwg/html/issues/555) @annevk proposes that the embeddee opt-in should be done using a HTTP header, e.g. `Expose-Height-Cross-Origin: 1`, instead of the CSS `intrinsic-width` and `intrinsic-height` properties we propose here.

This is significantly more difficult for developers to deploy than CSS properties, and so we prefer the CSS-based approach. It could potentially improve the layout latency, as we don't need to wait until style is resolved on the `<html>` element of the iframe to start the auto-resizing process; we have the appropriate opt-in before any bytes of the body are even sent. But on balance that is probably not the right tradeoff.

### Using a JavaScript API

In [w3c/csswg-drafts#1771 (comment)](https://github.com/w3c/csswg-drafts/issues/1771#issuecomment-805117925), @tabatkins proposes that the embeddee opt-in should be using `window.resizeTo()` inside the iframe, instead of the CSS `intrinsic-width` and `intrinsic-height` properties we propose here.

Given the strong desire expressed by web developers, in that thread and others, for a solution that only requires CSS and not JavaScript, we do want to introduce the `intrinsic-size` property to provide that capability. But once we have that, the CSSOM API gives use a JavaScript API via `document.documentElement.style.intrinsicSize`. So `window.resizeTo()` doesn't seem to add anything.

### Using existing properties instead of `intrinsic-size` on the root element

In [w3c/csswg-drafts#1771 (comment)](https://github.com/w3c/csswg-drafts/issues/1771#issuecomment-810633323), @fantasai proposes using `width` and `height`, instead of the `intrinsic-width` and `intrinsic-height` properties we propose here.

Doing so removes the opt-in necessary for [security and privacy](#security-and-privacy-considerations), so it doesn't seem workable. We need a new property, which no existing content is using. Indeed, see [below](#naming-choices) for ideas on making the opt-in property name even more explicit, to emphasize its security and privacy impact.

### Automatic opt-in for same-origin iframes

In the case where the iframe is same-origin with its embedder, there is no security or privacy concern that requires the bidirectional opt-in. However, there is still a compatibility concern: we cannot suddenly make all same-origin iframes automatically size to their content. So we would need at least one of the two opt-ins.

But changing the model in this way, only for same-origin iframes, seems more likely to lead to developer confusion. Which opt-in becomes optional? And how does that impact our [explanation of how the cooperative sizing works](#the-proposal), via the from-element intrinsic width/height? It seems best to keep the same model everywhere, wherein the embedded content signals its desired intrinsic size, and the embedder decides whether to respect that request or not.

### Automatic opt-in for `srcdoc=""` iframes

In [w3c/csswg-drafts#1771 (comment)](https://github.com/w3c/csswg-drafts/issues/1771#issuecomment-806208058), @prlbr proposes that `srcdoc=""` iframes should not require one of the two opt-ins. As with same-origin iframes, this is sound from a security perspective. However, similar to that case, we believe adding such exceptions is likely to create confusing developer experiences, and so it's not worth it just to save some typing.

### Naming choices

We are not firmly set on the naming choices in [the proposal](#the-proposal). Other ideas could be:

* A more explicit name for `intrinsic-size`, e.g. `requested-intrinsic-size`, or `container-intrinsic-size`, or `intrinsic-size-request-exposed-to-embedder-which-they-may-or-may-not-respect`.

* A different value than `auto` to communicate that the from-element intrinsic size should be set based on the laid-out size: e.g. `intrinsic-size: from-content`.

* A different value than `from-element` to communicate that we're using the requested intrinsic size, e.g. `contain-intrinsic-size: requested`, or `contain-intrinsic-size: from-embeddee`.

We welcome feedback on the naming.

## Acknowledgments

This proposal is the latest in a long lineage of proposals. Most recent work in this area was done by @tabatkins, in [w3c/csswg-drafts#1771](https://github.com/w3c/csswg-drafts/issues/1771); this proposal is largely created by fleshing out and trimming down his original [comment](https://github.com/w3c/csswg-drafts/issues/1771#issuecomment-805117925) there. Thanks also to the other participants in that discussion.

Other historical requests for this functionality in the web standards world include:

* [Marc Attinasi on Mozilla Bugzilla](https://bugzilla.mozilla.org/show_bug.cgi?id=80713) in 2001;
* Hixie's work on [seamless iframes](https://html.spec.whatwg.org/commit-snapshots/784c7c3f0e63d5809742f4c32696406186f364a2/#attr-iframe-seamless), and Ben Vinegar's [feedback](https://lists.w3.org/Archives/Public/public-whatwg-archive/2014Feb/0015.html) on it ~2014;
* Craig Francis [on whatwg/html](https://github.com/whatwg/html/issues/555) in 2016 and [on www-style](https://lists.w3.org/Archives/Public/www-style/2017Aug/0045.html) in 2017;
* and likely many others which I have not yet found.

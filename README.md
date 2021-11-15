# Cooperatively-sized iframes

Developers have [continually asked for](https://stackoverflow.com/search?q=resize+iframe) the ability for an iframe to change size based on its contents. This proposal is trying to make that happen!

## Examples

This iframe is communicating to its parent page that it would like to be 480x320 pixels in size:

```html
<iframe style="contain-intrinsic-size: from-element 500px 500px"
        src="iframe.html"></iframe>

<!-- In iframe.html --->
<html requestedwidth="480" requestedheight="320">
```

This example illustrates the two-sided opt-in for this feature:

* The containing page styles the `<iframe>` element with `contain-intrinsic-size: from-element`. It can optionally provide an initial intrinsic size, in this case `500px 500px`. The `from-element` indicates that the `<iframe>` element's [intrinsic size](https://developer.mozilla.org/en-US/docs/Glossary/Intrinsic_Size) will be taken from whatever the contained page indicates.
* The iframed page sets the `requestedwidth=""` and `requestedheight=""` elements on its `<html>` element, to communicate the desired intrinsic size up to the containing page.

The final result is that the iframe will start out at 500x500, but as soon as the iframed content loads enough to parse its `<html>` element, the browser will do appropriate layout synchronization work to resize the iframe to 480x320. (This work could potentially span multiple processes, and so could take a couple of frames to go through.) If we just use `contain-intrinsic-size: from-element` without the `500px 500px` default, then iframe will start out at the usual 300x150 size until such layout work happens.

(Note: [`contain-intrisic-size`](https://developer.mozilla.org/en-US/docs/Web/CSS/contain-intrinsic-size) is a preexisting CSS property; the `from-element` value is new in this proposal.)

### Control from the outside

The containing page can use the usual properties to place some restrictions on the iframe's size, here to make sure it never goes above 600x600:

```html
<iframe style="contain-intrinsic-size: from-element 500px 500px;
               max-width: 600px; max-height: 600px;"
        src="iframe.html"></iframe>
```

Similarly, it could restrict the width to always be 500px (disregarding the requested width), while letting the height vary freely:

```html
<iframe style="contain-intrinsic-size: from-element 500px 500px;
               width: 500px;"
        src="iframe.html"></iframe>
```

An alternate way of specifying this same thing (TODO double-check that this is truly equivalent?) would be

```html
<iframe style="contain-intrinsic-width: 500px;
               contain-intrinsic-height: from-element 500px;"
        src="iframe.html"></iframe>
```

### Requests from the inside

The iframed content can change its `requestedwidth=""` and `requestedheight=""` attributes at any time to communicate up a new request. As a silly example, the following will randomly request a resize every second:

```js
setInterval(() => {
  // You can use the requestedWidth/requestedHeight properties
  document.documentElement.requestedWidth = Math.random() * 1_000;

  // Or directly modify the attributes
  document.documentElement.setAttribute("requestedheight", Math.random() * 1_000);
}, 1_000);
```

More realistically, we expect this to be used by the inner page to request a resize to the current content size. This could be done on `load` (or `DOMContentLoaded`), like so:

```js
document.addEventListener("load", () => {
  document.documentElement.requestedWidth = window.scrollWidth;
  document.documentElement.requestedHeight = window.scrollHeight;
});
```

or it could be done continually, by monitoring appropriate resizes, using code such as the following:

```js
const ro = new ResizeObserver(entries => {
  const size = entries.at(-1).borderBoxSize;
  document.documentElement.requestedWidth = size.inlineSize;
  document.documentElement.requestedHeight = size.blockSize;
});

ro.observe(document.documentElement);
```

Be careful with this sort of automatic resizing, however, as it can easily cause infinite loops. That is, if you have absolutely- or fixed-positioned elements on your page, changing the viewport size (by changing the outer iframe size) will cause them to shift, which could in turn cause the inner `<html>` element to grow, which would trigger another resize observer callback, etc.

### Observing the results from the inside

Using the existing `resize` event, code inside an iframe can already observe when the iframe resizes. This is particularly useful in the context of this proposal, where it can tell you whether a resize request was granted by the outer frame:

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
document.documentElement.requestedWidth = 900;
```

Note that these resizes will always be asynchronous, i.e., the `resize` event will not fire, and `window.innerWidth` will not change, until at least another event loop turn. TODO see if we can say anything more specific, once we get into spec details.

## The proposal

We introduce new `requestedwidth=""` and `requestedheight=""` content attributes for the `<html>` element, and maybe for the `<svg>` element as well.

When these attributes are changed, they communicate up the frame tree to set the `<iframe>` element's "from-element intrinsic width/height". The values are [parsed as a floating point number](https://html.spec.whatwg.org/#rules-for-parsing-floating-point-number-values). If one or both of the attributes are absent or unparseable, then then the from-element intrinsic width and/or height are set to null.

These attributes are [reflected](https://html.spec.whatwg.org/#reflecting-content-attributes-in-idl-attributes) via `innerWidth`/`innerHeight` properties. The exact reflection algorithm is tricky here:

* We probably want to use floating-point instead of integer types, since CSS pixels are pretty coarse these days
* We need a special indicator for absent or unparseable. Falling back to 300 and 150 loses a bit of information; e.g., `requestedwidth="asdf"` will use any provided fallback value from the parent's `contain-intrinsic-size`, whereas an explicit `requestedwidth="300"` will ignore the parent's fallback value, so reflecting them both as 300 might be subpar. `null` might work.
* We want to preserve space for future expansion into non-numeric values like [`"auto-on-load"`](#but-what-about-auto-resizing). Maybe that's best done by just expanding the property's type from `double` to `(double or DOMString)` at that time.

We then introduce new values for the [existing `contain-intrisic-width` and `contain-intrinsic-height` properties](https://drafts.csswg.org/css-sizing-4/#intrinsic-size-override): `from-element` and `from-element <length>`. (These affect `contain-intrinsic-block-size`, `contain-intrisic-inline-size`, and `contain-intrinsic-size` in the usual ways.) They work as follows:

* If the value is `from-element`, then the element's explicit intrinsic inner width/height is equal to its from-element intrinsic width/height if that is not null, or the per-element default value if it is null (e.g., 300px width/150px height for iframes).
* If the value is `from-element <length>`, then the element's explicit intrinsic inner width/height is equal to its from-element intrinsic width/height if that is not null, or to the given length value if it is null.

Note that the viewport sizing inside the iframe (exposed via, e.g., `window.innerWidth` and `window.innerHeight`, or the firing of `resize` events) already derives from the `<iframe>` element's intrinsic size. So, we automatically get the appropriate updates to those APIs via the above mechanisms.

## But what about auto-resizing?

The above proposal requires the developer of content inside the iframe to provide explicit pixel values for width and height. We've heard in many discussions about this problem space that web developers would prefer to just be able to set something once, and have the iframe automatically expand or contract as its content changes. For example, it'd be ideal if we could do something like

```html
<!-- Does not work -->
<html requestedwidth="auto" requestedheight="auto">
```

and then never have to update these values again.

The problem here is the one mentioned above with the `ResizeObserver` example: it's easy for this to cause infinite resize loops, where the changing viewport size causes an changing requested size, which causes a changing viewport size, etc. The feedback we have so far, at least from the Chrome team, is that it's not OK to give developers this sort of automatic footgun. If they want to potentially get themselves into infinite loops, they'll need to hook up the `ResizeObserver` themselves.

There are a number of ways we could attempt to overcome this issue. For example, we could introduce one-shot auto-resizing when the `load` event fires inside the iframe, using `requestedwidth="auto-on-load"`. Or we could introduce a different layout mode for such iframes, which would avoid the infinite loops at the cost of making absolutely- and fixed-position content, or percentage measurements, behave in unexpected ways. See [#3](https://github.com/domenic/cooperatively-sized-iframes/issues/3) for some discussion of the options there.

Our current thinking is that we should start with only explicit sizing, while leaving open the potential for future expansion into some form of auto-sizing as discussions continue there. We welcome feedback from the community on that approach.

## Security and privacy considerations

The main security and privacy issue with this problem space is that we cannot leak information about cross-origin content without explicit opt-in. That is, if the outer page were able to create an auto-resizing iframe without the embedded page's consent, then the outer page would be able to measure the width/height of the inner page, which leaks potentially sensitive information (such as logged-in state).

This proposal solves that by requiring the inner page to opt in, by using the `requestedwidth=""` and `requestedheight=""` attributes on its document element. This serves as an explicit sign-off that it wants to communicate the given intrinsic size request to the outer page.

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

* Coordination: providing a standard mechanism for cooperatively-sizing iframes would avoid both sides needing to agree on the message format or library to use. Instead, everyone could use the standard pattern.

* Performance: especially given cross-origin iframes, passing messages across the frame boundary via JavaScript, and then performing a resize of the containing frame, can be expensive. By providing a direct declaration of intent to the layout engine, browsers can integrate the resizing process into their usual layout passes.

### Using a HTTP header

In [whatwg/html#555](https://github.com/whatwg/html/issues/555) @annevk proposes that the embeddee opt-in should be done using a HTTP header, e.g. `Expose-Height-Cross-Origin: 1`, instead of the CSS `requestedwidth=""` and `requestedheight=""` properties we propose here.

This is significantly more difficult for developers to deploy than HTML attributes. Additionally, since that comment was made, we have realized that [auto-resizing is hard](#but-what-about-auto-resizing). So we need a way of updating the requested width/height at runtime, to allow manual resizing over time. So any header would have to be in addition to some other mechanism.

### Using a JavaScript API

In [w3c/csswg-drafts#1771 (comment)](https://github.com/w3c/csswg-drafts/issues/1771#issuecomment-805117925), @tabatkins proposes that the embeddee opt-in should be using `window.resizeTo()` inside the iframe, instead of the `requestedwidth=""` and `requestedheight=""` attributes we propose here.

This is definitely a viable alternative, and we welcome feedback if people prefer it. We think the declarative version has some slight advantages, in that it can be parsed earlier in the page lifecycle (so as to avoid a flash of unresized content), and it provides an easy mechanism for expanding in the future to [auto-resizing](#but-what-about-auto-resizing).

### Using a CSS API

In [w3c/csswg-drafts#1771 (comment)](https://github.com/w3c/csswg-drafts/issues/1771#issuecomment-805117925), @tabatkins proposes a new `intrinsic-size` property (including `intrinsic-width`/`intrinsic-height`) that would be applied to the document element. This could be used instead of the `requestedwidth=""` and `requestedheight=""` attributes we propose here.

This is also a viable alternative. We heard some feedback that it's weird to have CSS properties that only apply to a single element, however, so for now we lean toward the attributes.

### Using existing CSS properties

In [w3c/csswg-drafts#1771 (comment)](https://github.com/w3c/csswg-drafts/issues/1771#issuecomment-810633323), @fantasai proposes using the existing `width` and `height` CSS properties, instead of the `requestedwidth=""` and `requestedheight=""` attributes we propose here.

Doing so removes the opt-in necessary for [security and privacy](#security-and-privacy-considerations), so it doesn't seem workable. We need a new API, which no existing content is using. Indeed, see [below](#naming-choices) for ideas on making the opt-in names even more explicit, to emphasize their security and privacy impact.

### Automatic opt-in for same-origin iframes

In the case where the iframe is same-origin with its embedder, there is no security or privacy concern that requires the bidirectional opt-in. However, there is still a compatibility concern: we cannot suddenly make all same-origin iframes automatically size to their content. So we would need at least one of the two opt-ins.

But changing the model in this way, only for same-origin iframes, seems more likely to lead to developer confusion. Which opt-in becomes optional? And how does that impact our [explanation of how the cooperative sizing works](#the-proposal), via the from-element intrinsic width/height? It seems best to keep the same model everywhere, wherein the embedded content signals its desired intrinsic size, and the embedder decides whether to respect that request or not.

### Automatic opt-in for `srcdoc=""` iframes

In [w3c/csswg-drafts#1771 (comment)](https://github.com/w3c/csswg-drafts/issues/1771#issuecomment-806208058), @prlbr proposes that `srcdoc=""` iframes should not require one of the two opt-ins. As with same-origin iframes, this is sound from a security perspective. However, similar to that case, we believe adding such exceptions is likely to create confusing developer experiences, and so it's not worth it just to save some typing.

### Naming choices

We are not firmly set on the naming choices in [the proposal](#the-proposal). Other ideas could be:

* A more explicit name for `requestedwidth=""`/`requestedheight=""`, e.g. `requestedintrinsicwidth=""`, or `containerintrinsicwidth=""`, or `intrinsicsizerequestexposedtoembedderwhichtheymayormaynotrespect=""`.

* A different value than `from-element` to communicate that we're using the requested intrinsic size, e.g. `contain-intrinsic-size: requested`, or `contain-intrinsic-size: from-embeddee`.

We welcome feedback on the naming.

## Acknowledgments

This proposal is the latest in a long lineage of proposals. Most recent work in this area was done by @tabatkins, in [w3c/csswg-drafts#1771](https://github.com/w3c/csswg-drafts/issues/1771); this proposal is largely created by fleshing out and trimming down his original [comment](https://github.com/w3c/csswg-drafts/issues/1771#issuecomment-805117925) there. Thanks also to the other participants in that discussion.

Other historical requests for this functionality in the web standards world include:

* [Marc Attinasi on Mozilla Bugzilla](https://bugzilla.mozilla.org/show_bug.cgi?id=80713) in 2001;
* Hixie's work on [seamless iframes](https://html.spec.whatwg.org/commit-snapshots/784c7c3f0e63d5809742f4c32696406186f364a2/#attr-iframe-seamless), and Ben Vinegar's [feedback](https://lists.w3.org/Archives/Public/public-whatwg-archive/2014Feb/0015.html) on it ~2014;
* Craig Francis [on whatwg/html](https://github.com/whatwg/html/issues/555) in 2016 and [on www-style](https://lists.w3.org/Archives/Public/www-style/2017Aug/0045.html) in 2017;
* and likely many others which I have not yet found.

---
up: null
collection:
  - technology
slug: whats-next-after-atomic-utility-css
---
# What’s Next After Atomic-Utility CSS

## My Atomic-Utility Experiences & Problems

I recently recreated my personal website as a version 2. I had learnt a lot more since version 1, and one major area that I wanted to step up in was in **CSS Methodology**.

My first-ever Jekyll website used Bootstrap. Yucky bloated component classnames that I had to overwrite every time. V1 was gladly immersed in Tailwind. But I soon realised I didn't like the DX of working with atomic styles in `class`, either.

I reached for [Inline fold - VS Code Extension](https://marketplace.visualstudio.com/items?itemName=moalamri.inline-fold) to collapse away the mess. But for very long `class`es with many declarations, it doesn't/can't/shouldn't collapse the number of lines, so we're left with an XML tag with multiple empty lines in the editor.

CSS selectors are awkward with Tailwind. I started using more `[&>foo]:bar` and psuedo-class/element selectors — *inheritance* and *context* are actually really really useful selectors. But Tailwind gets lengthy when you want to declare multiple styles for one selector (e.g. on hover, multiple styles should be added). [UnoCSS](https://unocss.dev/) and [Master CSS](https://css.master.co/) were interesting to consider.

Then it's also hard to scan for a specific style declaration in such a long one-line string. I tried [prettier-plugin-tailwindcss](https://www.npmjs.com/package/prettier-plugin-tailwindcss), which helped to an extent. But with [260-character-long strings](https://github.com/chuangcaleb/v1.chuangcaleb.com/blob/master/src/components/buttonVariants.ts#L4), it still gets very lost.

These declarations tend to be very repetitive too. Imagine a list of buttons, each having a 260+ characters-long `class` string. Strings that may 90-100% be the same. Doesn't feel very DRY.

Then there's the diff issue: since every atomic style is in one line of string, so it's really hard to see git diffs. Contrast traditional CSS, where each declaration is its own line and gets its own diff.

Tailwind also populates the stylesheet with a bunch of boilerplate CSS variable declarations. I'm probably smarter today to find out how to fix that, but the `@tailwind` injection is pre-configured and black-boxed (which is appropriate and desirable for quick one-off scaffolding!). But I desired having more control to exactly define what gets into the stylesheet.

Component variants was the biggest hurdle. Make a Tailwind-styled button with a matrix of style vs. size vs. color-scheme vs. font-styles, etc. There's a lot of variants with overlapping style conflicts (e.g. a solid button has the accent be the background color, but transparent buttons has no background color but accented text) then you may end up with undesirable declaration conflict resolutions in the same `class` string (e.g. `"bg-accent ... bg-none"`). I tried solutions like [clsx](https://github.com/lukeed/clsx) and [cva](https://cva.style/docs) to pre-process the issues. (I still like the idea of type-safe/hinted variants — may revisit this!)

Even with abstraction, compiled HTML output is cluttered. CSS clutter is not too bad, but when debugging the HTML, it's unruly to sift through the lines-spanning XML tags. Doesn't it also increase the initial HTML file size by a lot? (EDIT: with minification and style sorting, HTML size isn't as affected.)

## Semantic CSS

I realized I needed something like [daisyUI](https://daisyui.com/) — which [I used](https://github.com/chuangcaleb/v1.chuangcaleb.com/commit/46b3276792ea40a8159f6a549c9c6255427681e9)! I like the how `.btn` and `.card` group a bunch of declarations. "Utility-only was slow and bloated"[^1] — Semantic CSS classnames gave more succinctness over Utility-Only.

[^1]: [My Journey to build daisyUI: Why Tailwind CSS was not enough? — DaisyUI Blog](https://daisyui.com/blog/my-journey-to-build-daisyui/)

I did also like how it hooks into Tailwind CSS’ plugin API to generate tokens. But DaisyUI had an opinionated design system and base styles, I wanted more fine control.

And in the end, DaisyUI is still a utility-first methodology where you construct styles and variants in `class` strings, while manually handling CSS (with @apply directives) was to be the exception to the rule. I still struggled to concisely express CSS selectors and variants. Since every style rule was still on the same one atomic "layer", it wouldn't be clear when two classnames may conflict (e.g. `btn` and `p-4`) and how to resolve them.

I would still reach for DaisyUI if I had to work in Tailwind.

But in the end, I realize that **all this atomic-utility abstraction is actually adding so much unnecessary complexity**. Reject unnecessary abstraction. Return to CSS.

## CUBE CSS

After much searching, I landed on [CUBE CSS](https://cube.fyi/). I won't waste breath re-explaining it — the [blog post](https://piccalil.li/blog/cube-css) will do a better job than I!

CUBE is a *simplistic* **methodology** (not a toolchain) that **embraces the CSS Cascade** by preferring general contextual hinting over micro-management. Reduced abstraction via high-level style rules and predictable flat/inclusive selectors. Tool-agnostic, just work with the cascade.

Some instant benefits:

1. Authoring directly in CSS/SSCS allows for grouping declarations and fancier contextual selectors.
2. Let the Cascade (precedence and specificity) handle conflicts. Instead of atomic classnames, now every style rule distinctly lives in a particular "layer".
3. Exploit strength of atomic utilities: one job done well. Rather than `btn` and `btn-primary`, etc. that apply 10+ (possibly conflicting) declarations
4. Easier to understand the specific *layers* of styling that compose the final style stack.

Speaking CSS as a first-language has really made it so fluid and effective to work with (especially on a small-medium project like this). I use very simple `scss` for generating custom tokens and using mixins (allows modularizing rules into different files, while keeping it all under the same `:root` declaration lol)

And compare the bundle sizes sent over the wire, between v1 and v2. As of writing, it's improved initial HTML (25.9KB to 12.1KB) and CSS (6.1KB to 9.0KB) — the CSS should’ve been way more because I also added polyfills and more declarations, turning it from from single-themed barebones to dynamic-themed and heavily-styled masterpiece. See [chuangcaleb.com | CSS Stats](https://cssstats.com/stats/?url=chuangcaleb.com) for a breakdown.

And the DX? It’s so much easier to develop when there’s defined layers of responsibility. General layout rules and specific component rules.

## Modern Syntax + CSS Variables

Finally, CSS Variables (aka CSS Properties) (and some modern CSS tricks!) ties it all together.

Here are some shoutouts:

- [argyleink/open-props: CSS custom properties to help accelerate adaptive and consistent design](https://github.com/argyleink/open-props) provides some zero-specificity standardised style tokens in CSS Variable form.
- [Axiomatic CSS and Lobotomized Owls – A List Apart](https://alistapart.com/article/axiomatic-css-and-lobotomized-owls/) was REVOLUTIONARY for me for controlling (rather, letting go of control of!) flowing prose layout
- [CSS Grid full-bleed layout tutorial](https://www.joshwcomeau.com/css/full-bleed/) — I struggled with full-bleed alternate-background-color sections
- [Layout Breakouts with CSS Grid](https://ryanmulligan.dev/blog/layout-breakouts/) takes the above concept further with *named grid lines* for content layouts that breakout of the line-width.
- [utopia-core-scss](https://github.com/trys/utopia-core-scss) by [Utopia](https://utopia.fyi/) generates fluid-responsive CSS Variable tokens.

I still would like to make refinements for a one-size-fits-all CSS workflow. But for now, on simple websites like this one you’re seeing, I’m convinced that CUBE CSS is the most optimised methodology that you should reach for.

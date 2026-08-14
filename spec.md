# Achieve — Landing Page

Handoff notes. This documents the things you **cannot** work out by reading the
code: why decisions were made, how the numbers were derived, and which traps have
already been hit and fixed. For structure, read the components — they are
commented where the reasoning is non-obvious.

---

## What this is

A single-page marketing site for **Achieve**, a personal-finance app. It is a
redesign built to match a supplied mockup (the "Necta" template) section by
section, then rebranded to Achieve and re-themed.

Status: **complete** — 10 content sections plus navbar and footer, every one
matched to a supplied mockup. Remaining work is copy, real assets, and polish —
see [Outstanding](#outstanding).

## Stack

| | |
|---|---|
| Framework | Astro 7.2.2 — static, no framework components, no islands |
| CSS | Tailwind 4.3.3 via `@tailwindcss/vite` (no config file; theme lives in `src/styles/global.css`) |
| Font | Satoshi 400/500/700/900, self-hosted from `public/fonts/` |
| Animation | CSS + IntersectionObserver, plus GSAP 3.15 / ScrollTrigger for scrubbed effects |

There are **no React/Vue/Svelte components** and no Astro integrations. Keep it
that way unless there's a strong reason — see
[Why not Framer Motion](#why-not-framer-motion).

## Running it

```
npx astro dev --background     # manage with: astro dev stop | status | logs
npx astro build
```

Dev server runs on `http://localhost:4321`.

## Page structure

`src/pages/index.astro` composes, in order:

```
Navbar → Hero → LogoMarquee → Features → Capabilities → Control
       → MoreFeatures → Connect → Testimonials → Faq → DownloadCta → Footer
       → ScrollFx
```

`ScrollFx` renders no markup — it is the GSAP module, mounted last.

---

## The measurement method — read this before changing any size

**Nothing in this project was eyeballed.** Every font size, card dimension and
gap was derived by measuring the mockup screenshots and solving against the real
font metrics. If you change sizes by guessing, you will break a system that is
currently self-consistent.

The method, repeated for each section:

1. Measure ink widths of text runs in the screenshot, in displayed pixels.
2. Establish a scale factor `k` by anchoring on something of known size — a
   container edge, a card boundary, or an element whose CSS size is already
   fixed elsewhere on the page.
3. Convert measured widths to target CSS widths (`measured × k`).
4. Binary-search the font size whose rendered ink width hits that target, using
   the actual Satoshi TTF via `opentype.js`.

Two properties make this trustworthy:

- **Cross-validation.** Independent measurements of the same nominal size should
  agree. They did — e.g. in Capabilities, six separate card titles all solved to
  19.0–19.1px; in MoreFeatures, three titles landed at 20.09/20.14/20.14px.
  When two measurements *disagreed*, that was signal, not noise (see the
  App Store badge trap below).
- **Geometry checks.** In MoreFeatures, three 362px cards on an 18px gutter
  compute to 1121px against a 1120px container — a 1px error across the whole
  composition, which confirms both numbers.

The tooling is not committed. To redo it: `npm i --no-save opentype.js`, load
the TTFs (re-download from Fontshare — only `.woff2` is committed), and lay
glyphs out manually with `charToGlyph`/`getKerningValue` rather than
`getPath(text)`, because some fonts hit unsupported GSUB lookups in opentype.js.

### Where the derived numbers live

Type is fluid, tied to the container so the mockup's proportions hold at every
width. The pattern is `clamp(min, calc(<a>vw - <b>px), max)` where `max` is the
size at the 1120px container and the slope reproduces it as the container
shrinks.

| Section | Heading clamp | Body / cards |
|---|---|---|
| Hero | `2rem → 2.8rem` (44.8px) | body `0.9375 → 1.05rem` |
| Features | `1.625 → 1.9375rem` (31px) | titles 20px, body 16px |
| Capabilities | `1.625 → 1.875rem` (30px) | titles 17px, body 15px |
| Control | section `1.5 → 1.6875rem` (27px); rows `1.5 → 1.875rem` (30px) | body 16px, button 15px |
| MoreFeatures | `1.625 → 1.875rem` | titles 19px, body 16px |
| Connect | `1.375 → 1.875rem` | body `0.875 → 1rem` |
| Testimonials | `1.625 → 1.875rem` | quotes 17px, names 15px |
| FAQ | `1.625 → 1.875rem` | questions 15px, footer 16px |
| DownloadCta | `1.625 → 1.875rem` | body 16px |

Every section eyebrow is **16px** and uses the shared `.text-gradient-brand`
utility.

**Known inconsistency:** Features uses slope `2.77vw` / max `1.9375rem` (31px)
while every other section settled on `2.679vw` / `1.875rem` (30px). Leftover
from before the scale was normalised. Harmless but worth aligning.

### Line breaks are explicit

Several places use `<br class="hidden lg:block" />` rather than relying on
`max-width`. This is deliberate: the natural wrap point often did **not** match
the mockup. In the Features cards, for example, "Set savings goals and let the
system" measures 250px against a 280px column — it fits, so the text would not
have broken where the mockup breaks it. Mobile still wraps naturally.

---

## Design tokens

All in the `@theme` block of `src/styles/global.css`. The site is **dark-themed**.

```
--color-surface    #050b09   page
--color-surface-2  #0b1512   cards, inputs
--color-surface-3  #12201b   hover / raised
--color-line       #1b2a24   borders, dividers
--color-ink        #e8f2ee   primary text      (light — the name predates the dark theme)
--color-ink-soft   #c0d4cc   secondary text
--color-muted      #8ba59c   body copy
--color-accent     #14cca9   buttons
--color-accent-ink #04120e   button labels
--color-star       #ffa51f   rating stars
--color-brand{,-cyan,-green}  #14cca9 / #14bcc4 / #14d49d
```

The brand colours were **sampled from `logo.png`**, not chosen — the logo runs
cyan → teal → emerald, and the palette follows it.

Surfaces are near-black with a deliberate **green cast** rather than neutral
grey, so the page reads as part of the brand rather than merely unlit.

Contrast was verified against WCAG, not assumed. All pairings pass AA: body copy
7.5:1, primary text 17.4:1, accent 9.7:1, button label on accent 9.3:1.

---

## Animation architecture

Two layers, deliberately separated.

### Layer 1 — CSS + IntersectionObserver (~900 bytes)

Handles section reveals (46 targets, 7 staggered groups), the hero entrance,
ambient drift on the gradient washes, the FAQ accordion, and hover lifts.

The observer is an `is:inline` script in `Layout.astro`'s `<head>`. It must stay
inline and in the head: it adds `reveal-ready` to `<html>` **before first paint**,
which is what prevents a flash of shown-then-hidden content.

**The failure mode is the design constraint.** Elements are visible by default in
CSS; the script *opts in*. With JS disabled, blocked, or reduced-motion
requested, the page is fully readable. Reveals must always resolve to
`opacity: 1` — never to hidden.

Chosen over `animation-timeline: view()` because scroll-driven CSS *scrubs* to
scroll position, so reveals play backwards when scrolling up. IntersectionObserver
gives play-once and unobserves after firing.

### Layer 2 — GSAP + ScrollTrigger (44 KB gzipped)

`src/components/ScrollFx.astro`. Scoped narrowly to scrubbed, scroll-linked
motion that CSS cannot express:

- **Device parallax** — 8 targets via `data-parallax="<px>"`. Travel is in pixels,
  not percentages, so it is predictable regardless of element height.
- **Connect orbit** — the six app icons travel 52° around the ring on scroll. The
  rotation is on a wrapper (`[data-orbit]`) and each icon counter-rotates
  (`[data-orbit-item]`) to stay upright.

Everything runs inside `gsap.matchMedia('(prefers-reduced-motion: no-preference)')`,
which both skips setup and reverts inline styles on cleanup.

**Rules if you extend this:**

- GSAP must never touch an element that carries a CSS reveal or CSS animation.
  A CSS animation beats GSAP's inline transform and the two will fight silently.
  This is why the drifting washes and revealing cards are off-limits.
- Do not parallax the hero device. It is tightly clipped and it is the LCP
  element — drift risks exposing gaps for almost no payoff, since little
  scrolling happens before it leaves.
- Animate transform/opacity only. Never height/top/width (CLS), and never
  `filter: blur()` — the washes are blurred at 26–70px and animating that is
  brutally expensive.

**Cost check:** GSAP is the only runtime dependency and accounts for the entire
JS bundle. The parallax alone could be hand-rolled with a scroll listener and
rAF in ~30 lines. If the orbit gets cut, drop GSAP with it.

### Why not Framer Motion

It is React-only. This site has zero framework components, so adopting it means
`@astrojs/react` + `react` + `react-dom` — roughly 45 KB of runtime before a
single animation. Note the library was renamed to **Motion** (motion.dev) and now
ships a ~5 KB vanilla API; *that* is viable if GSAP ever needs replacing.

---

## Assets

`public/images/`:

| File | Status |
|---|---|
| `logo.png` | **Real.** Full lockup (wordmark + checkmark). Used in Navbar (h-9) and Footer (h-8). |
| `mid-phone.webp` | **Real.** 3D device render, real transparency. 154 KB, derived from `mid_phone.png`. |
| `hero-hand.webp` | **Real,** but keyed by me — see trap below. 204 KB. |
| `hero_image.avif` | **Real** app screenshot (Necta-era UI). Used in a Features card. |
| `hero_withhand.png` | Source of `hero-hand.webp`. **1.7 MB, unreferenced.** |
| `mid_phone.png` | Source of `mid-phone.webp`. **1.5 MB, unreferenced.** |
| `hand.webp` | Superseded. **Unreferenced.** |

Everything in `public/` ships whether referenced or not — those three unused
files are ~3.2 MB of dead weight in the deploy.

### Placeholder artwork (built in markup, not images)

Eight components draw app screens as HTML/SVG because no export existed:
`SavingsPhone`, `SendMoneyPhone`, `AnalyticsPhone`, `SpendingPhone`,
`SubscriptionsPhone`, `MoneySavedPhone`, `ReadyCashPhone`, `TransactionsPhone`.

They are crisp at any size and restyle with a class change, but they are
**approximations drawn by eye** — the Analytics chart geometry especially. Each is
a one-component swap if real exports arrive.

Also placeholder: the six marquee brand marks (`LogoMarquee`), all avatar
gradients (hero social proof, testimonials, contact carousel, transaction list),
and the Connect section's app icons — those approximate real trademarks
(PayPal, Alipay, Bitcoin, Cash App).

---

## Traps already hit — do not regress these

**Carousel loop threshold.** `Testimonials` rewinds at `setWidth`, not
`2 × setWidth`. The latter is unreachable: max `scrollLeft` is
`3w − clientWidth`, which on a 1920px display is 3570 against a 3660 threshold —
the track parks at the end and both autoplay and the next button jam. This bug
is **width-dependent**: fine below ~1830px, broken above. Verified by simulating
200 steps at four viewport widths.

**Never put `scroll-smooth` on the carousel track.** It would animate the rewind,
making the seam visible. `scrollBy()` opts into smooth per call instead.

**The supplied hero PNG had no alpha.** `hero_withhand.png` shipped with the
transparency checkerboard flattened into the pixels (two greys, 32px squares).
Dropped in as-is it renders a grey checkered rectangle. `hero-hand.webp` is the
keyed version — flood-filled inward from the borders so light UI *inside* the
phone survives. If that source is ever replaced, check alpha first.

**`items-center` on a grid collapses an absolute-only column to zero height,** so
`top` then measures from the row's midpoint instead of the section top. Bit me in
`DownloadCta` and `Capabilities`; both use `items-stretch`.

**`h-full` plus a width cap on an `<img>` stretches it** rather than preserving
aspect ratio. Use `object-contain` (see `PhoneMockup`).

**Device screens are near-invisible on the dark theme** — they are `#050506`–
`#08090A` against a `#050b09` page. All nine carry a `ring-1 ring-white/10`
hairline so they read as devices. Keep it.

**`group-hover/card:` needs a `group/card` ancestor.** Without one it silently
never fires. Caught in MoreFeatures.

**Reveal groups need `data-reveal` on the children,** not just
`data-reveal-group` on the container — otherwise the group has nothing to
stagger and nothing animates.

**Tailwind `class:list` appends,** so classes do not appear in source order in
the output. Greps that assume ordering will produce false negatives.

**When two measurements of the same thing disagree, dig.** "App Store" solved to
23.8px and "Google Play" to 20.7px — impossible for one design. Checking against
the badges' *overall* widths settled it at 20px for both; the text-edge reading
was the bad measurement.

---

## Outstanding

**Copy.** All body copy is the template's, lightly rebranded. FAQ answers,
testimonial quotes, and the two cut-off testimonials ("Automatic sync…" /
"Knowing my credit score…") were written by me. The footer still credits
"Template by Zain Malik" — decide whether that stays.

**Favicon** is still Astro's default. `logo.png` is a wide lockup and will not
crop to a square tab icon; extract the checkmark or supply a square mark.

**Newsletter form** (`Footer`) has no action — needs an endpoint or a service.

**Nav links point at `#features`, `#control` etc.** Only `#features` and
`#control` exist as ids; the rest are dead anchors.

**Carousel speed.** Autoplay is 1000ms per card (`AUTOPLAY_MS` in
`Testimonials`), set on explicit request. It is fast for 2–3 line quotes and now
competes with the scroll reveals — 3000–4000ms is the usual range.

**Unused images** — see the asset table.

**Features heading clamp** is 1px out of step with every other section.

**Alipay's 支 needs a CJK font.** Satoshi has no such glyph, so it falls back
through PingFang / Microsoft YaHei / Noto Sans SC. A machine with no CJK font
installed shows a placeholder box.

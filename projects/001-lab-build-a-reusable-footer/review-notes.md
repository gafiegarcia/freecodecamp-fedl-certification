# Review: Build a Reusable Footer

**Overall:** This is a solid first React lab. It meets every user story. The JSX is clean, the component boundary makes sense, and the CSS is more modern than the lab needs. There is one real JavaScript mix-up to fix, and a few accessibility and semantics points worth learning now.

---

## 1. Things to fix

### `Logo(...classes)` doesn't do what it looks like it does (`index.jsx:51`)

React calls every component with **one argument**, an object of the attributes you passed (called "props"). That means `...classes` (rest parameters, from the JS cert) collects that single object into an array:

```jsx
<Logo />                    // React calls Logo({})
function Logo(...classes)   // classes === [{}]
```

It's never used, so nothing breaks. But it suggests a picture of components that will trip you up in the next theory block, "Working with Data in React". For now, just remove it:

```jsx
function Logo() {
```

If you meant to let the logo take a custom class, keep that idea for the next lesson. It has a proper solution there.

### `aria-hidden="true"` hides your brand name (`index.jsx:4`)

`aria-hidden` removes an element _and everything inside it_ from screen readers. So "Headsmooth" disappears, not just the decorative icon. Move it onto the SVG only:

```jsx
<section className="footer-hero">
    <Logo /> {/* with aria-hidden="true" on the <svg> itself */}
    <span className="hero-text">Headsmooth</span>
</section>
```

Rule of thumb: hide **decoration**, never **content**.

### Links have a hover style but no keyboard focus style (`styles.css:44`)

Anyone using Tab never gets the accent color. Your `border-radius` on `a` (line 41) suggests you were already halfway to a nice focus ring:

```css
&:hover,
&:focus-visible {
    color: var(--accent-color);
}
```

---

## 2. Worth improving (semantics and polish, not bugs)

- **`<section>` without headings** (`index.jsx:4, 8`): a `<section>` is meant to be a titled chunk of content. Neither of yours has a heading, so `<div>` is more honest for the logo block. For the link block, `<nav aria-label="Footer">` describes what it actually is. You covered both elements in RWD.
- **Unlabeled link columns:** your three lists clearly have themes (learning, community, legal), but nothing says so. A small heading above each list helps both sighted users and screen reader users:
    ```jsx
    <div>
        <h2 className="footer-heading">Community</h2>
        <ul className="footer-links">…</ul>
    </div>
    ```
- **SVG cleanup** (`index.jsx:55–60`): `fill="#000000"` hardcodes black. `fill="currentColor"` makes the logo follow the CSS `color`, so a theme change carries through to the icon. You also don't need `version="1.1"`, `xmlns` or the two empty nested `<g>` tags in inline SVG; they're export leftovers.
- **"All Rights Reversed"** (`index.jsx:46`): if that's a pun (it's an old copyleft joke), it's great, and it fits the bald-head "Headsmooth" theme. If it's a typo, it should be _Reserved_.

**Just style, not a problem:** you repeat `@media (width < 900px)` three times. Keeping each query next to the rule it changes is a valid choice, and I'd keep it.

---

## 3. What you did well, so keep doing it

- **Extracting `<Logo />`**: you pulled the noisy part out so `Footer` reads like an outline. That's exactly what components are for.
- **Your README reasoning**: you chose _not_ to generate the links from data because there was no real data yet. Thinking "is this abstraction worth it here?" is better judgment than abstracting by reflex.
- **Clean JSX**: you used `className` correctly, there's a single `<footer>` root as the tests require, and it's a named export.
- **CSS that holds up**: custom properties for your design tokens, logical properties (`padding-block-start`), native nesting, relative color syntax, and generous link padding (`0.75rem 1rem`) that gives good tap targets.

---

## Look ahead

In the next theory block, "Working with Data in React", you'll learn to describe the columns as an array and render them with `.map()` (which you know from the JS cert) plus a `key`. Your Logo class idea also gets its real form there, as a prop:

```jsx
function Logo({ className }) {
  return <svg className={className} …>
```

Don't refactor yet. Come back to this footer after that lesson; it makes a good before-and-after.

---

# me

1. so I _will_ learn something later that allow me to pass arguments to components I want to render with `<Component />`?
2. regarding the aria-hidden, that was because I changed my mind... should've moved the aria-hidden back to the svg again. I think Iwas thinking about putting h2/h3 on the hero text, plus h3/h4 for the labels of each `.footer-links` (which I didn't do, because not required). I was thinking "if the goal is to make reusable component, putting an exact heading number might not be the best idea, since every page with the same footer may have different document hierarchy where 'h3' could mean very different". then after all that overthinking, I forgot to clean up xD
3. hover: actually, outline on keyboard focus style works ootb (blue outline), and border radius kinda plays a role in it. at least it does on my mac... idk, is this macOS default behavior or user-agent default? if user-agent default, I might just let it
4. <nav>: why `aria-label"Footer"`? wouldn't that be ambiguous with the parent `<footer>` element, which covers not just the links but also the logo and the copyright line?
5. Unlabeled link columns: my previous question: how do I decide the heading number if the document shaps is currently unknown... how should I deal with that... I took a peak at some websites (nestjs, cloudflare, tailscale) and found that some of them just fallback to <span> for almost all text, and div for the stylings and containers and all. one of them actually uses h3, but yeah... back to my previous question
6. can't wait for the lookahead.. I realy need to pass args to the component function (wait, what do I call them... just "Components"? what about the `<Component />`... a "render"/"component function call"/something else?)

# Claude

Honest note first: your questions caught me overstating two things (#3 and #4).

### 1. Passing arguments to `<Component />`

Yes. They're called **props**, and you'll learn them in the very next theory block, "Working with Data in React". They're written like HTML attributes:

```jsx
<Logo className="logo--large" />
```

React then gathers them into **one object** and hands it to your function:

```jsx
function Logo(props) {
    // props = { className: "logo--large" }
    return <svg className={props.className}>…</svg>;
}
```

That's why `...classes` gave you `[{}]`: there's only ever one argument.

### 2. aria-hidden

That makes sense. The leftover was a cleanup slip, not a misunderstanding, and your reasoning was sharp. See #5.

### 3. Focus outline

You're right, and my review was wrong to say keyboard users get nothing. That blue ring comes from the **browser's own default stylesheet** (the "user-agent stylesheet"), not from macOS. Chrome, Safari and Firefox all draw one on `:focus-visible`. Modern browsers also make the outline follow `border-radius`, which is what you're seeing.

Leaving it is a legitimate choice. The only thing missing is the accent color you show on hover, which is consistency polish, not an accessibility bug.

A Mac-specific gotcha: **Safari** doesn't Tab to links by default. You either press Option+Tab, or turn on Safari → Settings → Advanced → "Press Tab to highlight each item on a webpage". If Tab seems broken in Safari, that's why.

### 4. `<nav aria-label="Footer">`

The label wouldn't clash with `<footer>`, because they're different landmark types. Screen readers announce `<footer>` as "content info" and the nav as "Footer navigation". The label only answers "which navigation is this?", to tell it apart from the main nav in the header.

But you've caught a bigger point: the HTML spec itself says a short list of footer links doesn't need `<nav>`, because `<footer>` alone is enough. So `<nav>` was optional, not a fix. Your `<section>` → `<div>` change is the part that matters.

### 5. Heading levels when the page structure is unknown

Your worry is valid for components that can be **nested at different depths**, like a card inside an article inside a section. The spec once tried to auto-number headings that way (the "document outline algorithm"), but browsers never implemented it, and it was removed from the spec in 2022. So you have three options:

1. **Use h2 for the footer.** A site footer always sits at the top level of the page, next to `<main>`, never inside it. So its headings are top-level sections → h2, on every page. The level only becomes unpredictable for components that move around. The UK government's design system (GOV.UK) does this in its footer.
2. **Make the level a prop**, e.g. `<Footer headingLevel={3} />`. This is the general solution for components that move around. You'll be able to build it soon (see #1).
3. **No headings, just `<span>`s**, like the sites you checked. It isn't wrong, since link lists are still understandable. But screen reader users lose the ability to jump between columns by heading. It's a trade-off, not a bug.

For this footer, I'd pick option 1.

### 6. Names for things

| Thing                 | Name                                                                         |
| --------------------- | ---------------------------------------------------------------------------- |
| `function Logo() {…}` | a **component** (specifically a "function component")                        |
| `<Logo />`            | a **JSX element**; you're "**rendering** Logo" or "using the Logo component" |
| `className="…"` on it | a **prop**                                                                   |

It's not a "call", and the difference matters. `<Logo />` doesn't run your function. It creates a small description ("a Logo goes here, with these props"), and **React** calls `Logo` when it decides to render. That's also why you should never write `Logo()` yourself.

# me

wait... tab to focus on links work on Safari — nvm, I got the setting toggled on. don't even remember if I toggled that on myself (or if the default changes to it being on)

and on the heading level: yeah h2 for the footer heading itself makes sense! (perhaps I can replace the hero text with h2). but what about the links section header? do you think I should use h3 for them, or is <span> (like the examples I saw in the wild) enough to indicate "this group is 'legal'/'contact us'" ?

# Claude

Glad Tab works in Safari now. As far as I know, that setting is off by default on a new Mac, so you probably turned it on at some point. Either way, it's worth remembering when you test keyboard navigation on someone else's machine.

### Should the hero text be an h2?

I'd **not** make "Headsmooth" the h2, and I'd use **h2 for the column labels** instead.

A heading names the content that comes after it, like a chapter title. If "Headsmooth" is the h2, then Learn, Community and Legal become h3 sub-chapters _of Headsmooth_. That doesn't really describe the footer. "Headsmooth" is a brand mark, more like a logo than a title. Leave it as a `<span>`, or make it a link to the homepage, which is a common footer pattern.

This gives you:

```
h1  Page title            (from the page)
…   main content          (h2s, h3s…)
h2  Learn                 ← footer columns, siblings of main's sections
h2  Community
h2  Legal
```

That's also the GOV.UK footer pattern I mentioned: column headings at h2, with no footer-wide heading above them.

### Heading vs `<span>` for the columns

A `<span>` does work **visually**. A screen reader also reads "Legal" just before the list, so someone reading in order still gets the context. What a span loses is the **shortcuts**: screen reader users skim pages by jumping from heading to heading (in VoiceOver, the Rotor; ⌃⌥U), much as sighted users skim bold labels. With spans, "jump to Legal" doesn't exist; they have to arrow through every link to get there.

So for you:

- **`<h2>` per column (recommended):** you get the jump-to shortcuts, and it's one tag change. You can style it to look exactly like your spans would.
- **`<span>`:** acceptable, as the sites you checked show, but it gives up that navigation for no real gain.

The cost is identical either way, since you're writing a tag and styling it regardless, so pick the one that carries meaning.

# me

then if using h2 per column, should the column be <div> or <section> or what?

# Claude

Either works. The h2 is what does the job here; the wrapper is mostly up to you.

- **`<div>`**: just a styling box. The h2 already marks where each group starts, so nothing is lost.
- **`<section>`**: also correct now. My earlier "use a div" point was about sections _without_ headings, and these columns would have one. Screen readers don't treat a plain `<section>` as anything special, so it behaves like a `<div>` for them. The benefit is that your code reads more clearly.

**Avoid** linking each column's heading to its section with `aria-labelledby`. That would turn each column into a landmark (a region screen readers list in their quick-jump menu), and three tiny landmarks for footer columns just clutter that menu. The headings already give users the jump points.

I'd use `<div>` for simplicity, but `<section>` is fine if you like what it says in the code:

```jsx
<div className="footer-column">
    <h2 className="footer-heading">Legal</h2>
    <ul className="footer-links">…</ul>
</div>
```

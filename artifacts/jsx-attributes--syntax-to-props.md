# JSX Attributes: from syntax to props

> "Why do we pass `{...developerObj}` in `<Component {...developerObj} />` instead of just `developerObj` directly?" (the answer is, the JSX compiler expects only two shapes when parsing the attributes input)

can we get a bit nerdy and tell me how the jsx compiler does the conversion from jsx to the final "object"? oh I also forgot what this "object" is, you know, the one that React finally "renders".

I want to see step by step how the compiler "reads and cooks" that JSX syntax for both name="Alice" format and the {...developerObj} format. not that seeing what the compiler does is helping me write better JSX, but I have a feeling seeing how the data "transforms" step by step will help me understand the compiler better that I may never forget why things are written and shaped as such in JSX

---

---

Sure. One thing to fix first, since it's the source of a lot of confusion: **the compiler never makes the object.** The compiler turns text into other text, at build time. The object only exists later, in the browser, when that new code runs.

## The pipeline

```
 BUILD TIME (Vite/Babel, on your Mac)          RUN TIME (the browser)
 ─────────────────────────────────────         ───────────────────────────────
 ① JSX source text                             ④ jsx() runs → React element object
 ② parse → syntax tree (AST)                   ⑤ React renders: calls DeveloperCard(props)
 ③ transform tree → generate plain JS text     ⑥ … eventually real DOM nodes
```

We'll follow this line through every step:

```jsx
<DeveloperCard name="Alice" age={30} {...developerObj} />
```

---

## ① Source: just characters

To the compiler this is a string. Nothing has any meaning yet.

```
"<DeveloperCard name=\"Alice\" age={30} {...developerObj} />"
```

## ② Parse: reading characters into a tree

The parser walks through the characters one at a time and keeps track of which **mode** it's in.

- It sees `<` where an expression could go, so it switches to **JSX mode**.
- `DeveloperCard` is read as the tag name.
- Now it reads **attributes** until it hits `/>` or `>`. At the start of each attribute, it checks the next character:

| Next char         | Parser's rule                                             | Node it builds       |
| ----------------- | --------------------------------------------------------- | -------------------- |
| a letter (`n`ame) | read a name, expect `=`, then a value                     | `JSXAttribute`       |
| `{`               | must be followed by `...`, then a JS expression, then `}` | `JSXSpreadAttribute` |

For attribute **values** after `=`, there's a second rule:

| Value starts with | Meaning                                                 | Node                     |
| ----------------- | ------------------------------------------------------- | ------------------------ |
| `"`               | string, read until the closing `"`                      | `StringLiteral`          |
| `{`               | switch to **JS mode**, parse one expression, expect `}` | `JSXExpressionContainer` |

This table is the answer to your earlier question. When the parser sees `{` at the start of an attribute, it **requires `...` next**. `{developerObj}` fails at exactly that spot, which is why Babel's error says it expected `...`.

The tree it produces looks like this (simplified, but these are Babel's real node names):

```js
JSXElement {
  openingElement: JSXOpeningElement {
    name: JSXIdentifier { name: "DeveloperCard" },
    selfClosing: true,
    attributes: [
      JSXAttribute {
        name:  JSXIdentifier { name: "name" },
        value: StringLiteral { value: "Alice" }
      },
      JSXAttribute {
        name:  JSXIdentifier { name: "age" },
        value: JSXExpressionContainer {
          expression: NumericLiteral { value: 30 }
        }
      },
      JSXSpreadAttribute {
        argument: Identifier { name: "developerObj" }
      }
    ]
  },
  children: []
}
```

Notice what happened to the braces:

- `name="Alice"`: the quotes are gone. All that's left is the value `"Alice"`.
- `age={30}`: the braces became a **wrapper** node (`JSXExpressionContainer`) holding a plain number.
- `{...developerObj}`: the braces and the `...` are **gone entirely**. The node just says "spread this identifier."

So the braces were punctuation the parser used to know when to switch modes, and that's all they were. They were never an object.

## ③ Transform: rewriting the tree into plain JS

Babel's React JSX plugin walks the tree and swaps every JSX node for a normal JS node. The rules are mechanical:

**The tag name:**

- Capitalized `DeveloperCard` becomes `Identifier` (a variable reference to your function).
- Lowercase `div` would become `StringLiteral "div"` (a built-in HTML tag).

That's why components **must** start with a capital letter. It's literally the rule this step uses to decide between a variable and a string.

**Each attribute becomes one piece of a new `ObjectExpression`:**

```
JSXAttribute  name="Alice"
  → ObjectProperty { key: "name", value: StringLiteral "Alice" }

JSXAttribute  age={30}
  → ObjectProperty { key: "age", value: NumericLiteral 30 }
               ↑ the JSXExpressionContainer wrapper is unwrapped and thrown away

JSXSpreadAttribute  {...developerObj}
  → SpreadElement { argument: Identifier "developerObj" }
```

All the pieces go, **in order**, into one `ObjectExpression`. That's the invisible object from last time. Then everything gets wrapped in a call:

```js
CallExpression {
  callee: Identifier "_jsx",
  arguments: [
    Identifier "DeveloperCard",
    ObjectExpression { properties: [ ...the three pieces above... ] }
  ]
}
```

Then the code generator prints that tree back out as text:

```js
import { jsx as _jsx } from "react/jsx-runtime";

_jsx(DeveloperCard, { name: "Alice", age: 30, ...developerObj });
```

That's the end of the compiler's job. Order is preserved, so `...developerObj` comes last here and **overwrites** `name`. You get `"Alice"` from the object either way, but if the object had `name: "Zed"`, Zed would win. That's ordinary JS object-spread behavior, because by now it _is_ ordinary JS.

## ④ Run time: `jsx()` builds the React element

In the browser, the call runs. First JS evaluates the object literal, which gives you a real props object:

```js
{ name: "Alice", age: 30, country: "USA" }
```

Then `jsx(type, props)` wraps it. **This is the object you forgot about: the React element.** It's tiny:

```js
{
  $$typeof: Symbol.for("react.transitional.element"),  // "I'm a React element"
  type: DeveloperCard,      // the actual function, not a string
  key: null,
  props: { name: "Alice", age: 30, country: "USA" }
  // + a couple of dev-only bookkeeping fields
}
```

(That's React 19. React 18 used `Symbol.for("react.element")`.)

A React element is a **description** ("I want a DeveloperCard with these props"). It's not a DOM node, and `DeveloperCard` hasn't run yet.

A couple of extra nerdy bits:

- `key` is the one prop that gets **pulled out** of props and stored next to them. That's why a component can't read `props.key`.
- `$$typeof` is a Symbol for security. JSON can't contain Symbols, so if an attacker sneaks a fake `{ type: ..., props: ... }` object in through API data, React refuses to treat it as an element.

## ⑤ React renders

React looks at `element.type`:

- **It's a function**, so React calls `DeveloperCard(element.props)`. That's where your `({ name, age, country })` destructuring receives the object. Your component returns **more elements**, e.g. `{ type: "div", props: { children: [...] } }`.
- **It's a string** like `"div"`, so it's a host element and React creates a real `<div>` DOM node.

React keeps going down the tree until everything is host elements, then puts the DOM nodes on the page.

---

## The whole journey on one screen

```
name="Alice"   →  JSXAttribute + StringLiteral            →  name: "Alice"
age={30}       →  JSXAttribute + JSXExpressionContainer   →  age: 30
{...devObj}    →  JSXSpreadAttribute                      →  ...devObj
                                    ↓
                  _jsx(DeveloperCard, { name, age, ...devObj })
                                    ↓ (browser)
                  { $$typeof, type: DeveloperCard, key, props: {...} }
                                    ↓ (React)
                  DeveloperCard(props) → more elements → DOM
```

## See each step with your own eyes

1. **Step ② (the tree):** go to **astexplorer.net**, set the parser to `@babel/parser`, and paste the JSX line. Click the attribute nodes and you'll find `JSXSpreadAttribute` with no brace node inside it.
2. **Step ③ (the output code):** go to **babeljs.io/repl**, turn on the React preset and set runtime to automatic.
3. **Step ④ (the element):** in your app, run `console.log(<DeveloperCard {...developerObj} />)` and expand the result in DevTools. Look for `$$typeof`, `type` (it'll show the function) and `props`.

Number 3 is worth doing. Once you've seen that a "component in JSX" is just a small object holding a function reference and a props bag, a lot of React gets less mysterious.

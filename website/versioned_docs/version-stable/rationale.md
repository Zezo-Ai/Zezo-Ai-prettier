---
id: rationale
title: Rationale
---

Prettier is an opinionated code formatter. This document explains some of its choices.

## What Prettier is concerned about

### Correctness

The first requirement of Prettier is to output valid code that has the exact same behavior as before formatting. Please report any code where Prettier fails to follow these correctness rules — that’s a bug which needs to be fixed!

### Strings

Double or single quotes? Prettier chooses the one which results in the fewest number of escapes. `"It's gettin' better!"`, not `'It\'s gettin\' better!'`. In case of a tie or the string not containing any quotes, Prettier defaults to double quotes (but that can be changed via the [singleQuote](/docs/options#quotes) option).

JSX has its own option for quotes: [jsxSingleQuote](/docs/options#jsx-quotes).
JSX takes its roots from HTML, where the dominant use of quotes for attributes is double quotes. Browser developer tools also follow this convention by always displaying HTML with double quotes, even if the source code uses single quotes. A separate option allows using single quotes for JS and double quotes for "HTML" (JSX).

Prettier maintains the way your string is escaped. For example, `"🙂"` won’t be formatted into `"\uD83D\uDE42"` and vice versa.

### Empty lines

It turns out that empty lines are very hard to automatically generate. The approach that Prettier takes is to preserve empty lines the way they were in the original source code. There are two additional rules:

- Prettier collapses multiple blank lines into a single blank line.
- Empty lines at the start and end of blocks (and whole files) are removed. (Files always end with a single newline, though.)

### Multi-line objects

By default, Prettier’s printing algorithm prints expressions on a single line if they fit. Objects are used for a lot of different things in JavaScript, though, and sometimes it really helps readability if they stay multiline. See [object lists], [nested configs], [stylesheets] and [keyed methods], for example. We haven’t been able to find a good rule for all those cases, so by default Prettier keeps objects multi-line if there’s a newline between the `{` and the first key in the original source code. Consequently, long single-line objects are automatically expanded, but short multi-line objects are never collapsed.

You can disable this conditional behavior with the [`objectWrap`](options.md#object-wrap) option.

**Tip:** If you have a multi-line object that you’d like to join up into a single line:

```js
const user = {
  name: "John Doe",
  age: 30,
};
```

…all you need to do is remove the newline after `{`:

<!-- prettier-ignore -->
```js
const user = {  name: "John Doe",
  age: 30
};
```

…and then run Prettier:

```js
const user = { name: "John Doe", age: 30 };
```

And if you’d like to go multi-line again, add in a newline after `{`:

<!-- prettier-ignore -->
```js
const user = {
 name: "John Doe", age: 30 };
```

…and run Prettier:

```js
const user = {
  name: "John Doe",
  age: 30,
};
```

[object lists]: https://github.com/prettier/prettier/issues/74#issue-199965534
[nested configs]: https://github.com/prettier/prettier/issues/88#issuecomment-275448346
[stylesheets]: https://github.com/prettier/prettier/issues/74#issuecomment-275262094
[keyed methods]: https://github.com/prettier/prettier/pull/495#issuecomment-275745434

:::note[A note on formatting reversibility]

The semi-manual formatting for object literals is in fact a workaround, not a feature. It was implemented only because at the time a good heuristic wasn’t found and an urgent fix was needed. However, as a general strategy, Prettier avoids _non-reversible_ formatting like that, so the team is still looking for heuristics that would allow either to remove this behavior completely or at least to reduce the number of situations where it’s applied.

What does **reversible** mean? Once an object literal becomes multiline, Prettier won’t collapse it back. If in Prettier-formatted code, we add a property to an object literal, run Prettier, then change our mind, remove the added property, and then run Prettier again, we might end up with a formatting not identical to the initial one. This useless change might even get included in a commit, which is exactly the kind of situation Prettier was created to prevent.

:::

### Decorators

Just like with objects, decorators are used for a lot of different things. Sometimes it makes sense to write decorators _above_ the line they're decorating, sometimes it’s nicer if they're on the _same_ line. We haven’t been able to find a good rule for this, so Prettier keeps your decorator positioned like you wrote them (if they fit on the line). This isn’t ideal, but a pragmatic solution to a difficult problem.

```ts
@Component({
  selector: "hero-button",
  template: `<button>{{ label }}</button>`,
})
class HeroButtonComponent {
  // These decorators were written inline and fit on the line so they stay
  // inline.
  @Output() change = new EventEmitter();
  @Input() label: string;

  // These were written multiline, so they stay multiline.
  @readonly
  @nonenumerable
  NODE_TYPE: 2;
}
```

There’s one exception: classes. We don’t think it ever makes sense to inline the decorators for them, so they are always moved to their own line.

<!-- prettier-ignore -->
```ts
// Before running Prettier:
@observer class OrderLine {
  @observable price: number = 0;
}
```

```ts
// After running Prettier:
@observer
class OrderLine {
  @observable price: number = 0;
}
```

Note: Prettier 1.14.x and older tried to automatically move your decorators, so if you've run an older Prettier version on your code you might need to manually join up some decorators here and there to avoid inconsistencies:

```ts
@observer
class OrderLine {
  @observable price: number = 0;
  @observable
  amount: number = 0;
}
```

### Template literals

Template literals can contain interpolations. Deciding whether it's appropriate to insert a linebreak within an interpolation unfortunately depends on the semantic content of the template - for example, introducing a linebreak in the middle of a natural-language sentence is usually undesirable. Since Prettier doesn't have enough information to make this decision itself, it uses a heuristic similar to that used for objects: it will only split an interpolation expression across multiple lines if there was already a linebreak within that interpolation.

This means that a literal like the following will not be broken onto multiple lines, even if it exceeds the print width:

<!-- prettier-ignore -->
```js
`this is a long message which contains an interpolation: ${format(data)} <- like this`;
```

If you want Prettier to split up an interpolation, you'll need to ensure there's a linebreak somewhere within the `${...}`. Otherwise it will keep everything on a single line, no matter how long it is.

The team would prefer not to depend on the original formatting in this way, but it's the best heuristic we have at the moment.

### Semicolons

This is about using the [noSemi](options.md#semicolons) option.

Consider this piece of code:

<!-- prettier-ignore -->
```js
if (shouldAddLines) {
  [-1, 1].forEach(delta => addLine(delta * 20))
}
```

While the above code works just fine without semicolons, Prettier actually turns it into:

<!-- prettier-ignore -->
```js
if (shouldAddLines) {
  ;[-1, 1].forEach(delta => addLine(delta * 20))
}
```

This is to help you avoid mistakes. Imagine Prettier _not_ inserting that semicolon and adding this line:

```diff
 if (shouldAddLines) {
+  console.log('Do we even get here??')
   [-1, 1].forEach(delta => addLine(delta * 20))
 }
```

Oops! The above actually means:

<!-- prettier-ignore -->
```js
if (shouldAddLines) {
  console.log('Do we even get here??')[-1, 1].forEach(delta => addLine(delta * 20))
}
```

With a semicolon in front of that `[` such issues never happen. It makes the line independent of other lines so you can move and add lines without having to think about ASI rules.

This practice is also common in [standard] which uses a semicolon-free style.

Note that if your program currently has a semicolon-related bug in it, Prettier _will not_ auto-fix the bug for you. Remember, Prettier only reformats code, it does not change the behavior of the code. Take this buggy piece of code as an example, where the developer forgot to place a semicolon before the `(`:

<!-- prettier-ignore -->
```js
console.log('Running a background task')
(async () => {
  await doBackgroundWork()
})()
```

If you feed this into Prettier, it will not alter the behavior of this code, instead, it will reformat it in a way that shows how this code will actually behave when ran.

```js
console.log("Running a background task")(async () => {
  await doBackgroundWork();
})();
```

[standard]: https://standardjs.com/rules.html#semicolons

### Print width

The [printWidth](options.md#print-width) option is more of a guideline to Prettier than a hard rule. It is not the upper allowed line length limit. It is a way to say to Prettier roughly how long you’d like lines to be. Prettier will make both shorter and longer lines, but generally strive to meet the specified print width.

There are some edge cases, such as really long string literals, regexps, comments and variable names, which cannot be broken across lines (without using code transforms which [Prettier doesn’t do](#what-prettier-is-not-concerned-about)). Or if you nest your code 50 levels deep your lines are of course going to be mostly indentation :)

Apart from that, there are a few cases where Prettier intentionally exceeds the print width.

#### Imports

Prettier can break long `import` statements across several lines:

```js
import {
  CollectionDashboard,
  DashboardPlaceholder,
} from "../components/collections/collection-dashboard/main";
```

The following example doesn’t fit within the print width, but Prettier prints it in a single line anyway:

```js
import { CollectionDashboard } from "../components/collections/collection-dashboard/main";
```

This might be unexpected by some, but we do it this way since it was a common request to keep `import`s with single elements in a single line. The same applies for `require` calls.

#### Testing functions

Another common request was to keep lengthy test descriptions in one line, even if it gets too long. In such cases, wrapping the arguments to new lines doesn’t help much.

```js
describe("NodeRegistry", () => {
  it("makes no request if there are no nodes to prefetch, even if the cache is stale", async () => {
    // The above line exceeds the print width but stayed on one line anyway.
  });
});
```

Prettier has special cases for common testing framework functions such as `describe`, `it` and `test`.

### JSX

Prettier prints things a little differently compared to other JS when JSX is involved:

```jsx
function greet(user) {
  return user
    ? `Welcome back, ${user.name}!`
    : "Greetings, traveler! Sign up today!";
}

function Greet({ user }) {
  return (
    <div>
      {user ? (
        <p>Welcome back, {user.name}!</p>
      ) : (
        <p>Greetings, traveler! Sign up today!</p>
      )}
    </div>
  );
}
```

There are two reasons.

First off, lots of people already wrapped their JSX in parentheses, especially in `return` statements. Prettier follows this common style.

Secondly, [the alternate formatting makes it easier to edit the JSX](https://github.com/prettier/prettier/issues/2208). It is easy to leave a semicolon behind. As opposed to normal JS, a leftover semicolon in JSX can end up as plain text showing on your page.

```jsx
<div>
  <p>Greetings, traveler! Sign up today!</p>; {/* <-- Oops! */}
</div>
```

### Comments

When it comes to the _content_ of comments, Prettier can’t do much really. Comments can contain everything from prose to commented out code and ASCII diagrams. Since they can contain anything, Prettier can’t know how to format or wrap them. So they are left as-is. The only exception to this are JSDoc-style comments (block comments where every line starts with a `*`), which Prettier can fix the indentation of.

Then there’s the question of _where_ to put the comments. Turns out this is a really difficult problem. Prettier tries its best to keep your comments roughly where they were, but it’s no easy task because comments can be placed almost anywhere.

Generally, you get the best results when placing comments **on their own lines,** instead of at the end of lines. Prefer `// eslint-disable-next-line` over `// eslint-disable-line`.

Note that “magic comments” such as `eslint-disable-next-line` and `$FlowFixMe` might sometimes need to be manually moved due to Prettier breaking an expression into multiple lines.

Imagine this piece of code:

```js
// eslint-disable-next-line no-eval
const result = safeToEval ? eval(input) : fallback(input);
```

Then you need to add another condition:

<!-- prettier-ignore -->
```js
// eslint-disable-next-line no-eval
const result = safeToEval && settings.allowNativeEval ? eval(input) : fallback(input);
```

Prettier will turn the above into:

```js
// eslint-disable-next-line no-eval
const result =
  safeToEval && settings.allowNativeEval ? eval(input) : fallback(input);
```

Which means that the `eslint-disable-next-line` comment is no longer effective. In this case you need to move the comment:

```js
const result =
  // eslint-disable-next-line no-eval
  safeToEval && settings.allowNativeEval ? eval(input) : fallback(input);
```

If possible, prefer comments that operate on line ranges (e.g. `eslint-disable` and `eslint-enable`) or on the statement level (e.g. `/* istanbul ignore next */`), they are even safer. It’s possible to disallow using `eslint-disable-line` and `eslint-disable-next-line` comments using [`eslint-plugin-eslint-comments`](https://github.com/mysticatea/eslint-plugin-eslint-comments).

## Disclaimer about non-standard syntax

Prettier is often able to recognize and format non-standard syntax such as ECMAScript early-stage proposals and Markdown syntax extensions not defined by any specification. The support for such syntax is considered best-effort and experimental. Incompatibilities may be introduced in any release and should not be viewed as breaking changes.

## Disclaimer about machine-generated files

Some files, like `package.json` or `composer.lock`, are machine-generated and regularly updated by the package manager. If Prettier were to use the same JSON formatting rules as with other files, it would regularly conflict with these other tools. To avoid this inconvenience, Prettier will use a formatter based on `JSON.stringify` on such files instead. You may notice these differences, such as the removal of vertical whitespace, but this is an intended behavior.

## What Prettier is _not_ concerned about

Prettier only _prints_ code. It does not transform it. This is to limit the scope of Prettier. Let’s focus on the printing and do it really well!

Here are a few examples of things that are out of scope for Prettier:

- Turning single- or double-quoted strings into template literals or vice versa.
- Using `+` to break long string literals into parts that fit the print width.
- Adding/removing `{}` and `return` where they are optional.
- Turning `?:` into `if`-`else` statements.
- Sorting/moving imports, object keys, class members, JSX keys, CSS properties or anything else. Apart from being a _transform_ rather than just printing (as mentioned above), sorting is potentially unsafe because of side effects (for imports, as an example) and makes it difficult to verify the most important [correctness](#correctness) goal.

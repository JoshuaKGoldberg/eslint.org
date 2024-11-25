---
layout: post
title: "The differences between linting and type checking"
teaser: "Why linters such as ESLint and type checkers such as TypeScript catch different areas of code defects -- and are best used in conjunction with each other."
authors:
  - joshuakgoldberg
categories:
  - Storytime
tags:
  - Linting
---

Linters such as ESLint and type checkers such as [TypeScript](https://typescriptlang.org) are two kinds of tools that both analyze code and report on detected issues.
They may seem similar at first.
However, the ways they operate are very different internally.
The two kinds of tools detect very different issues and are useful in different ways.

## Definitions

*"Static analysis"* is the term for tooling that checks code for issues without running it.
There are three forms of static analysis commonly used in web development projects today:

1. **Formatters** : which only format your code quickly, without worrying about logic
2. **Linters**: which execute individually configurable checks known as "lint rules"
3. **Type checkers**: which collect all files into a full understanding of the project to make sure

We'll focus in this blog post on *linters*, namely ESLint, and *type checkers*, namely [TypeScript](https://typescriptlang.org).
ESLint and TypeScript are often used together to detect defects in code.

## Differences in Purpose

Type checkers make sure the *intent* behind values in code matches the *usage* of those values.
They check that code is "type safe": meaning values are used according to the ways their types describe as being allowed.

Type checkers do not attempt to look for defects that are only likely, rather than certain.
Nor do type checkers enforce subjective opinions that might change between projects.
That means TypeScript does not attempt to enforce best practices -- only that values are used in type safe ways.
TypeScript won't detect likely mistakes that work within those types.

Linters, on the other hand, primarily target *likely* defects and can also be used to enforce subjective opinions.
ESLint and other linters catch issues that are technically type safe but are potential sources of bugs.

For example, developers sometimes leave out the `break` or `return` at the end of a `switch` statement's `case`.
Doing so is type safe and permitted by JavaScript and TypeScript.
In practice, this is almost always a mistake that allows the next `case` statement to run accidentally.
ESLint's [`no-fallthrough`](https://eslint.org/docs/latest/rules/no-fallthrough) can catch that likely mistake:

```ts
function logFruit(value: "apple" | "banana" | "cherry") {
    switch (value) {
        case "apple":
            console.log("🍏");
            break;

        case "banana":
            console.log("🍌");

        // eslint(no-fallthrough):
        // Expected a 'break' statement before 'case'.

        case "cherry":
            console.log("🍒");
            break;
    }
}

// Logs:
// 🍌
// 🍒
logFruit("banana");
```

Another way of looking at the differences between linters and type checkers is that type checkers enforce what you *can* do, whereas linters enforce what you *should* do.

### Granular Extensibility

Another difference between linters and type checkers is how flexibly their options generally can be configured.

Type checkers are generally configured with a set list of compiler options on a project level.
TypeScript's [`tsconfig.json` ("TSConfig") files](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html) allow for compiler options that change type checking for all files in the project.
Those compiler options are set by TypeScript and generally change large swathes of type checking behavior.

Linters, on the other hand, can be granularly augmented by **plugins** that add new lint rules.
Plugin-specific lint rules extend the breadth of code checks that linter configurations can pick and choose from.

For example, this ESLint configuration enables the recommended rules from [`eslint-plugin-jsx-a11y`](https://github.com/jsx-eslint/eslint-plugin-jsx-a11y), a plugin that adds checks for accessibility in JSX libraries such as Solid.js and React:

```js title="eslint.config.js"
import js from "@eslint/js";
import jsxA11y from "eslint-plugin-jsx-a11y"

export default [
    js.configs.recommended,
    jsxA11y.flatConfigs.recommended,
    // ...
];
```

A project using the JSX accessibility rules would then be told if they render a native `<img>` tag without descriptive text per [`jsx-a11y/alt-text`](https://github.com/jsx-eslint/eslint-plugin-jsx-a11y/blob/main/docs/rules/alt-text.md):

```tsx
const MyComponent = () => <img src="source.webp" />;
//                        ~~~~~~~~~~~~~~~~~~~~~~~~~
// eslint(jsx-a11y/alt-text):
// img elements must have an alt prop, either with meaningful text, or an empty string for decorative images.
```

By adding in rules from plugins, linter configurations can be tailored to the specific best practices and common issues to the frameworks and platforms a project is built for.

## Is a Linter Still Useful With a Type Checker?

**Yes.**

If you are using TypeScript, it is still very useful to use ESLint.
In fact, linters and type checkers are most useful when used in conjunction with each other.

### Linters, With Type Checking

Traditional lint rules run on a single file at a time and have no knowledge of other files in the project.
They can't make decisions on files based on contents of other files.

However, if your project is set up using TypeScript, you can opt into "type checked" lint rules: rules that can pull in type information.
In doing so, type checked lint rules can make decisions based on other files.

For example, [`@typescript-eslint/no-for-in-array`](https://typescript-eslint.io/rules/no-for-in-array) is able to detect `for...in` loops over values that are array types, even if the values come from other files.
TypeScript would not report a type error for a `for...in` loop over an array because doing so is technically type safe and might be what a developer intended.
A linter, however, could be configured to notice that the developer probably made a mistake and meant to use a `for...of` loop instead:

```ts
// declare function getArrayOfNames(): string[];
import { getArrayOfNames } from "./my-names";

for (const name in getArrayOfNames()) {
    // eslint(@typescript-eslint/no-for-in-array):
    // For-in loops over arrays skips holes, returns indices as strings,
    // and may visit the prototype chain or other enumerable properties.
    // Use a more robust iteration method such as for-of or array.forEach instead.
    console.log(name);
}
```

Augmenting the linter with information from the type checker makes for a more powerful set of lint rules.
See [Typed Linting: The Most Powerful TypeScript Linting Ever](https://typescript-eslint.io/blog/typed-linting) for more details on typed linting with typescript-eslint.

### Type Checkers, With Linting

TypeScript adds extra complexity to JavaScript.
That complexity is very often worth it, but any added complexity brings with it the potential for misuse.
Linters are useful for stopping developers from making TypeScript-specific blunders in code.

For example, TypeScript's `{}` ("empty object") type is often misused by developers new to TypeScript.
It visually looks like it should mean any `object`, but actually means any non-`null`, non-`undefined` value --- including primitives such as `number`s and `string`s.
[`@typescript-eslint/no-empty-object-type`](https://typescript-eslint.io/rules/no-empty-object-type) catches uses of the `{}` type that likely meant `object` or `unknown` instead:

```ts
export function logObjectEntries(value: {}) {
    //                                  ~~
    // eslint(@typescript-eslint/no-empty-object-type):
    // The `{}` ("empty object") type allows any non-nullish value, including literals like `0` and `""`.
    // - If that's what you want, disable this lint rule with an inline comment or configure the 'allowObjectTypes' rule option.
    // - If you want a type meaning "any object", you probably want `object` instead.
    // - If you want a type meaning "any value", you probably want `unknown` instead.
    console.log(Object.entries(value));
}

logObjectEntries(0); // No type error!
```

Enforcing language-specific best practices with a linter helps developers learn about and correctly use type checkers such as TypeScript.

# JavaScript Code Style Guide

This document explains some code style guidelines to make JavaScript code more
readable and easier to understand.

## Prefer Array methods

> With Array methods common collection-based operations can be solved without
> imperative loops.

::: danger BAD

```javascript
const persons = [
  { firstName: "John", lastName: "Doe" },
  { firstName: "Jane", lastName: "Doe" },
  { firstName: "Bob", lastName: "Bean" },
  { firstName: "Bea", lastName: "Bean" },
];
const doeFirstNames = [];
for (const person of persons) {
  if (person.lastName !== "Doe") continue;
  doeFirstNames.push(person.firstName);
}
```

:::

::: tip BETTER

```javascript
const persons = [
  { firstName: "John", lastName: "Doe" },
  { firstName: "Jane", lastName: "Doe" },
  { firstName: "Bob", lastName: "Bean" },
  { firstName: "Bea", lastName: "Bean" },
];
const doeFirstNames = persons
  .filter((person) => person.lastName === "Doe")
  .map((person) => person.firstName);
```

:::

Solving typical collection-based problems (e.g. finding or mapping items) using
imperative loops introduces additional complexity in the code because it does
not only express _what_ to do but also _how_ it is done. This makes it harder to
understand what the real purpose of the operation is.

It is recommended to use the various methods available on `Array` instances
which allow for a declarative implementation of common operations. See the
[MDN reference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array#instance_methods)
for a list of the available functions.

There are some important things to keep in mind though:

- Some `Array` methods like `filter` and `map` return new instances with the
  resulting items. They do not change the original instance they are invoked on.
  Other methods like `sort` modify the instance they are invoked on.
- When using `Array` methods like `forEach`, `filter` and `map` each of them
  will start its own iteration over the items and there is no way to break the
  internal iteration before all items have been visited. If it turns out to be a
  performance issue, e.g. when processing huge lists of items, consider using a
  normal `for-of` loop instead.

## Prefer arrow functions

> Arrow functions make code more concise and help preserve the `this` context.

::: danger BAD

```javascript
class Component {
  filteredLastName = "Doe";
  persons = [
    { firstName: "John", lastName: "Doe" },
    { firstName: "Jane", lastName: "Doe" },
    { firstName: "Bob", lastName: "Bean" },
    { firstName: "Bea", lastName: "Bean" },
  ];
  get filteredPersons() {
    const component = this;
    return persons.filter(function (person) {
      return person.lastName === component.filteredLastName;
    });
  }
}
```

:::

::: tip BETTER

```javascript
class Component {
  filteredLastName = "Doe";
  persons = [
    { firstName: "John", lastName: "Doe" },
    { firstName: "Jane", lastName: "Doe" },
    { firstName: "Bob", lastName: "Bean" },
    { firstName: "Bea", lastName: "Bean" },
  ];
  get filteredPersons() {
    return persons.filter((person) =>
      person.lastName === this.filteredLastName;
    );
  }
}
```

:::

Especially when used as a callback traditional functions suffer from the fact
that `this` inside the function is not the same as in the outer closure. A very
common pattern to work around this problem is to set a variable referring to
`this` in the outer closure and then accessing that variable inside the callback
function. Such tricks pollute the code with unnecessary complexity.

It is recommended to use arrow functions whenever possible to write more concise
code and to allow access to `this` of the outer function. This is especially
useful for callbacks inside object methods.

However, it is important to note that arrow functions cannot be bound to a
specific `this` scope. They always preserve the scope they have been declared
inside. Therefore, arrow functions are not suitable for the declaration of
object methods.

```javascript
const brokenObject = {
  objectValue: "some value",
  getObjectValue: () => {
    return this.objectValue; // This does not work as expected!
  },
};
const correctObject = {
  objectValue: "some value",
  getObjectValue() {
    return this.objectValue; // This works
  },
};
```

## Prefer template strings over concatenation

> Template strings improve the readability of strings with variable values.

::: danger BAD

```javascript
const firstName = "John";
const lastName = "Doe";
const greeting = "Hello " + firstName + " " + lastName + "!";
```

:::

::: tip BETTER

```javascript
const firstName = "John";
const lastName = "Doe";
const greeting = `Hello ${firstName} ${lastName}!`;
```

:::

Traditional string concatenation requires a lot of plus signs to chain the
segments of literal texts and variables.

It is recommended to use template strings to compose texts containing variable
values. It improves readability.

## Prefer const for immutable variables

> Constant declaration makes the code's behavior easier to understand.

::: danger BAD

```javascript
let pi = 3.1415926535897932;
let names = ["John", "Jane"];
let longestName;
for (let name of names) {
  if (!longestName || longestName.length < name.length) {
    longestName = name;
  }
}
```

:::

::: tip BETTER

```javascript
const pi = 3.1415926535897932;
const names = ["John", "Jane"];
let longestName;
for (const name of names) {
  if (!longestName || longestName.length < name.length) {
    longestName = name;
  }
}
```

:::

Declaring all variables using `let` or `var` makes it harder to distinguish
between those intended to be assigned a value only once and the ones that may
change their value multiple times during execution.

It is recommended to use the `const` keyword for the declaration of variables
which are not expected to change value after their declaration. This makes it
easier to understand the behavior of the code because it is explicit which
variables will not change during execution. The JavaScript interpreter will also
throw an error when attempting to reassign a value.

## Prefer explicit radix when parsing integer numbers

> Explicit definition of the integer radix helps avoid unexpected runtime
> behavior.

::: danger BAD

```javascript
const value1 = parseInt("015"); // Yields 15
const value2 = parseInt("0x15"); // Yields 21
```

:::

::: tip BETTER

```javascript
const value1 = parseInt("015", 10); // Yields 15
const value2 = parseInt("0x15", 16); // Yields 21
```

:::

The radix used in the standard function `parseInt` does **not** default to `10`!
If omitted the radix is determined based on the given string format. This may
confuse developers who are not aware of that and can even introduce unexpected
behavior if the parsed string comes from an external data source. For example,
in some programming languages preceding a string with `0` indicates it is in
octal number format. However, in JavaScript `"015"` is parsed to the number `15`
by default. The radix 8 must be passed explicitly to get the result `13` which
may be expected in this case.

It is recommended to define the radix in `parseInt` calls explicitly to make it
clear which number format is expected in the parsed string. This avoids
confusion among developers and helps prevent unexpected results when parsing
data from external data sources.

## Prefer async/await for asynchronous code

> Usage of async/await produces more linear code with less callbacks.

::: danger BAD

```javascript
const produceAsyncValue = () =>
  new Promise((resolve, reject) => {
    // Calculate something
    const result = "some result";
    resolve(result);
  });
produceAsyncValue().then((result) => {
  // Do something with result
});
```

:::

::: tip BETTER

```javascript
const produceAsyncValue = async () => {
  // Calculate something
  const result = "some result";
  return result;
};
const result = await produceAsyncValue();
// Do something with result
```

:::

While the `Promise` API already helpes a lot making asynchronous code more
readable it still requires wrapping certain logic in callback functions to
execute it once the promise is fulfilled.

It is recommended to use `async/await` to write asynchronous functions and wait
for `Promise` results. This avoids callback functions and therefore gives the
code a linear look and feel.

However, pay attention to avoid unnecessary `async` function declarations
because every one produces a new `Promise` instance. When overused, it can lead
to inefficient memory usage.

```javascript
// Unnecessary async function, wrapping fetch promise in another promise
const fetchApiResult = async () => {
  const url = "https://api.somedomain.com/";
  return fetch(url);
};
```

In the example above the `async` keyword should be left out as long as the
function itself does not need to `await` other asynchronous results.

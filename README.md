# "Get Intrinsic" Proposal

EcmaScript Proposal, specs, and reference implementation to provide a robust way to retrieve intrinsics to first-run code.

Spec drafted by [@ljharb](https://github.com/ljharb).

This proposal is currently at stage 1 of the [process](https://tc39.github.io/process-document/).

## "First-run" code

This entire proposal assumes what is currently an axiom of JavaScript: that much of the language can only be relied upon if code that ran previously has not done anything malicious. In other words, all use cases for this proposal assume that the state of the runtime JavaScript environment matches the expectations of the author of the code at the time it is first evaluated (those expectations could be "no changes from what the host provides", "missing/broken features are shimmed/polyfilled", "builtins are locked down, a la SES", etc).

## Rationale / Problem Statement

Any time a portion of code evaluates, it may create functions to be executed later. At the time of function creation, the environment can be relied upon (per [above](#first-run-code)). However, at the time the function(s) are executed, this expectation may not hold true, as later code may have modified the environment in unexpeted ways (advertisement code, browser extensions, etc).

Here's a contrived example of a package that is vulnerable to builtin modification by later-run code:
```jsx
const rainbowColors = ['red', 'orange', 'yellow', 'green', 'blue', 'indigo', 'violet'];

export default function isRainbowColor(color) {
  return rainbowColors.includes(String(color).toLowerCase());
}
```

The typical approach for package authors to author their code in a way that is robust against later modification is to "cache" - to store a copy in a closed-over lexical variable - the functions they'll need in their package. Note that borrowing a prototype method in this way requires use of `.call` to set the receiver (the "this" value), so to be robust against that as well it will need to "call-bind" the methods:
```jsx
const rainbowColors = ['red', 'orange', 'yellow', 'green', 'blue', 'indigo', 'violet'];

const $String = String;
const includes = Function.call.bind(Array.prototype.includes);
const toLowerCase = Function.call.bind(String.prototype.toLowerCase);

export default function isRainbowColor(color) {
  return includes(rainbowColors, toLowerCase($String(color));
}
```

(Please note that nothing about this pattern is ergonomic, and this proposal is **not** attempting to improve the ergonomics here - that will fall to other proposals)

In an application there are often many of these kinds of packages (via transitive dependencies of larger frameworks/libraries). Due to code splitting and lazy loading, often these packages will not all be evaluated at the same time, which breaks the "first-run" expectation established above. To minimize the chance of this being a problem, they will often use a common "shared" package to hold all the intrinsics. My [es-shims](https://npmjs.com/~es-shims) packages all use one package to do this, [get-intrinsic](https://npmjs.com/get-intrinsic), which currently has about 15.1M downloads/week.

This means that whichever shim is evaluated *first* will also cause `get-intrinsic` to evaluate first, providing safe, robust access to intrinsics for all other shims, even ones loaded after the environment has been changed. Here's the example code modified to use this npm package, via another one, [call-bind](https://npmjs.com/call-bind) (~18.2M downloads/week):
```jsx
import getIntrinsic from 'get-intrinsic';
import callBound from 'call-bind/callBound';

const rainbowColors = ['red', 'orange', 'yellow', 'green', 'blue', 'indigo', 'violet'];

const $String = getIntrinsic('%String%');
const includes = callBound('Array.prototype.includes');
const toLowerCase = callBound('String.prototype.toLowerCase');

export default function isRainbowColor(color) {
  return includes(rainbowColors, toLowerCase($String(color));
}
```

The cost of this `get-intrinsic` abstraction is 9.7KB shipped to all browsers that depend on any portion of this code. Additionally, since not every intrinsic is accessible from the global - some requires modern syntax that would not parse in all browsers - a form of `eval` is required to obtain these intrinsics, which clashes with CSP requirements on some sites.

See also, committee discussion on this proposal in [July 2021](https://github.com/tc39/notes/blob/596ea8e5b199af3bfe578f605066c5b7bd6dd943/meetings/2021-07/july-15.md#getoriginals-for-stage-1).

## Proposed Solution

Provide a function property on `Reflect`, `getIntrinsic`, that takes a string argument and effectively replicates the `get-intrinsic` package API, but explicitly does not expose the specification's internal `%a.b.c%` notation. It would return the "original" value being requested.

Shims/polyfills that added new methods would of course need to also wrap/replace the `getIntrinsic` function so that it also provided the new methods; "lockdown" libraries like SES would similarly need to wrap/replace the `getIntrinsic` function so it provided whatever forms of these methods it desired.

In modern browsers/engines that ship this API, this cost will reduce to nothing, and the eval/CSP clash will go away.

Additionally, `Reflect.getIntrinsics` is provided, that produces an iterator of "intrinsic names" - each of which can be passed to `Reflect.getIntrinsic` to retrieve the desired value.

## Spec

You can view the spec for the `Reflect.getIntrinsic`/`Reflect.getIntrinsics` solution rendered as [HTML](https://tc39.github.io/proposal-get-intrinsic/).

## Implementations

None yet - it would be inappropriate to publish a runtime implementation of a language proposal for API prior to stage 3.

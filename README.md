# ECMAScript proposal: ParallelMap, ParallelFilter

## Authors

- Michele Riva

## Champions

- Ujjwal Sharma

## Summary

**ParallelMap** and **ParallelFilter** are a proposal for extending the JavaScript `Array` class by providing an easy but powerful way to parallelize computation while performing `map` and `filter` operations over arrays.

## Motivation

Due to the nature of the JavaScript computational model, callbacks in higher-order functions (such as `map`, `reduce`, or `filter`), gets executed asynchronously. We can observe this behaviour by looking at the following code example (in Node.js):

```js
import { setTimeout } from 'node:timers/promises';

const input = [1, 2, 3];

async function mapPredicate(value) {

  if (value === 2) {
    await setTimeout(0);
  } else {
    await setTimeout(value * 1000);
  }

  return value * 10;
}

const asyncPredicateResult = input.map(mapPredicate);

// asyncPredicateResult is now:
// [Promise<pending>, Promise<pending>, Promise<pending>]
// But once resolved, .map keeps the order of the original array:
// [Promise<10>, Promise<20>, Promise<30>]
```

As written in the `Array.prototype.map` specification ([https://tc39.es/ecma262/#sec-array.prototype.map](https://tc39.es/ecma262/#sec-array.prototype.map)), the order of elements should remain untouched.

In the example above, in fact, the order of the elements is the same as for the original array, even if the callbacks got executed asynchronously and the second element got resolved way earlier than the others.

While this is a totally fine and expected behaviour, there are times where we might want to perform a similar operation without any guarantee on the order of the elements in the returning array, in parallel. This is where `Array.prototype.parallelMap` can help:


```js
import { setTimeout } from 'node:timers/promises';

const input = [1, 2, 3];

async function asyncMapPredicate(value) {
  
  if (value !== 2) {
    await setTimeout(100);
  }
  
  return value * 10;
}

const parallelPredicateResult = input.parallelMap(mapPredicate);

// parallelPredicateResult is now:
// [Promise<pending>, Promise<pending>, Promise<pending>]
// But once resolved, we can observe the following:
// [Promise<20>, Promise<10>, Promise<30>]
```

The same concept applies to the `Array.prototype.filter` with `parallelFilter`, as shown in the code example above:

```js
import { setTimeout } from 'node:timers/promises';

const input = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

async function filterPredicate(value) {
  if (value % 2 === 0) {
    await setTimeout(value * 100);
  }
  
  return value < 5;
}

const parallelPredicateResult = input.parallelFilter(mapPredicate);

// parallelPredicateResult is now:
// [Promise<pending>, Promise<pending>, Promise<pending>, Promise<pending>, Promise<pending>]
// But once resolved, we can observe the following:
// [Promise<1>, Promise<3>, Promise<2>, Promise<4>]
```

This opens to a world of optimizations in performance-sensitive programs, where the order of elements in an array should be determined by the resolution time of an asynchronous (but even synchronous) function.

## What about ParallelReduce?

With reduce, things can get more complicated. The reduce operation can guarantee a deterministic result only when operating on operations closed under the associative property, i.e:

```js
const associativeBinaryOperationResult = [1,2,3,4].reduce((x, y) => x + y); // Will always output 10
const nonAssociativeBinaryOperationResult = [1,2,3,4].reduce((x, y) => x / y); // Result may change depending on callback execution order
```

For that reason, the `ParallelReduce` method is out of scope for this proposal.

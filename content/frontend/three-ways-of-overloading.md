---
title: "Three Ways of Overloading"
date: 2022-08-27T00:14:17+08:00
draft: false
categories: ["TypeScript"]
---

## Definition

`overload` means to provide multiple definitions for a function with the same name but different signatures. In TypeScript, we have three ways to achieve this.

### 1. multiple function definitions

```ts
function sum(a: number): number;
function sum(a: number, b: number): string;

// actual implementation
function sum(a: any, b?: any) {
  // do something
}
```

### 2. Generic

```ts
function sum<Args extends any[]>(...args: Args): Args['length'] extends 1 ? number : string;

// actual implementation
function sum(a: any, b?: any) {
    // do something   
}
```

### 3. Interface

```ts
interface Sum {
  (a: number): number;
  (a: number, b: number): string;
}
const sum: Sum = (a: any, b?: any) => {
  // do something
};

```

## Real World Example

curried `sum` function has two definitions:
 - if arguments(numbers) are given, return the collector function
 - instead, return the sum of all accumulated numbers

### 1. multiple function definitions
```ts
function sum(...initialArgs: number[]) {
  const acc: number[] = [...initialArgs]

  /**
   * @overload 1. if no arguments given, return the sum result.
   */
  function collector(): number

  /**
   * @overload 2. if {@link args} given, return the collector itself.
   * @param args
   */
  function collector(...args: number[]): typeof collector

  function collector(...args) {
    if (args.length) {
      acc.push(...args)
      return collector
    }
    return acc.reduce((a, b) => a + b)
  }
  return collector
}

const sixSum = sum(1, 2)(3) // a collector function
const twelve = sixSum(6)() // a number: 12
console.log(twelve) // 12

```


### 2. Generic

```ts
function sum(...initialArgs: number[]) {
  const acc: number[] = [...initialArgs]

  function collector<T extends number[]>(
    ...args: T
  ): T['length'] extends 0 ? number : typeof collector {
    if (args.length) {
      acc.push(...args)
      return collector
    }
    return acc.reduce((a, b) => a + b)
  }
  return collector
}
```

### 3. Interface

```ts
interface Collector {
  (): number
  (...args: number[]): Collector
}

function sum(...initialArgs: number[]) {
  const acc: number[] = [...initialArgs]

  const collector = (
    ...args: number[]
  ) => {
    if (args.length) {
      acc.push(...args)
      return collector
    }
    return acc.reduce((a, b) => a + b)
  }
  return collector as Collector
}
```


### Conclusion

The `multiple definition` approach is more concise and readable, but the `interface` one is more flexible. `Generic` is not recommended in such scenario.

# TypeScript String Enums

Typesafe string enums in TypeScript.

[![Build Status](https://travis-ci.org/dphilipson/typescript-string-enums.svg?branch=master)](https://travis-ci.org/dphilipson/typescript-string-enums)

## Installation

```
npm install --save typescript-string-enums
```

## Usage

Define an enum as follows:

``` javascript
// Status.ts
import { Enum } from "typescript-string-enums";

export const Status = Enum("RUNNING", "STOPPED");
export type Status = Enum<typeof Status>;
```

Use it elsewhere:

``` javascript
import { Status } from "./Status";

console.log(Status.RUNNING); // -> "RUNNING";

// Works fine.
const goodStatus: Status = Status.RUNNING;

 // TypeScript error: Type '"hello"' is not assignable to type '"RUNNING" | "STOPPED"'
const badStatus: Status = "hello";

// Enum can be used for discriminated unions:

interface RunningState {
    status: typeof Status.RUNNING;
    pid: number;
}

interface StoppedState {
    status: typeof Status.STOPPED;
    shutdownTime: Date;
}

type State = RunningState | StoppedState;

function saySomethingAboutState(state: State) {
    // The following typechecks.
    if (state.status === Status.RUNNING) {
        console.log("The pid is " + state.pid);
    } else if (state.status === Status.STOPPED) {
        console.log("The shutdown time is " + state.shutdownTime);
    }
}
```

## Motivation

Enums are useful for cleanly specifying a type that can take one of a few specific values.
TypeScript users typically implement enums in one of two ways: built-in
[TypeScript enums](https://www.typescriptlang.org/docs/handbook/enums.html) or string literals, but
each of these has drawbacks.

### Why not built-in enums?

Built-in enums have one big drawback. Their runtime value is a number, which is annoying during
development and makes them unsuitable for use with external APIs.

``` javascript
enum Status {
    RUNNING, STOPPED
}

const state = { status: Status.RUNNING, pid: 12345 };
console.log(state);
// -> { status: 0, pid: 12345 }. What status was that again?
// I hope you're not expecting other services to send you objects that look like this.
```

### Why not string literals

Using string literals throughout a program leaves it vulnerable to bugs caused by typos or
incomplete refactors. For example:

``` javascript
type Status = "RUNNING" | "STOPPED";

...

// Typo "SOTPPED" is not caught by typechecker.
// Pretty bad bug- this block will never run.
if (state.status === "SOTPPED") {
    soundTheAlarm();
}
```

Further, string literals make refactoring difficult. Suppose I have two enums:

``` javascript
type Status = "RUNNING" | "STOPPED";
type TriathlonStage = "SWIMMING" | "CYCLING" | "RUNNING";
```

Then if at a later stage I want to change `Status` to be `"STARTED" | "STOPPED"`, there's no easy
way to do it. I can't globally find/replace `"RUNNING"` to `"STARTED"` because it will also change
the unrelated string constants representing `TriathlonStage`. Instead, I have to examine every
occurrance of the string `"RUNNING"` to see if it needs to change.

I might try to make this step easier by introducing constants for the string literals, but this has
problems as well:

``` javascript
type Status = "RUNNING" | "STOPPED";

// Naive attempts to define constants for these don't work.
const StatusNaive = {
    RUNNING: "RUNNING",
    STOPPED: "STOPPED",
};

// Type error even though it shouldn't be, because StatusNaive.RUNNING has type
// string which is not assignable to Status.
const status: Status = StatusNaive.RUNNING;

// Correctly defining constants is annoyingly repetitive.
const Status = {
    RUNNING: "RUNNING" as "RUNNING",
    STOPPED: "STOPPED" as "STOPPED",
};
```

This library is effectively a programmatic version of these repetitive definitions. It attempts to
provide the best of both worlds: string enums with the convenience of built-in enums.

## How It Works

This section is not necessary to use this library, but for those curious about how it is implemented, read on.

The entire source of this library is

``` javascript
export function Enum<V extends string>(...values: V[]): { [K in V]: K } {
    const result: any = {};
    values.forEach((value) => result[value] = value);
    return result;
}

export type Enum<T> = T[keyof T];
```

We are creating a function named `Enum` and a type named `Enum`, so both can be imported with a single symbol.

The type signature

``` javascript
function Enum<V extends string>(...values: V[]): { [K in V]: K } {
```

can be read as follows: take in an array of strings and call the different types `V`. A string
constant is a type in TypeScript (for example, `const foo = "hello"` assigns the type `"hello"`
to `foo`), so `V` is actually multiple string literal types. Then `{ [K in V]: K }` describes an
object whose keys are the types that make up `V` and for each such key has a value equal to that
key. Hence, the type of `Enum("RUNNING", "STOPPED")` is

``` javascript
// This is a type, not an object literal.
{
    RUNNING: "RUNNING";
    STOPPED: "STOPPED";
}
```

Next, consider the definition

``` javascript
type Enum<T> = T[keyof T];
```

This describes, for a given keyed type `T`, the type obtained by taking the values of `T` when
passing in each key (the syntax `T[keyof T]` is meant to evoke `t[key]` for each `key` in `t`). When
applied to the type from the previous step, we end up with the union of the types of the values,
hence `Enum<typeof Enum("RUNNING", "STOPPED")>` evaluates to `"RUNNING" | "STOPPED"`, which is what
we want.

## Acknowledgements

This libary is heavily inspired by posts in
[this thread](https://github.com/Microsoft/TypeScript/issues/3192). In particular, credit goes to
users **[@igrayson](https://github.com/igrayson)** and **[@nahuel](https://github.com/nahuel)**.

Copyright © 2017 David Philipson

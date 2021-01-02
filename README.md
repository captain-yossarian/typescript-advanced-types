# I. TypeScript-advanced-types

[1. Generic class for API requests](#1generic-class-for-api-requests---link)

[2. Math operations](#2math-operations---link)

[3. Typed React children](#3typed-react-children---link)

[4. React, return component type](#4react-component-return-type---link)

[5. Compare arguments](#5compare-arguments-in-typescript---link)

[6. Generate numbers in range](#6-generate-numbers-in-range---link)

[8. Constraints are matter](#8-constraints-are-matter)

[9. Recursive types](#9-recursive-types---link1-link2)

[10. Typeguards](#10-typeguards---link)

[11. Handle tuples](#11-handle-tuples---link)

[12. Transform union to array](#12-transform-union-to-array---link)

## II. Advanced data structures

[1. Bit representation of object](#1-bit-representation-of-simple-object)

## III. Patterns

[1. Type state pattern](#1-typestate-and-builder-patterns)

[2. Publish subscribe pattern](#2-publish-subscribe-pattern)

## 1.Generic class for API requests

Let's assume that we have next allowed endpoints:

```typescript
const enum Endpoints {
  /**
   * Have only GET and POST
   */
  users = "/api/users",
  /**
   * Have only POST and DELETE
   */
  notes = "/api/notes",
  /**
   * Have only GET
   */
  entitlements = "/api/entitlements",
}
```

You might have noticed, that I used `const enum` instead of `enum`. This technique will reduce you code output.
Please, keep in mind, it works bad with babel.
You allowed to make:

- `GET` | `POST` requests for `/users`
- `POST` | `DELETE` requests for `/notes`
- `GET` requests for `/entitlements`

Let's define interfaces of our allowed fetch methods:

```typescript
interface HandleUsers {
  get<T>(url: Endpoints.users): Promise<T>;
  post(url: Endpoints.users): Promise<Response>;
}

interface HandleNotes {
  post(url: Endpoints.notes): Promise<Response>;
  delete(url: Endpoints.notes): Promise<Response>;
}

interface HandleEntitlements {
  get<T>(url: Endpoints.entitlements): Promise<T>;
}
```

Now, we can define our main `API` class

```typescript
class Api {
  get = <T = void>(url: Endpoints): Promise<T> =>
    fetch(url).then((response) => response.json());
  post = (url: Endpoints) => fetch(url, { method: "POST" });
  delete = (url: Endpoints) => fetch(url, { method: "DELETE" });
}
```

For now, class `Api` does not have any constraints.
Let's define them:

```typescript
// Just helper
type RequiredGeneric<T> = T extends void
  ? { __TYPE__ERROR__: "Please provide generic parameter" }
  : T;

interface HandleHttp {
  <T extends void>(): RequiredGeneric<T>;
  <T extends Endpoints.users>(): HandleUsers;
  <T extends Endpoints.notes>(): HandleNotes;
  <T extends Endpoints.entitlements>(): HandleEntitlements;
}
```

As You see, `HandleHttp` is just overloading for function. Nothing special except the first line. I will come back to it later.
We have class `Api` and overloadings for function. How we can combine them? Very simple - we will just create a function which returns instance of Api class

```typescript
const handleHttp: HandleHttp = <_ extends Endpoints>() => new Api();
```

Take a look on generic parameter of `httpHandler` and `HandleHttp` interface, there is a relation between them.

Let's test our result:

```typescript
const request1 = handleHttp<Endpoints.notes>(); // only delete and post methods are allowed
const request2 = handleHttp<Endpoints.users>(); // only get and post methods are allowed
const request3 = handleHttp<Endpoints.entitlements>(); // only get method is allowed
```

If you have forgotten to provide generic parameter, return type of `request` will be

```typescript
const request = {
    __TYPE__ERROR__: 'Please provide generic parameter';
}
```

Drawbacks:

- Without generic parameter, using `request.TYPE _ERROR` will be perfectly valid from TS point of view
- `Api` class is not singleton, you should create it every time

## 2.Math operations - [link](https://stackoverflow.com/questions/65280785/is-it-possible-to-declare-a-typescript-function-which-works-on-both-numbers-and)

Let's assume, You want to make some math operations either on number or bigint

Please keep in mind, this code is not valid in JS/TS:

```typescript
const sum = 2 + 2n; // Error
```

So, we want to accept only number or only bigints.
Let's start with function definition:

```typescript
function sum<A extends number | bigint>(a: A, b: A) {
  return a * a + b * b;
}
```

Unfortunately, this function don't work as expected. Let's test it:

```typescript
const x = 3n;
let y: number | bigint;
if (Math.random() < 0.5) y = 4;
else y = 4n;

sum(x, y); // OK
```

In above case, `y` can be either `number` or `bigint`. So, from TS point of view it is ok, but I'd willing to bet,
that it will throw at least 1 error in dev environment and 1K errors in production. It was a joke.

Ok, what we can do?
We can define two generic parameters:

```typescript
type Numbers = number | bigint;

function sumOfSquares<A extends Numbers, B extends Numbers>(
  a: A,
  b: B
): Numbers {
  return a * a + b * b;
}

const result = sumOfSquares(x, y); // There should be an error
const result2 = sumOfSquares(3n, 4); // There should be an error
const result3 = sumOfSquares(3, 4n); // There should be an error
const result4 = sumOfSquares(3, 4); // ok
const result5 = sumOfSquares(3n, 4n); // ok
```

Unfortunately, above example still don't work as we expect.

```typescript
<A extends Numbers, B extends Numbers>
// same as
<A extends number | bigint, B extends number | bigint>
// so A can be number and B in the same time can be bigint
```

Only overloadings might help us here. We should explicitly say, that `B` generic parameter should have same type as `A`

```typescript
type Numbers = number | bigint;

function sumOfSquares<A extends number, B extends number>(a: A, b: B): number;
function sumOfSquares<A extends bigint, B extends bigint>(a: A, b: B): bigint;
function sumOfSquares<A extends Numbers, B extends A>(a: A, b: B): Numbers {
  return a * a + b * b;
}

const result = sumOfSquares(x, y); // There should be an error
const result2 = sumOfSquares(3n, 4); // There should be an error
const result3 = sumOfSquares(3, 4n); // There should be an error
const result4 = sumOfSquares(3, 4); // ok
const result5 = sumOfSquares(3n, 4n); // ok
```

## 3.Typed React children - [link](https://stackoverflow.com/questions/64967447/adding-required-props-to-child-react-elements-with-typescript)

Let's assume you want to create component which will accept array of children components with certain props ?
First approach:

```typescript
function One() {
  return <div>Child One</div>;
}

const Two: React.FC<{ label: string }> = ({ label }) => {
  return <div>{label}</div>;
};

function Parent({ children }: { children: JSX.Element[] }) {
  return (
    <div>
      {children.map((child) => (
        <div key={child.props.label}>{child}</div> // Props: any
      ))}
    </div>
  );
}

function App2() {
  return <Parent children={[<One />, <Two label={"hello"} />]} />;
}
```

As you see, there is nothing highlighted, code is ok, but we still need to accept components with `label` props.
Let's change out `children` prop type.

```typescript
function One() {
  return <div>Child One</div>;
}

const Two: React.FC<{ label: string }> = ({ label }) => {
  return <div>{label}</div>;
};

function Parent({
  children,
}: {
  children: React.ReactElement<{ label: string }>[]; // change is here
}) {
  return (
    <div>
      {children.map((child) => (
        <div key={child.props.label}>{child}</div>
      ))}
    </div>
  );
}

function App2() {
  return <Parent children={[<One />, <Two label={"hello"} />]} />;
}
```

Does it work - no! But, why????
Because `React.ReactElement<{label: string}>` is still union type, and in fact, it accepts `(props:any)=>{}`
It is looks like, we all forgot about React native syntax, did not we?
On my second approach, I will change a bit declaration of `One` component.

```typescript
// One is explicitly typed now
const One: React.FC = () => {
    return <div>Child One</div>;
}

const Two: React.FC<{ label: string }> = ({ label }) => {
    return <div>{label}</div>;
};

function Parent({
    children
}: {
    children: React.ReactElement<{ label: string }>[];
}) {
    return (
        <div>
            {children.map((child) => (
                <div key={child.props.label}>{child}</div>
            ))}
        </div>
    );
}

function App2() {
    return React.createElement(Parent, {
        children: [
            // error, all components should have label prop
            React.createElement(One, null),
            React.createElement(Two, { label: 'd' }) // no error
        ]
    });
}

function App3() {
    return React.createElement(Parent, {
        children: [
            /**
             * still error, because One don't expect {label: string}
             * If you add typings for One, error will disappear
             */
            React.createElement(One, { label: 'd' }), //error
            React.createElement(Two, { label: 'd' }) // no error
        ]
    })
```

So, we can write now the code which will meet our requirements:

```typescript
const One: React.FC<{ label: string }> = ({ label }) => {
  return <div>{label}</div>;
};

const Two: React.FC<{ label: string }> = ({ label }) => {
  return <div>{label}</div>;
};

function Parent({
  children,
}: {
  children: React.ReactElement<{ label: string }>[];
}) {
  return (
    <div>
      {children.map((child) => (
        <div key={child.props.label}>{child}</div>
      ))}
    </div>
  );
}

function App4() {
  return React.createElement(Parent, {
    children: [
      React.createElement(One, { label: "d" }),
      React.createElement(Two, { label: "d" }),
    ],
  });
}
```

## 4.React component return type - [link](https://stackoverflow.com/questions/65406516/react-typescript-difference-between-react-fc-t-and-function)

There is a common pattern for typing return value of component:

```typescript
type Props = {
  label: string;
  name: string;
};

const Result: FC<Props> = (prop: Props): JSX.Element => <One label={"hello"} />;

type ComponentReturnType = ReturnType<typeof Result>; // React.ReactElement<any, any> | null
```

Is it correct ? - Yes.
Is it helpful ? - Not really.
What if you need to make sure that Result component will always return component with some particular props.
For example I'm interested in `{label:string}` property.

```typescript
type Props = {
  label: string;
};
type CustomReturn = React.ReactElement<Props>;

const MainButton: FC<Props> = (prop: Props): CustomReturn => <Two />;
```

Unfortunately, there is no error. This code compiles.
Native React syntax comes to help!

```typescript
const Two: React.FC = () => <div></div>;

type Props = {
  label: string;
};
type CustomReturn = React.ReactElement<Props>;

const MainButton: FC<Props> = (prop: Props): CustomReturn =>
  React.createElement(Two); // Error
```

Finally, we have an error:

```
Type 'FunctionComponentElement<{}>' is not assignable to type 'CustomReturn'. Types of property 'props' are incompatible.
Property 'label' is missing in type '{}' but required in type '{ label: string; }'.ts(2322)
```

This code works as expected:

```typescript
type Props = {
  label: string;
};
type CustomReturn = React.ReactElement<Props>;

const One: React.VFC<{ label: string }> = ({ label }) => <div>{label} </div>;

const MainButton: FC<Props> = (props: Props): CustomReturn =>
  React.createElement(One, props);
```

Btw, small reminder, how to use generics with React components:

```typescript
import React from "react";

type Props<D, S> = {
  data: D;
  selector: (data: D) => S;
  render: (data: S) => any;
};

const Comp = <D, S>(props: Props<D, S>) => null;

const result = (
  <Comp<number, string>
    data={2}
    selector={(data: number) => "fg"}
    render={(data: string) => 42}
  />
); // ok

const result2 = (
  <Comp<number, string>
    data={2}
    selector={(data: string) => "fg"}
    render={(data: string) => 42}
  />
); // expected error

const result3 = (
  <Comp<number, string>
    data={2}
    selector={(data: number) => "fg"}
    render={(data: number) => 42}
  />
); // expected error
```

## 5.Compare arguments in TypeScript - [link](https://stackoverflow.com/questions/65361696/arguments-of-same-length-typescript)

Let's say you want to make a function with next constraints:

- First argument should be an array
- Second arguments should be 2D array, where each nested array has same length as first argument
  Pseudocode:

```
const handleArray=(x: number[], y:number[][])=>void
```

Let's start from defining our function. This is very naive approach, we will improve it later

```typescript
type ArrayElement = number;
type Array1D = ReadonlyArray<ArrayElement>;

function array<X extends Array1D, Y extends readonly ArrayElement[]>(
  x: X,
  y: readonly Y[]
) {}

const result = array([1, 2, 3], [[1, 1, 1, 1, 1]]); // no error, unexpected behaviour
```

So, now we should find a way to compare length of first argument with length of all inner arrays of second argument.
First, of all we should define `Length` util.

```typescript
/**
 * Get length of the array
 */
type Length<T extends ReadonlyArray<any>> = T extends { length: infer L }
  ? L
  : never;
/**
 * There is another approach to get the length of the array
 */
type Length2<T extends ReadonlyArray<any>> = T["length"];

type ArrayMock1 = readonly [1, 2, 3];
type ArrayMock2 = readonly [4, 5, 6, 7];
type ArrayMock3 = readonly [8, 9, 10];
type ArrayMock4 = readonly [[11, 12, 14]];
type ArrayMock5 = readonly [[15, 16]];
type ArrayMock6 = readonly [[1], [2], [3]];

type TestArrayLength1 = Length<ArrayMock1>; // 3
type TestArrayLength2 = Length<ArrayMock2>; // 4
```

Now, when we know how to get length of the array, we should create comparison util.

```typescript
/**
 * Compare length of the arrays
 */
type CompareLength<
  X extends ReadonlyArray<any>,
  Y extends ReadonlyArray<any>
> = Length<X> extends Length<Y> ? true : false;

/**
 * Tests
 */
type TestCompareLength1 = CompareLength<ArrayMock1, ArrayMock2>; // false
type TestCompareLength2 = CompareLength<ArrayMock1, ArrayMock3>; // true
```

It is looks like we have all our necessary utils. If you still have't head ake, here You have other portion of types to think about.
Try to figure out whats going on here:

```typescript
{
  type Foo = {
    x: number;
  };

  type FooX = {
    x: number;
  }["x"];

  type Result = FooX extends number ? true : false; // true

  const obj = {
    y: 42,
  }["y"];

  obj; // 42
}

/**
 * A bit complicated with weird conditional statement
 */
{
  type Foo = {
    x: number;
  };

  type FooX<T> = {
    x: number;
    y: string;
  }[T extends number ? "x" : "y"];

  type Result = FooX<number> extends number ? true : false; // true

  type Result2 = FooX<string> extends string ? true : false; // true

  const condition = 2;

  const obj = {
    y: 42,
    x: 43,
  }[condition === 2 ? "y" : "x"];

  obj; // 42
}
```

Now, when You are familiar with such a weird syntax, we can go further. Here is our function definition with 1 overloading.

```typescript
function array<
  X extends Array1D,
  Y extends {
    0: readonly ArrayElement[];
  }[CompareLength<X, Y> extends true ? 0 : never]
>(x: X, y: readonly Y[]): void;

function array<X extends Array1D, Y extends readonly ArrayElement[]>(
  x: X,
  y: readonly Y[]
) {}
```

And here is our whole code placed in one block with type tests

```typescript
type ArrayElement = number;
type Array1D = ReadonlyArray<ArrayElement>;

type Length<T extends ReadonlyArray<any>> = T extends { length: infer L }
  ? L
  : never;

type CompareLength<
  X extends ReadonlyArray<any>,
  Y extends ReadonlyArray<any>
> = Length<X> extends Length<Y> ? true : false;

function array<
  X extends Array1D,
  Y extends {
    0: readonly ArrayElement[];
  }[CompareLength<X, Y> extends true ? 0 : never]
>(x: X, y: readonly Y[]): void;
function array<X extends Array1D, Y extends readonly ArrayElement[]>(
  x: X,
  y: readonly Y[]
) {}

const arr = [1, 2, 3] as const;

/**
 * No errors expected
 */
const result = array(
  [1, 2, 3] as const,
  [
    [1, 1, 1],
    [1, 2, 3],
  ] as const
); // ok
const result0 = array([1, 2, 3] as const, [[1, 1, 1]] as const); // ok

/**
 * Errors expected
 */
const result1 = array([1, 2, 3], [[1, 1, 1], [1]]); // no error, but should be
const result2 = array(
  [1, 2, 3] as const,
  [
    [1, 1, 1],
    [1, 2],
  ] as const
); // no error, but should be
const result3 = array([1, 2, 3] as const, [[1, 2]] as const); // error
const result5 = array([1, 2, 3] as const, [1] as const); // error
const result6 = array([1, 2, 3] as const, [[1, 2, 3], []] as const); // no error, but should be
const result7 = array(arr, [[1, 1, 1]]); // no error, but should be
```

It is look like we made logical error somewhere in the code. Ok, not you, I made :D.
Maybe we should test again our `Length` type util. What exactly are we expect from this util ? Answer is - literal integer. Let's test it again:

```typescript
type Length<T extends ReadonlyArray<any>> = T extends { length: infer L }
  ? L
  : never;
const array1 = [1, 2, 3];

type Test1 = Length<[1, 2, 3]>; // ok - 3 literal type
type Test2 = Length<typeof array1>; // not ok - number

/**
 * Q: But why TS does not complain here?
 * I have explicitly defined that Length argument should extend ReadonlyArray
 *
 * A: Because, literal array type extends ReadonlyArray
 * When you use `typeof array1`, TS treats it as number[], because array1 is mutable.
 * Hence TS can't infer the length of array1. It is possible only in runtime
 */
```

So, we should provide extra restrictions for mutable arrays? Correct? - Yes. Let's provide them:

```typescript
// Type of mutable array length will be always - number
type MutableLength = unknown[]["length"]; // number

type CheckCompatibility = number extends 5 ? true : false; // false
type CheckCompatibility2 = 5 extends number ? true : false; // true

/**
 * This code works because  number does not extends literal 5
 * but literal 5 does extends number
 */
export type Length<T extends ReadonlyArray<any>> = T extends { length: infer L }
  ? MutableLength extends L
    ? MutableLength
    : L
  : MutableLength;

/**
 * Tests
 */
const array = [1, 2, 3];
const arrayImm = [1, 2, 3] as const;

type Test1 = Length<number[]>; // number
type Test2 = Length<unknown[]>; // number
type Test3 = Length<typeof array>; // number
type Test4 = Length<typeof arrayImm>; // 3

type CompareLength<
  X extends ReadonlyArray<any>,
  Y extends ReadonlyArray<any>
> = Length<X> extends Length<Y> ? true : false;

/**
 * Let's test again CompareLength
 */
const arr1 = [1, 2, 3];
const arr2 = [1, 2];
type Test1 = CompareLength<typeof arr1, typeof arr2>; // true, BANG! this is not what we are expect!
```

`CompareLength` should be rewritten as follow:

```typescript
type CompareLength<
  X extends ReadonlyArray<any>,
  Y extends ReadonlyArray<any>
> = MutableLength extends Length<X>
  ? false
  : MutableLength extends Length<Y>
  ? false
  : Length<X> extends Length<Y>
  ? true
  : false;

/**
 * Tests
 */
const arr1 = [1, 2, 3];
const arr2 = [1, 2];
type Test1 = CompareLength<typeof arr1, typeof arr2>; // false, expected
```

Ok, I'm exhausted now, it should work. Let's test our code:

```typescript
type ArrayElement = number;
type Array1D = ReadonlyArray<ArrayElement>;

type MutableLength = unknown[]["length"]; // number

/**
 * Get length of the array
 * Allow only immutable arrays
 */
export type Length<T extends ReadonlyArray<any>> = T extends { length: infer L }
  ? MutableLength extends L
    ? MutableLength
    : L
  : MutableLength;

/**
 * Compare length of the arrays
 */
type CompareLength<
  X extends ReadonlyArray<any>,
  Y extends ReadonlyArray<any>
> = MutableLength extends Length<X>
  ? false
  : MutableLength extends Length<Y>
  ? false
  : Length<X> extends Length<Y>
  ? true
  : false;

/**
 * CompareLength, compares length of X and filtered Y,
 * if true - return zero index element - ReadonlyArray<ArrayElement>
 * if false - return never
 *
 * So, if it will return never, then you can't pass second argument,
 * but if you did not provide second argument, you will receive another error -
 * function expects two arguments
 */
function array<
  X extends Array1D,
  Y extends {
    0: readonly ArrayElement[];
  }[CompareLength<X, Y> extends true ? 0 : never]
>(x: X, y: readonly Y[]): "put here your returned type";

function array<
  X extends Array1D,
  Y extends readonly ArrayElement[],
  Z extends CompareLength<X, Y>
>(x: X, y: readonly Y[]) {
  return [1, 2, 3] as any;
}
const result = array(
  [1, 2, 3] as const,
  [
    [1, 1, 1],
    [1, 2, 3],
  ] as const
); // ok
const result0 = array([1, 2, 3] as const, [[1, 1, 1]] as const); // ok

const arr = [1, 2, 3] as const;

const result1 = array([1, 2, 3], [[1, 1, 1], [1]]); // error
const result2 = array(
  [1, 2, 3] as const,
  [
    [1, 1, 1],
    [1, 2],
  ] as const
); // no error, but SHOULD BE
const result3 = array([1, 2, 3] as const, [[1, 2]] as const); // error
const result5 = array([1, 2, 3] as const, [1] as const); // error
const result5 = array([1, 2, 3] as const, [[1, 2, 3], []] as const); // error
const result6 = array(arr, [[1, 1, 1]]); // error, because TS is unable to fidure out length of mutable array.
```

Ohh, what a .... What's going on here ? We still need to fix one failed test, see `result3`.
It looks like if second arguments contains at least one array which feet requirements,
TS ok with it. So we should compare arrays only if their length are equal.

```typescript
/**
 * Check if all inner arrays have same length as X
 */
type Filter<
  X extends ReadonlyArray<any>,
  Y extends ReadonlyArray<any>
> = X["length"] extends Y["length"]
  ? Y["length"] extends X["length"]
    ? Y
    : never
  : never;

type Test1 = Filter<[1, 2, 3], [1, 2]>; // never
type Test2 = Filter<[], [1, 2]>; // never
type Test3 = Filter<[1, 2], [1, 2]>; // [1,2]

/**
 * Please keep in mind that Y can be and will be a union type of inner arrays
 *
 */
type Test4 = Filter<[], [1, 2] | [1, 2, 3]>; // never
type Test5 = Filter<[1, 2], [1] | [1, 2] | [1, 2, 3]>; // never

/**
 * Btw, try to rewrite Filter without second conditional
 */
type Filter<
  X extends ReadonlyArray<any>,
  Y extends ReadonlyArray<any>
> = X["length"] extends Y["length"] ? Y : never;

type Test5 = Filter<[1, 2], [1] | [1, 2] | [1, 2, 3]>; // not never, not ok
/**
 * Why ?
 */
type O = 5 extends 1 | 2 | 5 ? true : false; // true
type O2 = 1 | 2 | 5 extends 5 ? true : false; // false
```

Finall working code:

```typescript
type ArrayElement = number;
type Array1D = ReadonlyArray<ArrayElement>;
type MutableLength = unknown[]["length"]; // number

export type Length<T extends ReadonlyArray<any>> = T extends { length: infer L }
  ? MutableLength extends L
    ? MutableLength
    : L
  : MutableLength;

type CompareLength<
  X extends ReadonlyArray<any>,
  Y extends ReadonlyArray<any>
> = MutableLength extends Length<X>
  ? false
  : MutableLength extends Length<Y>
  ? false
  : Length<X> extends Length<Y>
  ? true
  : false;

type Filter<
  X extends ReadonlyArray<any>,
  Y extends ReadonlyArray<any>
> = X["length"] extends Y["length"]
  ? Y["length"] extends X["length"]
    ? Y
    : never
  : never;

function array<
  X extends Array1D,
  Y extends {
    0: readonly ArrayElement[];
  }[CompareLength<X, Filter<X, Y>> extends true ? 0 : never]
>(x: X, y: readonly Y[]): "put here your returned type";

function array<
  X extends Array1D,
  Y extends readonly ArrayElement[],
  Z extends CompareLength<X, Y>
>(x: X, y: readonly Y[]) {
  return [1, 2, 3] as any;
}

/**
 * Positive Tests
 */
const result = array(
  [1, 2, 3] as const,
  [
    [1, 1, 1],
    [1, 2, 3],
  ] as const
); // ok
const result0 = array([1, 2, 3] as const, [[1, 1, 1]] as const); // ok

const arr = [1, 2, 3] as const;

/**
 * Negative Tests
 */
const result1 = array([1, 2, 3], [[1, 1, 1], [1]]); // error
const result2 = array(
  [1, 2, 3] as const,
  [
    [1, 1, 1],
    [1, 2],
  ] as const
); // error
const result3 = array([1, 2, 3] as const, [[1, 2]] as const); // error
const result4 = array([1, 2, 3] as const, [[1], [1, 2], [1, 2, 3]] as const); // error
const result5 = array([1, 2, 3] as const, [1] as const); // error
const result6 = array([1, 2, 3] as const, [[1, 2, 3], []] as const); // error
const result7 = array(arr, [[1, 1, 1]]); // error, because TS is unable to figure out length of mutable array.
```

## 6. Generate numbers in range - [link](https://stackoverflow.com/questions/65307438/how-to-define-properties-in-a-typescript-interface-with-dynamic-elements-in-the)

Let's take a look on `type Values = T[keyof T]` utility.
Maybe you wonder, what does it mean ?
Before we continue, please make sure you are aware of [distibutive types](https://www.typescriptlang.org/docs/handbook/advanced-types.html#distributive-conditional-typ)

Let's start with simple example:

```typescript
interface Foo {
  age: number;
  name: string;
}

type Alias1 = Foo["age"]; // number
type Alias2 = Foo["name"]; // stirng
type Alias3 = Foo["age" | "name"]; // string | number

type Check = keyof Foo; // 'age'|'name
```

Our `Values` utility works perfect with objects, but not with arrays.
To get all keys of object, we use - `keyof`.
To get all array elements we use `[number]` because arrays have numeric keys.

```typescript
type Arr = [1, 2, 3];
type Val1 = Arr[0]; // 1
type Val2 = Arr[1]; // 2
type Val3 = Arr[0 | 1]; // 1|2
type Val4 = Arr[0 | 1 | 2]; // 3 | 1 | 1
type Val5 = Arr[number]; // 3 | 1 | 1
```

Now, we can keep going. Let's define out utility types.
For clarity, I will use simple Assert test type

```typescript
type Assert<T, U extends T> = T extends U ? true : false;

type Values<T> = T[keyof T];

{
  type Test1 = Values<{ age: 42; name: "John" }>; //  42 | "John"
  type Test2 = Assert<Test1, "John" | 42>;
}

type LiteralDigits = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;

type NumberString<T extends number> = `${T}`;

{
  type Test1 = Assert<NumberString<6>, "6">; // true
  type Test2 = Assert<NumberString<42>, "42">; // true
  type Test3 = Assert<NumberString<6>, 6>; // false
  type Test4 = Assert<NumberString<6>, "6foo">; // false
}

type AppendDigit<T extends number | string> = `${T}${LiteralDigits}`;

{
  /**
   * If you don't understand why, please read again about distributivenes here
   * https://www.typescriptlang.org/docs/handbook/advanced-types.html#distributive-conditional-types
   */
  type Test1 = Assert<
    AppendDigit<2>,
    "20" | "21" | "22" | "23" | "24" | "25" | "26" | "27" | "28" | "29"
  >; // true
  type Test2 = Assert<
    AppendDigit<9>,
    "90" | "91" | "92" | "93" | "94" | "95" | "96" | "97" | "98" | "99"
  >; // true
}

type MakeSet<T extends number> = {
  [P in T]: AppendDigit<P>;
};

{
  type Test1 = Assert<
    MakeSet<1>,
    {
      1: "10" | "11" | "12" | "13" | "14" | "15" | "16" | "17" | "18" | "19";
    }
  >;

  type Test2 = Assert<
    MakeSet<1 | 2>,
    {
      1: "10" | "11" | "12" | "13" | "14" | "15" | "16" | "17" | "18" | "19";
      2: "20" | "21" | "22" | "23" | "24" | "25" | "26" | "27" | "28" | "29";
    }
  >;
}

type RemoveTrailingZero<
  T extends string
> = T extends `${infer Fst}${infer Rest}`
  ? Fst extends `0`
    ? RemoveTrailingZero<`${Rest}`>
    : `${Fst}${Rest}`
  : never;

{
  /**
   * Because nobody uses 01 | 02 ... | 0n
   * Everybody use 1 | 2 | 3 ... | n
   */
  type Test1 = Assert<RemoveTrailingZero<"01">, "1">;
  type Test2 = Assert<RemoveTrailingZero<"02" | "03">, "2" | "3">;
  type Test3 = Assert<RemoveTrailingZero<"002" | "003">, "2" | "3">;
}

type From_1_to_999 = RemoveTrailingZero<
  Values<
    {
      [P in Values<MakeSet<LiteralDigits>>]: AppendDigit<P>;
    }
  >
>;

type By<V extends NumberString<number>> = RemoveTrailingZero<
  Values<
    {
      [P in V]: AppendDigit<P>;
    }
  >
>;

/**
 * Did not use recursion here,
 * because my CPU will blow up
 */
type From_1_to_99999 =
  | From_1_to_999
  | By<From_1_to_999>
  | By<From_1_to_999 | By<From_1_to_999>>;
```

## 8. Constraints are matter

Please take a look on next example and answer a question: `Is your generic is helpful?`

```typescript
const makeGenericArray = <T>(arr: Array<T>) => arr;
const colors = makeGenericArray(["red", "green", "blue"]); //type string[]
const colors2 = makeGenericArray([1, 2, 3]); //type number[]If your answer is: Yes, but actually - no, we are on the same boat
```

TypeScript doing his best to narrow types and made them helpful.
So if you are expect T parameter to be either string or number, please provide constraints:

```typescript
const makeStringArray = <T extends string | number>(arr: Array<T>) => arr;
const colorsLiteral = makeStringArray(["red", "green", "blue"]); // type ("red" | "green" | "blue")[]
const colorsLiteral2 = makeStringArray([1, 2, 3]); // type ("red" | "green" | "blue")[]
```

Now, your types are more helpful.

## 9. Recursive types - [link1](https://stackoverflow.com/questions/64899511/deepexclude-type-for-typescript), [link2](https://stackoverflow.com/questions/65503728/defining-a-type-for-this-function-that-works-on-arbitrary-length-tuples)

Simple example:

```typescript
type Immutable<T> = { readonly [K in keyof T]: Immutable<T[K]> };

function foo<T>(t: T): Immutable<T> {
  return t;
}

const result1 = foo({ age: { name: "John" } }); // { readonly age: { name: string }; }
```

Make all properties immutable except name children

```typescript
type Immutable<T> = {
  readonly [K in keyof T]: K extends "name" ? T[K] : Immutable<T[K]>;
};

function foo<T>(t: T): Immutable<T> {
  return (t as any) as Immutable<T>;
}

const result1 = foo({ age: { name: { surname: "Doe" } } });
result1.age.name = "2"; // error
result1.age.name.surname = "2"; // ok
```

## 10. Typeguards - [link](https://stackoverflow.com/questions/65429424/need-help-in-understanding-confusing-typescript-function)

I'd willing to bet, you are using Array.prototype.filter 1 hundred times per day.
And I bet you want to do it like a PRO.
I have found this example in this [book](https://typescriptnapowaznie.pl/)
Let's say you have an array, and you want to get rid of 4s and 9s

```typescript
const arr = [85, 65, 4, 9] as const;
type Arr = typeof arr;

/**
 * Naive approach
 */
const result_naive = arr.filter((elem) => elem !== 4 && elem !== 9); // (85 | 65 | 4 | 9)[]
```

You still have 4 | 9 in your union type. This is not what we expected.
Here is much better approach:

```typescript
type Without_4_and_9 = Exclude<Arr[number], 4 | 9>;
const result = arr.filter(
  (elem): elem is Without_4_and_9 => elem !== 4 && elem !== 9
); // (85 | 65)[]
```

By using simple utility types, we can emulate JS Array.prototype.findIndex

```typescript
const arr = [85, 65, 4, 9] as const;
type Arr = typeof arr;

type Values<T> = T[keyof T];

type ArrayKeys = keyof [];

type FindIndex<
  T extends ReadonlyArray<number>,
  Value extends number = 0,
  Keys extends keyof T = keyof T
> = {
  [P in Keys]: Value extends T[P] ? P : never;
};

type Result = Values<Omit<FindIndex<Arr, 65>, ArrayKeys>>; // '1'
```

Is second example is useful in practive? Of course not) Will it help you to understand better TS type system? - Definitely
[Here](https://stackoverflow.com/questions/65429424/need-help-in-understanding-confusing-typescript-function) you can find very interesting example with typeguards:

## 11. Handle tuples - [link](https://stackoverflow.com/questions/65466617/typescript-filter-out-rest-parameter)

Let's say you have a literal type of array and you want to filter this type

```typescript
/**
 * We should get rid of all numbers
 */
type Arr = [number, string, ...number[]];

type Filter<T, U>= // @todo

// Our test suite
type Test0 = Filter<[], number>; // []
type Test1 = Filter<Arr, number>; // [string, ...number[]]
type Test2 = Filter<Arr, number[]>; // [number, string]

type Test3 = Filter<Arr, number | number[]>; // [string]
type Test4 = Filter<Arr, number | number[] | string>; // []
```

First of all, we should handle all possible cases. What if first generic parameter of `Filter` is empty array?

```typescript
type Filter<T extends any[], F> = T extends [] ? [] : T; //

type Test0 = Filter<[], number>; // []
```

Before, we continue, make sure you are aware of some basics of functional programming lists operations.
Each list has Head - the first element and Tail - all elements but first.
To iterate recursively through the tuple, we should every time to separate `Head` and `Tail`

```typescript
type Head<T extends ReadonlyArray<unknown>> = T extends readonly [
  infer H,
  ...infer Tail
]
  ? H
  : never;

{
  type Test1 = Assert<Head<[1, 2, 3]>, 1>;
  type Test2 = Assert<Head<["head", "tail"]>, "head">;
  type Test3 = Assert<Head<[]>, never>;
}

type Tail<List extends ReadonlyArray<unknown>> = List extends readonly [
  infer H,
  ...infer Tail
]
  ? Tail
  : never;

{
  type Test1 = Assert<Tail<[1, 2, 3]>, [2, 3]>;
  type Test2 = Assert<Tail<["head", "tail"]>, ["tail"]>;
  type Test3 = Assert<Head<[]>, never>;
}
```

Very straightforward.
I hope you have noticed, that in case of empty list, I returned never, because empty list has not neither head nor tail
Now, we can define our recursive type:

```typescript
type Filter<T extends any[], F> = T extends []
  ? []
  : T extends [infer Head, ...infer Tail]
  ? Head extends F
    ? Filter<Tail, F>
    : [Head, ...Filter<Tail, F>]
  : /* --> */ [];

{
  type Test1 = Filter<Arr, number>; // [string, ...number[]]
  type Test2 = Filter<Arr, number[]>; // [number, string]

  type Test3 = Filter<Arr, number | number[]>; // [string]
  type Test4 = Filter<Arr, number | number[] | string>; // []
  type Test5 = Filter<[number[]], number | number[] | string>; // []
}
```

I hope you have noticed the `-->` symbol at the and of conditional type.
It is mean, that if list has not neither Head nor Tail - return empty array.
I used empty array here instead of never, because we want to filter an array, not to get either Head or Tail

Is it possible to reuse above pattern for other cases ? Sure!

Take a look on this [question](https://stackoverflow.com/questions/65476787/how-to-dynamically-create-an-object-based-on-a-readonly-tuple-in-typescript/65478618#65478618)

Let's say you have an array and you want to map it to other array. How to do it with type system?

```typescript
type Mapped<
  Arr extends Array<unknown>,
  Result extends Array<unknown> = []
> = Arr extends []
  ? []
  : Arr extends [infer H]
  ? [...Result, [H, true]]
  : Arr extends [infer Head, ...infer Tail]
  ? Mapped<[...Tail], [...Result, [Head, true]]>
  : Readonly<Result>;

type Result = Mapped<[1, 2, 3, 4]>; // [[1, true], [2, true], [3, true], [4, true]]
```

If you want to restrict maximum array (tuple) length - this is not a problem.
[Here](https://stackoverflow.com/questions/65495285/typescript-restrict-maximum-array-length) you can find how to do it

```typescript
type ArrayOfMaxLength4 = readonly [any?, any?, any?, any?];
```

Ok, ok. I know what you are thinking about. How we can reduce the array to object? Is it possible at all with typings?
Sure, you can take a look on this [answer](https://stackoverflow.com/questions/65517583/create-an-object-type-in-typescript-derived-from-another-objects-values-using-a/65522869#65522869)

We should transform `Data` type to `ExpectedType` type

```typescript
export const myArray = [
  { name: "Relationship", options: "foo" },
  { name: "Full name of family member as shown in passport", options: "bar" },
  { name: "Country family member lives in", options: "baz" },
] as const;

type Data = typeof myArray;

type ExpectedType = Array<{
  Relationship: "foo";
  "Full name of family member as shown in passport": "bar";
  "Country family member lives in": "baz";
}>;

type Values<T> = T[keyof T];

type Data = typeof myArray;

type Elem = { readonly name: string; readonly options: string };

type MapObject<T extends Elem, Key extends keyof T, Val extends keyof T> = {
  [P in Values<Pick<T, Key>> & string]: T[Val];
};

type Mapper<
  Arr extends ReadonlyArray<Elem>,
  Result extends Record<string, any> = {}
> = Arr extends []
  ? Result
  : Arr extends [infer H]
  ? H extends Elem
    ? Result & MapObject<H, "name", "options">
    : never
  : Arr extends readonly [infer H, ...infer Tail]
  ? Tail extends ReadonlyArray<Elem>
    ? H extends Elem
      ? Mapper<Tail, Result & MapObject<H, "name", "options">>
      : never
    : never
  : never;

type Result = Mapper<Data>[] extends ExpectedType ? true : false;
```

## 12. Transform union to array - [link](https://stackoverflow.com/questions/65533827/get-keys-of-an-interface-in-generics/65534971#65534971)

Let's say you have a `Union`, and you want to convert it to `ExpectedArray`

```typescript
type Union = "one" | "two" | "three";

type ExpectedArray = ["one", "two", "three"];
```

There is a naive way to do it:

```typescript
type Result = Union[]; // ('one' | 'two' | 'three')[]

type Test1 = Union extends ["one", "one", "one"] ? true : false; // true
```

As you see, it is not what we are looking for.

```typescript
//Credits goes to https://stackoverflow.com/questions/50374908/transform-union-type-to-intersection-type/50375286#50375286
type UnionToIntersection<U> = (U extends any ? (k: U) => void : never) extends (
  k: infer I
) => void
  ? I
  : never;

// Credits goes to ShanonJackson https://github.com/microsoft/TypeScript/issues/13298#issuecomment-468114901
// Converts union to overloaded function
type UnionToOvlds<U> = UnionToIntersection<
  U extends any ? (f: U) => void : never
>;

// Credits goes to ShanonJackson https://github.com/microsoft/TypeScript/issues/13298#issuecomment-468114901
type PopUnion<U> = UnionToOvlds<U> extends (a: infer A) => void ? A : never;

// Credit goes to Titian Cernicova-Dragomir  https://stackoverflow.com/questions/53953814/typescript-check-if-a-type-is-a-union#comment-94748994
type IsUnion<T> = [T] extends [UnionToIntersection<T>] ? false : true;

// Finally me)
type UnionToArray<T, A extends unknown[] = []> = IsUnion<T> extends true
  ? UnionToArray<Exclude<T, PopUnion<T>>, [PopUnion<T>, ...A]>
  : [T, ...A];

interface Person {
  name: string;
  age: number;
  surname: string;
  children: number;
}

type Result = UnionToArray<keyof Person>; // ["name", "age", "surname", "children"]

const func = <T>(): UnionToArray<keyof T> => null as any;

const result = func<Person>(); // ["name", "age", "surname", "children"]
```

Please keep in mind, this solution is not CPU friendly and there is no order guarantee.

# II. Advanced data structures

## 1. Bit representation of simple object

How to opack JS regular objects into bits?

```typescript
const obj = { top: true, category: 242, id: 123_456 }; // boolean and numbers, not strings
```

Constraints:
`top` - can be either 1 or 0, flag - (T)

`category` - can be from `0` to `999`, flag - (C)

`id` - can be from `1` to `1_000_000`, flag - (I)

Let's start with encoding.
How many bits we should allocate for ID's?

`1_000_000..toString(2).length` -> we should allocate 20 bits

How many bits for category?

`999..toString(2).length` -> 10

And for top? - 1 bit, because it is a boolean

TOP: 1

CATEGORY: 10

ID's: 20

Result:

T(1)-CCCCCCCCCC(10)-IIIIIIIIIIIIIIIIIIII(20)

TCCCCCCCCCCIIIIIIIIIIIIIIIIIIII -> length 31

```typescript
const id_hex = (123_456).toString(16); // 1e240 // 11110001001000000
const category_hex = (242).toString(16); // f2 // 11110010
const top_hex = (1).toString(16); // 1
const result = 0x1f21e240; // 1 - f2 - 1e240, binary representation 1 111100100 0011110001001000000
```

And decoder:

```typescript
const bits = (from, to, number) => ((1 << to) - 1) & (number >> (from - 1));
```

# III. Patterns

## 1. TypeState and Builder patterns

```typescript
interface Active {
  active: true;
  disable(): Disabled;
}

interface Disabled {
  active: false;
  activate(): Active;
}

class ConnectionActive<T> implements Active {
  active: true;
  data: T;
  constructor(data: T) {
    this.data = data;
  }

  disable = () => new ConnectionDisabled<T>(this.data);
}

class ConnectionDisabled<T> implements Disabled {
  active: false;
  data: T;
  constructor(data: T) {
    this.data = data;
  }

  activate = () => new ConnectionActive<T>(this.data);
}

const socket = { foo: 42 };

const result = new ConnectionDisabled(socket);
```

You are able to call only `activate` methods when connection is disabled and `disable` when connection is enabled.

This pattern was inspired by these 3 articles:

- [first article](https://docs.google.com/presentation/d/1po3-zRQCp8m8cwg-CF5dUL_6RPe9gIaKIT5P_DNbGE8/edit#slide=id.g6baf2c25cf_0_33)
- [second article](http://cliffle.com/blog/rust-typestate/)
- [third SO answer](https://stackoverflow.com/questions/65431379/type-property-relying-on-return-type-of-another-property/65433418#65433418)

The main goal here - is to make illegal states unrepresentable. This is always my first goal, when I'm trying to type smth.

## 2. Publish subscribe pattern

Like I said in previous chapter, main goal - is to make illegal states unrepresentable.
Please see next example. This pattern is your friend if you want to make event driven app (sockets, etc...)

```typescript
const enum Events {
  foo = "foo",
  bar = "bar",
  baz = "baz",
}

/**
 * Single sourse of true
 */
interface EventMap {
  [Events.foo]: { foo: number };
  [Events.bar]: { bar: string };
  [Events.baz]: { baz: string[] };
}

type Values<T> = T[keyof T];

// All credits goes here :
// https://stackoverflow.com/questions/50374908/transform-union-type-to-intersection-type/50375286#50375286
type UnionToIntersection<U> = (U extends any ? (k: U) => void : never) extends (
  k: infer I
) => void
  ? I
  : never;

type EmitRecord = {
  [P in keyof EventMap]: (name: P, data: EventMap[P]) => void;
};

type ListenRecord = {
  [P in keyof EventMap]: (
    name: P,
    callback: (arg: EventMap[P]) => void
  ) => void;
};

type MakeOverloadings<T> = UnionToIntersection<Values<T>>;

type Emit = MakeOverloadings<EmitRecord>;
type Listen = MakeOverloadings<ListenRecord>;

const emit: Emit = <T>(name: string, data: T) => {};

emit(Events.bar, { bar: "1" });
emit(Events.baz, { baz: ["1"] });
emit("unimplemented", { foo: 2 }); // expected error

const listen: Listen = (name: string, callback: (arg: any) => void) => {};

listen(Events.baz, (arg /* { baz: string[] } */) => {});
listen(Events.bar, (arg /* { bar: string } */) => {});
```

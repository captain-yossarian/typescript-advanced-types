# typescript-advanced-types

[1. Generic class for API requests](#1generic-class-for-api-requests)
[2. Math operations](#2math-operations)

[4. React, return component type](#4react-component-return-type)

## 1.Generic class for API requests

Let's assume that we have next allowed endpoints:

```typescript
const enum Endpoints {
    /**
     * Have only GET and POST
     */
    users = '/api/users',
    /**
     * Have only POST and DELETE
     */     
    notes = '/api/notes',  
    /**
     * Have only GET
     */
    entitlements = '/api/entitlements'
}
```
You might have noticed, that I used const enum instead of enum. This technique will reduce you code output. 
Please, keep in mind, it works bad with babel. 
Let's assume that backend developer allowed You to make:

 - GET | POST requests for users
 - POST | DELETE requests for notes
 - GET requests for entitlements
 
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

Now, we can define our main API class

```typescript
class Api {
    get = <T = void>(url: Endpoints): Promise<T> => fetch(url).then(response => response.json())
    post = (url: Endpoints) => fetch(url, { method: 'POST' })
    delete = (url: Endpoints) => fetch(url, { method: 'DELETE' })
}
```
For now, class Api does not have any constraints.
So let's define them:
```typescript
// Just helper
type RequiredGeneric<T> = T extends void
    ? { __TYPE__ERROR__: 'Please provide generic parameter' }
    : T

interface HandleHttp {
    <T extends void>(): RequiredGeneric<T>
    <T extends Endpoints.users>(): HandleUsers;
    <T extends Endpoints.notes>(): HandleNotes;
    <T extends Endpoints.entitlements>(): HandleEntitlements;
}
```
As You see, HandleHttp is just overloading for function. Nothing special except the first line. I will come back to it later.
We have class Api and overloadings for function. How we can combine them? Very simple - we will just create a function which returns instance of Api class
```typescript
const handleHttp: HandleHttp = <_ extends Endpoints>() => new Api();
```
Take a look on generic parameter of httpHandler and HandleHttp interface, there is a relation between them.
Let's test our result:
```typescript
const request1 = handleHttp<Endpoints.notes>() // only delete and post methods are allowed
const request2 = handleHttp<Endpoints.users>() // only get and post methods are allowed
const request3 = handleHttp<Endpoints.entitlements>() // only get method is allowed
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
Let's assume You want to make some math operations either on number or bigint 

Please keep in mind, this code is not valid in JS/TS: 
```typescript
const sum = 2 + 2n // Error
```
So, we want to accept only number or only bigints. Let's start with function definition:
```typescript
function sum<A extends number | bigint>(a: A, b: A){
    return a * a + b * b
}
```
Unfortunately, this function don't work as expected. Let's test it:
```typescript
const x = 3n;
let y: number | bigint;
if (Math.random() < 0.5) y = 4;
else y = 4n;

sum(x,y) // OK
```
In above case, y can be either number or bigint. So, from TS point of view it is ok, but I'willing to bet that it will throw at least 1 error in dev environment and 1K in production. It was a joke :D
Ok, what we can do?
We can define two generic parameters:
```typescript
type Numbers = number | bigint

function sumOfSquares<A extends Numbers, B extends Numbers>(a: A, b: B): Numbers {
    return a * a + b * b
};

const result = sumOfSquares(x, y) // There should be an error
const result2 = sumOfSquares(3n, 4) // There should be an error
const result3 = sumOfSquares(3, 4n) // There should be an error
const result4 = sumOfSquares(3, 4) // ok
const result5 = sumOfSquares(3n, 4n) // ok
```
Unfortunately, above example still don't work as we expect. 
```typescript
<A extends Numbers, B extends Numbers>
// same as 
<A extends number | bigint, B extends number | bigint>
// so A can be number and B in the same time can be bigint
```
Only overloadings might help us here. We should explicitly say, that B parameter should have same type as A
```typescript
type Numbers = number | bigint

function sumOfSquares<A extends number, B extends number>(a: A, b: B): number
function sumOfSquares<A extends bigint, B extends bigint>(a: A, b: B): bigint
function sumOfSquares<A extends Numbers, B extends A>(a: A, b: B): Numbers {
    return a * a + b * b
};

const result = sumOfSquares(x, y) // There should be an error
const result2 = sumOfSquares(3n, 4) // There should be an error
const result3 = sumOfSquares(3, 4n) // There should be an error
const result4 = sumOfSquares(3, 4) // ok
const result5 = sumOfSquares(3n, 4n) // ok
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

function Parent({
    children
}: {
    children: JSX.Element[]
}) {
    return (
        <div>
            {children.map((child) => (
                <div key={child.props.label}>{child}</div> // Props: any
            ))}
        </div>
    );
}

function App2() {
    return <Parent children={[<One />, <Two label={'hello'} />]} />;
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
    children
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
    return <Parent children={[<One />, <Two label={'hello'} />]} />;
}
```
Does it work - no! But, why????
Because `React.ReactElement<{label: string}>` is still union type, and in fact, it accepts (props:any)=>....
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

So , we can write now the code which will meet our requirements:
```typescript
const One: React.FC<{ label: string }> = ({label}) => {
    return <div>{label}</div>;
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

function App4() {
    return React.createElement(Parent, {
        children: [
            React.createElement(One, { label: 'd' }),
            React.createElement(Two, { label: 'd' })
        ]
    });
}
```
## 4.React component return type - [link](https://stackoverflow.com/questions/65406516/react-typescript-difference-between-react-fc-t-and-function)
There is a common pattern for typing return value of component:
```typescript
type Props = {
  label: string;
  name: string;
}

const Result: FC<Props> = (prop: Props): JSX.Element => <One label={'hello'} />

type O = ReturnType<typeof Result> // React.ReactElement<any, any> | null
```
Is it correct ? - Yes.
Is it helpful ? - Not really.
What if you need to make sure that Result component will always return component with some particular props. For example I'm interested in `{label:string}` property.
```typescript
type Props = {
  label: string;
}
type CustomReturn = React.ReactElement<Props>;

const MainButton: FC<Props> = (prop: Props): CustomReturn => <Two />
```
Unfortunately, there is no error. This code compiles.
Native React syntax comes to help!
```typescript
const Two: React.FC = () => <div></div>

type Props = {
  label: string;
}
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
}
type CustomReturn = React.ReactElement<Props>;

const One: React.VFC<{ label: string }> = ({ label }) => <div>{label} </div>;

const MainButton: FC<Props> = (props: Props): CustomReturn => 
    React.createElement(One, props);
```
Btw, small reminder, how to use generics with React components:
```typescript
import React from 'react'

type Props<D, S> = {
    data: D
    selector: (data: D) => S
    render: (data: S) => any
}


const Comp = <D, S>(props: Props<D, S>) => null

const result =
    <Comp<number, string>
        data={2}
        selector={(data: number) => 'fg'}
        render={(data: string) => 42}
    /> // ok

const result2 =
    <Comp<number, string>
        data={2}
        selector={(data: string) => 'fg'}
        render={(data: string) => 42}
    /> // expected error

const result3 =
    <Comp<number, string>
        data={2} selector={(data: number) => 'fg'}
        render={(data: number) => 42}
    /> // expected error
```

## 5.Compare arguments in TypeScript - [link](https://stackoverflow.com/questions/65361696/arguments-of-same-length-typescript)
Let's say you want to make a function with next constraints:
 - First argument should be an array
 - Second arguments should be 2D array, where each nested array has same length as first argument
Pseudocode:
```
const handleArray=(x: number[], y:number[][])=>void
```
Let's start from defining each type and some type utils.
Let's start from defining our function. This is very naive approach, we will improve it later
```typescript
type ArrayElement = number
type Array1D = ReadonlyArray<ArrayElement>

function array<
    X extends Array1D,
    Y extends readonly ArrayElement[]>
    (x: X, y: readonly Y[]) { }

const result = array([1, 2, 3], [[1, 1, 1,1,1]]) // no error, unexpected behaviour

```
So, now we should find a way to compare length of first argument with length of all inner arrays of second argument.
First, of all we should define Length util.
```typescript
/**
 * Get length of the array     
 */
type Length<T extends ReadonlyArray<any>> = 
 T extends { length: infer L } ? L : never
/**
 * There is another approach to get the length of the array
 */
type Length2<T extends ReadonlyArray<any>> = T['length']

type ArrayMock1 = readonly [1, 2, 3]
type ArrayMock2 = readonly [4, 5, 6, 7]
type ArrayMock3 = readonly [8, 9, 10]
type ArrayMock4 = readonly [[11, 12, 14]]
type ArrayMock5 = readonly [[15, 16]]
type ArrayMock6 = readonly [[1],[2],[3]]


type TestArrayLength1 = Length<ArrayMock1> // 3
type TestArrayLength2 = Length<ArrayMock2> // 4
```
Now, when we know how to get length of the array, we should create comparison util.
```typescript
/**
 * Compare length of the arrays
 */
type CompareLength<X extends ReadonlyArray<any>, Y extends ReadonlyArray<any>> = 
  Length<X> extends Length<Y> ? true : false

 
/**
 * Tests
 */
 type TestCompareLength1=CompareLength<ArrayMock1, ArrayMock2> // false 
 type TestCompareLength2=CompareLength<ArrayMock1, ArrayMock3> // true
 ```
 It is looks like we have all our necessary utils. If you still have not head ake, here You have other portion of types to think about.Try to figure out whats going on here:
 ```typescript
 {
   type Foo = {
      x: number
   }

   type FooX = {
      x: number
   }['x']

   type Result = FooX extends number ? true : false // true

   const obj = {
      y: 42
   }['y']

   obj // 42
}

/**
 * A bit complicated with weird conditional statement
 */
{
   type Foo = {
      x: number
   }

   type FooX<T> = {
      x: number;
      y: string
   }[T extends number ? 'x' : 'y']

   type Result = FooX<number> extends number ? true : false // true

   type Result2 = FooX<string> extends string ? true : false // true

   const condition = 2;

   const obj = {
      y: 42,
      x: 43
   }[condition === 2 ? 'y' : 'x']

   obj // 42
}

```
Now, when You are familiar with such a weird syntax, we can go further. Here is our  function definition with 1 overloading.
```typescript
function array<X extends Array1D, Y extends {
    0: readonly ArrayElement[]
}[CompareLength<X, Y> extends true ? 0 : never]>
(x: X, y: readonly Y[]): void;

function array<X extends Array1D, Y extends readonly ArrayElement[]>
(x: X, y: readonly Y[]) { }
```
And here is our whole code placed in one block with type tests
```typescript
type ArrayElement = number
type Array1D = ReadonlyArray<ArrayElement>

type Length<T extends ReadonlyArray<any>> =
   T extends { length: infer L } ? L : never


type CompareLength<X extends ReadonlyArray<any>, Y extends ReadonlyArray<any>> =
   Length<X> extends Length<Y> ? true : false


function array<X extends Array1D, Y extends {
   0: readonly ArrayElement[]
}[CompareLength<X, Y> extends true ? 0 : never]>
   (x: X, y: readonly Y[]): void;
function array<X extends Array1D, Y extends readonly ArrayElement[]>
   (x: X, y: readonly Y[]) { }


const arr = [1, 2, 3] as const

/**
 * No errors expected
 */
const result = array([1, 2, 3] as const, [[1, 1, 1], [1, 2, 3]] as const) // ok
const result0 = array([1, 2, 3] as const, [[1, 1, 1]] as const) // ok

/**
 * Errors expected
 */
const result1 = array([1, 2, 3], [[1, 1, 1], [1]]) // no error, but should be
const result2 = array([1, 2, 3] as const, [[1, 1, 1], [1, 2]] as const) // no error, but should be
const result3 = array([1, 2, 3] as const, [[1, 2]] as const) // error
const result5 = array([1, 2, 3] as const, [1] as const) // error
const result6 = array([1, 2, 3] as const, [[1, 2, 3], []] as const) // no error, but should be
const result7 = array(arr, [[1, 1, 1]]) // no error, but should be
```
It is look like we made logical error somewhere in the code. Ok, not you, but I made :D. 
Maybe we should test again our `Length` type util. What exactly are we expect from this util ? Answer is - literal integer. Let's test it again:
```typescript
type Length<T extends ReadonlyArray<any>> = T extends { length: infer L } ? L : never
const array1 = [1, 2, 3]

type Test1 = Length<[1, 2, 3]> // ok
type Test2 = Length<typeof array1> // not ok - number

/**
 * Q: But why TS does not complain here? 
 * I have explicitly defined that Length argument should extend ReadonlyArray
 * 
 * A: Because, literal array type extends ReadonlyArray
 * When you use [typeof array1], TS treats it as number[], because array1 is mutable. 
 * Hence TS cant infer the length of array1. It is possible only in runtime
 */
 ```
 So, we should provide extra restrictions for mutable arrays? Correct? - Yes. Let's provide them:
 ```typescript
 // Type of mutable array length will be always - number
type MutableLength = unknown[]['length'] // number

type CheckCompatibility = number extends 5 ? true : false // false
type CheckCompatibility2 = 5 extends number ? true : false // true

/**
 * This code works because  number does not extends literal 5
 * but literal 5 does extends number
 */
export type Length<T extends ReadonlyArray<any>> = T extends { length: infer L } ? MutableLength extends L ? MutableLength : L : MutableLength;

/**
 * Tests
 */
const array = [1,2,3]
const arrayImm = [1,2,3] as const

type Test1=Length<number[]> // number
type Test2=Length<unknown[]> // number
type Test3=Length<typeof array> // number
type Test4=Length<typeof arrayImm> // 3

type CompareLength<X extends ReadonlyArray<any>, Y extends ReadonlyArray<any>> =
   Length<X> extends Length<Y> ? true : false

/**
 * Let's test again CompareLength
 */
const arr1 = [1, 2, 3]
const arr2 = [1, 2]
type Test1 = CompareLength<typeof arr1, typeof arr2> // true, BANG! this is not what we are expect!
```

`CompareLength` should be rewritten as follow:
```typescript
type CompareLength<X extends ReadonlyArray<any>, Y extends ReadonlyArray<any>> =
   MutableLength extends Length<X> ? false : MutableLength extends Length<Y> ? false : Length<X> extends Length<Y> ? true : false;

/**
 * Tests
 */
const arr1 = [1, 2, 3]
const arr2 = [1, 2]
type Test1 = CompareLength<typeof arr1, typeof arr2> // false, expected
```
Ok, I'm exhausted now, it should work. Let's test our code:
```typescript
type ArrayElement = number
type Array1D = ReadonlyArray<ArrayElement>

type MutableLength = unknown[]['length'] // number

/**
 * Get length of the array
 * Allow only immutable arrays
 */
export type Length<T extends ReadonlyArray<any>> = T extends { length: infer L } ? MutableLength extends L ? MutableLength : L : MutableLength;

/**
 * Compare length of the arrays
 */
type CompareLength<X extends ReadonlyArray<any>, Y extends ReadonlyArray<any>> =
    MutableLength extends Length<X> ? false : MutableLength extends Length<Y> ? false : Length<X> extends Length<Y> ? true : false;

/**
 * CompareLength, compares length of X and filtered Y, 
 * if true - return zero index element - ReadonlyArray<ArrayElement>
 * if false - return never
 * 
 * So, if it will return never, then you can't pass second argument,
 * but if you did not provide second argument, you will receive another error - 
 * function expects two arguments
 */
function array<X extends Array1D, Y extends {
    0: readonly ArrayElement[]
}[CompareLength<X, Y> extends true ? 0 : never]>(x: X, y: readonly Y[]): 'put here your returned type'

function array<X extends Array1D, Y extends readonly ArrayElement[], Z extends CompareLength<X, Y>>(x: X, y: readonly Y[]) {
    return [1, 2, 3] as any
}
const result = array([1, 2, 3] as const, [[1, 1, 1], [1, 2, 3]] as const) // ok
const result0 = array([1, 2, 3] as const, [[1, 1, 1]] as const) // ok

const arr = [1, 2, 3] as const

const result1 = array([1, 2, 3], [[1, 1, 1], [1]]) // error
const result2 = array([1, 2, 3] as const, [[1, 1, 1], [1, 2]] as const) // no error, but SHOULD BE
const result3 = array([1, 2, 3] as const, [[1, 2]] as const) // error
const result5 = array([1, 2, 3] as const, [1] as const) // error
const result5 = array([1, 2, 3] as const, [[1, 2, 3], []] as const) // error
const result6 = array(arr, [[1, 1, 1]]) // error, because TS is unable to fidure out length of mutable array. 
```
Ohh, what a .... What's going on here ? We still need to fix one failed test, see `result3`. It looks like if second arguments contains at least one array which feet requirements, TS ok with it.  So we should compare arrays only if their length are equal.
```typescript
/**
 * Check if all inner arrays have same length as X
 */
type Filter<X extends ReadonlyArray<any>, Y extends ReadonlyArray<any>> =
    X['length'] extends Y['length']
    ? Y['length'] extends X['length'] ? Y : never : never

type Test1 = Filter<[1, 2, 3], [1, 2]> // never
type Test2 = Filter<[], [1, 2]> // never
type Test3 = Filter<[1, 2], [1, 2]> // [1,2]

/**
 * Please keep in mind that Y can be and will be a union type of inner arrays
 * 
 */
type Test4 = Filter<[], [1, 2] | [1, 2, 3]> // never
type Test5 = Filter<[1, 2], [1] | [1, 2] | [1, 2, 3]> // never

/**
 * Btw, try to rewrite Filter without second conditional
 */
type Filter<X extends ReadonlyArray<any>, Y extends ReadonlyArray<any>> =
    X['length'] extends Y['length'] ? Y : never;
    
type Test5 = Filter<[1, 2], [1] | [1, 2] | [1, 2, 3]> // not never, not ok
/**
 * Why ?
 */
type O = 5 extends 1 | 2 | 5 ? true : false // true
type O2 = 1 | 2 | 5 extends 5 ? true : false // false
```
Finall working code:
```typescript
type ArrayElement = number
type Array1D = ReadonlyArray<ArrayElement>
type MutableLength = unknown[]['length'] // number

export type Length<T extends ReadonlyArray<any>> = T extends { length: infer L } ? MutableLength extends L ? MutableLength : L : MutableLength;

type CompareLength<X extends ReadonlyArray<any>, Y extends ReadonlyArray<any>> =
    MutableLength extends Length<X> ? false : MutableLength extends Length<Y> ? false : Length<X> extends Length<Y> ? true : false;

type Filter<X extends ReadonlyArray<any>, Y extends ReadonlyArray<any>> =
    X['length'] extends Y['length']
    ? Y['length'] extends X['length'] ? Y : never : never

function array<X extends Array1D, Y extends {
    0: readonly ArrayElement[]
}[CompareLength<X, Filter<X, Y>> extends true ? 0 : never]>(x: X, y: readonly Y[]): 'put here your returned type'

function array<X extends Array1D, Y extends readonly ArrayElement[], Z extends CompareLength<X, Y>>(x: X, y: readonly Y[]) {
    return [1, 2, 3] as any
}

/**
 * Positive Tests
 */
const result = array([1, 2, 3] as const, [[1, 1, 1], [1, 2, 3]] as const) // ok
const result0 = array([1, 2, 3] as const, [[1, 1, 1]] as const) // ok

const arr = [1, 2, 3] as const

/**
 * Negative Tests
 */
const result1 = array([1, 2, 3], [[1, 1, 1], [1]]) // error
const result2 = array([1, 2, 3] as const, [[1, 1, 1], [1, 2]] as const) // error
const result3 = array([1, 2, 3] as const, [[1, 2]] as const) // error
const result4 = array([1, 2, 3] as const, [[1], [1, 2], [1, 2, 3]] as const) // error
const result5 = array([1, 2, 3] as const, [1] as const) // error
const result6 = array([1, 2, 3] as const, [[1, 2, 3], []] as const) // error
const result7 = array(arr, [[1, 1, 1]]) // error, because TS is unable to figure out length of mutable array. 
```

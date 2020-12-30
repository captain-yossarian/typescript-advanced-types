# typescript-advanced-types

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

[![Build Status](https://travis-ci.org/microstates/microstates.js.svg?branch=master)](https://travis-ci.org/microstates/microstates.js)
[![Coverage Status](https://coveralls.io/repos/github/microstates/microstates.js/badge.svg?branch=master)](https://coveralls.io/github/microstates/microstates.js?branch=master)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Gitter](https://badges.gitter.im/microstates/microstates.js.svg)](https://gitter.im/microstates/microstates.js?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

# Microstates

* [Intro](#intro)
* [microstate constructor](#microstate-constructor)
* [Composition](#composition)
* [Static values](#static-values)
* [Computed Properties](#computed-properties)
* [Transitions](#transitions)
* [Batch Transitions](#batch-transitions)
* [Changing structure](#changing-structure)
* [Parameterized Arrays & Objects](#parameterized-arrays--objects)
* [Observable interoperability](#observable-interoperability)
* [Built-in types](#built-in-types)
  * [`Boolean`](#boolean)
    * [set(value: any) => microstate](#setvalue-any--microstate)
    * [toggle() => microstate](#toggle--microstate)
  * [`Number`](#number)
    * [set(value: any) => microstate](#setvalue-any--microstate-1)
    * [increment(step: Number = 1) => microstate](#incrementstep-number--1--microstate)
    * [decrement(step: Number = 1) => microstate](#decrementstep-number--1--microstate)
  * [`String`](#string)
    * [set(value: any) => microstate](#setvalue-any--microstate-2)
    * [concat(str: String [, str1: String]) => microstate](#concatstr-string--str1-string--microstate)
  * [`Array`](#array)
    * [set(value: any) => microstate](#setvalue-any--microstate-3)
    * [push(value: any) => microstate](#pushvalue-any--value1-any--microstate)
    * [pop() => microstate](#pop--microstate)
    * [shift() => microstate](#shift--microstate)
    * [unshift(value: any) => microstate](#unshift-any--value1-any--microstate)
    * [filter(fn: value => boolean) => microstate](#filterfn-value--boolean--microstate)
    * [map(fn: (microstate, index) => any) => microstate](#mapfn-microstate-index--any--microstate)
    * [clear() => microstate](#clear--microstate)
  * [`Object`](#object)
    * [set(value: any) => microstate](#setvalue-any--microstate-4)
    * [assign(object: Object) => microstate](#assignobject-object--microstate)
    * [put(key: string, value: any) => microstate]()
    * [delete(key: string) => microstate]()
* [FAQ](#faq)
  * [What if I can't use class syntax?](#what-if-i-cant-use-class-syntax)
  * [What if I can't use Class Properties?](#what-if-i-cant-use-class-properties)
* [Run Tests](#run-tests)

## Intro

Microstates are a _typed_, _composable_ and _immutable_ state container.

Let's get some terminology out of the way so we're on the same page for the rest of the README.

* _Typed_ means that microstates know about the kind of data it contains and transitions that can be
  performed on that data.
* _Composable_ means that simple types can be grouped to describe complex data structures.
  Microstates knows how to transition complex data structures in an immutable way.
* _Immutable_ means that state in the microstate cannot be modified. To change the state, you must invoke a transition.
* _Transition_ is an operation that has access to the current state, might perform computations/transformations on it and returns a new state.

To add microstates to your project,

```bash
npm install --save microstates

# yarn add microstates
```

Let's see some code,

```js
import { create } from 'microstates';

create(Number, 42).increment().state;
//=> 43

class MyCounter {
  count = Number;
}

// creating microstate from composed types
create(MyCounter).state;
// => { count: 0 }

create(MyCounter).count.increment().state;
// => { count: 1 }

class MyModal {
  isOpen = Boolean;
  title = String;
}

// you can restore a microstate from previous value
create(MyModal, { isOpen: true, title: 'Hello World' }).state;
// => { isOpen: true, title: 'Hello World' }

create(MyModal, { isOpen: true, title: 'Hello World' }).isOpen.toggle().state;
// => { isOpen: false, title: 'Hello World' }

// lets compose multiple types together
class MyType {
  modal = MyModal;
  counter = MyCounter;
}

create(MyType).state;
// => {
//  modal: { isOpen: false, title: '' },
//  counter: { count: 0 }
// }

create(MyType)
  .counter.increment()
  .modal.title.set('Hello World')
  .modal.isOpen.set(true).state;
// => {
//  modal: { isOpen: true, title: 'Hello World' },
//  counter: { count: 1 }
// }
```

## microstate constructor

Microstate constructor creates a microstate. It accepts two arguments, `Type` class that describes the structure of your state and `value` that is the initial state.

```js
// initial value is undefined so state will be default value of String which is an empty string
create(String).state;
// => ''

// initial value is 'hello world' so state will be hello world.
create(String, 'hello world').state;
// => 'hello world'
```

The object returned from the constructor has all of the transitions for the structure.

```js
import { create } from 'microstates';

let ms = create(Object);

console.log(ms);
//  {
//    assign: Function
//    set: Function
//  }

ms.assign({ hello: 'world', hi: 'there' }).state;
// => { hello: 'world', hi: 'there' }
```

## Composition

Microstates are designed to be composable. You can define custom microstate types and nest them as 
necessary. Microstates will take care of ensuring that all transitioning composed microstates is
done immutably.

Defining custom types is done using [ES2016 classes syntax](http://exploringjs.com/es6/ch_classes.html). In JavaScript, classes are functions. So
any function can work as a type, but we recommend using class syntax.

```js
class Counter {
  count = Number;
}

create(Counter);
// => {
//      merge: Function
//      set: Function
//      count: {
//        increment: Function
//        decrement: Function
//        sum: Function
//        subtract: Function
//      }
//    }

Microstate.create(Counter).state;
// => { count: 0 }
```

We use Class Properties syntax to define the composition of microstates. If you're using Babel, you need the
[Class Properties transform](https://babeljs.io/docs/plugins/transform-class-properties/) for Class
Properties to work.

```js
class Person {
  name = String;
  age = Number;
}

create(Person);
// => {
//      name: { concat: Function, set: Function },
//      age: { increment: Function, decrement: Function, sum: Function, subtract: Function }
//    }

// state objects maintain their type's class
create(Person).state instanceof Person;
// => true

create(Person).state;
// => { name: '', age: 0 }
```

You can restore a composed microstate by providing initial value that matches the structure of the
microstate.

```js
create(Person, { name: 'Taras', age: 99 }).state;
// => { name: 'Taras', age: 99 }
```

You can compose custom types into other custom types.

```js
class Address {
  street = String;
  number = Number;
  city = String;
}

class Person {
  name = String;
  age = Number;
  address = Address;
}

create(Person, { name: 'Taras', address: { city: 'Toronto' } }).state;
// => {
//      name: 'Taras',
//      age: 0,
//      address: {
//        street: '',
//        number: 0,
//        city: 'Toronto'
//      }
//    }
```

You can access transitions of composed states by using object property notation. Every time that you
call a transition, you receive a new microstate and you start from the root of the original type.

```js
create(Person)
  .age.increment()
  .address.city.set('San Francisco')
  .address.street.set('Market St').state;
// => {
//      name: 'Taras',
//      age: 1,
//      address: {
//        street: 'Market St',
//        number: 0,
//        city: 'San Francisco'
//      }
//    }
```

The objects can be of any complexity and can even support recursion.

```js
class Person {
  name = String;
  father = Person;
}

create(Person, { name: 'Stewie' })
  .father.name.set('Peter')
  .father.father.name.set('Mr Griffin')
  .father.father.father.name.set('Mr Giffin Senior').state;
// => { name: 'Stewie',
//      father: {
//        name: 'Peter',
//        father: {
//          name: 'Mr Griffin,
//          father: {
//            name: 'Mr Griffin Senior'
//          }
//        }
//      }
//    }
```

We currently do not support composition inside of arrays but we're looking into supporting it in the
future.

Composed states have two default transitions `set` and `merge`.

`set` will replace the state of the current microstate.

```js
create(Person).father.father.set({ name: 'Peter' }).state;
// => { name: '', father: { name: 'Peter' }}
```

`merge` will recursively merge the object into the current state.

```js
create(Person, { name: 'Peter' }).merge({ name: 'Taras', father: { name: 'Serge' } }).state;
// { name: 'Taras', father: { name: 'Serge' }}
```

## Static values

You can define static values on custom types. Static values are added to state but do not get
transitions.

```js
class Ajax {
  content = null;
  isLoaded = false;
}

create(Ajax).state.content;
// => null;

create(Ajax).content;
// undefined

create(Ajax).state.isLoaded;
// => false

create(Ajax).isLoaded.set(false);
// Error: calling set of undefined
```

## Computed Properties

Composed states can have computed properties. Computed Properties make it possible define properties
that derive their values from the state. Use getter syntax to define computed properties on composed
states.

```js
class Measure {
  length = Number;
  get inInches() {
    return `${this.height / 2.54} inches`;
  }
}

create(Measure, { length: 170 }).state.inInches;
// => '66.9291 inches'

create(Measure, { length: 170 }).height.set(160).state.inInches;
// => '62.9921 inches'
```

## Transitions

You can define transitions on custom types. Inside of transitions, you have access to
current state and transition local state.

```js
class Person {
  home = String;
  location = String;
  get isAtHome() {
    return this.home === this.location;
  }
  goHome() {
    let { home, location, isAtHome } = this.state;
    if (!isAtHome) {
      return this.location.set(home);
    } else {
      return this.state;
    }
  }
  goSomewhere(newLocation) {
    return this.location.set(newLocation);
  }
}

create(Person, { home: 'Toronto', location: 'San Francisco' }).goHome().state;
// => { home: 'Toronto', location: 'Toronto', isAtHome: true }

create(Person, { home: 'Toronto', location: 'San Francisco' }).goSomewhere('Bangalore').state;
// => { home: 'Toronto', location: 'Bangalore', isAtHome: false }
```

## Batch Transitions

Transitions can be used to perform multiple transformations in sequence. This is useful when
you have a deeply nested microstate and you're applying several transformations to one branch of the
microstate. All of the operations we'll perform before the transformation is complete.

```js
class MyModal {
  isOpen = Boolean;
  title = String;
  content = String;

  show(title, content) {
    return this
      .isOpen.set(true),
      .title.set(title)
      .content.set(content)
  }
}

class MyComponent {
  modal = MyModal;
  counter = Number
}

create(MyComponent).modal.show('Hello World', 'Rise and shine!').state;
// => { modal: { isOpen: true, title: 'Hello World', content: 'Rise and shine!' }, counter: 0 }
```

## Changing structure

When modeling state machines, it's often helpful to be able to initialize into a particular
state based on value. This can be accomplished by adding a static `create` method. This method
should return the next state that you want the node to initialize into.

```js
import { create } from 'microstates';

class Session {
  content = null;

  static create(session) {
    if (session) {
      return create(AuthenticatedSession, session);
    } else {
      return create(AnonymousSession);
    }
  }
}
```

Once your state is initialized, you can return new microstates from transition to transition
time of the node that the transition is attached to.

```js
class AuthenticatedSession extends Session {
  isAuthenticated = true;
  content = Object;

  logout() {
    return create(AnonymousSession);
  }
}

class AnonymousSession extends Session {
  content = null;
  isAuthenticated = false;
  authenticate(user) {
    return create(AuthenticatedSession, { content: user });
  }
}

class MyApp {
  session = Session;
}

// without initial state it initializes into AnonymousSession state
create(MyApp).state;
// => { session: { content: null, isAuthenticated: false }}

// transition to AuthenticatedSession state with authenticate
create(MyApp).authenticate({ name: 'Taras' }).state;
// => { session: { content: { name: 'Taras' }, isAuthenticated: true }};

// restore into AuthenticatedSession state
create(MyApp, { session: { content: { name: 'Taras' } } }).state;
// => { session: { content: { name: 'Taras' } }, isAuthenticated: true }

// transition to AnonymousSession state with logout
create(MyApp, { session: { content: { name: 'Taras' } } }).logout().state;
// => { session: { content: null, isAuthenticated: false }}
```

## Parameterized Arrays & Objects

It's often useful to be able to indicate that an array consists of a certain type of items, 
for example, an array of todo items. Microstates provides a special syntax for this.

```js
class Todo {
  title = String
  isCompleted = Boolean
}

class TodoApp {
  todos = [Todo]
}
```

This tells Microstates that we're composing an array of Todo items. Microstates will use this information to generate state 
and transitions for items in the array.

```js
let today = create(TodoApp, { todos: [{ title: 'Buy milk' }] }).state;
// => { todos: [ Todo { title: 'Buy milk', isCompleted: false } ] }
``` 

It will also provide you with transitions array with objects for each data item. You can invoke,
transitions from this array and they will behave as all other transitions in microstates.

```js
today.todos[0].title.set('Buy 2% milk').state;
// => { todos: [ Todo { title: 'Buy 2% milk', isCompleted: false } ] }

today.todos.push({ title: 'Review PRs' }).state
// => { todos: [ Todo { title: 'Buy 2% milk', isCompleted: false }, Todo { title: 'Review PRs', isCompleted: false } ] }
```

You can do the same with Objects using `{Type}` syntax. 

```js
let today = create({Todo}, { t1: { title: 'Buy milk' }, t2: { title: 'Review PRs' } }).state
// => { t1: Todo { title: 'Buy milk', isComplete: false }, t2: Todo { title: 'Review PRs', isComplete: false } }

today.t2.isComplete.toggle().state
// => { t1: Todo { title: 'Buy milk', isComplete: false }, t2: Todo { title: 'Review PRs', isComplete: true } }
```

## Observable interoperability

You can create an observable from any microstate using the `Observable.from` method.
The resulting Observable will stream the next microstate for every transition.
When you subscribe to the observable microstate, you'll immediately receive a microstate through the stream.
This microstate will have transitions that you can call to cause the next microstate to come through the stream.

```js
import { create } from 'microstates';
import { Observable } from 'rxjs';

let ms = create(Number, 42);
let observable = Observable.from(ms);

let last;

observable.subscribe(microstate => {
  last = microstate;
  console.log(microstate.state);
});
// => 42

last.increment().increment().increment();
// => 43
// => 44
// => 45
```

## Built-in types

Microstates package provides base building blocks for your composing your own types. `Number`, `String`, `Boolean`,
`Object` and `Array` come with predefined transitions.

### `Boolean`

`Boolean` type presents a `true` or `false` value. `Boolean` has `toggle` and `set` transitions.

#### set(value: any) => microstate

Returns a new microstate with boolean value replaced. Value will be coerced with
`Boolean(value).valueOf()`.

```js
Microstate.create(Boolean).set(true).state;
// => true
```

#### toggle() => microstate

Returns a new microstate with state of boolean value switched to the opposite.

```js
Microstate.create(Boolean).state;
// => false

Microstate.create(Boolean).toggle().state;
// => true;

Microstate.create(Boolean, true).toggle().state;
// => false;
```

### `Number`

`Number` type represents any numeric value. `Number` has `sum`, `subtract`, `increment`, `decrement` and `set` transitions.

#### set(value: any) => microstate

Replaces current state with value. The value will be coerced same as `Number(value).valueOf()`.

```js
Microstate.create(Number).set(10).state;
// => 10
```

#### sum(number: Number [, number: Number]) => microstate

Returns a new microstate with the sum of passed in values with the current state.

```js
Microstate.create(Number).sum(5, 10).state;
// => 15
```

#### subtract(number: Number, [, number: Number]) => microstate

Returns a new microstate with the difference between the current state and the passed in values.

```js
Microstate.create(Number, 42).subtract(2, 10).state;
// => 30
```

#### increment(step: Number = 1) => microstate

Returns a new microstate with the current state incremented by the passed in step value (defaults to 1).

```js
Microstate.create(Number).increment().state;
// => 1

Microstate.create(Number).increment(5).state;
// => 5

Microstate.create(Number)
  .increment(5)
  .increment().state;
// => 6
```

#### decrement(step: Number = 1) => microstate

Returns a new microstate with current state value decreased by passed in step value (defaults to 1).

```js
Microstate.create(Number).decrement().state;
// => -1

Microstate.create(Number).decrement(5).state;
// => -5

Microstate.create(Number)
  .decrement(5)
  .decrement().state;
// => -6
```

### `String`

`String` represents string values. `String` has `concat` and `set` transitions. You can vote for additional transitions to be included in #27.

#### set(value: any) => microstate

Replaces the current state with the passed in value and returns a new microsate with the new state. The value will be coerced using
`String(value).valueOf()`.

```js
Microstate.create(String).set('hello world').state;
// => 'hello world'
```

#### concat(str: String [, str1: String]) => microstate

Combines the current state with passed in string and returns a new microstate containing the new state.

```js
Microstate.create(String, 'hello ').concat('world').state;
// => 'hello world'
```

### `Array`

Represents an indexed collection of iterable instances of other types.

#### set(value: any) => microstate

Replaces state with passed in value and returns a new microstate containing the new state. The value will be coerced with
`Array(value).valueOf()`

```js
Microstate.create(Array).set('hello world');
// ['hello world']
```

#### push(value: any [, value1: any]) => microstate

Pushes value to the end of the array and returns a new microstate with the new array as its state.

```js
Microstate.create(Array).push(10, 15, 25).state;
// => [ 10, 15, 25 ]

Microstate.create(Array, ['a', 'b'])
  .push('c')
  .push('d').state;
// => [ 'a', 'b', 'c', 'd' ]
```

#### filter(fn: value => boolean) => microstate

Applies filter fn to every element in the array and return a new microstate with the result as the state.

```js
Microstate.create(Array, [10.123, 1, 42, 0.01]).filter(value => Number.isNumber(value)).state;
// => [ 1, 42 ];
```

#### map(fn: (value, index) => any) => microstate

Maps every item in the array and returns a new microstate with the new array as the state.

```js
Microstate.create(Array, ['a', 'b', 'c']).map(v => v.toUpperCase()).state;
// => ['A', 'B', 'C']
```

#### replace(item: any, replacement; any) => microstate

Replaces the first occurrence of `item` in the array with `replacement` using exact(`===`) comparison.

```js
Microstate.create(Array, ['a', 'b', 'c']).replace('b', 'B').state;
// => [ 'a', 'B', 'c' ]
```

### `Object`

Represents a collection of values keyed by strings. Object types have `assign` and `set` transitions.

```js
Microstate.create(Object).state;
// => {}
```

#### set(value: any) => microstate

Replaces state with the passed in value and return a new microstate with the new state. The value will be coerced to
object with `Object(value).valueOf()`.

```js
Microstate.create(Object).set({ hello: 'world' }).state;
// => { hello: 'world' }
```

#### assign(object: Object) => microstate

Creates a new object and copy values from existing state and passed in object. Return a new microstate with the new state.

```js
Microstate.create(Object, { color: 'red' }).assign({ make: 'Honda' }).state;
// => { color: 'red', make: 'Honda' }
```

## FAQ

### What if I can't use class syntax?

Classes are functions in JavaScript, so you should be able to use a function to do most of the same
things as you would with classes.

```js
class Person {
  name = String;
  age = Number;
}
```

☝️ is equivalent to 👇

```js
function Person() {
  this.name = String;
  this.age = Number;
}
```

### What if I can't use Class Properties?

Babel compiles Class Properties into class constructors. If you can't use Class Properties, then you
can try the following.

```js
class Person {
  constructor() {
    this.name = String;
    this.age = Number;
  }
}

class Employee extends Person {
  constructor() {
    super();
    this.boss = Person;
  }
}
```

## Run Tests

```shell
$ npm install
$ npm test
```

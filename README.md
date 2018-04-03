# Private methods and fields for JavaScript.

Stage 0

The following proposes a vision for private fields that conforms to the harmony of a JavaScript prototype based paradigm.

## Why

Consider the following class.

```js
class A {
  constructor() {
    this.id = Symbol('unique')
  }
  equal(instance, property) {
    return this[property] == other[property]
  }
}

const x = new A()

x.equal(x, 'id')
```

Both active proposals that make use of the [.#](https://github.com/tc39/proposal-class-fields) and [->](https://github.com/zenparsing/js-classes-1.1) sigils suffer from the same inability to implement this design pattern.

For example given the [.#](https://github.com/tc39/proposal-class-fields) sigil, we might try to implement it in the following way, but the lack of computed access prevents us from achieving this goal.

```js
class A {
  constructor() {
    this.#id = Symbol('unique')
  }
  get(instance, property) {
    // how do I get the computed private properties of another instance of A?
    return this.#id == instance.#id
  }
}

const x = new A()

x.equal(x, 'id')
```

The same problem affects the [->](https://github.com/zenparsing/js-classes-1.1) sigil.

```js
class A {
  constructor() {
    this->id = Symbol('unique')
  }
  get(instance, property) {
    // same problem, how do I get the computed private properties of another instance of X?
    return this->id == instance->id
  }
}

const x = new A()

x.equal(x, 'id')
```

## What

This proposal aims to achieve these requirements without introducing an asymmetrical new syntax for private fields, with the additional goal of not restricting the future of private properties to class syntax in consideration of JavaScripts primary roots within a prototype-based paradigm.

The afor-mentioned affected pattern would be implemented in following:

```js
class A {
  private id = Symbol('unique')
  equal(instance, property) {
    return private(this)[property] == private(instance)[property]
  }
}

const x = new A()

x.equal(x, 'id')
```

This introduces the use of the reserved keyword `private` that acts symmetrical to the use of `super` or `import` where it is both a syntax i.e `private[property]` and function `private(this)[property]`.

This means that `private(this)[property]` is equivalent to `private[property]`. 

The following examples demonstrates how this might look and work with other active/future proposals.

```js
class A {
  // uses the private keyword to create private fields, methods, getters etc.
  private value = 'string' // [1]
  private method () {} // [2]

  // uses the static keyword to create static fields, methods, getters etc
  static value = 'string' // [3]
  static method () {} // [4]

  // uses neiher to create instance fields, methods, getters etc
  value = 'string' // [5]
  method () {} // [6]

  constructor() {
    // invoke instance method
    this['method'](this.value) // [7]

    // use private.name or private['name'] to access private fields, methods, getters etc.
    private['method'](private['value']) // [8]

    // all of the following invocations are equivalent to the previous
    private.method(private.value)
    private(this)['method'](private(this)['value'])
    private(this).method(private(this).value)

    // assign private values
    private.length = 1 // [9]
    private['length'] = 1

    static['method'](static['value']) // [10]
    static.method(static.value)
  }
}
```

This has the additional benefit of allowing future proposals to introduce something akin to private symbols that translates the above class syntax to an equivelent prototype-based constructor design.

```js
var PrivateSymbolForValue = Symbol.private('value')
var PrivateSymbolForMethod = Symbol.private('method')
var PrivateSymbolForLength = Symbol.private('length')

function A () {
  this[PrivateSymbolForValue] = 'string' // [1]
  this.value = 'string' // [5]
  this['method']() // [7]
  this[PrivateSymbolForMethod](this[PrivateSymbolForValue]) // [8]
  this[PrivateSymbolForLength] = 1 // [9]
  this.constructor['method'](this.constructor['value']) // [10]
}

A.prototype[PrivateSymbolForMethod] = function () {} // [2]
A.prototype.method = function () {} // [6]

A.value = 'string' // [3]
A.method = function () {} // [4]
```

## Related

- [Yet another approach to a more JS-like syntax](https://github.com/tc39/proposal-private-methods/issues/28)
- [private and protected are like static and super](https://github.com/tc39/proposal-class-fields/issues/90)
- [proposal-about-private-symbol](https://esdiscuss.org/topic/proposal-about-private-symbol)

# Private methods and fields for JavaScript.

Stage 0

Champion Needed

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

This proposal aims to achieve these requirements without introducing an asymmetrical new syntax for private fields.

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

## FAQ

>What does private(this)[property] do?

"private(this)[property]" and alternatively "private[property]" or "private.property" all invoke access of a private "property" on the instance of the class, symmetrical to the syntax/function nature of both the "super" and "import" keywords.

>What's private about private fields?

Outside of a private fields provider class, private fields/methods would not be accessible.

>How do you prevent them from being forged or stuck onto unrelated objects?

Given the following:

```js
class A {
  private id = 0;
  private method(value) {
    return value;
  }
  write(value) {
    private(this)["id"] = private["method"](value);
  }
}
```

and then invoking the above write method with a "this" value that is not an instance of A, for example `(new A()).write.call({}, 'pawned');`, would fail – the private syntax call site is scoped to the surrounding provider class. For example imagine the current possible transpilation of this with WeakMaps:

```js
(function (){
  var registry = WeakMap()

  function A () {
    registry.set(this, {id: 0})
  }
  A.prototype.write: function () {
    registry.get(this)["id"] = registry.get(this.constructor)["method"].call(this, value)
  }

  // shared(i.e private methods)
  registry.set(A, {
    method: function (value) {
      return value
    }
  })

  return A
})()
```

Trying to do the the afore-mentioned forge here would currently fail along the lines of cannot read property "id" of  "undefined".

> An instance has a fixed set of private fields which get created at object creation time.

The implications of this alternative do not limit the creation of private fields to creation time, for example writing to a private field in the constructor or at any arbitrary time within the lifecycle of the instance.

```js
class HashTable {
  constructor() {
    private[Symbol.for('length')] = 0
  }
  set(key, value) {
    private[key] = value
  }
  get(key) {
    return private[key]
  }
}
```

>That would contradict your previous answer to the hijacking question. In the transpilation you created the field using "registry.set(this, {id: 0})", in the constructor.  If you then claim that any write to the field can also create it, then you get the hijacking behavior which you wrote doesn't happen.

The difference between the following

```js
class A {
  private id = 0
}
```

and the following

```js
class A {
  constructor() {
    private.id = 0
  }
}
```

is similar to the difference between

```js
(function (){
  var registry = WeakMap()

  function A () {
    registry.set(this, {id: 0})
  }

  return A
})()
```

and

```js
(function () {
  var registry = WeakMap()

  function A () {
    registry.set(this, {})
    registry.get(this)["id"] = 0
  }

  return A
})
```

This in no way permits the hijacking behavior previously mentioned – `(new A()).write.call({}, 'pawned')`

>Do you limit classes to creating only the private fields declared in the class, or can they create arbitrarily named ones?

Just as you could write to arbitrary named fields with the mentioned WeakMap approach, you can also do the same for this alternative, for example –

```js
private[key] = value
// or
private(this)[key] = value
```

## Related

- [Yet another approach to a more JS-like syntax](https://github.com/tc39/proposal-private-methods/issues/28)
- [Private and Protected are like Static and Super](https://github.com/tc39/proposal-class-fields/issues/90)
- [Proposal About Private Symbols](https://esdiscuss.org/topic/proposal-about-private-symbol)
- [Related Esdiscuss Thread](https://esdiscuss.org/topic/ecmascript-proposal-private-methods-and-fields-proposals)

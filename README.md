# Private methods and fields for JavaScript.

Stage 0

Champion Needed

The following proposes a vision for private fields that conforms to the harmony of JavaScripts current syntax synergy.

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
  private value = 'string'
  private method () {}

  // uses the static keyword to create static fields, methods, getters etc
  static value = 'string'
  static method () {}

  // uses neiher to create instance fields, methods, getters etc
  value = 'string'
  method () {}

  constructor() {
    // invoke instance method
    this['method'](this.value)

    // use private.name or private['name'] to access private fields, methods, getters etc.
    private['method'](private['value'])

    // all of the following invocations are equivalent to the previous
    private.method(private.value)
    private(this)['method'](private(this)['value'])
    private(this).method(private(this).value)

    // assign private values
    private.length = 1
    private['length'] = 1

    // future facing
    static['method'](static['value'])
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
  private id = 0
  private method(value) {
    return value
  }
  write(value) {
    private(this)["id"] = private["method"](value)
  }
}
```

and then invoking the above write method with a `this` value that is not an instance of `A`, for example `(new A()).write.call({}, 'pawned');`, would fail – the private syntax call site is scoped to the surrounding provider class. For example imagine the current possible transpilation of this using WeakMaps:

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

Trying to do the the afore-mentioned forge here would currently fail along the lines of cannot read property `id` of  `undefined`.

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

>What does static['method'] and static.method refer to?

This hints to a future facing proposal for static fields/methods that can share the same syntax symmetry of the proposed private fields/methods.

## Related

- [Yet another approach to a more JS-like syntax](https://github.com/tc39/proposal-private-methods/issues/28)
- [Private and Protected are like Static and Super](https://github.com/tc39/proposal-class-fields/issues/90)
- [Proposal About Private Symbols](https://esdiscuss.org/topic/proposal-about-private-symbol)
- [Related Esdiscuss Thread](https://esdiscuss.org/topic/ecmascript-proposal-private-methods-and-fields-proposals)

## Syntax Synergy

Syntax synergy is a strong contributing factor to the introduction of this alternative and a much stronger contributing factor in the community push back against a `#sigil` direction to private fields/methods. In demonstration, the following is non-exhaustive collection documenting issues/disagreement/outlets expressed with the current `#sigil` syntax.

<details>

<summary>Details</summary>

#### Github

- [Why not use the "private" keyword, like Java or C#?](https://github.com/tc39/proposal-private-fields/issues/14)
- [This proposal does not address the actually existing needs for private fields](https://github.com/tc39/proposal-private-methods/issues/22)
- [Yet another approach to a more JS-like syntax](https://github.com/tc39/proposal-private-methods/issues/28)
- [Proposal: keyword to replace `#` sigil](https://github.com/tc39/proposal-class-fields/issues/56)
- [Please do not use "#"](https://github.com/tc39/proposal-class-fields/issues/77)
- [Private and Protected are like Static and Super](https://github.com/tc39/proposal-class-fields/issues/90)
- [New, more JS-like syntax](https://github.com/tc39/proposal-private-methods/issues/20)
- [Stop this proposal](https://github.com/tc39/proposal-private-methods/issues/10)
- [Why not use private keyword instead of #?](https://github.com/tc39/proposal-private-methods/issues/8)
- [Independent private field](https://github.com/tc39/proposal-private-methods/issues/12)
- [Syntax change suggestion](https://github.com/tc39/proposal-private-methods/issues/24)
- [Yes, the negative reaction of #priv is worse, I think it's the worst in our history. I'll try to explain it.](https://github.com/zenparsing/js-classes-1.1/issues/26#issuecomment-374171662)
- [Hash -> Underscore](https://github.com/tc39/proposal-private-fields/issues/100)
- [Can we just use private keyword?](https://github.com/tc39/proposal-private-fields/issues/92)
- [Is this really needed if we need a sigil](https://github.com/tc39/proposal-private-fields/issues/61)
- [Accessing private fields](https://github.com/tc39/proposal-private-fields/issues/50)
- [Use `private.x` syntax for referencing private fields](https://github.com/tc39/proposal-private-fields/issues/18)
- [Unrelated but the syntax looks really weird.](https://github.com/prettier/prettier/pull/2837#discussion_r139260878)
- [This is by far my least favourite proposal that has advanced through tc39.](https://github.com/prettier/prettier/pull/2837#issuecomment-329939107)
- [Why not use obj#prop instead obj.#prop](https://github.com/tc39/proposal-private-fields/issues/39)
- [Use this#prop instead this.#prop](https://github.com/tc39/proposal-private-fields/issues/35)

#### Twitter

- [Not a fan of # for this, would prefer something with clearer semantics. # looks like bash comments or hashtags.](https://twitter.com/unthunk/status/913271895620857856)
- [Agreed - I would prefer the keyword `private`.](https://twitter.com/joeattardi/status/913395181688369152)
- [Python @ThePSF would be disappointed in JS. Definitely a feature I'd prefer not to have.](https://twitter.com/niklaskorz/status/913416022262190080)
- [Even if the feature is interesting, I am really not a fan of the syntax](https://twitter.com/sbegaudeau/status/913388802877657088)
- [Why dont we use a keyword instead of a semantic character?](https://twitter.com/codeaholicguy/status/913602347208601602)
- [If this is how I would have to write private methods, then I am not going to use them.](https://twitter.com/jjb_official/status/914082976262230016)
- [So ugly and unnecessary](https://twitter.com/mcfedr/status/915491622523228160)

#### Medium

- [Is it necessary to use a #hashtag? To me it seems too distracting...](https://medium.com/@marianban/is-it-necessary-to-use-a-hashtag-c575af85cab8)
- [That honestly add noise and complexity to the language for little gain. That # suffix looks terrible.](https://medium.com/@richard_lopes/that-honestly-add-noise-and-complexity-to-the-language-for-little-gain-778ff951fcc3)
- [How do I stop this from getting into JavaScript?](https://medium.com/@allycw/how-do-i-stop-this-from-getting-into-javascript-817813af7dc1)
- [is the # required or is it just a casual (/horrible) convention?](https://medium.com/@charlesdeuter/is-the-required-or-is-it-just-a-casual-horrible-convention-b81b851f3be)

</details>

This proposal is a best affort to afford these issues an alternative syntax without sacrificing the current syntax synergy of the JavaScript language.

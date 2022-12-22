# [[Internal Slot]]s, `Symbol`s, `WeakMap`s, and `#private` fields

One issue that comes up when trying to polyfill built-ins (or a "[host-defined facility](https://tc39.es/ecma262/#host-defined)", such as the DOM), is that a lot of built-ins make use of [[[Internal Slots]]](https://tc39.es/ecma262/#sec-object-internal-methods-and-internal-slots) to store information that can only be accessed via native code; examples include the [[StringData]] slot on [String` exotic objects](https://tc39.es/ecma262/#sec-string-exotic-objects), the [[RegExpMatcher]] slot on `RegExp`s, etc.

A key feature of internal slots is that they are 'tamper-proof', and cannot be modified by manipulating the properties of the object, only by (native) methods.  Notably, they are unaffected by whatever properties might exist on the object.  So, for example, if we have `const myString = new String('hello')`, we can always obtain the value `'hello'` using `String.prototype.toString.call(myString)`; none of the following attempts to modify the stored value will work:

```JS
// Creates unrelated property named 'StringData':
myString.StringData = 'goodbye';

// Tries to create a property whose name is defined by the variable
// StringData (after first putting that value in an array and then
// converting the array back to a string); throws if StringData is not
// already defiend:
myString[[StringData]] = 'goodbye';

// Creates an unrelated property named '[[StringData]]':
myString['[[StringData]]'] = 'goodbye';  
```

Unfortunately ES5.1 (and prior versions) provided no mechanism by which a JS developer might emulate internal slots; this was one of the the things that the ES6 and subsequent versions have sought to correct.

## [EcmaScript 2015](https://262.ecma-international.org/6.0/) ("ES6")

ES6 introduced two new mechanisms to emulate internal slots:

### `Symbol`-keyed properties
Objects can now have properties whose key is a [symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) and symbols generated using the `Symbol` function (rather than [`Symbol.for`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/for)) are guaranteed to be unique, so a class can use a unique symbol to **store information in a property that is guaranteed not to clash with any user-defined properties**:

```JS
const privateKey = Symbol('MyClass private property');
class MyClass {
  constructor(value) {
    this[privateKey] = value;
  }
  getValue() {
    return this[privateKey];
  }
}
```

Unfortunately, this technique won't give one truly _private_ data—even if `myKey` is hidden in a closure or module-local variable—because objects can be introspected to find the complete list of their property-key sybols using [`Object.getOwnPropertySymbols`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertySymbols).

### `Map`s and `WeakMap`s

The introduction of [`Map` objects](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map), which can have objects as keys, allows private data about an object to be stored separately from the object.  Unfortunately, objects used as map keys (and any values associated with those keys in the map) are retrievable via [`Map.prototype.keys()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map/keys) (amongst other methods), so remain accessible even if all other references to the object are no longer accessible; this means that using a `Map` to store private information about an object in this way prevents it from ever being garbage collected.

Fortunately, ES6 also introduced the [`WeakMap`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap), which is much like `Map` but accepts _only_ objects as keys, is non-iterable (and has no `.keys()` method or the like) and thus prevents exfiltration of keys, allowing keys and their associated values to be garbage collected once no other references to them remain.  If a WeakMap is held in a closure (or module-local) variable, the data held will be truly private:

```JS
const MyClass = (function() {
  const privateStuff = new WeakMap();

  return class MyClass {
    constructor(value) {
      privateStuff[this] = {value};
    }
    getValue() {
      return privateStuff[this].value;
    }
  }
})();
```

Unfortunately, while weakmaps can provide a private, non-conflictable location to store internal data, this approach is not very ergonomic.

## [EcmaScript 2022](https://tc39.es/ecma262/2022/)
### `#private` Fields

One of the main features of ES2022 was [the Class Fields proposal](https://github.com/tc39/proposal-class-fields), which introduced [private fields](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_class_fields#private_fields), which were [designed to be analogous to internal slots](https://docs.google.com/presentation/d/1lPEfTLk_9jjjcjJcx0IAKoaq10mv1XrTZ-pgERG5YoM/edit#slide=id.g4c6616ed54_0_16).

Private fields are quite ergonomic; the previous example can be rewritten thus:

```JS
class MyClass {
  #value;  // Declare private field (mandatory);
  constructor(value) {
    this.#value = value;
  }
  getValue() {
    return this.#value;
  }
}
```

# Functions

## External Parameter Names

In Swift, functions can have external parameters names:

```swift
func distance (from p1: Point, to p2: Point) -> Int
```

External parameters names are supported through ECMAScript’s [object destructuring assignment](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment#Object_destructuring). The above code is translated as:

```javascript
function distance ({ from: p1, to: p2 })
```

### Omitted External Parameter Names

External parameter names can be omitted by giving `_` as its external name:

```swift
func select (a: Int, _ b: Int,
             selector: (Int, Int) -> Int) -> Int
```

By default, the first parameter omits its external parameter name. Also, starting from the second parameter, if the programmer does not specify its external name, its external name would be identical to its internal name. Thus, the above function has the “signature” `select(_:_:selector:)`.

During the translation, each parameter without an external name will be given an external name of the form `$n`, where `n` indicates that the parameter is the nth parameter without an external name. The above Swift function translates to:

```javascript
function select ({ $0: a, $1: b, selector: selector})
```

If each parameters of a function does not have an external name, it is translated as an ordinary ECMAScript function. Namely,

```swift
func min (a: Int, _ b: Int) -> Int
```

translates to

```javascript
function min (a, b)
```

### “External Name Free” Form

Closures in Swift do not support external parameter names. Any functions passed to other functions will be converted to an “external name free” form implicitly. Also, a translated function may be passed to higher order functions like `Array.map`. Thus, an `externalNameFreeForm` property is provided for each translated function that has some parameters with external names:

```javascript
function distance ({ from: p1, to: p2 }) {
  return Math.sqrt(/* ... */);
}
distance.externalNameFreeForm = (p1, p2) =>
  distance({from: p1, to: p2});
```

#### Note

`externalNameFreeForm` is assigned on demand and depends on access control level. If a function is `public`—namely it will be exported by the Swift module—this property will always be set. However, if a function is `internal` or `public`, this property will be set only if the function is passed as value for at least once in the Swift source code.

### External Parameter Names is Unordered

Due to the unordered nature of object’s properties in ECMAScript, the following code cannot be differentiated with the above translation mechanism:

```swift
func select (x1 x1: Int, x2: Int) -> Int {
  if x1 > x2 { return x1 }
  else { return x2 }
}

func select (x2 x2: Int, x1: Int) -> Int {
  if x1 > x2 { return x1 }
  else { return x2 }
}
```

Since we believe that such a naming convention is a bad practice, we do not plan to support it. An **error** will be raised when such a function is encountered.

Note that other forms of function overloading are supported, as discussed in the next section.

## Function Overloading

Function overloading has two different cases. The first is that multiple candidates share the same “base” name but different parameter “signature”—i.e. different numbers of parameters and different external parameter names. The second is that multiple candidates share both the same “base” name and parameter “signature”, but parameters’ types in candidates differ from each other.

In either case, each candidate has its own translated function whose name is of the form `\(base name)$\(unique suffix)`. For example, each of the following candidates

```swift
func print (x: Printable)
func print (context context: String, errorMessage: String)
func print (context context: String, object: Printable)
```

will have its own translated function:

```javascript
function print$_TZ1F (x)
function print$_F2sI({ context, errorMessage })
function print$_a5X3({ context, object })
```

### Different Parameter “Signature”

In this case, a dispatcher function will be generated. It will analyze the shape of its arguments and determine which function to call. The above set of candidates should share a dispatcher function whose behavior is identical to:

```javascript
function print () {
  if (argument.length != 1)
    // throw some error...
  const arg1 = argument[0];
  if (typeof arg1 == 'object') {
  	if (arg1.hasOwnProperty('context') &&
        arg1.hasOwnProperty('errorMessage'))
      return print$_F2sI(arg1);
    if (arg1.hasOwnProperty('conext') &&
        arg1.hasOwnProperty('object'))
      return print$_a5X3(arg1);
  }
  return print$_TZ1F(arg1);
}
```

Function calls to these candidates will go through the dispatcher function for better code readability. However, in the future, with optimization enabled, such call should be eliminated.

As for `public` function that is overloaded, only the dispatcher function is exposed. If the set of `public` candidates is different from all candidates available in that module, a private dispatcher function will also be created, whose name is `\(base name)$`.

### Different Parameter Types

Overall, it is not difficult to create a similar dispatcher function that determines each parameter’s type. However, doing so might require the translator to differentiate numeral types like `Int8`, `Int16`, `Int32`, `Int64`, etc. **Under Discussion.**

## Operators

ECMAScript does not allow custom operators. Therefore, all custom operators in Swift will be translated to a normal function. All invocations to these operators will be translated as normal function applications, with precedence taken into account.

To ensure the translated codes’ readability, operator names are translated to `$x1x2..xn` where `xi` is the i-th character’s Unicode name. Users can also use pragmas to give operators’ corresponding function names directly. **Under Discussion:** syntax of pragmas.

Operators of access control level `public` will *not* be exported. Keep in mind that a good practice is to provide a function for each custom operator you create.
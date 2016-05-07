# Control Flow

## Loops

### For-In Loops

A `for-in` loop is translated to `for-of` in ECMAScript. Accordingly, all types that support `SequenceType` protocol will have [generators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) implemented in the translated codes. Namely,

```swift
for index in 1...5 {
    print("\(index) times 5 is \(index * 5)")
}  
```

translates to

```javascript
for index of Swift.makeClosedRange(1, 5) {
    console.log(`${index} times 5 is \(index * 5)`);
}
```

### While Loops

`while` is translated in the canonical way.

### Repeat-While

This translates to `do-while` loop. The following code

```swift
func fibonacci (n: Int) -> Int {
    var a = 0
    var b = 1
    repeat {
        let c = a + b
        a = b
        b = c
        n -= 1
    } while n > 0
    return a
}
```

translates to

```javascript
function fibonacci (n) {
    let a = 0;
    let b = 1;
    do {
		const c = a + b;
        a = b;
        b = c;
        n -= 1;
    } while (n > 0)
    return a;
}
```

## Conditional Statements

### If

Issues related with optional type `T?` is deferred to the section _Optional Chaining_.

`if` is translated in the canonical way.

### Switch

`switch` is tricky to handle, as Swift supports many features like pattern matching over algebraic data types (i.e. enumerations with associated values), ranges, and tuples.

#### Simplest Form

In the simplest form, we only consider `switch` over primitive types, namely integers, float numbers, and strings. Two things to consider:

1. In Swift, `case` does not fallthrough by default, therefore the translator will automatically add `break` unless the last statement is `return`. Still, an explicit `fallthrough` is possible in Swift. In such case, `break` is removed from the translated code.
2. In Swift, a `case` clause can have multiple values. This is translated into several empty cases with the last one doing the real work.

An example:

```swift
let hello = "Hello"
switch hello {
case "Hello", "Bonjour", "Salve":
    print("European languages")
case "你好", "こんにちは":
	print("Eastern Asian languages")
case "Kamusta":
	print("Filipino")
	fallthrough
default:
	print("Other languages")
}
```

translates to

```javascript
const hello = "Hello"
switch hello {
case "Hello":
case "Bonjour":
case "Salve":
	console.log("European languages");
	break;
case "你好":
case "こんにちは":
	console.log("Eastern Asian languages");
case "Kamusta":
	console.log("Filipino");
default:
	console.log("Other languages");
}
```

In all the following cases, `if-else` sequence is used to stimulate `switch`. There are two benefits:

1. `if-else` introduces a new scope, which makes value-bindings much easier.
2. if we use closure to introduce new scopes, differentiating the `return` in Swift code from `return` that exists purely for exiting these closures would be hard.

Nevertheless, pattern matching over algebraic data types is implemented by a combination of `switch` and `if-else`. This is discussed in section _Enumerations_.

#### Interval Matching

Interval matching is implemented by `if-else` sequence. Consider the example from Apple:

```swift
let approximateCount = 62
let countedThings = "moons orbiting Saturn"
var naturalCount: String
switch approximateCount {
case 0:
    naturalCount = "no"
case 1..<5:
    naturalCount = "a few"
case 5..<12:
    naturalCount = "several"
case 12..<100:
    naturalCount = "dozens of"
case 100..<1000:
    naturalCount = "hundreds of"
default:
    naturalCount = "many"
}
print("There are \(naturalCount) \(countedThings).")
// Prints "There are dozens of moons orbiting Saturn."
```

This translates to

```javascript
const approximateCount = 62
const countedThings = "moons orbiting Saturn"
let naturalCount
if (approximateCount == 0) {
	naturalCount = "no";
} else if (approximateCount >= 1 && approximateCount < 5) {
	naturalCount = "a few";
}
} else if (approximateCount >= 5 && approximateCount < 12) {
	naturalCount = "several";
}
} else if (approximateCount >= 12 && approximateCount < 100) {
	naturalCount = "dozens of";
}
} else if (approximateCount >= 100 && approximateCount < 1000) {
	naturalCount = "hundreds of";
} else {
	naturalCount = "many";
}
console.log(`There are ${naturalCount} ${countedThings}.`);
```

#### Tuples and Value Binding

Pattern matching against tuples also translates to `if-else`. Consider another example from Apple:

```swift
let somePoint = (1, 1)
switch somePoint {
case (0, 0):
    print("(0, 0) is at the origin")
case (_, 0):
    print("(\(somePoint.0), 0) is on the x-axis")
case (0, _):
    print("(0, \(somePoint.1)) is on the y-axis")
case (-2...2, -2...2):
    print("(\(somePoint.0), \(somePoint.1)) is inside the box")
default:
    print("(\(somePoint.0), \(somePoint.1)) is outside of the box")
}
// Prints "(1, 1) is inside the box"
```

translates to

```javascript
const somePoint = [1, 1];
if (somePoint[0] == 0 && somePoint[1] == 0) {
    console.log("(0, 0) is at the origin");
} else if (somePoint[1] == 0) {
    console.log(`(${somePoint[0]}, 0) is on the x-axis`);
} else if (somePoint[0] == 0) {
    console.log(`(0, ${somePoint[1]}) is on the y-axis`);
} else if (somePoint[0] >= -2 && somePoint[0] <= 2 &&
           somePoint[1] >= -2 && somePoint[1] <= 2) {
	console.log(`(${somePoint[0]}, ${somePoint[1]}) is inside the box`);
} else {
	console.log(`(${somePoint[0]}, ${somePoint[1]}) is outside of the box`);
}
```

Value bindings are done by binding the variables inside `if-else`:

```swift
let anotherPoint = (2, 0)
switch anotherPoint {
case (let x, 0):
    print("on the x-axis with an x value of \(x)")
case (0, let y):
    print("on the y-axis with a y value of \(y)")
case let (x, y):
    print("somewhere else at (\(x), \(y))")
}
// Prints "on the x-axis with an x value of 2"
```

translates to

```javascript
const anotherPoint = [2, 0];
if (anotherPoint[1] == 0) {
    const x = anotherPoint[0];
    console.log(`on the x-axis with an x value of ${x}`);
} else if (anotherPoint[0] == 1) {
    const y = anotherPoint[1];
    console.log(`on the y-axis with an y value of ${y}`);
} else {
    let [x, y] = anotherPoint;
    console.log(`somewhere else at (${x}, ${y})`);
}
```

#### Where

In Swift, a `case` can have a `where` clause for additional checks:

```swift
let yetAnotherPoint = (1, -1)
switch yetAnotherPoint {
case let (x, y) where x == y:
    print("(\(x), \(y)) is on the line x == y")
case let (x, y) where x == -y:
    print("(\(x), \(y)) is on the line x == -y")
case let (x, y):
    print("(\(x), \(y)) is just some arbitrary point")
}
// Prints "(1, -1) is on the line x == -y"
```

The `where` clause is implemented by adding a closure whose return value is boolean. The closure is put conjunctively into the `if` of its corresponding `case`:

```javascript
const yetAnotherPoint = (1, -1)
if (((x, y) => x == y)(yetAnotherPoint[0], yetAnotherPoint[1])) {
    let [x, y] = yetAnotherPoint;
    console.log(`(${x}, ${y}) is on the line x == y`);
} else if (((x, y) => x == -y)(yetAnotherPoint[0], yetAnotherPoint[1])) {
    let [x, y] = yetAnotherPoint;
    console.log(`(${x}, ${y}) is on the line x == -y`);
} else {
    let [x, y] = yetAnotherPoint;
    console.log(`(${x}, ${y}) is just some arbitrary point`);
}
```

## Control Transfer Statements

There are five control transfer statements:

* `continue`
* `break`
* `fallthrough`
* `return`
* `throw`

`continue`, `break`, and `return` maps well to ECMAScript. [Labeled `continue` and `break`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/label) are also supported. They will be translated in the canonical way. `fallthrough` is tricky and will be discussed below. `throw`, as well as exception handling in general, will not be supported in the initial version of the translator and is thus omitted.

### Fallthrough

While `fallthrough` in the simplest form of `switch` is simple, it is a bit tricky in other cases. Note that in Swift, `fallthrough` cannot transfer the control into a `case` that introduces new variables. Hence, it is theoretically possible to replace the `fallthrough` with the statements of the next `case`.

Still, it is crucial that once we execute all the fallthrough-ed statements, we should immediately exit the `if-else` sequence. Consider the following nonsense:

```swift
switch someNumber {
case 1..<10:
    for item in collection {
        if item == 1 {
            fallthrough
        }
        print(item)
    }
default:
    print("a strange branch")
}
```

Simply replacing `fallthrough` with statements in `default` is wrong:

```javascript
if (someNumber >= 1 && someNumber < 10) {
    for (item of collection) {
        if (item == 1) {
            console.log("a strange branch");
		}
      	console.log(item);
	}
} else {
	console.log("a strange branch");
}
```

To solve the issue, we wrap the `if-else` sequence in a loop and uses a labeled `break` for early exit. More specifically, the `if-else` sequence is wrapped into a `do-while` loop that has a constant false condition so that it will execute for only once. A “jump-out” label is put before  the loop, and a labeled `break` is put after the expanded `fallthrough`:

```javascript
$jumpout_1:
do {
    if (someNumber >= 1 && someNumber < 10) {
        for (item of collection) {
            if (item == 1) {
                console.log("a strange branch");
              	break $jumpout_1;
    		}
          	console.log(item);
    	}
    } else {
    	console.log("a strange branch");
    }
} while (false)
```

## Early Exit

A `guard` clause in Swift may or may not introduce a new variable. If it does not, it is equivalent to an `if` statement in ECMAScript:

```swift
guard collection.count != 0 else {
	return nil
}
```

translates to

```javascript
if (!(collection != 0)) {
    return null
}
```

The other case is to unwrap an optional value. However, in ECMAScript, everything can be `null`. Therefore, it is unnecessary to have a separate assignment to unwrap an optional value; we only need to check the assigned variable is not `null`:

```swift
let person = [ "Alexander": "the Great"]
guard let alex = person["Alexander"] else {
    return
}
```

translates to

```javascript
const person = {
    "Alexander": "the Great"
};
const alex = person["Alexander"];
if (alex == null) {
    return
}
```

## Check API Availability

Swift has built-in support to check API availability:

```swift
if #available(iOS 9, OSX 10.10, *) {
    // Use iOS 9 APIs on iOS, and use OS X v10.10 APIs on OS X
} else {
    // Fall back to earlier iOS and OS X APIs
}
```

This will be supported. However, the specific platform name and version of our backend is still **under discussed**.




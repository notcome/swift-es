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

1. In Swift, `case` does not fall through by default, therefore the translator will automatically add `break` unless the last statement is `return`. Still, an explicit `fallthrough` is possible in Swift. In such case, `break` is removed from the translated code.
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

#### Interval Matching

Pattern matching over both ranges and tuples are translated to `if-else` sequence. One benefit of this is that `if-else` introduces a new scope, which makes value-bindings much easier.

For interval matching, consider the case from Apple:

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

As mentioned above, pattern matching against tuples is translated as `if-else` sequence:

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

In Swift, a `case` can have a `where` clause for additional checks. **Under discussion.**
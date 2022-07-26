# Guide: Calling Swift APIs from C++

A Swift library author might want to expose their interface to C++, to allow a C++ codebase to interoperate with the Swift library.  This document describes how this can be accomplished, by first describing how Swift can expose its interface to C++, and then going into the details on how to use Swift APIs from C++.

**NOTE:** This is a work-in-progress, living guide document for how Swift APIs can be imported and used from C++.

**NOTE:** This document does not go over the following Swift language features yet:

* Closures
* Class types & inheritance
* Existential types (any P)
* Nested types
* Operators
* Tuples & functions returning multiple parameters
* class subclass generic constraint
* Type casting
* Recursive/indirect enums
* Associated types in generic where clauses
* Error handling
* Opaque return type `-> some P` (should we not support it)
* Character type & character literal


## Exposing Swift Codebase to C++

A Swift codebase is organized into units called modules. A module typically corresponds to a specific Xcode or Swift package manager target. Swift can generate a module interface file that presents a source view of the public Swift interface provided by the module. In addition to a Swift module interface, Swift can also generate a header file that contains C++ functions and classes that allow us to work with the Swift functions and types. We can import this header file into our C++ program to start using the Swift APIs from C++.

### C++ Language And Library Requirements

Importing Swift APIs into C++ requires certain C++ features introduced in newer C++ language standards. The following C++ standards are expected to work:

* C++20. It is the recommended standard, as C++ 20 concepts enable type checking for imported Swift generic APIs.
* C++17 and C++14 are supported with some restrictions. Some generic APIs might not be available prior to C++20.

## Importing Swift Modules

A Swift module can be imported over into C++ by using an  `#include` that imports the generated C++ header for that module:

```
// Swift module 'MyModule'
func myFunction();

// C++
#include "MyModule-Swift.h"
```

A C++ namespace is used to represent the Swift module. Namespacing provides a better user experience for accessing APIs from different modules as it encapsulates the different module interfaces in their own namespace. For example, in order to use a Swift module called `MyModule` from C++, you have to go through the `MyModule::` namespace in C++:

```
// C++
#include "MyModule-Swift.h"

int main() {
  MyModule::myFunction(); // calls into Swift.
  return 0;
}
```

## Calling Swift Functions

Swift functions that are callable from C++ are available in their corresponding module namespace. Their return and parameter types are transcribed to C++ primitive types and class types that represents the underlying Swift return and parameter types.

Fundamental primitive types have a C++ fundamental type that represents them in C++:

|Swift Type    |C++ Type    |C Type (if different)    |    |target specifc    |
|---    |---    |---    |---    |---    |
|Void (or no return)    |void    |    |    |    |
|Int    |swift::Int    |ptrdiff_t    |long or long long (windows)    |YES    |
|UInt    |size_t    |    |unsigned long or unsigned long long (windows)    |YES    |
|Float    |float    |    |    |    |
|Double    |double    |    |    |    |
|    |    |    |    |    |
|CInt    |int    |    |    |    |
|CUnsignedInt    |unsigned int    |    |    |    |
|CShort    |short    |    |    |    |
|CUnsignedShort    |unsigned short    |    |    |    |
|CLong    |long    |    |    |    |
|CUnsignedLong    |unsigned long    |    |    |    |
|CLongLong    |long long    |    |    |    |
|CUnsignedLongLong    |unsigned long long    |    |    |    |
|    |    |    |    |    |
|OpaquePointer     |void *    |    |    |    |
|UnsafePointer<T>    |const T *    |    |    |    |
|UnsafeMutablePointer<T>    |T *    |    |    |    |

**NOTES**: Need static_assert that std::is_same(size_t, unsigned long) or unsigned long long to ensure we can match the right type metadata using a template specialization.

A function that takes or return primitive Swift types behaves like any other C++ function, and you can pass in the C++ types when calling them, just as you’d expect.

```
// Swift module 'MyModule'
func myFunction(x: float, _ c: Int) -> Bool

// C++
#include "MyModule-Swift.h"

int main() {
  return !MyModule::myFunction(2.0f, 3); // myFunction(float, swift::Int) -> bool
}
```

### In-Out Parameters

A Swift `inout` parameter is mapped to a C++ reference type in the C++ function signature that’s generated in the C++ interface. You can then pass in a value directly to an `inout` parameter from C++ side, like the example below:

```
// Swift module 'MyModule'
func swapTwoInts(_ a: inout Int, _ b: inout Int)

// C++ interface snippet
void swapTwoInts(swift::Int &a, swift::Int &b) noexcept;

// C++
#include "MyModule-Swift.h"

void testSwap() {
  swift::Int x = 0, y = 42;
  MyModule::swapTwoInts(x, y);
}
```

### Function Overloading

Swift allows you to specify which overload of the function you would like to call using argument labels. For example, the following snippet is explicitly calling the second definition of `greet` because of the call using `greet(person:,from:)` argument labels:

```
func greet(person: String, in city: String) {
  print("Hello \(person)! Welcome to \(city)!")
}

func greet(person: String, from hometown: String) {
  print("Hello \(person)!  Glad you could visit from \(hometown).")
}

greet(person: "Bill", from: "San Jose") // calls the second overload of greet.
```

C++ only allows us to select which overload of the function we want to call by using type-based overloading. In cases where type-based overloading isn’t sufficient, like when the arguments have the same type but a different argument label, you can use the `exposed` attribute to provide a different C++ name for the C++ function, like in the following example:

```
@expose(C++, greetPersonIn)
func greet(person: String, in city: String) {
  print("Hello \(person)! Welcome to \(city)!")
}

@expose(C++, greetPersonFrom)
func greet(person: String, from hometown: String) {
  print("Hello \(person)!  Glad you could visit from \(hometown).")
}
```

### Default Parameter Values

Default parameter values allow you to provide a default value for Swift function parameter, allowing the program to not specify it when calling such function. The generated C++ interface for a Swift function contains default parameter values as well, just like in the following example:

```
// Swift module 'MyModule'
func someFunction(parameterWithoutDefault: Int, parameterWithDefault: Int = 12) {
}

// C++ interface snippet
void someFunction(swift::Int parameterWithoutDefault, swift::Int parameterWithDefault = 12) noexcept;

// C++
#include "MyModule-Swift.h"
using namespace MyModule;

void testSwap() {
  someFunction(3, 6); // parameterWithDefault is 6
  someFunction(4);    // parameterWithDefault is 12
}
```

Swift default parameter values that are set to a `#file` or `#line` call site specific literal are not represented in the generated C++ interface. The user need to pass them explicitly from the C++ call site instead.

**TODO:** Any constraints for default parameter values?
**OPEN QUESTIONS:** Are there any problems here?

* YES : a problem with Swift allowing non-last parameter orders.

### Variadic Parameters

A variadic parameter is a parameter that accepts zero or more values of the specified type. It gets exposed in C++ using a `swift::variadic_parameter_pack` class template. You can pass values to a variadic parameter using the C++ initializer list syntax. For example, the following Swift function with a `Double` variadic parameter:

```
func arithmeticMean(_ numbers: Double...) -> Double {
   ...
}
```

can be called from C++ using a C++ initializer list:

```
arithmeticMean({ 1.0, 2.0 });
```

## Using Swift Structure Types

Swift structures that are usable from C++ are available in their corresponding module namespace. They’re bridged over as a C++ `class` that has an opaque representation and layout whose size and alignment matches the size and alignment of the Swift structure.

You can construct an instance of a structure using the static `init` method in the C++ class:

```
// Swift module 'Weather'
struct WeatherInformation {
  `var`` temperature``:`` ``Int`
}

// C++ use site.
#include "Weather-Swift.h"

int main() {
   auto weather = Weather::WeatherInformation::init(/*temperature=*/ 25);
}
```

### Initialization

Swift’s structures that have a default initializer are given a default C++ constructor. For example, the following structure:

```
struct ScreenSize {
    var width = 0
    var height = 0
}
```

Will have a default initializer that will initialize a `width` and `height` to zero, which you can then use from C++ directly:

```
void constructScreenSize() {
  auto size = ScreenSize();
  // size.width and size.height is 0
}
```

The other initializers are bridged over as static `init` methods. The C++ initializers use type-based overloading to select the right overload for the initializer.

For example, given the following structure with two initializers:

```
struct Color {
  let red, green, blue`:`` Float``
  
  public init(red: Float, green: Float, blue: Float) {
    self.red = red
    self.green = green
    self.blue = blue
  }
  public init(white: Float) {
    self.red = white
    self.green = white
    self.blue = white
  }`
}
```

The following C++ `init` methods will be available:

```
class Color {
public:
  Color() = delete;

  static Color init(float red, float green, float blue);
  static Color init(float white);
};
```

**NOTE**: Swift doesn’t allow calling constructor without argument labels. Is that a problem for us?

### Providing renamed C++ overloads for Swift Initializers

The C++ `init` overloads for Swift initializers can sometimes conflict between each other because C++ doesn’t allow us to use argument labels to select the correct overload, and so instead we need to rely on the type of the argument when calling it from C++. In order to avoid ambiguities on the C++ side, you can rename one specific initializer using something like the `@expose` attribute.

As an example, this structure renames its second `init` overload in C++ to expose them both to C++:

```
// Swift module 'Weather'
struct Celcius {
  `var`` temperatureInCelcius``:`` ``Double`
`  `
`  ``// FEEDBACK: could provide a constructor here?
  // NOTE: concern about encouraging people not to use labels`
`  init``(``_ t``:`` ``Double``)`` ``{`` ``self``.``temperatureInCelcius ``=`` t ``}`
`  `
`  ``// FEEDBACK: could the compiler construct the 'initFromFahrenheit' c++ name?`
`  ``@expose``(``c``++,`` initFromFahrenheit``)`
`  init``(``fromFahrenheit fahrenheit``:`` ``Double``)`` ``{`` ``...`` ``}`
}
```

Both initializers can then be used from C++:

```
#include "Weather-Swift.h"
using namespace Weather;

void makeSunnyDay() {
  auto morningTemperature = Celcius::init(25);
  auto noonTemperature    = Celcius::initFromFahrenheit(90);
}
```

**NOTE**: The compiler should warn here about overload ambiguities.

### Convenience Initialization of Swift Structures that conform to ExpressibleBy...Literal protocol

Certain types like Swift’s `String` and `Array` are bridged over with convenience initializers that are inferred from their conformance to the `ExpressibleByStringLiteral` and the `ExpressibleByArrayLiteral` protocols. In general, any type that conforms to a protocol like `ExpressibleByStringLiteral` will receive a C++ constructor in it’s interface that resembles the following constructor for Swift’s `String` type:

```
   String(const char *value) {
     *this = String::init(value);
   }
```

Similarly, any type that conforms to `ExpressibleByArrayLiteral` will receive a C++ constructor that takes in a C++ initializer list so that it can be initialized from a C++ array literal.

### Resilient Swift Structures

Swift resilient structures are bridged over as a C++ class that boxes the Swift value on the heap. Their generated C++ interface resembles the C++ interface for a non-resilient structure, so all the methods and properties can be accessed in the same manner. For example, you can call methods and access the properties on `Foundation::URL` , which is a resilient Swif structure, in the same manner as you would for any other fixed layout Swift structure:

```
#include "Foundation-Swift.h"
using namespace Weather;

void workWithURL() {
  auto url = Foundation::URL::init("https://swift.org");
  std::cout << "Is File URL:" << url.isFileURL() << "\n";
  auto absoluteURL = URL.absoluteURL();
}
```

The boxing implies that the following operations will allocate and store a new value on the heap:

* Returning a value from a call to a Swift method/function allocates a new value on the heap.
* Returning a value from a getter call to a Swift property `get` accessor allocates a new value on the heap.
* Creating a new instance of a resilient structure in C++ using the static `init` method allocates a new value on the heap.
* Copying a C++ `class` that represents a resilient Swift structure using a C++ copy constructor allocates a new value on the heap.

**NOTE**: A fixed-layout structure that contains a resilient structure as a stored property is also boxed on the C++ side.

## Calling Swift Methods

Swift’s structures, enumerations and classes can define instance methods. An instance method that’s declared in a Swift type gets its own C++ member function declaration in the C++ class that corresponds to the underlying Swift type in the generated C++ interface for a Swift module.

Instance methods in structures and enumerations are marked as `const` in C++, unless they’re marked as `mutating` in Swift. Here's how one could call a mutating method on a Swift structure from C++:

```
// Swift module 'Geometry'
struct Point {
  var x = 0.0, y = 0.0
  mutating func moveBy(x deltaX: Double, y deltaY: Double) {
    x += deltaX
    y += deltaY
  }
}

// C++ use site:
#include "Geometry-Swift.h"
using namespace Geometry;

int main() {
  auto point = Point();
  point.moveBy(1.0, 2.0);
  std::cout << "The point is now at " << point.getX() << ", " << point.getY() << "\n";
  // Prints "The point is now at 1.0, 2.0"
  return 0;
}
```

Calling `mutating` methods on a value that is declared as `const` is not allowed:

```
int main() {
  const auto point = Point();
  point.moveBy(1.0, 2.0);
  //` reports a compile time error.`
}
```

### Static Methods

A static method declared in a Swift structure or enumeration gets its own C++ static member function declaration in the C++ class that corresponds to the underlying Swift type in the generated C++ interface for a Swift module. It can be called using its qualified name directly from C++, like in the following example:

```
// Swift module 'Geometry'
struct Rectangle {
  var left, right: Point

  static func computeDeviceScreenSize() -> Rectangle {
    ...
  }
}

// C++ use site:
#include "Geometry-Swift.h"

int main() {
  auto screenSize = Geometry::Rectangle::computeDeviceScreenSize();
  // Use screen size...
  return 0;
}
```

**TODO:** Dispatching overriding / class methods

## Using Swift Enumeration Types

A Swift enumeration is imported as class in C++. That allows C++ to invoke methods and access properties that the enumeration provides. Each enumeration case that doesn’t have associated value is exposed as a static variable in the structure.

For example, given the following enum:

```
// Swift module 'Navigation'
enum `CompassDirection {`
  case north
  case south
  case east
  case west
}
```

The following interface will be generated:

```
// "Navigation-Swift.h" - C++ interface for Swift's Navigation module.
class CompassDirection {
public:
  static const CompassDirection north;
  static const CompassDirection south;
  static const CompassDirection east;
  static const CompassDirection west;
};
```

### Matching Swift Enumeration Values with a C++ Switch Statement

Swift’s enumerations can not be used directly in a switch, as C++ does not allow a `switch` to operate on C++ classes. However, For Swift enumerations that have an underlying integer representation, the generated C++ interface provides a convenience  C++ enum called `cases` inside of the generated C++ class that represents the enumeration. This C++ enum can then be used in a switch, as the class that represents the enumeration implicitly converts to it. The `cases` C++ enum allows us to switch over the `CompassDirection` class from the example above in the following manner:

```
#include "Navigation-Swift.h"
using namespace Navigation;

CompassDirection getOpposite(CompassDirection cd) {
  switch (cd) {                       // implicit conversion to CompassDirection::cases
  using enum CompassDirection::cases; // allow name lookup to find enum cases.
  case north:
    return CompassDirection::south;
  case south:
    return CompassDirection::north;
  case east:
    return CompassDirection::west;
  case west:
    return CompassDirection::east;
  }
}
```

### Enumerations With Raw Values

Swift allows you to declare enumerations whose cases are represented using an underlying raw value type. The C++ interface for such a Swift enumeration allows you access both the raw value of such an enumeration, and also to construct such an enumeration from a raw value.

For example, given the following enum with a String type:

```
// Swift module 'Airport'
enum Airport : String` {`
  case LosAngeles   = "LAX"
  case SanFrancisco = "SFO"
}
```

You can access the underlying rawValue from C++ using the `getRawValue` method:

```
#include "Airport-Swift.h"
using namespace Airport;

void printAirport(Airport dest) {
  swift::String airportCode = dest.getRawValue();
  std::cout << "landing at " << airportCode << "\n";
}
```

You can use the static `init` method to construct an optional enumeration from a raw value:

```
void constructRoute() {
  swift::Optional<Airport> arrivingTo = Airport::init("LAX");
  // arrivingTo is now Airport::LosAngeles
  
  auto departingFrom = Airport::init("HTX");
  // departingFrom is none
}
```

### Enumerations With Associated Values

Swift allows an enumeration to store values of other types alongside the enumeration’s case values. This additional information is called an associated value in Swift. Enums with associated values are represented in a different manner than enums without associated values.

For example, the following enum with two cases with associated values:

```
// Swift module 'Store'
enum Barcode {
  case upc(Int, Int, Int, Int)
  case qrCode(String)
}
```

Will get a C++ interface that resembles this class:

```
// "Store-Swift.h" - C++ interface for Swift's Store module.
class Barcode {
public:
  Barcode() = delete;
 
  bool isUpc() const;

  using UpcType = swift::Tuple<swift::Int, swift::Int, swift::Int, swift::Int>;

  bool isUpc() const;

  // Extracts the associated valus from Barcode.upc enum case
  UpcType getUpc() const;

  bool isQrCode() const;

  // Extracts an associated value from Barcode.qrCode enum case
  swift::String getQrCode() const;

  static Barcode initUpc(swift::Int, swift::Int, swift::Int, swift::Int);
  static Barcode initQrCode(swift::String);
};
```

The C++ user of this enumeration can then use it by checking the type of the value and getting the associated value using the `is` and `get` member functions:

```
#include "Store-Swift.h"
using namespace Store;

Barcode normalizeBarcode(Barcode barcode) {
  if (barcode.isQrCode()) {
    auto qrCode = barcode.getQrCode();
    swift::Array<swift::Int> loadedBarcode = loadQrCode(qrCode);
    return Barcode::initUpc(loadedBarcode[0], loadedBarcode[1], loadedBarcode[2], loadedBarcode[3]);
  }

  return barcode;
}
```

## Accessing Properties In C++

Swift allows structures and classes to define stored and computed properties. Stored properties store constant and variable values as part of an instance, whereas computed properties calculate (rather than store) a value. The stored and the computed properties from Swift types are bridged over as getter `get...` and setter `set...` methods in C++. Setter methods are not marked as `const` and should only be invoked on non `const` instances of the bridged types.

For example, given the following structure with a stored and a computed property:

```
// Swift module 'Weather'
struct WeatherInformation {
  `var`` temperature``:`` ``Int`
` `
`  ``var`` temperatureInFahrenheit``:`` ``Int`` ``{`
`    ``...`
`  ``}`
}
```

Both properties can be accessed with getters and setters, as demonstrated by the interface and example below:

```
// "Weather-Swift.h" - C++ interface for Swift's Weather module.
class WeatherInformation {
public:
  WeatherInformation() = delete;

  swift::Int getTemperature() const;
  void setTemperature(swift::Int);

  swift::Int getTemperatureInFahrenheit() const;

private:
  // opaque storage representation for the Swift struct.
};

// C++ use site.
#include "Weather-Swift.h"
#include <iostream>

void printWeatherInformation(const Weather::WeatherInformation &info) {
  std::cout << "Temperature (C): " << info.getTemperature() << "\n";
  std::cout << "Temperature (F): " << info.getTemperatureInFahrenheit() << "\n";
}

void updateWeather(Weather::WeatherInformation &info) {
  info.setTemperature(25);
}
```

Please note, however, that a getter method for property returns a copy of the value stored in the property. This means that when you mutate a value returned by the getter, it does not update the original property value. We can mutate property values using `withMutable...` member function described in the next section.

Getter-only properties of type `bool` that start with `is` or `has` can be used by their exact name from C++. For example Array’s `isEmpty` maps to `isEmpty()` call in C++:

```
int printArray(const swift::Array<int> &array) {
  if (array.isEmpty()) {
    std::cout << "[]";
    return
  }
  ...
}
```

### Mutating Property Values

Swift allows you to mutate a property by using additional operations like assignments, mutating method calls, or property mutations when accessing a property:

```
// Swift module 'Shapes'
struct Point {
    var x = 0.0, y = 0.0
}
struct Size {
    var width = 0.0, height = 0.0
}
struct Rectangle {
  var position: Point
  var size: Size
}

func updatePosition(shape: inout Rectangle, by value: Double) {
  shape.position.x += value // mutate `position.x` inside of given shape
  shape.position.y += value // mutate `position.y` inside of given shape
}
```

The generated C++ interface allows you to mutate a property value using a `withMutating...` method, which takes in a C++ lambda that receives a reference to the underlying value that can be safely mutated within the lambda:

```
#include "Shapes-Swift.h"

void updatePosition(Shapes::Rectangle &shape, double value) {
  shape.withMutablePosition([&](auto &position) {
    position.withMutableX(  [&](auto &x)        { x += value; }
    position.withMutableY(  [&](auto &y)        { y += value; }
  });
}
```

It’s illegal to escape the passed reference to the value from the lambda, as that can create a dangling reference in your program.

### Static Properties

Type properties are mapped as `static` getter, setter, and mutation member functions in the C++ class that represents the Swift type. They can be accessed directly from C++ by invoking the function using its qualified name directly, like in the following example:

```
// Swift module 'GlobalSettings'
struct Config {
  static var binaryName = ""
}

// C++
#include "GlobalSettings-Swift.h"

int main(const char *argv[], int argc) {
  if (!GlobalSettings::Config::getBinaryName().isEmpty())
    GlobalSettings::Config::setBinaryName(swift::String::init(argv[0]));
  ...
}
```

Open Property Questions:

* What happens when we have a name collision between a Swift `get` method that we’d like to bridge and the bridged property getter?

## Accessing Subscripts In C++

Swift subscripts allow users to use the `[]` operator to access elements in a collection.  The getter of a Swift subscript is bridged over as `operator []` to C++. It takes in the index parameter and returns the subscript’s value over to C++. This is how you would use the subscript to access an element from a Swift `Array` :

```
#include "Swift-Swift.h"
#include <iostream>

void printElementsInArray(const swift::Array<swift::Int> &elements) {
  for (size_t i = 0; i < elements.getCount(); ++i) {
    std::cout << elements[i] << "\n";
  }
}
```

The setter of a Swift subscript is bridged over as method named `setElementAtIndex` . It takes in the index parameter and a new value that’s being set. This how you would invoke the subscript setter for a Swift `Array`:

```
#include "Swift-Swift.h"

void updateArrayElement(swift::Array<swift::String> &elements) {
  elements.setElementAtIndex(0, "hello world!");
}
```

### Mutating Subscript Values

Swift allows you to mutate a value that’s yielded by the subscript. For example, you can append an element to an array inside of another array by using the subscript operator:

```
// Swift module 'Matrix'
func appendColumn(to matrix: inout [[Int]], value: Int) {
  for rowIndex in matrix.indices() {
    matrix[rowIndex].append(value)
  }
}
```

The generated C++ interface allows you to mutate a subscript value using a `mutateElementAtIndex` method, which takes in a C++ lambda that receives a reference to the underlying value that can be safely mutated within the lambda:

```
#include "Matrix-Swift.h"

void appendColumn(swift::Array<swift::Array<int>> &matrix, swift::Int value) {
  for (auto rowIndex : matrix.indices()) {
    elements.mutateElementAtIndex(rowIndex, [](auto &row) {
      row.append(value);
    });
  }
}
```

It’s illegal to escape the passed reference to the value from the lambda, as that can create a dangling reference in your program.

Open Questions:

* Bridging over overloaded subscripts.

## Using Swift Optional Values

An optional type represents a value that may be absent. Swift’s optional type can be used from C++ using the `swift::Optional` class template. It must be instantiated with a C++ type that represents some Swift type.

### Constructing an Optional

The `swift::Optional` class provides a default constructor that can be used to initialize it to `none`:

```
auto x = swift::Optional<int>();         // x is none
```

The optional can be initialized to be `Some` using a constructor which takes the value that should be stored in the optional:

```
swift::Optional<int> y = 0;              // y is some(0)
```

The optional class also provides a constructor that receives `nullptr_t` , so that it can be initialized from a `nullptr`, similar as to how you could initialize an optional from `nil` in Swift:

```
swift::Optional<double> a = nullptr;     // a is none
```

An alternative constructor can receive `nullopt_t` type, so that it can be initialized from `nullopt`, just like an `std::optional`:

```
swift::Optional<float> b = std::nullopt; // b is none
```

### Checking If an Optional Has Value

The `swift::Optional` class provides an explicit `operator bool` that be used to check if it contains a value using an `if` statement:

```
void printOptionalInt(const swift::Optional<int> &x) {
  if (x) {
    std::cout << ".some(" << x.value() << ")";
  } else {
    std::cout << ".none";
  }
}
```

You can also use the `hasValue` member function to check if it has a value as well.

### Extracting Value From an Optional

The `swift::Optional` class provides a `value` member function that can be used to extract the value from the optional. The C++ dereference operator `*`  can also be used to extract the stored value:

```
void getXOrDefault(const swift::Optional<int> &x) {
  return x.hasValue() ? *x : 42;
}
```

It’s illegal to try to extract a value from an optional when it has no value. A fatal error will be reported at runtime if one attempts to do that:

```
swift::Optional<int> x = nullptr;
std::cout << x.value() << "\n";
// Fatal error: Unexpectedly found nil while unwrapping an Optional value
```

### Mutating Value In an Optional

Swift provides optional chaining syntax that allows you to invoke mutating methods and property accessors on the stored value in a convenient manner:

```
func getXPerhaps() -> [Int]? { ... }

var x = getXPerhaps()
x?.append(42);  // append `42` to x when it's not nil
```

The C++ interface for `Optional` provides a similar mutation mechanism, where the mutation occurs only when an optional has a value in it. The provided  `withMutableValue` method allows you to pass a lambda that receives a reference to the underlying value that can be safely mutated within the lambda:

```
swift::Optional<swift::Array<swift::Int>> x = getXPerhaps();
x.withMutableValue([](auto &val) {
  // append `42` to the array x only when x is not nil
  val.append(42);
});
```

It’s illegal to escape the passed reference to the value from the lambda, as that can create a dangling reference in your program.

## Extensions

Swift extensions can be used to add new functionality to an existing class, structure, enumeration or a protocol in Swift. The C++ interface generator in the Swift compiler is capable of exposing an extension for a type or a protocol that’s defined in the same Swift module as the type/protocol itself. An extension that’s exposed to C++ can add the following members to the C++ class that represents a Swift type in the generated C++ interface:

* Getter and setter methods that expose computed instance or type properties added in the extension
* Instance and static methods that expose Swift methods added in the extension
* Static `init` methods that expose new initializers added in the extension
* Subscript operator and `setElementAtIndex` method that expose subscripts added in the extension
* Nested types added in the extension


**Note:** C++ does not have a language feature that would allow us to represent Swift extensions in their full fidelity. This is why the current implementation of the C++ interface generator in the Swift compiler only lets us expose extensions defined in the same module as the type that’s being extended.

### Accessing Extension Members

The exposed extension members are added to the C++ class that corresponds to the underlying Swift type. For example, the following extensions for a Swift type:

```
// Swift module 'Geometry'
struct Rect {
  var x, y, width, height: Double
}

extension Rect {
  init(size: Int) {
    self.init(x: 0, y: 0, width: size, height: size)
  }
  
  func squareThatFits() -> Rect {
    let size = max(width, height)
    return Rect(x: x, y: y, width: size, height: size)
  }
}

extension Rect: `CustomDebugStringConvertible` {
  var debugDescription: String {
    return ""
  }
}
```

Are exposed in the C++ `class` Rect, as per the sample interface below:

```
// C++ interface for 'Geometry'

class Rect {
public:
  // init(x:,y:,width:,height:)
  static Rect init(double x, double y, double width, double height);
  
  // init(size:)
  static Rect init(double size);
  
  Rect squareThatFits() const { ... }
  
  swift::String getDebugDescription() const { ... }
};
```

### Protocol Extensions

Swift protocols can be extended to provide method, initializer, subscript, and computed property implementations to the conforming types. The exposed members from such a protocol extension are added to the C++ class that corresponds to the underlying Swift type. For example, if `Rect` receives a conformance for `Shape` like below:

```
`protocol ``Shape`` ``{`
`  ``var`` area``:`` double`` ``{`` ``get`` ``}`
`}`
`extension ``Rect``:` Shape {
  var area: double { width * height }
}
extension Shape {
  func fits(inArea otherArea: double) -> Bool {
    area < otherArea
  }
}
```

The members from the extension of `Shape` are then added to the C++ class that corresponds to the `Rect` Swift structure:

```
// C++ interface for 'Geometry'

class Rect {
public:
  ...
  
  bool fits(double inArea) const { ... }
};
```

A protocol extension need to be in the same Swift module as the type that conforms to such protocol in order for the extension to get exposed in the C++ interface for the module.

## Generics

Swift’s generics allow programmer to write reusable functions and types that can work with any type, subject to any requirements that are specified by the programmer. C++ templates provide similar facilities for generic programming in C++. While Swift’s generics and C++ templates look similar, there are some important differences between them:

* Generic Swift functions and types are type checked at their definition using their stated requirements. C++ templates type check the generic code only after a template is specialized for a concrete type.
* Generic Swift functions and types provide generic implementation of their generic code, that can work with any type that conforms to their stated requirements. C++ templates, however, **do not** provide generic implementation of C++ functions or classes, as they only provide concrete implementations that operate on specific types that get generated whenever a C++ template is instantiated.

Even though C++ templates have different semantics than Swift generics, they are used in the generated C++ interface to provide type parameters for Swift functions or types that are then passed to Swift generic code. A generic Swift function or a generic Swift type is represented using a C++ function template, or a C++ class template, with certain constraints on the template parameters. The constraints are checked at compile time in C++ using `requires` in C++20 , or `enabled_if` when compiling using an older C++ standard.

Generic Swift code that’s invoked from C++ always goes through Swift’s generic codepath. A programmer that’s calling Swift generic APIs from C++ should keep that in mind, as the generic Swift code is most likely going to be less performant than a comparable C++ code generated by a template instantiation, as the C++ code is specialized for a specific type instead of being generic.

### Calling Generic Functions from C++

A generic function is represented using a function template in C++. The template type parameters must represent a type that is usable from a generic context in Swift, and must conform to any other generic constraints that are specified in Swift. These requirements are verified at compile time by the template requirements specified alongside the C++ function.

A generic function can be called from C++ by calling the C++ function template that represents it. For example, the generic function `swapTwoValues` with one type parameter `T` :

```
// Swift module 'Swapper'
func swapTwoValues<T>(_ a: inout T, _ b: inout T) {
  ...
}
```

Gets exposed to C++ via the following C++ function template:

```
template<typename T>
void swapTwoValues(T &a, T& b)
  requires swift::isUsableInGenericContext<T> {
  ...
}
```

And can then be called from C++ just like any other Swift function, as long as `T` is a type that can be used in a generic context in Swift:

```
#include "Swapper-Swift.h"

int main() {
  int x, y;
  Swapper::swapTwoValues(x, y); // ok.

  std::string s1, s2;
  Swapper::swapTwoValues(s1, s2);
  // error: no matching function for call to 'Swapper::swapTwoValues'
  // `because 'swift::isUsableInGenericContext<...>' evaluated to false`
  return 0;
}
```

When compiling in C++ 17 mode, the C++ function template relies on `std::enable_if` to verify that `T` is a type that can be used in a generic context instead of `requires`:

```
template<typename T,
         typename = std::enable_if_t<swift::isUsableInGenericContext<T>>>
void swapTwoValues(T& a, T& b)
```

Generic methods from Swift types are represented using a member function template in C++. They must obey the same requirements as Swift generic functions as well.

### Using Generic Types

A generic Swift structure, enumeration or class is represented using a class template in C++. The template type parameters must represent a type that is usable from a generic context in Swift, and must conform to any other generic constraints that are specified in Swift. These requirements are verified at compile time by the template requirements specified alongside the C++ class.

A generic Swift type can be used in C++ by specifying its class name and type parameters using the C++ template syntax. For example, the generic structure `Stack` with one type parameter `Element` :

```
// Swift module 'Datastructures'
struct Stack<Element> {
  mutating func push(_ item: Element) {
    ...
  }
  ...
}
```

Can then be used in C++ just like a C++ class template:

```
#include "Datastructures-Swift.h"

void useSwiftStack() {
  Datastructures::Stack<int> intStack;
  intStack.push(22);
}
```

It’s illegal to instantiate a class template for a Swift generic type like `Stack` with a type parameter that can’t be represented in a generic context in Swift. The compiler will verify that at compile-time by checking the constraints specified in the `requires` clause of the class template:

```
// Snippet from Datastructures-Swift.h
template<class Element>
requires swift::isUsableInGenericContext<Element>
class Stack {
  ...
};

// C++ use site
#include "Datastructures-Swift.h"

void useSwiftStackIncorrectly() {
  Datastructures::Stack<std::string> cxxStringStack;
  // error: constraints not satisfied for class template 'Stack'
  // note: because 'swift::isUsableInGenericContext<...>' evaluated to false
}
```

**Open Questions:**

* How do the opaque layout type type parameters that affect structs layout work - do they need template specializations, or can we do this with constexpr if - they need to be boxed?

### Generic Type Constraints

Swift programers can specify type constraints on the types that can be used with generic functions and generic types. These constraints are exposed to C++‘s type system through a set of requirements that must be satisfied by the C++ function or class template that represents a Swift generic function or type. They are verified in C++ at compile-time to ensure that the program is not invoking generic Swift code with types that don’t satisfy the specified constraints.

A generic constraint that specifies that a generic type parameter must conform to a particular protocol or protocol composition is verified using a `swift::conformsTo` type trait in C++. For example, the following generic function with a `Comparable` protocol constraint on `T`:

```
// Swift module 'MyModule'
func isWithinRange<T: Comparable>(_ value: T, lowerBound: T, upperBound: T) -> Bool {
  ...
}
```

Gets exposed to C++ via the following C++ function template with a `requires` clause that verifies conformance of the C++ type that represents some Swift type:

```
template<typename T>
bool isWithinRange(const T& value, const T& lowerBound, const T& upperBound)
  requires swift::isUsableInGenericContext<T> &&
           swift::conformsTo<T, swift::Comparable>
```

And can then be called from C++ as long as the Swift type that’s being represented by the C++ type `T` actually conforms to `Comparable` in Swift:

```
#include "MyModule-Swift.h"

int main() {
  MyModule::isWithinRange(1, 0, 2);
  return 0;
}
```

It’s illegal to instantiate a template with such a requirement when the template type parameter does not conform to `Comparable` in Swift. The compiler will verify that at compile-time by checking the template constraints:

```
void useWithinRangeWithCustomSwiftType() {
  MyModule::isWithinRange<MyModule::SomeSwiftStruct>({}, {}, {});
  // error: no matching function for call to 'MyModule::isWithinRange'
  // because 'swift::conformsTo<MyModule::SomeSwiftStruct, swift::Comparable>' evaluated to false
}
```


**TODO:** Inherit from a specific class constraint

### Extensions with a Generic Where Clause

Swift extensions can use a generic `where` clause to limit the extension to types that match certain generic constraints. The members of such extensions get exposed to C++ as described in the “Extensions” section above. These exposed members receive additional template requirements in C++ to ensure that they are available only from a C++ class template that satisfies the template requirements imposed by the generic `where` clause of the extension.

For example, an extension to the generic `Stack` structure from the previous example:

```
extension Stack where Element: Equatable {
  `func isTop(_ item: Element) -> Bool {`
    ...
  }
}
```

Gets exposed to C++ inside of the `Stack` class template with the added member constraint which validates that the C++ `Element` type represents a Swift type that conforms to `Equatable` :

```
template<class Element>
requires swift::isUsableInGenericContext<Element>
class Stack {
  ...

  bool isTop(const Element &) const
    requires swift::conformsTo<Element, swift::Equatable> {
    ...
  }
};
```

It’s illegal to access extension members in C++ class templates that don’t satisfy the where clause imposed by such an extension. The compiler will verify that at compile-time by enforcing the requirements on the member:

```
void useStackInCxx() {
  Datastructures::Stack<MyModule::SomeSwiftStruct> stack;

  stack.push(MyModule::SomeSwiftStruct()); // ok
  
  stack.isTop(MyModule::SomeSwiftStruct());
  // error: invalid reference to function 'isTop': constraints not satisfied
  // note: because 'swift::conformsTo<MyModule::SomeSwiftStruct, swift::Equatable>' evaluated to false
}
```

**TODO:** Support nested types inside of generic where clause extensions. - add a requires on them.
**TODO:** Add an example for Protocol Extensions With Contextual Where Clauses>


## Using Swift’s Standard Library Types

### Using String

String conforms to `StringLiteralConvertible` , so you can implicitly construct instances of `swift::String` directly from a C++ string literal, or any `const char *` C string:

```
swift::String string = "Hello world";

string.hasPrefix("Hello"); // Implicit construction of swift::String.
```

You can convert a `swift::String` to a `std::string` using `std::to_string`:

```
void printSwiftString(const swift::String &swStr) {
  std::string str = std::to_string(swStr);
  std::cout << "swift string is " << str << "\n";
}
```

You can convert an `std::string` into a `swift::String` using an explicit constructor:

```
void setSwiftString(swift::String &other, const std::string &str) {
  other = swift::String(str);
}
```

Open questions:

* How do the  `StringLiteralConvertible` rules work in practice?
* What happens when String.init fails for a literal? (fatalError most likely). Check what Swift does, does it ever allow an invalid utf8 sequence, and will the actual initializer fail at runtime?
* String.init - implicit initializer from C++ string is problematic potentially. Make Pointers have to go through force casting? (I think that’s probably not a problem - C++ string literal type just won’t conform to swift::isUsableInGenericContext)

### Using Array

A Swift array type can be used from C++ using the `swift::Array` class template. It must be instantiated with a C++ type that represents a Swift type.

An array can be initialized using a C++ initializer list because it conforms to `ArrayLiteralConvertible`:

```
swift::Array<int> intArray = {};

swift::Array<swift::String> languages = { "Swift", "C++", "Objective-C" };
```

You can iterate over the array elements using a `for` loop:

```
for (auto language : languages)
  std::cout << std::to_string(language) << "\n";
```

You can modify the elements in the array using the `setElementAtIndex` member function:

```
for (size_t i = 1; i < languages.getCount(); ++i)
  languages.setElementAtIndex(i, languages[i] + languages[i - 1]);
```

You can convert a `swift::Array` to a `std::vector` using the following explicit constructor of `std::vector`:

```
auto cxxVector = std::vector<int>(intArray.begin(), intArray.end());
```

This constructor copies over the elements from the Swift array into the constructed C++ vector.

You can also convert a vector to a `swift::Array` using the following explicit constructor of `swift::Array`:

```
auto swiftIntArray = swift::Array<int>(cxxVector);
```

This constructor copies over the elements from the C++ vector into the constructed Swift array.

## Appendix A: Type Traits That Model Swift’s Type System In C++

The C++ interface that’s generated by the Swift compiler for a Swift module uses a number of C++ type traits that can be used to query information about Swift’s type system from C++. These type traits are listed in this section.

### swift::isUsableInGenericContext

```
template<class T>
inline constexpr const bool swift::isUsableInGenericContext
```

This type trait can be used to check if a type can be passed to a generic Swift function or used as a generic type parameter for a Swift type. It  evaluates to `true` in the following cases:

* When `T` is a primitive type like `int` , `float`, `swift::Int`, etc. that has a corresponding Swift primitive type.
* When T is a class or a class template that acts a proxy for a Swift type and is defined in the C++ interface generated by the Swift compiler for a Swift module.
* **TODO:** Objective-C ARC pointers , etc?

The following example illustrates how this type trait evaluates to true for types that have a Swift representation, but to false for regular C++ types:

```
static_assert(swift::isUsableInGenericContext<int> == true);
static_assert(swift::isUsableInGenericContext<swift::String> == true);

static_assert(swift::isUsableInGenericContext<std::string> == false);
```

### swift::conformsTo

```
template<class T, class P>
inline constexpr const bool swift::conformsTo
```

This type trait evaluates to true when a specific Swift type that is being proxied by the given C++ type `T` conforms to a Swift protocol that’s being proxied by the given C++ class `P`.
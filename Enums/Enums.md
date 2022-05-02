# Enums

## Boiler plate code
```swift
struct Point: Equatable, Codable, Hashable {
    
  var x, y: Double
}

struct Size: Equatable, Codable {
  var width, height: Double
}

struct Rectangle: Equatable {
  var origin: Point
  var size: Size
}

extension Point {
    static var zero: Self {
        Point(x: 0, y: 0)
    }
}
```

## Enumerations
```swift
enum Quadrant {
    case i, ii, iii, iv
    
    init?(_ point: Point) {
        guard !point.x.isZero && !point.y.isZero else {
            return nil
        }
        
        switch(point.x.sign, point.y.sign ) {
        
        case (.plus, .plus):
            self = .i
        case (.minus, .plus):
            self = .ii
        case (.minus, .minus):
            self = .iii
        case (.plus, .minus):
            self = .iv
        }
    }
}

Quadrant(Point(x: 10, y: -3)) // evaluates to .iv
Quadrant(.zero)
```

## Associated Value With Enums
```swift
enum Shape {
    case point(Point)
    case segment(start: Point, end: Point)
    case circle(center: Point, radius: Double)
    case rectangle(Rectangle)
}

let shape: Shape = .rectangle(Rectangle(origin: Point(x: 3, y: 4), size: Size(width: 3, height: 4)))


switch shape {
    
case .point(let point):
    print(point.y)
case .segment(start: let start, end: let end):
    break
case .circle(center: let center, radius: let radius):
    break
case .rectangle(let rect):
    print(rect.size.width)
    break
}
```

## Using RawRepresentable
```swift
enum Bevarage: String, CaseIterable {
    case coffee = "Coffee", tea = "Tea", juice = "Juice"
}

let bevarage = Bevarage(rawValue: "Coffee")
```

## Iterate over enum
```swift
Bevarage.allCases.forEach { caseType in
//    print(caseType.rawValue)
}
```

## RawRepresentable directly to create simple checked types
```swift
struct Email: RawRepresentable {
    var rawValue: String
    
    init?(rawValue: String) {
        guard rawValue.contains("@") else { return nil }
        self.rawValue = rawValue
    }
}

// Extend String to Error for throwing String as Error
extension String: Error { }

func send(message: String, to recipient: Email?) throws {

    guard recipient != nil else {
        throw "Invalid email obtained"
    }
}

// Driver Code
do {
    try send(message: "Hello", to: Email(rawValue: "afaqahmad6296gmail.com"))
} catch {
    print("Failed to send message with error:", error)
}

```

## How to use enum for custom error
```swift
private enum AppError: String {
    case unkown = "An unknown error occured"
}

extension AppError: LocalizedError {
    public var errorDescription: String? {
        return NSLocalizedString(self.rawValue, comment: "AppError")
    }
}
```
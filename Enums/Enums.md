# Enums

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
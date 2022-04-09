# CoreImage Pipline 

# Input
## How to create CIImage
> How to create `ciimage` that's the object that represents image within the `CoreImage` Framework and behaves as a model object, very much like a `CALayer` from `CoreAnimation`
```swift
// Create url to image
let url = Bundle.main.url(forResource: "IMG_123", withExtension: "heic")
let cImage = CIImage(contentsOf: url!)

// For respecting the orientation pass options
let image = CIImage(contentsOf: url!, options: [CIImageOption.applyOrientationProperty : true])

// For scaling the size down call transform method on CIImage
let image = CIImage(contentsOf: url!, options: [CIImageOption.applyOrientationProperty : true])?.transformed(by: CGAffineTransform(scaleX: 0.125, y: 0.125))
```

### Exploring more options
```swift
let options = [
    CIImageOption.applyOrientationProperty: true,
    CIImageOption.auxiliaryDepth: true //  maps that approximated using the sterioscopic vision camera on the back of the phone
]

// Depth map
let depthImage = CIImage(contentsOf: url!, options: options)
```

## Filtering
> One of the key things about the **CoreImage** is it's pipeline is incredibly efficient. Taken indvidually, each of the filters has a kernal, that's the bit that actually does the image processing and that is built in Metal. So that gets pushed on to the GPU and operated incredibly quickly.

### Stringly typed filters
```swift
let filter = CIFilter(name: "CIVibrance", parameters: ["inputImage" : image,
                                                       "inputAmount": 0.9
                                                      ])
filter?.outputImage
```

### Strongly typed filters
> import import CoreImage.CIFilterBuiltins
```swift
let filter = CIFilter.vibrance()
filter.inputImage = image
filter.amount = 0.9
filter.outputImage
```

### Filters Chain
```swift
let filter2 = CIFilter.gloom()
filter2.radius = 10
filter2.intensity = 0.7
filter2.inputImage = filter.outputImage

let filter3 = CIFilter.vignette()
filter3.radius = 2
filter3.intensity = 0.9
filter3.inputImage = filter2.outputImage
filter3.outputImage
```

> Convolutional filters like gloom and guassian blur increase the size of image, so to get back on original size use cropped method as below
```swift
let finishedImage = filter3.outputImage?.cropped(to: image!.extent)
```

## Saving/Outputing Images
```swift
let context = CIContext()
let jpegUrl = URL(fileURLWithPath: "fileOutput.jpeg")

try! context.writeJPEGRepresentation(of: finishedImage, to: jpegUrl, colorSpace: .sRGB)
```
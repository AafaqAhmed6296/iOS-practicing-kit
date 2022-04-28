## Extracing Images Mattes

```swift
let url = Bundle.main.url(forResource: "IMG_5276", withExtension: "HEIC")!
let image = CIImage(contentsOf: url, options: [CIImageOption.applyOrientationProperty: true])

var options = [
  CIImageOption.applyOrientationProperty: true,
]

options[.auxiliaryPortraitEffectsMatte] = true
let matte = CIImage(contentsOf: url, options: options)!

let resized = image?.transformed(by: CGAffineTransform(scaleX: 0.125, y: 0.125))
let mattResized = matte.transformed(by: CGAffineTransform(scaleX: 0.25, y: 0.25))

// Invert the color so background will be blured not the foreground
let invertFilter = CIFilter.colorInvert()
invertFilter.inputImage = mattResized

let filter = CIFilter.maskedVariableBlur()
filter.inputImage = resized
filter.mask = invertFilter.outputImage
filter.radius = 50
filter.outputImage
```


## Applying a Masked Filter
```swift

// Manually Filtering the background

let bgVibrance = CIFilter.vibrance()
bgVibrance.amount = -0.7
bgVibrance.inputImage = resized

let bgDiscBlur = CIFilter.discBlur()
bgDiscBlur.radius = 8
bgDiscBlur.inputImage = bgVibrance.outputImage

let bgVignette = CIFilter.vignette()
bgVignette.radius = 20
bgVignette.intensity = 0.7
bgVignette.inputImage = bgDiscBlur.outputImage?.cropped(to: resized!.extent)

let background = bgVignette.outputImage


// Compositing images

let fgVibrance = CIFilter.vibrance()
fgVibrance.inputImage = resized
fgVibrance.amount = 0.7

let foreground = fgVibrance.outputImage

// Blend means joining two things together
let compositeFilter = CIFilter.blendWithMask()
compositeFilter.inputImage = foreground
compositeFilter.backgroundImage = background
compositeFilter.maskImage = mattResized

let composite = compositeFilter.outputImage
```

> * You can use this blendWithMask filter with your own filter chain and using these portait mask build up some really quite impressive photography.
> * In iOS 13+ in addition to portrait effect matte, you also get teeth, skin and hair mattes so you can use segmentation of the images to apply some special effects to just hair for example.

## Generator Filters

### Sunbeam Generator

```swift
// Sunbeam generator
let sunbeamGen = CIFilter.sunbeamsGenerator()
sunbeamGen.sunRadius = 25
sunbeamGen.striationStrength = 0.5

let sun = sunbeamGen.outputImage?.transformed(by: CGAffineTransform(translationX: -40, y: 300))

let sunCompositFilter = CIFilter.lightenBlendMode()
sunCompositFilter.backgroundImage = resized
sunCompositFilter.inputImage = sun

// Use this image before starting filter chaining
let withSun = sunCompositFilter.outputImage?.cropped(to: resized!.extent)
```

### QRCode Generator
```swift

let qrGen = CIFilter.qrCodeGenerator()
qrGen.message = "Hello world".data(using: String.Encoding.ascii)!

qrGen.outputImage?.transformed(by: CGAffineTransform(scaleX: 3, y: 3))
```
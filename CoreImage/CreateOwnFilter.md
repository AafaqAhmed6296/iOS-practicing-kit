# Create Custom Kernels and Filters.
> With CoreImage you can create your own CIFilters and that means building your own filter ***Kernal***. The Kernal is the Core of the filter is the bit that actually processess on a pixel by pixel. In order to keep kernal really efficient they are executed on *GPU* and therefore we need to write them in. There are three types of Kernal that you can create in CoreImage.

> * Color Kernal
>   * A Color kernal recieves a single pixel as input and its output will be the new value of that pixel.
> 
> * Warp Kernal
>   * A Warp Kernal recieves a single position of the pixel as its input, and its output will be where that position should be in the ouput image.
> 
> * General Kernal
>   * Genereal Kernel do pretty much what you want. e.g. Convolution. The kernel will be called for every single pixel in the output image and you have got acess to the entirety of the input image should you desire.

**The difference is the specificiation of the input and output**


# Create a Metal Kernel
## Compile settings for metal kernel
1. Selet the project in navigator
2. Select build settings 
3. Select All settings
4. Search for metal, you will find `Other Metal Compiler Flags` and add custom flag `-fcikernel`. That means when it compiles metal it will be sure to inclue CIKernel library.
> Make sure to link.
5. Search for `Other Metal Linker flags` and add custom flag `-cikernel`, hit cmd+B that's all!!

---
## Passthrough Kernel

> Passthrough kernel does nothing. Give it a pixel and does return the same pixel as it is. Below implementation is as Color kernel but it can be of other types too.

```C++
/*
> Metal shader language is variant of C++
1. Create Metal File
2. import CoreImage header 
*/

#include <CoreImage/CoreImage.h>

// Since we are publishing something we need to make it extern C

extern "C" {
    // To acces the functionality we need our kernel insert in coreimage namespace

    // Passthrough KERNEL
    // sample_t is of type float4 too.
    float4 passthroughFilterKernel(sample_t s) {
      return s; // just pass it back
    }
}
```

## Wrap a kernel in a Filter

```swift
import CoreImage

/// Filter class
class PassthroughFilter: CIFilter {
  private lazy var kernel: CIKernel = {
      // URL will be always same for every Metal kernel because by default metal will be compiler in to file called default.metallib.
    guard
      let url = Bundle.main.url(forResource: "default", withExtension: "metallib"),
      let data = try? Data(contentsOf: url) else {
        fatalError("Unable to load metallib")
    }
    
    // Function name is the name that we gave the kernel inside the metal code.
    guard let kernel = try? CIColorKernel(functionName: "passthroughFilterKernel", fromMetalLibraryData: data) else {
      fatalError("Unable to create the CIColorKernel for passthoughFilterKernel")
    }
    
    return kernel
  }()
  
  var inputImage: CIImage?
  
  // When creating your own filter its your responsibilty to override output image property to perform filtering operation.
  override var outputImage: CIImage? {
    guard let inputImage = inputImage else { return .none }
    
    return kernel.apply(extent: inputImage.extent,
                        roiCallback: { (index, rect) -> CGRect in
        // For us here it's really simple the area of the input we need is just the area of output that we are working on.
                          return rect
    }, arguments: [inputImage])
  }
}
```

> kernal.apply method args
> * `extent` : how much of the input image do we actually pass to the kernel for filtering. in most cases entire image
> * `roiCallBack`: ROI stands for region of interest. This a closure, you get passed and index and CGRect and you have to return a CGRect.
>   * The purpose of this closure is to answer you question from CoreImage. It's saying I'v got this rectangle of the outputImage that I want to generate. What area of the inputImage do I need provide access to? It does this because of the way it **slices up** images to pass them over to **GPU** for optimal processing. If you have enormous image it won't necessarily all be in the GPU at once. CoreImage will slice it up intelligentaly as pass it across in little pieces. This gives you the opportunity to say, don't forget that when I am doing this bit of the image, make sure you provide me this bit of image at the same time.
> * `arguments`: These are the ordered parameters to the function you wrote the metal kernel (passthroughFilterKernel in this case) in metal. 

---
---

# Create a Color Space Transform

<p> What is color space? Color Spaces in digital devices are usually RGB. That means in order to produce full color picture you would have a Red, green and blue channel and this is the Color Space CoreImage works with. <p>

However, for various reasons, we use other Color Spaces one of them is **YCbCr**. Where Y is the intensity (Grayscale), Cb is the blue color difference and Cr is the red color difference.

> One of the reason that you might want to use this Color space is because it compresses better. The majority of information's actually carried in Intensity (Y) channel, which mean you can downsample the color channels and up sample them at the other end of the transmission and actually you don't actually lose any perceptual quality. 
---
> You will also find that the many filters work well on the intensity channel and don't works well on color channel, and tha't the reason we can do conversion of Color Space!

#### More Information about Color Space and Conversion.
1. [Color Space](https://www.dynamsoft.com/blog/insights/image-processing/image-processing-101-color-models/#:~:text=A%20color%20space%20identifies%20a,on%20the%20RGB%20color%20model.)
2. [Conversion of Color Space](https://www.dynamsoft.com/blog/insights/image-processing/image-processing-101-color-space-conversion/)

<p> Since CoreImage does not natively understand YCbCr, we will approximate it by using the red channel as the intensity channel the green as the Cb and the blue as Cr channel.<p>

``` 
Formuala for RGB to YCbCr conversion.
Y = 0.299R + 0.587G + 0.114B
Cb= (B-Y)*0.565
Cr= (R-Y)*0.713
```

## RGBToYCbCr and YCbCrToRGB Filter Kernel

```C++
#include <metal_stdlib>
using namespace metal;

#define K_R 0.2126
#define K_G 0.7152
#define K_B 0.0722

#include <CoreImage/CoreImage.h>

extern "C" {
    namespace coreimage {

        // Utilities
        // Formula for YCbCr to RGB Conversion
        float4 ycbcrToRgb(float4 pixel) {
            float y  = pixel.r;
            float cb = pixel.g;
            float cr = pixel.b;
            float blue  = 2 * cb * (1 - K_B) + y;
            float red   = 2 * cr * (1 - K_R) + y;
            float green = (y - K_R * red - K_B * blue) / K_G;
            return float4(red, green, blue, pixel.a);
        }
        
        // TODO
        float intensityY(float4 pixel) {
            return (pixel.r * K_R) + (pixel.g * K_G) + (pixel.b * K_B);
        }
        
        float blueDifference(float4 pixel) {
            return (pixel.b - intensityY(pixel)) * 0.565;
        }
        
        float redDifference(float4 pixel) {
            return (pixel.r - intensityY(pixel)) * 0.713;
        }
        
        // KERNEL
        float4 rgbToYcbcrFilterKernal(sample_t s) {
            float y = intensityY(s);
            float cb = blueDifference(s);
            float cr = redDifference(s);
            return float4(y, cb, cr, s.a);
        }

        // KERNEL  
        float4 ycbcrToRgbFilterKernel(sample_t s) {
            return ycbcrToRgb(s);
        }
        
    }
}
```

## Wrap Kernels in Filters.

> Wrap up kernel inside Filters as we done for Passthrough Filter Kernal above.

### Results after Conversion from RGB to YCbCr
![RGBToYCbCr]()

### Results after Reverse Operation
![YCbCrToRGB]()


# Create BilateralFiler
--- 
## Create BilateralKernal

```C++
// KERNEL
    float4 bilateralFilterKernel(sampler src, float kernelRadius_f, float sigmaSpatial, float sigmaRange) {
      float4 input = src.sample(src.coord());
      float3 premultipliedRunningSum = 0;
      float weightRunningSum = 0;
      int kernelRadius = int(kernelRadius_f);
      
      for (int i = -kernelRadius; i <= kernelRadius; i++) {
        for (int j = -kernelRadius; j <= kernelRadius; j++) {
          float4 referenceInput = src.sample(src.coord() + float2(i, j) / src.size());
          float weight = exp ( - (i * i + j * j) / (2 * sigmaSpatial * sigmaSpatial)
                               - pow((input.r - referenceInput.r), 2.0) / (2 * sigmaRange * sigmaRange));
          weightRunningSum += weight;
          premultipliedRunningSum += weight * referenceInput.rgb;
        }
      }
      
      return float4(premultipliedRunningSum / weightRunningSum, input.a);
    }
```

## Use Inside Filter

```swift
   

class BilateralFilter: CIFilter {
    private lazy var kernel: CIKernel = {
        guard
            let url = Bundle.main.url(forResource: "default", withExtension: "metallib"),
            let data = try? Data(contentsOf: url) else {
                fatalError("Unable to load metallib")
            }
        
        // TODO
        guard let kernel = try? CIKernel(functionName: "bilateralFilterKernel", fromMetalLibraryData: data) else {
            fatalError("Failed to load bilateralFilterKernel")
        }
        
        return kernel
    }()
    
    var inputImage: CIImage?
    var kernelRadius: Float = 15
    var sigmaSpatial: Float = 13
    var sigmaRange: Float = 0.1
    
    
    override var outputImage: CIImage? {
        guard let inputImage = inputImage else {
            print("fall here")
            return .none }
        
        // TODO
        return kernel.apply(extent: inputImage.extent, roiCallback: { (index, rect) -> CGRect in
            let out = rect.insetBy(dx: CGFloat(-self.kernelRadius), dy: CGFloat(-self.kernelRadius)).intersection(inputImage.extent)
            return out
        }, arguments: [inputImage, kernelRadius, sigmaSpatial, sigmaRange])
    }
}

```

> Above Filter is Convolutional and works better on intensity channel. So we can first switch to YCbCr Color space using filtering chain and then after applying bilateral filter switch back to RGB Color Space

### Final Bilateral Filter.
![Bilaterla Filter]()
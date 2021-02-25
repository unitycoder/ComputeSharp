![](https://user-images.githubusercontent.com/10199417/108635546-3512ea00-7480-11eb-8172-99bc59f4eb6f.png)
![.NET](https://github.com/Sergio0694/ComputeSharp/workflows/.NET/badge.svg) [![NuGet](https://img.shields.io/nuget/v/ComputeSharp.svg)](https://www.nuget.org/packages/ComputeSharp/) [![NuGet](https://img.shields.io/nuget/dt/ComputeSharp.svg)](https://www.nuget.org/stats/packages/ComputeSharp?groupby=Version)

# What is it?

**ComputeSharp** is a .NET 5 library to run C# code in parallel on the GPU through DX12 and dynamically generated HLSL compute shaders. The available APIs let you access GPU devices, allocate GPU buffers and textures, move data between them and the RAM, write compute shaders entirely in C# and have them run on the GPU. The goal of this project is to make GPU computing easy to use for all .NET developers! 🚀

# Table of Contents

- [Installing from NuGet](#installing-from-nuget)
- [Quick start](#quick-start)
  - [Capturing variables](#capturing-variables)
  - [GPU resource types](#gpu-resource-types)
  - [HLSL intrinsics](#hlsl-intrinsics)
  - [Working with images](#working-with-images)
  - [Shader metaprogramming](#shader-metaprogramming)
- [Requirements](#requirements)
- [Sponsors](#sponsors)
- [Special thanks](#special-thanks)

# Installing from NuGet

To install **ComputeSharp**, run the following command in the **Package Manager Console**

```
Install-Package ComputeSharp
```

More details available [here](https://www.nuget.org/packages/ComputeSharp/).

# Quick start

**ComputeSharp** exposes a `Gpu` class that acts as entry point for all public APIs. The available `Gpu.Default` property that lets you access the main GPU device on the current machine, which can be used to allocate buffers and perform operations. If your machine doesn't have a supported GPU (or if it doesn't have a GPU at all), **ComputeSharp** will automatically create a [WARP device](https://docs.microsoft.com/windows/win32/direct3darticles/directx-warp) instead, which will still let you use the library normally, with shaders running on the CPU instead through an emulation layer. This means that you don't need to manually write a fallback path in case no GPU is available - **ComputeSharp** will automatically handle this for you.

Let's suppose we want to run a simple compute shader that multiplies all items in a target buffer by two. The first step is to create the GPU buffer and copy our data to it:

```csharp
// Get some sample data
int[] array = Enumerable.Range(0, 100).ToArray();

// Allocate a GPU buffer and copy the data to it.
// We want the shader to modify the items in-place, so we
// can allocate a single read-write buffer to work on.
using ReadWriteBuffer<float> buffer = Gpu.Default.AllocateReadWriteBuffer(array);
```

The `AllocateReadWriteBuffer` extension takes care of creating a `ReadWriteBuffer<T>` instance with the same size and contents of the input array. There are a number of overloads available as well, to create buffers of different types and with custom length.

Next, we need to define the GPU shader to run. To do this, we'll need to define a `struct` type implementing the `IComputeShader` interface. This type will contain the code we want to run on the GPU, as well as fields representing the values we want to capture and pass to the GPU (such as GPU resources, or arbitrary values we need). In this case, we only need to capture the buffer to work on, so the shader type will look like this:

```C#
[AutoConstructor]
public readonly struct MultiplyByTwo : IComputeShader
{
    public readonly ReadWriteBuffer<float> buffer;

    public void Execute()
    {
        buffer[ThreadIds.X] *= 2;
    }
}
```

We're using the `[AutoConstructor]` attribute included in **ComputeSharp**, which creates a constructor for our type automatically. The shader body is also using a special `ThreadIds` class, which is one of the available special classes to access dispatch parameters from within a shader body. In this case, `ThreadIds` lets us access the current invocation index for the shader, just like if we were accessing the classic `i` variable from within a `for` loop.

We can now finally run the GPU shader and copy the data back to our array:

```csharp
// Launch the shader
Gpu.Default.For(buffer.Length, new MyShader(buffer));

// Get the data back
buffer.CopyTo(array);
```

## Capturing variables

Shaders can store either GPU resources or custom values in their fields, so that they can be accessed when running on the GPU as well. This can be useful to pass some extra parameters to a shader (eg. some factor to multiply values by), that don't belong to a GPU buffer of their own. The captured variables need to be of a supported scalar or vector type so that they can be correctly used by the GPU shader in HLSL. Here is a list of the variable types currently supported by the library:

✅ .NET scalar types: `bool`, `int`, `uint`, `float`, `double`

✅ .NET vector types: `System.Numerics.Vector2`, `Vector3`, `Vector4`

✅ HLSL types: `Bool`, `Bool2`, `Bool3`, `Bool4`, `Float2`, `Float3`, `Float4`, `Int2`, `Int3`, `Int4`, `UInt2`, `Uint3`, etc.

✅ Custom `struct` types containing any of the types above, as well as other valid custom `struct` types

## GPU resource types

There are a number of extension APIs for the `GraphicsDevice` class that can be used to allocate GPU resources. Here is a breakdown of the main resource types that are available:

- `ReadWriteBuffer<T>`: this type can be viewed as the equivalent of the `T[]` type, and can contain a writeable sequence of any of the supported types mentioned above. It is very flexible and works well in most situations. If you're just getting started with this library and are not sure about what kind of buffer to use, this is usually a good choice.
- `ReadOnlyBuffer<T>`: this type represents a sequence of items that the GPU cannot write to. It is particularly useful to declare intent from within a compute shader and to avoid accidentally writing to data that is not supposed to change during the execution of a shader.
- `ConstantBuffer<T>`: this type is meant to be used for small sequences of items that never change during the execution of a shader. Compared to `ReadOnlyBuffer<T>` is has more constraints, but can benefit from better caching on the GPU side (it is recommended to verify with proper benchmarking that this type is appropriate to use). Items within a `ConstantBuffer<T>` instance are packed to 16 bytes, which helps the GPU to have a particularly fast access time to them, but the total size of the buffer is limited to around 64KB. Copying to and from this buffer can also have additional overhead, as the GPU needs to account for the possible padding for each item (as the 16 bytes alignment is not present on the CPU side). If you're in doubt about which buffer type to use, just use either `ReadOnlyBuffer<T>` or `ReadWriteBuffer<T>`, depending on whether or not you also need write access to that buffer on the GPU side.
- `ReadOnlyTexture2D<T>` and `ReadWriteTexture2D<T>`: these types represent a 2D texture with elements of a specific type. Note that textures are not just 2D arrays, and have additional characteristics and limitations. Items in a texture are stored with a tiled layout instead of with the classic row-major order that .NET `T[,]` arrays have, and this allows them to be extremely fast when accessing small areas of neighbouring items (due to better cache locality). This can offer a big performance speedup in operations that have a similar memory access pattern, such as blur effect or convolutions in general. Textures also have limitations in the type of items they can contain (eg. custom `struct` types are not supported), and you can check if a specific type is supported at runtime with the `GraphicsDevice.IsReadOnlyTexture2DSupportedForType<T>()` and `IsReadWriteTexture2DSupportedForType<T>()` methods.
- `ReadOnlyTexture3D<T>` and `ReadWriteTexture3D<T>`: these are just like 2D textures, but in 3 dimensions. The same characteristics and limitations apply, with the addition of the fact that the depth axis has a much smaller limit on the size it can have. The `GraphicsDevice.IsReadOnlyTexture3DSupportedForType<T>()` and `IsReadWriteTexture3DSupportedForType<T>()` methods can be used to check for type support at runtime.
- `ReadOnlyTexture2D<T, TPixel>` and `ReadWriteTexture2D<T, TPixel>`: these texture types are particularly useful when doing image processing from a compute shader, as they allow the CPU to perform pixel format conversion automatically. The two type parameters indicate the type of items in a texture when exposed on the CPU side (as `T` items) or on the GPU side (as `TPixel` items). The type parameter `T` can be any of the existing pixel formats available in **ComputeSharp** (such as `Rgba32` and `Bgra32`), while the `TPixel` parameter represents the type of elements the GPU will be working with (for instance, `Float4` for `Rgba32`). These texture types have a number of benefits such as lower memory usage and reduced overhead on the CPU (as there is no need to manually do the pixel format conversion when copying data back and forth from the GPU).
- `ReadOnlyTexture3D<T, TPixel>` and `ReadWriteTexture3D<T, TPixel>`: these types are just like their 2D equivalent types, with the same support for automatic pixel format conversion on the GPU side.
- `UploadBuffer<T>`: this type can be used in more advanced scenarios where performance is particularly important. It represents a buffer that can be copied directly to a GPU buffer (such as a `ReadOnlyBuffer<T>`), without the need to create a temporary transfer buffer to do so. If there is a large number of copy operations being performed to GPU buffers, it can be beneficial to create a single `UploadBuffer<T>` instance to load data to copy to the GPU, and execute the copy from there.
- `ReadBackBuffer<T>`: this type is another advanced buffer type that is analogous to `UploadBuffer<T>`, but that can be used to copy data back from a GPU buffer. In particular, this buffer can also be accessed quickly by the GPU when reading or writing data from it, so it can also be used for further processing of data on the CPU side, without the need to copy the data onto another buffer first (such as a .NET array).
- `UploadTexture2D<T>`, `UploadTexture3D<T>`, `ReadBackTexture2D<T>` and `ReadBackTexture3D<T>`: these types are conceptually similar to `UploadBuffer<T>` and `ReadBackBuffer<T>`, with the main difference being that they can be used to copy data back and forth from 2D and 3D textures respectively.

> **NOTE:** although the various APIs to allocate buffers are simply generic methods with a `T : unmanaged` constrain, they should only be used with C# types that are supported (see notes above). Additionally, the `bool` type should not be used in buffers due to C#/HLSL differences: use the `Bool` type instead (or just an `int` buffer).

## HLSL intrinsics

**ComputeSharp** offers support for all HLSL intrinsics that can be used from compute shaders. These are special functions (usually representing mathematical operations) that are optimized on the GPU and can execute very efficiently. Some of them can be mapped automatically from methods in the `System.Math` type (such as `Math.Sin` and `Math.Cos`), but if you want to have direct access to all of them you can just use the methods in the `Hlsl` class from a compute shader, and **ComputeSharp** will rewrite all those calls to use the native intrinsics in the compute shaders where those are used.

Here's an example of a shader that applies the [softmax function](https://en.wikipedia.org/wiki/Rectifier_(neural_networks)) to all the items in a given buffer:

```csharp
[AutoConstructor]
public readonly struct SoftmaxActivation : IComputeShader
{
    public readonly ReadWriteBuffer<float> buffer;
    public readonly float k;

    public void Execute(ThreadIds ids)
    {
        float exp = Hlsl.Exp(k * buffer[ThreadIds.X]);
        float log = Hlsl.Log(1 + exp);

        buffer[ThreadIds.X] = log / k;
    }
}
```

## Working with images

As mentioned in the [GPU resource types](#gpu-resource-types) paragraph, there are several texture types that are specialized to work on image pixels, such as `ReadWriteTexture2D<T, TPixel>`. Let's imagine we want to write a compute shader that applies some image processing filter: we will need to load an image (eg. using [ImageSharp](https://github.com/SixLabors/ImageSharp), or just the `System.Drawing` APIs) and then process it on the GPU, but without spending time on the CPU to convert pixels from a format such as RGBA32 to the normalized `float` values we want our shader to work on. We can do this by utilizing the `ReadWriteTexture2D<T, TPixel>` type as follows:

```csharp
// Load a bitmap from a specified path and then lock the pixel data.
// The locked pixels are in the BGRA32 format, which is the one we need.
var bitmap = new Bitmap("myImage.jpg");
var bitmapData = bitmap.LockBits(
    new Rectangle(0, 0, bitmap.Width, bitmap.Height),
    ImageLockMode.ReadWrite,
    PixelFormat.Format32bppArgb);

try
{
    // Create a span over the locked pixel data, of type Bgra32 (from ComputeSharp)
    var bitmapSpan = new Span<Bgra32>((Bgra32*)bitmapData.Scan0, bitmapData.Width * bitmapData.Height);

    // Allocate a 2D texture by directly reading from the pixel data
    using var texture = Gpu.Default.AllocateReadWriteTexture2D<Bgra32, Float4>(bitmapSpan, bitmap.Width, bitmap.Height);

    // Run our shader on the texture we just created
    Gpu.Default.For(bitmap.Width, bitmap.Height, new GaussianBlur(texture));

    // When the shader has completed, copy the processed texture back
    texture.CopyTo(bitmapSpan);
}
finally
{
    bitmap.UnlockBits(bitmapData);
}

bitmap.Save("myImage.jpg");
```

With the compute shader being like this:

```csharp
[AutoConstructor]
public readonly partial struct GaussianBlur : IComputeShader
{
    public readonly IReadWriteTexture2D<Float4> texture;

    // Other captured resources or values here...

    public void Execute()
    {
        // Our image processing logic here. In this example, we are just
        // applying a naive grayscale effect to all pixels in the image.
        Float3 rgb = texture[ThreadIds.XY].RGB;
        float avg = Hlsl.Dot(rgb, new(0.0722f, 0.7152f, 0.2126f));

        texture[ThreadIds.XY].RGB = avg;
    }
}
```

> **NOTE:** this is just an example to illustrate how these texture types can help with automatic pixel format conversion. You're free to use any library of choice to load and save image data, as well as to how to structure your compute shaders representing image effects. This is just one of the infinite possible effects that could be achieved by using **ComputeSharp**.

## Shader metaprogramming

One of the reasons why **ComputeSharp** compiles shaders at runtime is that it allows it to support a number of dynamic scenarios, including shader metaprogramming. What this means is that it's possible to have a shader that captures an instance of a `Delegate` type (provided the signature matches the supported types mentioned above), and reuse it with different methods. The library will automatically run different variations of the shaders depending on what methods they are capturing. Here is an example:

```csharp
// Define a function we want to use in our shader. Captured functions
// need to be static methods annotated with [ShaderAttribute].
[ShaderMethod]
public static float Square(float x) => x * x;

[AutoConstructor]
public readonly struct ApplyFunction : IComputeShader
{
    public readonly ReadWriteBuffer<float> buffer;
    public readonly Func<float, float> function;

    public void Execute()
    {
        buffer[ThreadIds.X] = function(buffer[ThreadIds.X]);
    }
}
```

Then we can simply create a new `Func<float, float>` instance and use that to construct a new instance of the `ApplyFunction` type, to execute that shader on a target buffer. **ComputeSharp** will build or reuse a version of that shader with the supplied captured function and execute it on the GPU:

```csharp
float[] array = LoadSampleData();

// Allocate the buffer and copy data to it
using ReadWriteBuffer<float> buffer = Gpu.Default.AllocateReadWriteBuffer(array);

// Run the shader
Gpu.Default.For(width, height, new ApplyFunction(buffer, Square));

// Copy the processed data back as usual
buffer.CopyTo(array);
```

# Requirements

The **ComputeSharp** library has the following requirements:
- .NET 5
- Windows x64

Additionally, in order to compile the library or a project using it, you need a recent build of Visual Studio 2019 or another IDE that has support for Roslyn source generators, as **ComputeSharp** relies on this feature to create the HLSL shader sources and to extract other shader metadata that is necessary to setup and execute the code at runtime. You can learn more about source generators [here](https://devblogs.microsoft.com/dotnet/introducing-c-source-generators/).

# Sponsors

Huge thanks to these sponsors for directly supporting my work on **ComputeSharp**, it means a lot! 🙌

<a href="https://github.com/xoofx"><img src="https://avatars.githubusercontent.com/u/715038" height="auto" width="80" style="border-radius:50%"></a>
<a href="https://github.com/sebastienros"><img src="https://avatars.githubusercontent.com/u/1165805" height="auto" width="80" style="border-radius:50%"></a>
<a href="https://github.com/hawkerm"><img src="https://avatars.githubusercontent.com/u/8959496" height="auto" width="80" style="border-radius:50%"></a>

# Special thanks

The **ComputeSharp** library was originally based on some of the code from the [DX12GameEngine](https://github.com/Aminator/DirectX12GameEngine) repository by [Amin Delavar](https://github.com/Aminator).

Additionally, **ComputeSharp** uses NuGet packages from the following repositories:

- [Microsoft.Toolkit.Diagnostics](https://github.com/windows-toolkit/WindowsCommunityToolkit)
- [TerraFX.Interop.Windows](https://github.com/terrafx/terrafx.interop.windows)
- [TerraFX.Interop.D3D12MemoryAllocator](https://github.com/terrafx/terrafx.interop.d3d12memoryallocator)
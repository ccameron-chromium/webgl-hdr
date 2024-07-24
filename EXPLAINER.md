# WebGL Extended Range Brightness

## Introduction

This proposal introduces a mechanism through which WebGL can draw colors
brighter than `white` (`#FFFFFF`).

This is the WebGL version of [this WebGPU feature](https://github.com/ccameron-chromium/webgpu-hdr/blob/main/EXPLAINER.md).

There is not an analogous CanvasRenderingContext2D feature, because CanvasRenderingContext2D does not have floating point buffer support.

## Background

## Existing WebGL Behavior

WebGL can change its drawing buffer's format using the function
[`drawingBufferStorage`](https://registry.khronos.org/webgl/specs/latest/1.0/#2.2).
Among the options for the drawing buffer's format is `RGBA16F`.

Using this format, values far outside of the range of the display's
capabilities may be specified. How such values are to be displayed is not
specified. In practice, all implementations project values outside of the
standard dynamic range of the screen to the standard dynamic range of the
screen.

## Goal

The goal of this proposal is to clarify that, by default, canvases must project
out of range values to the standard dynamic range gamut of the screen, and to
provide a mechanism through which a WebGL canvas can indicate that its colors
should not be restricted to the display's standard dynamic range.

## API proposal

Add the following new dictionaries.

```webidl
enum WebGLToneMappingMode {
  "standard",
  "extended",
};

dictionary WebGLToneMapping {
  WebGLToneMappingMode mode = "standard";
};

partial interface WebGLRenderingContextBase {
  undefined drawingBufferToneMapping(WebGLToneMapping toneMapping);
};
```

Calling the function `drawingBufferToneMapping` will result in the drawing
buffer being cleared.

How the colors specified in the drawing buffer are to be displayed on the screen
is determined by the properties specified in the `GLCanvasToneMapping`.

If the `mode` is set to `"standard"`, then all colors must be converted from the
color space indicated by `drawingBufferColorSpace` and to the color space of the
display, and then projected to the `[0, 1]` interval in that space. This space
is known as the standard dynamic range gamut of the display.

If the `mode` is set to `"extended"`, then all colors must be converted from the
color space indicated by `drawingBufferColorSpace` and to the color space of the
display, and then projected to the `[0, H]` interval in that space, where `H` is
the maximum color component value that the display is capable of producing.
This space is known as as the extended range gamut of the display.

## Example use

The following code would configure the canvas for high dynamic range, using the
Display P3 color space.

```
  gl.drawingBufferStorage(gl.RGBA16F,
                          gl.drawingBufferWidth,
                          gl.drawingBufferHeight);
  gl.drawingBufferColorSpace = "display-p3";
  gl.drawingBufferToneMapping({ mode:"extended" });
```

## Notes on CanvasRenderingContext2D and unification of APIs

Unlike WebGL and WebGPU, CanvasRenderingContext2D does not have the ability to
specify that its backbuffer (its bitmap) be of any other pixel format than 8-bit.
This limitation makes this API have no effect.

When that situation changes, it would be appropriate to create a
`CanvasToneMapping` dictionary and `CanvasToneMappingMode` enum, and update
WebGL and WebGPU to use those. This would then be similar to how
`PredefinedColorSpace` is used across WebGL, WebGPU, and CanvasRenderingContext2D.

## Question 1: Why not use an attribute, like `drawingBufferColorSpace`?

The IDL definition does not allow attributes to be dictionaries, so that is not
an option.

## Question 2: Why not roll this into a single function?

In the example usage, there are three separate calls that will all clear the
backbuffer. Why not add a function like WebGPU's `configure` function? Something
like:

```
  gl.drawingBufferStorage({
      gl.drawingBufferWidth,
      gl.drawingBufferHeight,
      "display-p3",
      { mode:"extended" });
```

This isn't totally unreasonable, but it feels a bit clunky. By what logic are
only these properties included, and not things like `antialias`, `depth`,
`stencil`, `premultipliedAlpha`, and `preserveDrawingBuffer` not to be included?

If we are to go down this path, then I would suggest creating a new API which
takes a dictionary of all of these properties (rather than an unending list).
That's a separate feature and separate proposal.

## Question 3: Why clear the contents of the buffer on `drawingBufferToneMapping`?

This is to avoid situations where the underlying buffer needs to be reallocated,
and so the contents need to be copied to preserve the illusion that the buffer
was unchanged.

This also matches the capabilities exposed by WebGPU.

## Question 4: Why no getter?

Match WebGPU, where canvas configuration properties are not query-able (including
the specified `colorSpace`, `toneMapping`, and maybe others).

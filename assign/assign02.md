---
layout: mathjax
title: "Assignment 2: Drawing functions"
---

**Due**:

* Milestone 1 due TBD
* Milestone 2 due TBD
* Milestone 3 due TBD

This is a pair assignment, so you may work with one partner.

## Overview

In this assignment, you will implement functions which perform drawing
operations into an in-memory image representation, in both C and assembly
language.

**Warning**: Assembly language programming is challenging! Make sure
you start each milestone as soon as possible, work steadily, and
ask questions early.  Also, writing unit tests and using `gdb` to
examine the detailed behavior of code under test will be critical
to successful implementation of the assembly language functions.

### Important requirement

In Milestones 2 and 3, you will be writing assembly language functions.
You **must** write these "by hand", and your assembly code must have
very detailed code comments explaining the purpose of each assembly language
instruction.

It is **not** allowed to generate assembly
code using a C compiler and submit this as your own code. We will assign
a grade of 0 to any submissions containing compiler-generated code
where hand-written assembly language is expected.

## Getting started

Downloading the skeleton code.

## Grading breakdown

Your grade for the assignment will be determined as follows:

## Drawing functions

Here are the prototypes of the drawing functions you will implement:

```c
void draw_pixel(struct Image *img, int32_t x, int32_t y, uint32_t color);

void draw_rect(struct Image *img,
               const struct Rect *rect,
               uint32_t color);

void draw_circle(struct Image *img,
                 int32_t x, int32_t y, int32_t r,
                 uint32_t color);

void draw_tile(struct Image *img,
               int32_t x, int32_t y,
               struct Image *tilemap,
               const struct Rect *tile);

void draw_sprite(struct Image *img,
                 int32_t x, int32_t y,
                 struct Image *spritemap,
                 const struct Rect *sprite);
```

Briefly:

* `draw_pixel` draws a single pixel, blending the specified
  color value with the current background color
* `draw_rect` draws a filled rectangle with the specified upper-left corner,
  width and height, blending the specified color with the
  colors of the existing pixels
* `draw_circle` draws a filled circle of specified radius centered at the
  specified $$x$$/$$y$$ coordinates, blending the specified color
  with the colors of the existing pixels
* `draw_tile` copies pixels from the specified rectangular region
  of a *tilemap* image to the specified location ($$x$$/$$y$$)
  of a destination image, without blending any of the copied colors
  with the existing colors
* `draw_sprite` is similar to `draw_tile`, but the pixels of the
  copied "sprite" are blended with the existing pixel colors

### struct Image and struct Rect data types

To understand the functionality of the drawing functions, it is necessary
to understand the `struct Image` and `struct Rect` data types, as well
as how colors are represented.

The `struct Image` type is defined as follows:

```c
struct Image {
  uint32_t width;
  uint32_t height;
  uint32_t *data;
};
```

The `width` and `height` fields define the width and height of an image,
in pixels. The `data` field is a pointer to a dynamically-allocated array
of `uint32_t` values, each one representing one pixel. The pixels are stored
in row-major order, starting with the top row of pixels.

A color is represented by a `uint32_t` value as follows:

* Bits 24-31 are the 8 bit red component value, ranging from 0–255
* Bits 16-23 are the 8 bit green component value, ranging from 0–255
* Bits 8–15 are the 8 bit blue component value, ranging from 0–255
* Bits 0–7 are the 8 bit alpha value, ranging from 0–255

The alpha value of a color represents its opacity, with 255 meaning
"fully opaque" and 0 meaning "fully transparent".

### Color blending

The color values of the destination image are always fully opaque,
with an alpha value of 255.

When a pixel is drawn to a destination image by any operation other than
`draw_tile`, the pixel's color, which we'll call the "foreground" color,
is blended with the existing color at the location where the pixel is being
drawn, which we'l call the "background" color. To find the correct color
value for the new pixel, the following computation is performed for
each color component, where $$f$$ is the foreground color component value,
$$b$$ is the background color component value, and $$\alpha$$ is the
alpha value of the foreground color:

$$\lfloor (\alpha f + (1 - \alpha)b) / 255 \rfloor$$

Note that the result of the division is truncated rather than being rounded,
so if you use integer division, it will behave in the expected way.

A blended color should have each color component value (red, green, and blue)
computed using the formula above, and the alpha value of the blended color
should be set to 255.

The `draw_pixel`, `draw_rect`, `draw_circle`, and `draw_sprite`
functions all should use color blending. The exception is the
`draw_tile` function, which does not blend the copied tile colors
with the existing pixel colors.

### Drawing a circle

The drawing functions should be implemented entirely using integer
arithmetic. When drawing a filled circle, all of the pixels within
$$r$$ units of distance from the circle's center should be drawn
with the specified color. Ordinarily, if the pixel's center is at
$$x,y$$ and the point being considered is at $$j,i$$,
determining if $$j,i$$ is in the circle would be determined by the
inequality

$$\sqrt{(x - j)^{2} + (y - i)^{2}} \le r$$

The square root operation would require floating point math.
However, we could square both sides of the inequality to give us

$$(x - j)^{2} + (y - i)^{2} \le r^{2}$$

This computation only requires integer arithmetic.

### Bounds checking

When any drawing function is executed, it should only attempt
to draw pixels that are in the boundaries of the destination
image.

For example, if `draw_pixel` is called, and either

* the $$x$$ coordinate is less than 0 or greater than or equal to the image width, or
* the $$y$$ coordinate is less than 0 or greater than or equal to the image height

then the destination image should not be modified.

The `draw_rect` and `draw_circle` functions could be asked to draw a rectangle
or circle that is partially or entirely outside the bounds of the destination
image. Only the pixels that are within the image bounds should be drawn.

The `draw_tile` or `draw_sprite` could be asked to draw pixels (copied
from the source tilemap or spritemap image) that are not within the bounds
of the destination image. Only pixels that are within the bounds of the
destination iamge should be modified.

Note also that the `draw_tile` and `draw_sprite` functions could be
passed `struct Rect` data such that the region described by the
rectangle is not entirely within the source tilemap or spritemap
image. In this case, the expected behavior is for the function to do
nothing. For example:

```c
void draw_tile(struct Image *img,
               int32_t x, int32_t y,
               struct Image *tilemap,
               const struct Rect *tile) {
  if (/* tile rectangle is not entirely within the bounds of tilemap */) {
    return;
  }

  // proceed to copy pixel data from tilemap to dest image...
}
```

## `c_draw` and `asm_draw` programs

As a demonstration of how the drawing functions could be used to implement
2D graphics, the `c_driver.c` program reads data from an "image description"
and renders the resulting operations to a `struct Image` in memory,
which is then written to a PNG file.

This program can be compiled as either `c_draw` or `asm_draw`. The only
difference is whether the C or assembly language implementations of the
drawing functions are used. To build them:

```
make c_draw
make asm_draw
```

To run the programs:

```
mkdir -p out
./c_draw out/example01.png input/example01.in
```

or

```
mkdir -p out
./asm_draw out/example01.png input/example01.in
```

There are other example input files in the `input` directory included in the
skeleton code.

Note that the `expected` directory contains the expected output for each
example input file. You can use the `compare` program to test your program's
put with the expected output. For example:

```
compare -metric RMSE expected/example01.png out/example01.png out/example01_diff.png
```

It is expected that your drawing operations produce results which are
*identical* to the expected images. If they are not identical, the
"diff image" produced by the `compare` program (in the example above,
`out/example01_diff.png`) will have a red pixel for each location
where the generated image differed from the expected image.

## Milestones

### Milestone 1: C implementation

Milestone 1 requires implementing all of the drawing functions in C.

The `test_drawing_functions.c` test program has a reasonably comprehensive
set of unit tests for the drawing functions themselves.
You can compile and run this program using the commands

```
make depend
make c_test_drawing_functions
./c_test_drawing_functions
```

However, you will want to write helper functions, and add your own
tests for your helper functions. These helper functions and their tests
will be *essential* for getting the assembly language implementations
of the drawing functions to work in Milestones 2 and 3.

Possible helper functions to implement:

```c
int32_t in_bounds(struct Image *img, int32_t x, int32_t y); 
uint32_t compute_index(struct Image *img, int32_t x, int32_t y);
int32_t clamp(int32_t val, int32_t min, int32_t max);
uint8_t get_r(uint32_t color);
uint8_t get_g(uint32_t color);
uint8_t get_b(uint32_t color);
uint8_t get_a(uint32_t color);
uint8_t blend_components(uint32_t fg, uint32_t bg, uint32_t alpha);
uint32_t blend_colors(uint32_t fg, uint32_t bg);
void set_pixel(struct Image *img, uint32_t index, uint32_t color);
int64_t square(int64_t x);
int64_t square_dist(int64_t x1, int64_t y1, int64_t x2, int64_t y2);
```

These are the helper functions used in the reference solution.

The `in_bounds` function checks `x` and `y` coordinates to determine
whether they are in bounds in the specified image.

The `compute_index` function computes the index of a pixel in
an image's `data` array given its `x` and `y` coordinates.
The `clamp` function constrains constrains a value so that it
is greater than or equal to `min` and less than or equal to
`max`. This is very useful when determining a rectangular area
that is entirely within the bounds of an image; for example,
when drawing a rectangle or circle, or copying a tile or sprite.

The `get_r`, `get_g`, `get_b`, and `get_a` functions return
the red, green, blue, and alpha components of a pixel color
value, respectively.

The `blend_components` function blends foreground and background
color component values using a specified alpha (opacity) value.

The `blend_colors` function blends foreground and background
colors using the foreground color's alpha value to produce an
opaque color. (It will call `blend_components`.)

The `set_pixel` function draws a single pixel
to a destination image, blending the specified foregroudn color
with the existing background color, at a specified pixel index.

The `square` function returns the result of squaring an
`int64_t` value. The `square_dist` function returns the sum
of the squares of the x and y distances between two points.

You are not required to implement this exact set of helper functions.
However, if you would like to base your implementation on these
functions, you are welcome to do that.

### Milestone 2: assembly language `draw_pixel`, helper functions

The goal of Milestone 2 is to fully implement the `draw_pixel` function, along with
required helper functions.

**Important**: your assembly language code should be a manual translation
of your C functions, including the helper functions, to assembly language.

Building the `asm_test_drawing_functions` executable will allow you
to run the unit tests you wrote for your C helper functions on the
assembly language implementations of those functions.

We suggest that you start out by adding "stub" implementations of
each helper function, where the body of the instruction consists only
of a single `ret` instruction. This will allow the test program to be
built and to execute, even though the assembly language functions aren't
implemented yet.

Once you have stub versions of each function added, you can start implementing
the implementations of the helper functions.

### Milestone 3: assembly language `draw_rect`, `draw_circle`, `draw_tile`, and `draw_sprite`

In Milestone 3, your task is to implement the remaining functions in assembly language.

**Important!** You are really only expected to implement `draw_rect` and `draw_circle`.
The other two functions (`draw_tile` and `draw_sprite`) are quite complicated to
write in assembly language, and implementing them is worth only 4 points
(out of 100 for the entire assignment.) If you attempt them at all, it should be
*after* `draw_rect` and `draw_circle` are completely working.

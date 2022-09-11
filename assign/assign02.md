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

Brief overview.

## Getting started

Downloading the skeleton code.

## You must hand-write your own assembly language code!

In Milestones 2 and 3, you will be writing assembly language functions.
You **must** write these "by hand", and your assembly code must have
very detailed code comment explaining the purpsoe of each assembly language
instruction.

It is **not** allowed to generate assembly
code using a C compiler and submit this as your own code. We will assign
a grade of 0 to any submissions containing compiler-generated code
where hand-written assembly language is expected.

## Drawing functions

Here are the prototypes of the drawing functions you will implement:

```c
// TODO: function prototypes
```

Brief synposis of each function.

### struct Image and struct Rect data types

Overview of the `struct Image` and `struct Rect` data types.

### Color blending

Explanation of color blending

## Milestones

### Milestone 1: C implementation

Milestone 1 requires implementing all of the drawing functions in C.

The `test_drawing_functions.c` test program has a reasonably comprehensive
set of unit tests for the drawing functions themselves.

However, you will want to write helper functions, and add your own
tests for your helper functions. These helper functions and their tests
will be *essential* for getting the assembly language implementations
of the drawing functions to work in Milestones 2 and 3.

Possible helper functions to implement:

```c
int32_t in_bounds(struct Image *img, int32_t x, int32_t y); 
unsigned compute_index(struct Image *img, int32_t x, int32_t y);
int32_t clamp(int32_t val, int32_t min, int32_t max);
uint8_t get_r(uint32_t color);
uint8_t get_g(uint32_t color);
uint8_t get_b(uint32_t color);
uint8_t get_a(uint32_t color);
uint8_t blend_components(uint32_t fg, uint32_t bg, uint32_t alpha);
uint32_t blend_colors(uint32_t fg, uint32_t bg);
void set_pixel(struct Image *img, unsigned index, uint32_t color);
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
opaque color. The `set_pixel` function draws a single pixel
to a destination image, blending the specified foregroudn color
with the existing background color, at a specified pixel index.
The `square` function returns the result of squaring an
`int64_t` value. The `square_dist` function returns the sum
of the squares of the x and y distances between two points.

You are not required to implement this exact set of helper functions.
However, if you would like to base your implementation on these
functions, you are welcome to do that.

### Milestone 2: assembly language `draw_pixel`, helper functions

Goal is to fully implement the `draw_pixel` function, along with
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

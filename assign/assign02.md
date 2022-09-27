---
layout: mathjax
title: "Assignment 2: Drawing functions"
---

**Due**:

* Milestone 1 due Tues Sept 20th by 11pm
* Milestone 2 due Thurs Sept 29th by 11pm
* Milestone 3 due Thurs Sept Oct 6th by 11pm

This is a pair assignment, so you may work with one partner.

*Update 9/22*: Added a link to the [Assembly language demo](https://jh.hosted.panopto.com/Panopto/Pages/Viewer.aspx?id=4b406360-7fbc-4875-b2e4-af1800e8286b)
screencast video to the [x86-64 tips](#x86-64-tips) section.

*Update 9/27*: Updated [Grading breakdown](#grading-breakdown)
section to indicate that your submitted unit tests should completely
pass when submitting MS2 and MS3.  For MS2, you should comment
out calls to tests functions that aren't related to `test_pixel`.

## Overview

In this assignment, you will implement functions which perform drawing
operations into an in-memory image representation, in both C and x86-64 assembly
language.

**Warning**: Assembly language programming is challenging! Make sure
you start each milestone as soon as possible, work steadily, and
ask questions early.  Also, writing unit tests and using `gdb` to
examine the detailed behavior of code under test will be critical
to successful implementation of the assembly language functions.

### Non-functional requirements

In Milestones 2 and 3, you will be writing assembly language functions.
You **must** write these "by hand", and your assembly code must have
very detailed code comments explaining the purpose of each assembly language
instruction.

It is **not** allowed to generate assembly
code using a C compiler and submit this as your own code. We will assign
a grade of 0 to any submissions containing compiler-generated code
where hand-written assembly language is expected.

Your submission for each milestone should include a `README.txt` file
describing how you and your partner divided the work, and letting
us know about any interesting implementation details. If there is
functionality you weren't able to get working completely, this is
a good place to mention that.

We expect you to follow the [style guidelines](style.html).
However, the expectations for function length will be relaxed considerably
for your assembly language code. It is not unusual for an assembly language
function to have 100 or more lines of code. In the reference solution,
the longest function was 170 lines, although there was extensive use of
comments and whitespace to improve readability.

Of course, you should strive to make your assembly language functions
as simple and readable as possible.

We expect your code to be free of memory errors. You should use
`valgrind` to test your code to make sure there are no uses
of uninitialized variables, out of bounds memory reads or writes,
etc.  This applies to both your C code and your assembly code.

## Getting started

To get started, download [csf\_assign02.zip](csf_assign02.zip) and
unzip it.

## Grading breakdown

Your grade for the assignment will be determined as follows:

* Milestone 1: 30%
  * Implementation of C drawing functions: 12.5%
  * Unit testing of helper functions: 12.5%
  * Design/coding style of C functions: 5%
* Milestone 2: 35%
  * Functional correctness of `draw_pixel`: 15%
  * Unit testing of helper functions (existence of tests, evidence that they pass): 15%
  * Design/coding style of assembly functions: 5%
* Milestone 3: 35%
  * Functional correctness of `draw_rect`: 13.5%
  * Functional correctness of `draw_circle`: 12.5%
  * Functional correctness of `draw_tile`: 2%
  * Functional correctness of `draw_sprite`: 2%
  * Design/coding style of assembly functions: 5%

Note that in each milestone, we expect all of the tests executed
by your unit test program to pass. For MS2 in particular, you can
comment out calls to test functions that aren't related to
`draw_pixel`. For example, your `main` function might have
code similar to the following:

```c
TEST(test_in_bounds);
TEST(test_blend_colors);
TEST(test_blend_components);
TEST(test_draw_pixel);
//TEST(test_draw_rect);
```

The tests for `in_bounds`, `blend_colors`, `blend_components`, and
`draw_pixel` are enabled because they all test functions involved in
the implementation of `draw_pixel`. The test for `draw_rect` is
commented out because it is not part of the functionality expected
for MS2.

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
"fully opaque" and 0 meaning "fully transparent". (See the
[Color blending](#color-blending) section for details on how
an alpha value allows two colors to be "blended".)

The `struct Rect` data type is defined as follows:

```c
struct Rect {
  int32_t x, y, width, height;
};
```

An instance of `struct Rect` describes a rectangle where the upper-left
corner is specified by `x` and `y`, the width of the rectangle (in pixels)
is specified by `width`, and the height of the rectangle (in pixels)
is specified by `height`.

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

$$\lfloor (\alpha f + (255 - \alpha)b) / 255 \rfloor$$

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
./c_draw out/example01.png < input/example01.in
```

or

```
mkdir -p out
./asm_draw out/example01.png < input/example01.in
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

The `test_drawing_funcs.c` test program has a reasonably comprehensive
set of unit tests for the drawing functions themselves.
You can compile and run this program using the commands

```
make depend
make c_test_drawing_funcs
./c_test_drawing_funcs
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

Building the `asm_test_drawing_funcs` executable will allow you
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

## Hints and tips

This section contains hints, tips, and general recommendations for how to
make progress on this assignment.

Note: it's possible that we will update this section. We will let you
know (in class and on Piazza) if we do.

### C implementation and helper functions

There are two related purposes for having you implement the drawing
functions in C in Milestone 1.

The first purpose is to fully understand how the drawing functions
are intended to work.

The second purpose is to find opportunities to create helper functions
to simplify the implementation of the drawing functions, and to
write unit tests for those helper functions.

The helper functions suggested in the [Milestone 1](#milestone-1-c-implementation)
section are one approach to managing the complexity of the drawing functions,
but are certainly not the only way. The thing to keep in mind is that writing
an assembly language function is, in general, *much* more complicated than
writing an equivalent C function. Arithmetic that would be straighforward
to implement using a single line of C code could involve dozens of lines of
assembly code. So, helper functions to perform arithmetic, bounds checking,
color blending, and so forth are valuable because they will allow you to
make progress on your assembly code one helper function at a time.
Without well-factored helper functions with good testing, you could
easily find your assembly languge code turning into an unmaintainable mess.

## x86-64 tips

We made a [screencast video](https://jh.hosted.panopto.com/Panopto/Pages/Viewer.aspx?id=4b406360-7fbc-4875-b2e4-af1800e8286b)
which walks through implementing, testing, and debugging assembly language
code. This video covers all of the essential techniques you will
need for this assignment.

Here are some x86-64 assembly language tips and tricks in no particular order.

Callee-saved registers are your best option to serve as local variables
in your assembly language functions. The callee-saved registers are
`%r12`, `%r13`, `%r14`, `%r15`, `%rbx`, and `%rbp`. If you are going
to store data in a callee-saved register, make sure that you use `pushq`
to save its value at the beginning of the function, and `popq` to restore
its value at the end of the function. (The `popq` instructions must be in
the opposite order as the `pushq` instructions.)

If you run out of callee-saved registers, then you can use memory in the
stack frame to store local variables. To reserve `N` bytes of memory,
use `subq $N, %rsp` at the beginning of the function. You will need
a corresponding `addq $N, %rsp` at the end of the function.  Note that
the `subq` must happen *after* you use `pushq` to save the values
of callee-saved registers, and the `addq` must happen *before* you
use `popq` to restore the values of callee-saved registers.
Also, don't forget that the amount by which the stack pointer is
changed needs to be an odd multiple of 8 (so 8, or 24, or 40, etc.),
and that each `pushq` subtracts 8 from `%rsp`.

If you've reserved memory for local variables on the stack, you can
refer to them by their offsets from `%rsp`. (See the example comment
below.)

Don't forget that you need to prefix constant values with `$`.  For example,
if you want to set register `%r10` to 16, the instruction is

```
movq $16, %r10
```

and not

```
movq 16, %r10
```

When calling a function, the stack pointer (`%rsp`) must contain an address
which is a multiple of 16.  However, because the `callq` instruction
pushes an 8 byte return address on the stack, on entry to a function,
the stack pointer will be "off" by 8 bytes.  You can subtract 8 from
`%rsp` when a function begins and add 8 bytes to `%rsp` before returning
to compensate.  (See the example `addLongs` function.)  Pushing an
odd number of callee-saved registers also works, and has the benefit
that you can then use the callee-saved registers freely in your function.

We *strongly* recommend that you have a comment in each function explaining
how it uses callee-saved registers and stack memory, since these are
the equivalent of local variables in assembly code. For exmaple,
here is a comment taken from one of the helper functions in the
reference solution:

```c
/*
 * Register use:
 *   %r12 - pointer to dest Image
 *   %r13d - x coord
 *   %r14d - y coord
 *   %r15 - pointer to blockmap Image
 *   %rbx - pointer to block Rect, then i (row counter)
 *   %ebp - j (column counter)
 *
 * Stack use (32 bytes):
 *   0(%rsp) - min_x
 *   4(%rsp) - max_x
 *   8(%rsp) - min_y
 *   12(%rsp) - max_y
 *   16(%rsp) - blend
 *   20(%rsp) - block->x
 *   24(%rsp) - block->y
 *   28(%rsp) - saved dest index
 */
```

Recall that your assembly language code must have detailed comments
explaining each line of assembly code. The following example
function illustrates the level of commenting that we expect to see:

```
/*
 * Determine the length of specified character string.
 *
 * Parameters:
 *   %rdi - pointer to a NUL-terminated character string
 *
 * Returns:
 *    number of characters in the string
 */
	.globl str_len
str_len:
	subq $8, %rsp                 /* adjust stack pointer */
	movq $0, %r10                 /* initial count is 0 */

.Lstr_len_loop:
	cmpb $0, (%rdi)               /* found NUL terminator? */
	jz .Lstr_len_done             /* if so, done */
	inc %r10                      /* increment count */
	inc %rdi                      /* advance to next character */
	jmp .Lstr_len_loop            /* continue loop */

.Lstr_len_done:
	movq %r10, %rax               /* return count */
	addq $8, %rsp                 /* restore stack pointer */
	ret
```

As illustrated in the example function, labels for control flow
should be *local labels*, with names beginning with "`.L`".
If you don't use local labels within functions, debugging with
`gdb` will be difficult because `gdb` will think that each control
flow label is the beginning of a function.

## Debugging tips

You primary means of determining whether or not your code works correctly
is running the unit test programs (`c_test_drawing_funcs` and `asm_test_drawing_funcs`.)

If a unit test fails, you should use `gdb` to debug the code to determine
why it is not working.

Setting a breakpoint on the specific test function that is failing is
one way to start. For example, if the `test_in_bounds` test function
is failing, in `gdb` set a breakpoint on that function, then run the
program so that it only runs that test function:

```
break test_in_bounds
run test_in_bounds
```

You will gain control of the program at the beginning of the test
function, at which point you can step through the code, inspect
variables, registers, and memory, etc.

Another good option for setting a breakpoint is the `siglongjmp`
function, because this is the function called when a test assertion
fails. For example, assuming `test_in_bounds` has an assertion failure:

```
break siglongjmp
run test_in_bounds
```

When the `sigjongjmp` breakpoint is reached, use the `up` command (as many
times as needed) to enter the stack frame for the failing assertion.
This can allow you to check variables at the location of the assertion.

Don't forget that you can inspect register values in `gdb` by prefixing the
register name with the "`$`" character. For example:

```
print $ebx
```

would show you the contents of the `%ebx` register. Using `print/x` allows
you to see integer values in hexadecimal (very useful for checking color values.)

Casting a register to a pointer allows you to interpret memory as values
belonging to C data types. For example, let's say `%r10` points to a
`struct Image` instance. You could check the value of the element
at index 18 of the `data` array using the command

```
print/x ((struct Image *)$r10)->data[18]
```

If you are storing local variables in stack memory, and using `%rsp` to
access them, it is easy to see their values. In particular, if all of the
local variables are the same size and type (e.g., they are all
4-byte integers), then you can think of them as an array.
For example, in the comment above about local variable allocation,
there are 8 local variables allocated in stack memory, each of which
is a 4 byte integer value. We can see all of the values at once
with the `gdb` comamnd

```
print (unsigned [8]) *((unsigned *)$rsp)
```

Here we are pretending that these variables belong to the `unsigned` type,
which is the same as the `uint32_t` type.  The `(unsigned [8])` at the
beginning of the expression tells `gdb` that we are interpreting the
memory as an array of 8 `unsigned` elements.

## Submitting

To submit your work for a milestone, create a zipfile with all of the code,
along with your `README.txt`, using a command such as

```
zip -9r solution.zip *.h *.c *.S Makefile README.txt
```

Please do not submit object files, executables, PNG image files,
or other files not mentioned above.

Upload the zipfile to [Gradescope](https://www.gradescope.com) as
**Assignment 2 MS1**, **Assignment 2 MS2**, or **Assignment 2 MS3**, depending
on which milestone you are submitting.

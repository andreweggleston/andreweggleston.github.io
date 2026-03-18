---
layout: post
category: perf
---

The (ELF) programs you run on your (UNIX) computer have these fun little 'sections' in them: `.text`, `.data`, `.bss`... and many more. But what are these sections, and what if you wanted to make your own?

If you have a compulsive computer addiction and like reading GCC docs for fun, you probably already know all about ELF, so skip to [the fun part](#the-fun-part) if that sounds like you.

_Otherwise_, here is a short overview of how ELF (the Executable and Linkable Format) describes a standard for executable code.

### the elf part

ELF is how Unix and Unix-like operating systems link and load compiled programs. When you compile a source file like a simple `hello_world.c` program with a compiler like `gcc`, what first is created is an **object** file: `hello_world.o`:

`sh-5.3$ cat hello_world.c`:
```
#include <stdio.h>
int main(void) {
    printf("Hello, World!\n");
	return 42;
}
```

[^1]`sh-5.3$ gcc -c hello_world.c; ls`:
```
hello_world.c  hello_world.o
```

This object file (ending in `.o`) isn't quite an executable yet, but it is an ELF file.
Using `objdump` to show the sections in an executable, we can see what `hello_world.o` is made up of:

[^2]`sh-5.3$ objdump --headers hello_world.o`:
```
hello_world.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000001a  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000000  0000000000000000  0000000000000000  0000005a  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  0000005a  2**0
                  ALLOC
  3 .rodata       0000000e  0000000000000000  0000000000000000  0000005a  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      00000013  0000000000000000  0000000000000000  00000068  2**0
                  CONTENTS, READONLY
  5 .note.GNU-stack 00000000  0000000000000000  0000000000000000  0000007b  2**0
                  CONTENTS, READONLY
  6 .note.gnu.property 00000030  0000000000000000  0000000000000000  00000080  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .eh_frame     00000038  0000000000000000  0000000000000000  000000b0  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
```
You can see this is an ELF file from the header `hello_world.o: file format elf64-x86-64`--this was compiled on my PC running Linux with an AMD x86_64 CPU.

You can also see that `hello_world.o` contains several sections. Your compiled code lives in `.text`--this will be made of actual x86_64 machine code. You can disassemble it with `objdump -d hello_world.o`. Data that your program uses to run is stored across `.data` and `.rodata`--things like strings and globals. You can see that `.rodata` has a non-zero size--this is where the "Hello, world\n" string is saved! The `.bss` section is the last important one, and its kind of special. It stands for "Block Started Symbol", and it holds metadata about the size of `static` variables in your code. The operating system's [loader](https://en.wikipedia.org/wiki/Loader_(computing)) uses `.bss` to determine how much space to allocate at load-time for static variables. This way, space can be saved in the executable file.


Finally, you can compile this `.o` file to an executable, again with `gcc`:

`sh-5.3$ gcc -o hello_world hello_world.o; ls`:
```
hello_world  hello_world.c  hello_world.o
```
`sh-5.3$ ./hello_world`:
```
Hello, World!
```


### [the fun part](#the-fun-part)

[GNU `ld`](https://sourceware.org/binutils/docs/ld.pdf)'s "linkerscripts" are the tool to reach for when you need control over your executable's layout. Let's say you were working on a project that required (for whatever reason) that certain source code ended up in a specific section:

```
src/
├─my_cool_section # <- this stuff should go in a new section ".my_cool_section"
└─everything_else # <- everything else should go in the regular sections

```
You could write a linker script that had a custom section defined as follows:

```
.my_cool_section ALIGN(0x200000) :
{
    PROVIDE (__mycoolsection_start = ABSOLUTE(.));
    KEEP(*/my_cool_section/*(.text))
    . = ALIGN(0x200000);
    PROVIDE (__mycoolsection_end = ABSOLUTE(.));
}
```

This would do a few cool things:
  1. Create a cool new section called `.my_cool_section`
  2. Align this cool new section to a 2 megabyte boundary
  3. Put all of the `.text` sections from code in the `my_cool_section` folder into `.my_cool_section`
  4. `PROVIDE` two new addresses: `__mycoolsection_start` and `__mycoolsection_end`, which you can access in your code using a declaration like `extern char[] __mycoolsection_start`

When you go to compile your program with `gcc`, you can pass your custom linker script with the `-T` option: `gcc -T myscript.ld ...`

Congratulations! You now have an executable with a cool custom section. Now what?


---


{: data-content="footnotes"}
[^1]: I used the `-c` option here so that `gcc` gave me an object file instead of a fully compiled and linked executable
[^2]: Technically if you followed me exactly you won't get this exact output. It looked like the default optimization level did something weird (`.text` had size 0!), so I actually ran the `gcc` command with `-O0` to disable optimization.

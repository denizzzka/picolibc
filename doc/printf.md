# Printf and Scanf in Picolibc

Embedded systems are often tightly constrained on code space, which
makes it important to be able to only include code and data actually
needed by the application. The format-string based interface to
printf and scanf makes it very difficult to determine which
conversion operations might be needed by the application.

Picolibc handles this by providing multiple printf and scanf
implementations in the library with varying levels of conversion
support. The application developer selects among these at compile time
based on knowledge of the requirements of the application. The
selection is done using a preprocessor define which maps the
public printf and scanf names to internal names.

The function name mapping happens in stdio.h, so the preprocessor
definition must be set before stdio.h is included the first time. This
is easily done by adding the flag to the compiler command line.

## Printf and Scanf levels in Picolibc

There are three levels of printf support provided by Picolibc that can
be selected when building applications:

 * Default level. This offers full printf functionality, including
   both float and double conversions. This is what you get with no
   pre-processor definition.

 * PICOLIBC_INTEGER_PRINTF_SCANF. This removes support for all
   float and double conversions.

 * PICOLIBC_FLOAT_PRINTF_SCANF. This provides support for float, but
   not double conversions. To use this mode, float values must be
   passed to printf using the printf_float macro.

PICOLIBC_FLOAT_PRINTF_SCANF requires a special macro for float
values, to make it easier to switch between that and the default
level, that macro is also correctly defined for the other two levels.

Here's an example program to experiment with these options:

	#include <stdio.h>

	void main(void) {
		printf(" 2⁶¹ = %lld π ≃ %g\n", 1ll << 61, printf_float(3.1415f));
	}

Now we can build and run it with the default options:

	$ riscv64-unknown-elf-gcc -Os -march=rv32imac -mabi=ilp32 --specs=picolibc.specs --oslib=semihost -Wl,--defsym=__flash=0x80000000 -Wl,--defsym=__flash_size=0x00200000 -Wl,--defsym=__ram=0x80200000 -Wl,--defsym=__ram_size=0x200000 -o printf.elf printf.c
	$ riscv64-unknown-elf-size printf.elf
	   text	   data	    bss	    dec	    hex	filename
	  12878	     16	      8	  12902	   3266	printf.elf
	$ qemu-system-riscv32 -chardev stdio,mux=on,id=stdio0 -semihosting-config enable=on,chardev=stdio0 -monitor none -serial none -machine virt,accel=tcg -kernel printf.elf -nographic -bios none
	 2⁶¹ = 2305843009213693952 π ≃ 3.1415

Switching to float-only reduces the size but lets this still work correctly:

	$ riscv64-unknown-elf-gcc -DPICOLIBC_FLOAT_PRINTF_SCANF -Os -march=rv32imac -mabi=ilp32 --specs=picolibc.specs --oslib=semihost -Wl,--defsym=__flash=0x80000000 -Wl,--defsym=__flash_size=0x00200000 -Wl,--defsym=__ram=0x80200000 -Wl,--defsym=__ram_size=0x200000 -o printf-float.elf printf.c
	$ riscv64-unknown-elf-size printf-float.elf
	   text	   data	    bss	    dec	    hex	filename
	   7844	     16	      8	   7868	   1ebc	printf-float.elf
	$ qemu-system-riscv32 -chardev stdio,mux=on,id=stdio0 -semihosting-config enable=on,chardev=stdio0 -monitor none -serial none -machine virt,accel=tcg -kernel printf-float.elf -nographic -bios none
	 2⁶¹ = 2305843009213693952 π ≃ 3.1415

Going to integer-only reduces the size even further, but now it doesn't output
the values correctly:

	$ riscv64-unknown-elf-gcc -DPICOLIBC_INTEGER_PRINTF_SCANF -Os -march=rv32imac -mabi=ilp32 --specs=picolibc.specs --oslib=semihost -Wl,--defsym=__flash=0x80000000 -Wl,--defsym=__flash_size=0x00200000 -Wl,--defsym=__ram=0x80200000 -Wl,--defsym=__ram_size=0x200000 -o printf-int.elf printf.c
	$ riscv64-unknown-elf-size printf-int.elf
	   text	   data	    bss	    dec	    hex	filename
	   2306	     16	      8	   2330	    91a	printf-int.elf
	$ qemu-system-riscv32 -chardev stdio,mux=on,id=stdio0 -semihosting-config enable=on,chardev=stdio0 -monitor none -serial none -machine virt,accel=tcg -kernel printf-int.elf -nographic -bios none
	 2⁶¹ = 0 π ≃ *float*

## Picolibc build options for printf and scanf options 

In addition to the application build-time options, picolibc includes a
number of picolibc build-time options to control the feature set (and
hence the size) of the library:

 * `-Dio-c99-formats=true` This option controls whether support for
   the C99 type-specific format modifiers 'j', 'z' and 't' is included
   in the library. Support for the C99 format specifiers like PRId8 is
   always provided.  This option is enabled by default.

 * `-Dio-long-long=true` This option controls whether support for long
   long types is included in the integer-only version of printf and
   scanf. long long support is always included in the full and
   float-only versions of printf and scanf. This option is disabled by
   default.

For even more printf and scanf functionality, picolibc can be compiled
with the original newlib stdio code. That greatly increases the code
and data sizes of the library, including adding a requirement for heap
support in the run time system. Here are the picolibc build options for that code:

 * `-Dnewlib-tinystdio=true` This option enables the tinystdio code in
   place of the original newlib stdio code. This option is enabled by
   default.

 * `-Dnewlib-io-pos-args=true` This option add support for C99
   positional arguments (e.g. "%1$"). This option is disabled by default.

 * `-Dnewlib-io-long-double=true` This option add support long double
   parameters. That is limited to systems using 80- and 128- bit long
   doubles, or systems for which long double is the same as
   double. This option is disabled by default

 * `-Dnewlib-stdio64=true` This option changes the newlib stdio code
   to use 64 bit values for file sizes and offsets. It also adds
   64-bit versions of stdio interfaces which are defined with types
   which may be 32-bits (like 'long'). This option is enabled by default.

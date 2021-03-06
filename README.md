C/C++ style for hardware interfacing
---
## A little background:
### Data types
Data types are used for efficiently working with constant or variable data.
Historically the smallest were the most efficient, but nowadays this only
applies to storage, as microcontrollers, even 8-bit ones, tend to have at
least some registers and instructions to handle "more-than-a-byte" data.
Mixing types in expressions can lead to some overhead hidden by the compiler,
and to failures where the programmer wasn't aware of how the overhead is
implemented. Most of the times it comes from "type promotion" arranged by
the C compiler and C++ compilers will force you produce more bloated source
code by explicitly doing the typecasts.
In C I usually stick to unsigned or signed integer types in expressions
where the sizes differ and only cast the result if needed.
- \_Bool (C99)
- \_Complex (C99)
- char: minimum size 8 bits, POSIX requires it to be exactly 8 bits
- double
- float
- int: minimum size 16 bits
- long (or as modifier to int or long or double): minimum size 32 bits
- short (or as modifier to int)
- union: storage type with element type choice
- struct: storage type for multiple elements
- data type modifiers:
  - signed
  - unsigned
- float size restrictions: float <= double <= long double
- int size restrictions: char <= short <= int <= long <= long long

### Type qualifiers
IO-functionality is often decribed as an address being a
constant pointer to a volatile aggregate type
containing the IO-registers and void fills for unused addresses.
Luckily even the old C89 standard has the words `const` and `volatile`:
- \_Atomic (C11)
- const (C89)
- volatile (C89)
- restrict (C99)

## My style for hardware-interfacing
Hardware control can use width-specified data types and ways of placing
registers at defined addresses. C99 defines width-specified data types, 
accessible with inclusion of `<inttypes.h>` or `<stdint.h>`, but if it is 
known which microcontroller and compiler are used, it is trivial to typedef
exact width data types for signed and unsigned types. 
I often use a small header file with typedefs for the names s08, u08, s16, u16,
s32, u32, s64, and u64. It should be clear which ones are signed and unsigned
respectively, and how many bits constitute them.

Address definitions are defined in capitals, being address constants, 
with a trailing underscore for single registers or aggregate types and a 
leading underscore for register-arrays, without underscore for the indexes 
into the register-arrays. 
Bit-constants for specific registers are named `bnn_R_E`, 
with nn the decimal lsb-shift designator in register R, 
known by the name E in the hardware-manual.
Multi-bit register-constants are similarly defined as `x#_R_E`, 
where \# is a hexadecimal value of register-width divided by 4 nibbles.
- `R_`, as a "private variable defined at address constant", example: `SDCMD_`
- `_A`, as an "array defined at address constant", example: `_IPR[17]`
- `AI`, as a "value constant", example: `_IPR[IRQ0]`
- `bnn_R_E`, example: `b04_DMINT_DTIE`
- `x#_R_E`, example: `xC000_ICU_IRQFLTC0_FCLKSEL7_PCLKBdiv64`

As a rather complex example if I had to code the IO-ports of an RX65N-ucon
"bare-metal", using C99 with exact-width data types, it would look something like:
```.c
    //HW 22. I/O Ports
#define IO_ (*(struct\
{	uint8_t _PDR[32],_PODR[32],_PIDR[32],_PMR[32];\
	struct {uint8_t _0,_1;} _ODR[32];\
	uint8_t _PCR[32],_DSCR[32],_100[0x28],_DSCR2[32];\
} volatile *const)0x0008C000)
```
The above can be explained by looking into the hardware manual in chapter 22, 
where it would be seen that there is room for 32 or `0x20` 8-bit IO-ports, 
divided into several 8-bit registers per IO-port:
1. PDR-array, size `0x20`, starting at hex-address `0x0008 C000`
2. PODR-array, size `0x20`, starting at hex-address `0x0008 C020`
3. PIDR-array, size `0x20`, starting at hex-address `0x0008 C040`
4. PMR-array, size `0x20`, starting at hex-address `0x0008 C060`
5. interleaved arrays ODR0 and ODR1, each sized `0x20`:
   - ODR0, adressable as `IO_._ODR[portnr]._0`, starting at hex-address `0x0008 C080`
   - ODR1, adressable as `IO_._ODR[portnr]._1`, starting at hex-address `0x0008 C081`
6. PCR-array, size `0x20`, starting at hex-address `0x0008 C0C0`
7. DSCR-array, size `0x20`, starting at hex-address `0x0008 C0E0`
8. fill-array of size `0x28`, starting at hex-address `0x0008 C100`
   - as its name I use the address-offset inside the aggregate type
9. DSCR2-array, size `0x20`, starting at hex-address `0x0008 C128`

So `IO_` would address the (volatile) struct-contents starting at absolute address
`0x0008 C000`, and selecting for example the output data register of port 5 would
be done with `IO_._PODR[5]` and address the contents of Byte-address `0x0008 C025`,
whether it would be a read- or a write action.

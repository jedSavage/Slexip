# SLEXIP

***Note: this project is currently in a conceptual phase - it is just an idea. No interpreter nor compiler have been made yet. Suggestions and requests to the specifications are welcome and encouraged. Please submit ideas/feedback/suggestions as [issues](https://github.com/jedSavage/Slexip/issues).***

SLEXIP is a low level programming language inspired by [Piet](http://www.dangermouse.net/esoteric/piet.html) and the [6502 microprocessor](http://www.6502.org). It is not as esoteric as Piet is but it does represents code visually as pixels and allows the execution of code in different directions across the image.

## Program Data Representation

Programs for SLEXIP are written using GIF (index) images. The GIF should contain 256 indices/colors and can be any size up to the maximum limited by GIF standard (65535 x 65535); however, only the first 64k pixels are addressable. The colors chosen for each index do not matter for program execution but can be modified by the program during execution.

SLEXIP uses the image as memory in addition to program data. As the program runs, altering data in memory directly alters the image. There is no console output - altering memory is the standard way of outputting data in SLEXIP.

## Numerical Representation

Throughout this document a `$` character before any value denotes hexadecimal. For decimal values no symbol will be used (`$1F` vs `31`) and binary values will be represented with a postfix `b` (e.g. `00110110b`).

## Memory

Pixels are addressable using 2-byte words. The first pixel (pixel index 0) is located at `$0000`. Pixel addresses increase by 1 running from left to right. Addresses wrap around to the next line of pixels. In a 10x10 image, the last pixel on the 3rd line is `$001D`; the 1st pixel on the 4th line is `$001E`; and the last pixel on the last line is `$0063`.

Each pixel can represent a value from `0` to `255` (1-byte). This value also denotes the color index which that pixel should be displayed with. Operators are 1-byte long and Memory addresses are 2-bytes (16-bits) long. Some operators do not require additional data (operands), others may require a value and/or memory address which will be fetched from the pixel(s) following the operator. The combination of operator and operand is an Instruction. For example: `$40` `$42` `$04` `$B5` is a 4-byte instruction that copies (`CVM` in Direct Mode or `$40`) the value of `$42` into memory address `$04B5`. 

## Program Initialization.

After loading a program (image) into memory, the interpreter caches the values contained in the first 18 pixel locations. These pixels contain **pointers** to various memory addresses that will be used by the interpreter. These pixels can be changed immediately upon program execution as the values are cached before any code is executed. Since memory addresses are 16-bits wide, each pointer takes up 2 pixels. The only way to change the cached locations after initialization is to execute a reset (`$FF` or `RST`) operator. Note that the values in these locations are **pointers** to memory addresses not the memory addresses themselves (unless of course they point to themselves).

Once these pointers are cached, the interpreter beings executing code starting the the address stored in the Program Counter (PC).

### Initialization Pointers

#### ![Pixels $0000, $0001](shields/pixels-%240000%2C%20%240001-brightgreen.svg) - Clock-Speed Register Initialization Pointer

The first pixel in the image contains the memory location of the clock-speed register (CS-Register).  This is a 24-bit register that the interpreter sets its speed by. A value of `0` halts program execution (if interpreter has debugging capabilities, manual stepping can be done). A value of 1 means to evaluate 1 pixel per second (useful for debugging). A value of `60` means to step through 60 pixels per second (60hz). Max value is `16777215` (`$FFFFFF`) or approx 16.7mhz. The value at the location pointed to can be changed during program execution to change interpreter speed.

#### ![Pixels $0002, $0003](shields/pixels-%240002%2C%20%240003-brightgreen.svg) - Stack-Pointer Initialization Pointer

This pixel contains the memory location of the stack-pointer (SP). The SP is a 16-bit pointer containing the memory location of the top of the stack. The stack can be relocated during code execution by changing the values at the SP. The stack grows backwards: when a value is pushed to the stack, the pointer value decrements by one; when a value is popped from the stack the pointer value is increased by one.

#### ![Pixels $0004, $0005](shields/pixels-%240004%2C%20%240005-brightgreen.svg) - Input-Key Register Initialization Pointer

This pixel contains the memory location of the Input-Key Register (IK-Register) - an 8-bit register whose lower 7-bits contain ascii key code of the last key pressed. Bit-7 of this register is set when a non-modifier key is pressed down.

#### ![Pixels $0006, $0007](shields/pixels-%240006%2C%20%240007-brightgreen.svg) - Modifier-Key Register Initialization Pointer

This pixel contains the memory location of the Modifier-Key Register (MK-Register). an 8-bit register with the following bits set when the appropriate key is being pressed down. Bits are cleared as soon as the key is released.

|Bit Position|Modifier Key|
|:---:|:---|
|`Bit 0`|Up Arrow|
|`Bit 1`|Down Arrow|
|`Bit 2`|Left Arrow|
|`Bit 3`|Right Arrow|
|`Bit 4`|Shift|
|`Bit 5`|Control|
|`Bit 6`|Alt|
|`Bit 7`|Command|
	
#### ![Pixels $0008, $0009](shields/pixels-%240008%2C%20%240009-brightgreen.svg) - LFSR Initialization Pointer

This pixel contains the memory location of a 16-bit Linear Feedback Shift Register (LFSR) with taps at bits `15`, `13`, `12`, and `10`. The LFSR progresses to its next state once after every access. Changing the LFSR value to anything other than zero seeds the LFSR with that value and advances to the next state. Placing a value of `$0000` into the register effectively turns the LFSR off.

![16-Bit LFSR](image/LFSR-F16.png)

#### ![Pixels $000A, $000B](shields/pixels-%24000A%2C%20%24000B-brightgreen.svg) - Program-Counter Initialization Pointer

This pixel contains the memory location of the program-counter (PC). The PC is a 16-bit pointer that contains the memory address of the next operator to be evaluated.

#### ![Pixels $000C, $000D](shields/pixels-%24000C%2C%20%24000D-brightgreen.svg) - Status/Direction Register Initialization Pointer

This pixel contains the memory location of the status/direction register (SD-Register). Bits 0-3 contain the status flags. Bit-0 is the Carry flag (C), Bit-1 is the Zero flag (Z), Bit-2 is the Overflow flag (V), bit-3 is the Negative flag (N). These flags are changed by operators.

|Bit Position|Description|
|:---:|:---|
|`Bit-0`|Carry Flag   |
|`Bit-1`|Zero Flag    |
|`Bit-2`|Overflow Flag|
|`Bit-3`|Negative Flag|

Bits 4-5 of this register contain the direction the program is currently evaluating in. With the default value of 00, the PC increments by `1` each clock cycle, rolling over at the end of the image. These bits also affect how indexing is applied.

|Bits 4 & 5|Program Direction|PC Change per Clock|Additional at Rollover|
|:---:|:---|:---|:---:|
|`00b` |Right|+ `1`          |None  |
|`01b` |Down |+ `CW-Register`| + `1`|
|`10b` |Left |- `1`          |None  |
|`11b` |Up   |- `CW-Register`| - `1`|

Bits 6 and 7 are used to control the behavior of the LFSR. If bit-6 is set, the taps are mirrored so that the output progresses in the reverse order. Bit 7 controls how often the LFSR progresses to the next output. When set, the LFSR progresses once per clock tick.

|Bit 6 State|LFSR Direction|
|:---:|:---|
|`0b`|Forward|
|`1b`|Reverse|

|Bit 7 State|LFSR Progression Behavior|
|:---:|:---|
|`0b`|Once after each read or write access|
|`1b`|Once every clock tick|

#### ![Pixels $000E, $000F](shields/pixels-%24000E%2C%20%24000F-brightgreen.svg) - Canvas-Width Register Initialization Pointer

This pixel contains the memory location of the canvas-width Register (CW-Register). The CW-Register holds a 16-bit value representing the width of the canvas in pixels. Changing the value at the CW-Register will dynamically change the width of the canvas. New pixels will be added with a default value of `$EE`. Clipped pixels will be permanently lost. The interpreter will update the displayed image as needed. Note that new pixels are added or removed from the end of memory, which can result in the image being distored after resizing.

#### ![Pixels $0010, $0011](shields/pixels-%240010%2C%20%240011-brightgreen.svg) - Canvas-Height Register Initialization Pointer

This pixel contains the memory location of the canvas-height Register (CH-Register), which work similarly to the CW-Register.

## Memory Addressing Modes
There are seven modes that can be used with operators. Not all modes are available with all operators.

### Implied (IM)
In implied mode, the data and/or memory that the operator works with is implied resulting in single-byte instructions. Status operators, for example, work solely on the bits of the status register.

### Non-Indexed

#### Relative (R)
This mode is only available with branch operators. The number provided to the operator is added to current PC and program execution continues from that address. Only one byte is passed and it is a signed byte. This allows a jump of `-128` to `+127` bytes relative to the current program counter.

#### Direct (D)
Uses the value directly as entered.  Example, using the JMP operator in direct mode, the PC is set to the value provided and program execution continues from that address.

#### Indirect (I)
Fetches the value from the provided memory address, and uses that value with the operator. Example, using the JMP operator in indirect mode: `JMP` `$44` `$02` - fetches the value from memory address `$4402` and sets the PC to that value.

### Indexed

Indexing follows the same rules as the program counter according to bits 4 & 5 of the `SD-Register`.

#### Direct Indexed (DX)
The value at the index location is added to the value provided. The sum is then used in the operation.

#### Indexed Indirect (XI)
The value at the index location is added to the memory address provided. The value at the new location is then used in the operation.

#### Indirect Indexed (IX)
The value at the memory location is fetched and the value at the index location is added to it. The sum is then used in the operation.

## Instruction Set:

There are 64 operators. The chart below shows their op-codes. All other opcodes are interpreted as `NOP` instructions. Program Counter offsets listed are based upon the Direction Register bits being set to `00b`. For other direction configurations, see [Direction Register](#pixels-000c-000d---statusdirection-register-initialization-pointer).

![](image/Opcode-Matrix.png)

### Data Transport operators

![$XX](shields/op-xx-red.svg) = 1-Byte Op-Code  
![VAL](shields/-VAL-pink.svg) = 1-Byte Value

Memory addresses are 16-bits wide. Instruction use a high-byte followed by a low-byte to specify an address.  
![MEM2-HB](shields/TMEM-HB-gray.svg) = Target Memory High Byte  
![MEM2-LB](shields/TMEM-LB-gray.svg) = Target Memory Low Byte

![MEM1-HB](shields/SMEM-HB-gray.svg) = Source Memory High Byte  
![MEM1-LB](shields/SMEM-LB-gray.svg) = Source Memory Low Byte  
  
Where two memory addresses are used in an indexed mode, indexing applies to both source and target address.  
![IND-HB](shields/IND-HB-gray.svg) = Index High Byte  
![IND-LB](shields/IND-LB-gray.svg) = Index Low Byte
  
---
![CVM](shields/op-CVM-red.svg) ![Flags: [-Z-N--]](shields/flags/Flags-ZN-white.svg)  
Copy a value into memory.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![40](shields/opcodes/oc-40-red.svg) ![VAL](shields/-VAL-pink.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg)            |+4|
|Indirect        |![60](shields/opcodes/oc-60-red.svg) ![VAL](shields/-VAL-pink.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg)            |+4|
|Direct Indexed  |![C0](shields/opcodes/oc-C0-red.svg) ![VAL](shields/-VAL-pink.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+6|
|Indexed Indirect|![80](shields/opcodes/oc-80-red.svg) ![VAL](shields/-VAL-pink.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+6|
|Indirect Indexed|![A0](shields/opcodes/oc-A0-red.svg) ![VAL](shields/-VAL-pink.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+6|

---
![CMM](shields/op-CMM-red.svg) ![Flags: [-Z-N--]](shields/flags/Flags-ZN-white.svg)  
Copy from first memory address to the second.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![41](shields/opcodes/oc-41-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)            |+5|
|Indirect        |![61](shields/opcodes/oc-61-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)            |+5|
|Direct Indexed  |![C1](shields/opcodes/oc-C1-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|
|Indexed Indirect|![81](shields/opcodes/oc-81-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|
|Indirect Indexed|![A1](shields/opcodes/oc-A1-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|

---
### Arithmetic operators

![ADC](shields/op-ADC-red.svg) ![Flags: [CZVN--]](shields/flags/Flags-CZVN-white.svg)  
Add (with carry) two memory values, replacing the first location with result. Indexed modes apply to the second memory address.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![42](shields/opcodes/oc-42-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)        |+5|
|Indirect        |![62](shields/opcodes/oc-62-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)        |+5|
|Direct Indexed  |![C2](shields/opcodes/oc-C2-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|
|Indexed Indirect|![82](shields/opcodes/oc-82-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|
|Indirect Indexed|![A2](shields/opcodes/oc-A2-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|

---
![SBC](shields/op-SBC-red.svg) ![Flags: [CZVN--]](shields/flags/Flags-CZVN-white.svg)  
Subtract (with borrow) two memory values, replacing the first memory location with result. The second memory value gets subtracted from the first. Indexed modes apply to the second memory address.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![43](shields/opcodes/oc-43-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)        |+5|
|Indirect        |![63](shields/opcodes/oc-63-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)        |+5|
|Direct Indexed  |![C3](shields/opcodes/oc-C3-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|
|Indexed Indirect|![83](shields/opcodes/oc-83-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|
|Indirect Indexed|![A3](shields/opcodes/oc-A3-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|

---
![INC](shields/op-INC-red.svg) ![Flags: [-Z-N--]](shields/flags/Flags-ZN-white.svg)  
Increment TMEMory by one.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![45](shields/opcodes/oc-45-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg)        |+3|
|Direct Indexed  |![C5](shields/opcodes/oc-C5-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+5|

---
![DEC](shields/op-DEC-red.svg) ![Flags: [-Z-N--]](shields/flags/Flags-ZN-white.svg)  
Decrement TMEMory by one.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![44](shields/opcodes/oc-44-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg)        |+3|
|Direct Indexed  |![C4](shields/opcodes/oc-C4-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+5|

---
### Status operators

![CLC](shields/op-CLC-red.svg) ![Flags: [C------]](shields/flags/Flags-C-white.svg)  
Clear the carry flag.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Implied|![E0](shields/opcodes/oc-E0-red.svg)|+1|

---
![SEC](shields/op-SEC-red.svg) ![Flags: [C------]](shields/flags/Flags-C-white.svg)  
Set the carry flag.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Implied|![E1](shields/opcodes/oc-E1-red.svg)|+1|

---
![CLV](shields/op-CLV-red.svg)![Flags: [--V----]](shields/flags/Flags-V-white.svg)  
Clear overflow flag.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Implied|![F0](shields/opcodes/oc-F0-red.svg)|+1|

---
### Logical operators

![AND](shields/op-AND-red.svg) ![Flags: [-Z-N--]](shields/flags/Flags-ZN-white.svg)  
Logical AND two memory locations, replacing the first memory location with result.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![50](shields/opcodes/oc-50-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)        |+5|
|Direct Indexed  |![D0](shields/opcodes/oc-D0-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|

---
![ORM](shields/op-ORM-red.svg) ![Flags: [-Z-N--]](shields/flags/Flags-ZN-white.svg)  
Logical OR two memory locations, replacing the first memory location with result.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![51](shields/opcodes/oc-51-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)        |+5|
|Direct Indexed  |![D1](shields/opcodes/oc-D1-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|

---
![XOR](shields/op-XOR-red.svg) ![Flags: [-Z-N--]](shields/flags/Flags-ZN-white.svg)  
Logical XOR two memory locations, replacing the first memory location with result.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![52](shields/opcodes/oc-52-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)        |+5|
|Direct Indexed  |![D2](shields/opcodes/oc-D2-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|

---
![SHL](shields/op-SHL-red.svg) ![Flags: [CZ-N--]](shields/flags/Flags-CZN-white.svg)  
Shifts all bits of a memory location left one position. 0 is shifted into bit-0 and the original bit-7 is shifted into the Carry.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![53](shields/opcodes/oc-53-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)        |+5|
|Direct Indexed  |![D3](shields/opcodes/oc-D3-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|

---
![SHR](shields/op-SHR-red.svg) ![Flags: [CZ-N--]](shields/flags/Flags-CZN-white.svg)  
Shifts all bits of a memory location right one position. 0 is shifted into bit-7 and the original bit-0 is shifted into the Carry.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![54](shields/opcodes/oc-54-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)        |+5|
|Direct Indexed  |![D4](shields/opcodes/oc-D4-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|

---
![ROL](shields/op-ROL-red.svg) ![Flags: [CZ-N--]](shields/flags/Flags-CZN-white.svg)  
Shifts all bits of a memory location left one position. The Carry is shifted into bit 0 and the original bit 7 is shifted into the Carry.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![55](shields/opcodes/oc-55-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)        |+5|
|Direct Indexed  |![D5](shields/opcodes/oc-D5-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|

---
![ROR](shields/op-ROR-red.svg) ![Flags: [CZ-N--]](shields/flags/Flags-CZN-white.svg)  
Shifts all bits of a memory location right one position. The Carry is shifted into bit 7 and the original bit 0 is shifted into the Carry.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![56](shields/opcodes/oc-56-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg)        |+5|
|Direct Indexed  |![D6](shields/opcodes/oc-D6-red.svg) ![MEM1-HB](shields/SMEM-HB-gray.svg) ![MEM1-LB](shields/SMEM-LB-gray.svg) ![MEM2-HB](shields/TMEM-HB-gray.svg) ![MEM2-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+7|

---
### Branch operators

![VAL](shields/-VAL-pink.svg) = 1-Byte Offset Value (Signed).

![BCC](shields/op-BCC-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Branch on carry clear - Branch to address if the carry flag is clear. (Increments the PC by VAL).

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Relative          |![20](shields/opcodes/oc-20-red.svg) ![VAL](shields/-VAL-pink.svg)|+2 \| `VAL`|

---
![BCS](shields/op-BCS-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Branch on carry set - Branch if the carry flag is set.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Relative          |![21](shields/opcodes/oc-21-red.svg) ![VAL](shields/-VAL-pink.svg)|+2 \| `VAL`|

---
![BNE](shields/op-BNE-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Branch on non-zero - Branch if the zero flag is clear.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Relative          |![22](shields/opcodes/oc-22-red.svg) ![VAL](shields/-VAL-pink.svg)|+2 \| `VAL`|

---
![BEQ](shields/op-BEQ-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Branch on zero - Branch if the zero flag is set.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Relative          |![23](shields/opcodes/oc-23-red.svg) ![VAL](shields/-VAL-pink.svg)|+2 \| `VAL`|

---
![BPL](shields/op-BPL-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Branch on positive - Branch if the negative flag is clear.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Relative          |![24](shields/opcodes/oc-24-red.svg) ![VAL](shields/-VAL-pink.svg)|+2 \| `VAL`|

---
![BMI](shields/op-BMI-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Branch on negative - Branch if the negative flag is set.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Relative          |![25](shields/opcodes/oc-25-red.svg) ![VAL](shields/-VAL-pink.svg)|+2 \| `VAL`|

---
![BVC](shields/op-BVC-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Branch on overflow clear - Branch if the overflow flag is clear.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Relative          |![26](shields/opcodes/oc-26-red.svg) ![VAL](shields/-VAL-pink.svg)|+2 \| `VAL`|

---
![BVS](shields/op-BVS-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Branch on overflow set - Branch if the overflow flag is set.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Relative          |![27](shields/opcodes/oc-27-red.svg) ![VAL](shields/-VAL-pink.svg)|+2 \| `VAL`|

---
### Comparison Operators

![CMP](shields/op-CMP-red.svg) ![Flags: [CZ-N--]](shields/flags/Flags-CZN-white.svg)  
Compare two TMEMory locations. Sets zero flag if values are identical. Sets carry flag if the first TMEMory value is equal to or greater than the second TMEMory value.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct          |![46](shields/opcodes/oc-46-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg)        |+3|
|Indirect        |![66](shields/opcodes/oc-66-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg)        |+3|
|Direct Indexed  |![C6](shields/opcodes/oc-C6-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+5|
|Indexed Indirect|![86](shields/opcodes/oc-86-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+5|
|Indirect Indexed|![A6](shields/opcodes/oc-A6-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg) ![IND-HB](shields/IND-HB-gray.svg) ![IND-LB](shields/IND-LB-gray.svg)|+5|

---
### Program Control

![JMP](shields/op-JMP-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Set PC to new value, altering flow of program.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct    |![5F](shields/opcodes/oc-5F-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg)|Set to TMEMory|
|Indirect  |![6F](shields/opcodes/oc-6F-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg)|Set to Value at TMEMory|

---
![JSR](shields/op-JSR-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Jump sub routine. Pushes the address of the next operator to the stack, then sets the PC to a new TMEMory value.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct    |![4F](shields/opcodes/oc-4F-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg)|Set to TMEMory|

---
![RSR](shields/op-RSR-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Return from subroutine, pop the stack value into the PC and continues evaluating from there.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Implied    |![EF](shields/opcodes/oc-EF-red.svg)|Set to Value from Stack|

---
### Stack operators

![PHS](shields/op-PHM-red.svg) ![Flags: [-Z-N--]](shields/flags/Flags-ZN-white.svg)  
Push TMEMory to stack.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct    |![4A](shields/opcodes/oc-4A-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg)|+3|

---
![PHS](shields/op-PHS-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Push status to stack.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Implied    |![FA](shields/opcodes/oc-FA-red.svg)|+1|

---
![PLM](shields/op-PLM-red.svg) ![Flags: [-Z-N--]](shields/flags/Flags-ZN-white.svg)  
Pop from stack into TMEMory locaton.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct    |![4B](shields/opcodes/oc-4B-red.svg) ![TMEM-HB](shields/TMEM-HB-gray.svg) ![TMEM-LB](shields/TMEM-LB-gray.svg)|+3|

---
![PLS](shields/op-PLS-red.svg) ![Flags: [CZVN--]](shields/flags/Flags-CZVN-white.svg)  
Pop from stack into status register.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Implied    |![FB](shields/opcodes/oc-FB-red.svg)|+1|

---
### Reset

![RST](shields/op-RST-red.svg) ![Flags: [CZVNDD]](shields/flags/Flags-CZVNDD-white.svg)  
Reset - Runs the interpreter's initialization routine, resets all cached references to pointers.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Implied|![FF](shields/opcodes/oc-FF-red.svg)|N/A|

---
### GIF Index operators

![IVAL](shields/-IVAL-lightgray.svg) = 1-Byte Index  
![RVAL](shields/-RVAL-red.svg) = 1-Byte Red Value  
![GVAL](shields/-GVAL-green.svg) = 1-Byte Green Value  
![BVAL](shields/-BVAL-blue.svg) = 1-Byte Blue Value

![IDX](shields/op-IDX-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
Index Color Modifier - Instructs the interpreter to change the color of an index.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|Direct    |![5E](shields/opcodes/oc-5E-red.svg) ![IVAL](shields/-IVAL-lightgray.svg) ![RVAL](shields/-RVAL-red.svg) ![GVAL](shields/-GVAL-green.svg) ![BVAL](shields/-BVAL-blue.svg)|+5|

---
### Null operators

![NOP](shields/op-NOP-red.svg) ![Flags: [------]](shields/flags/Flags-EMPTY-white.svg)  
No operator - Does nothing.

|Addressing Mode|Instruction Format|PC Offset|
|---:|:---|:---|
|N/A |All Other Op-Codes|+1|

---
## Operator Evaluation Cycle

1. Operator is fetched from the memory location reference by the `PC`.
2. Depending on the operator, the operands are fetched from the memory following the operator.
3. The full Instruction is evaluated, and any related memory is read from and written to.
4. The flags of the status-register are updated to reflect the operation.
5. The `PC` is incremented/decremented as neccesary depending on `SD-Register` bits `4` and `5`, and the length of the instruction, see [Direction Register](#pixels-000c-000d---statusdirection-register-initialization-pointer).
6. The `LFRS` is cycled the same amount as the instruction length.
7. The `CS-Register` is read and the interpreter speed is adjusted if needed.
8. The `IK-Register` and `MK-Register` are both updated, depending on the keys currently pressed.
9. The `CW-Register ` and `CH-Register` are read and the image size displayed is adjusted if needed.


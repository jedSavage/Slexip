# SLEXIP

***Note: this project is currently in a conceptual phase - it is just an idea. No interpreter has been made yet. Suggestions and requests to the specifications are welcome and encouraged. Please submit ideas/feedback/suggestions as [issues](https://github.com/jedSavage/Slexip/issues).***

SLEXIP is a low level programming language inspired by [Piet](http://www.dangermouse.net/esoteric/piet.html) and the [6502 microprocessor](http://www.6502.org). It is not as esoteric as Piet is but does represents code visually as pixels and allows the execution of code in different directions across the image.

## Data Representation

Programs for SLEXIP are written using GIF (index) images. The image must be 256 colors and can be any size up to the maximum limited by GIF standard (65535 x 65535), however only the first 64k pixels are addressable. The colors chosen for each index do not matter for program execution. When reaching the end of the image, the program counter rolls over.

SLEXIP uses the image as its code as well memory. As the program runs, altering data in memory directly alters the image. There is no console output - altering memory is the standard way of outputting data.

Each pixel can represent a value from 0 to 255 (1-byte). This value not only denotes the color index, but the value of the memory in that pixel location. Operators are 1-byte long and Memory addresses are 2-bytes (16-bits) long. Some operators do not require additional data, others may require a memory address or value which will be fetched from the pixel(s) following the operator pixel. The combination of operator and related data pixels is an Instruction. For example: $00 $42 $04 $B5 is a 4-byte instruction that copies (operator $00 or CVM) the value of $42 into memory address $04B5. The $ character before the values indicate hexadecimal values. For decimal values, a # symbol will be used throughout this document ($1F vs #31).

## Program initialization.

After loading a program (image) into memory, the interpreter caches the values contained in the first 18 pixel locations. These pixel locations contain pointers to various memory addresses that will be used by the interpreter. These pixels can be changed immediately upon program execution as the values are cached before any code is executed. Since memory addresses are 16-bits wide, each pointer takes up 2 pixels. The only way to change the cached locations after initialization is to execute a reset ($FF or RST) operator. Note that the values in these locations are **pointers** to memory addresses not the memory addresses themselves (unless of course they point to themselves).

**Pixel 0** (The first pixel) contains a pointer to the speed-value (24-bit) that the interpreter should evaluate each pixel at. A value of #0 halts program execution (if interpreter has debugging capabilities, manual stepping can be done). A value of 1 means to evaluate 1 pixel per second (useful for debugging). A value of #60 means to step through 60 pixels per second (60hz). Max value is #16777215 ($FFFFFF) or approx 16.7mhz. The value at the location pointed to can be changed during program execution to change interpreter speed.

Example: If upon initialization pixel 0 and 1 contain $02 and $FF reprectively, then the interpreter will look to pixels #767, #768, and #769 (24-bit) and base it's speed upon the values it finds there.  If any of those pixels are changed during program execution, the interpreter will update it's speed to match.

**Pixel 2** contains a pointer to the stack-pointer (SP). The top of the stack is pointed to at the location pointed to by this pixel **(double pointer)**. The stack extends backwards from the pointed to location as it grows. The stack can be relocated during code execution by changing the values at the pointer location. When a value is pushed to the stack, the pointer value decrements by one. When a value is popped from the stack the pointer value is increased by one.

Example: If upon initialization pixels 2 and 3 contain $55 and $0C respectively, the interpreter will look to pixels #21772 and #21773 for the location of the top of the stack. If pixel #21772 and #21773 contains $00 and $FF respectively, then the top of the stack is located at pixel #255.

**Pixel 4** contains a pointer to the IK (Input Key) register, which contains the 7-bit ascii key code of the last key pressed. Bit-7 of this register will be set if a non-modifier key is currently being pressed down.

Example: If upon initialization pixels 2 and 3 contain $FF and $20 repectively, then the ascii value of the last key pressed will be located at pixel #65312.

**Pixel 6** contains a pointer to the MK (Modifier Key) register. The following bits are set when the appropriate key is being pressed down. They are cleared as soon as the key is released.
	*Bit 0 - Up Arrow
	*Bit 1 - Down Arrow
	*Bit 2 - Left Arrow
	*Bit 3 - Right Arrow
	*Bit 4 - Shift
	*Bit 5 - Control
	*Bit 6 - Alt
	*Bit 7 - Command
	
**Pixel 8** contains a pointer to a 16-bit Linear Feedback Shift Register (LFSR). Setting this to 0 turns off the LFSR feature. The LFSR progresses to the next output once every pixel evaluation. Changing this value to anything other than zero seeds the LFSR with that value.

**Pixel 10** contains a pointer to the program counter. This points to the next instruction to be evaluated.

**Pixel 12** contains a pointer to the status and direction register. Bits 0-3 contain the status flags. Bit-0 is the Carry flag (C), Bit-1 is the Zero flag (Z), Bit-2 is the Overflow flag (V), bit-3 is the Negative flag (N). These flags are changed by operators.

Bits 4-5 of this register contain the direction the program is currently evaluating in. 00=right (default), 01=down, 10=left, 11=up. Bits 6 and 7 are unused but can be read from and written to. With the default calue of 00, the PC increments by 1 after each pixel is accessed, rolling over at the end of the image. Setting this to down adds the width of the image to the PC instead (moving down one pixel). Upon rolling over, 1 is also added to the PC; up subtracts the width (-1 upon roll over); left decrements the PC instead of incrementing it.

**Pixel 14** contains a pointer to the width of the canvas. Changing the value at the address pointed to by this value will dynamically change the width of the canvas. New pixels will be added with default values of $FF. Clipped pixels will be lost. The interpreter will update the image as needed.

**Pixel 16** contains a pointer to the height of the canvas. Changing the value at the address pointed to by this value will dynamically change the height of the canvas. New pixels will be added with default values of $FF. Clipped pixels will be lost. The interpreter will update the image as needed.

Upon initialization or when a reset instruction is encountered:
1) The interpreter caches the speed-value (SV) in pixel 1 and sets its speed to match 1/SV seconds. This is the speed at which each pixel should be evaluated.
2) The interpreter caches the stack-pointer pointer (SPP). Calls to stack-based operators will use the value located at this pointer to point to the top of the stack.
3) The interpreter caches the IK-pointer. The bits of this location will be modified accoring to keyboard input.
4) The interpreter caches the MK-pointer. The bits of this location will be modified accoring to keyboard input.
5) The interpreter caches the LFSR-pointer.
6) The interpreter caches the Program Counter (PC) Pointer. This is where the program counter will reside.
7) The interpreter caches the Status / Direction register.
8) The interpreter caches the Canvas Width Pointer (CW).
9) The interpreter caches the Canvas Height Pointer (CH).
10) The interpreter begins evaluating pixels at the address pointed to by the PC.

## Memory Addressing Modes
There are seven modes that can be used with operators. Not all modes are available with all operators.

### Implied (IM)
In implied mode, the data and/or memory that the operator works with is implied resulting in single-byte instructions. Status operators, for example, work solely on the bits of the status register.

### Non-Indexed

#### Relative (R)
This mode is only available with branch operators. The number provided to the operator is added to current PC and program execution continues from that address. Only one byte is passed and it is a signed byte. This allows a jump of -128 to +127 bytes relative to the current program counter.

#### Direct (D)
Uses the value directly as entered.  Example, using the JMP operator in direct mode, the PC is set to the value provided and program execution continues from that address.

#### Indirect (I)
Fetches the value from the provided memory address, and uses that value with the operator. Example, using the JMP operator in indirect mode: JMP $44 $02 - fetches the value from memory address $4402 and sets the PC to that value.

### Indexed

#### Direct Indexed (DX)
The value at the index location is added to the value provided. The sum is then used in the operation.

#### Indexed Indirect (XI)
The value at the index location is added to the memory address provided. The value at the new location is then used in the operation.

#### Indirect Indexed (IX)
The value at the memory location is fetched and the value at the index location is added to it. The sum is then used in the operation.

## Instruction Set:

There are 64 operators. The chart below shows their op-codes. All other opcodes are interpreted as NOP instructions.

![](/image/Opcode-Matrix.png)

### Data Transport operators

VAL = value; MHB = Memory High Byte; MLB = Memory Low Byte; IHB = Index High Byte; ILB = Index Low Byte

**CVM** Copy a value into memory.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$40 VAL MHB MLB|
|Indirect        |$60 VAL MHB MLB|
|Direct Indexed  |$C0 VAL MHB MLB IHB ILB|
|Indexed Indirect|$80 VAL MHB MLB IHB ILB|
|Indirect Indexed|$A0 VAL MHB MLB IHB ILB|

Flags Affected: [-Z-N--]

**CMM** Copy from first memory address to the second.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$41 VAL MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Indirect        |$61 VAL MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Direct Indexed  |$C1 VAL MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|
|Indexed Indirect|$81 VAL MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|
|Indirect Indexed|$A1 VAL MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|

Flags Affected: [-Z-N--]

### Arithmetic operators

**ADC** Add (with carry) two memory values, replacing the first location with result. Indexed modes apply to the second memory address.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$42 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Indirect        |$62 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Direct Indexed  |$C2 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|
|Indexed Indirect|$82 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|
|Indirect Indexed|$A2 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|

Flags Affected: [CZVN--]

**SBC** Subtract (with borrow) two memory values, replacing the first memory location with result. The second memory value gets subtracted from the first. Indexed modes apply to the second memory address.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$43 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Indirect        |$63 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Direct Indexed  |$C3 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|
|Indexed Indirect|$83 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|
|Indirect Indexed|$A3 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|

Flags Affected: [CZVN--]

**INC** Increment memory by one.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$45 MHB MLB|
|Direct Indexed  |$C5 MHB MLB IHB ILB|

Flags Affected: [-Z-N--]

**DEC** Decrement memory by one.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$44 MHB MLB|
|Direct Indexed  |$C4 MHB MLB IHB ILB|

Flags Affected: [-Z-N--]

### Status operators

**CLC** Clear the carry flag.

|Addressing Mode|Instruction Format|
|---:|:---|
|Implied|$E0|

Flags Affected: [C------]

**SEC** Set the carry flag.

|Addressing Mode|Instruction Format|
|---:|:---|
|Implied|$E1|

Flags Affected: [C------]

**CLV** Clear overflow flag.

|Addressing Mode|Instruction Format|
|---:|:---|
|Implied|$F0|

Flags Affected: [--V----]

### Logical operators

**AND** Logical AND two memory locations, replacing the first memory location with result.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$50 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Direct Indexed  |$D0 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|

Flags Affected: [-Z-N--]

**ORM** Logical OR two memory locations, replacing the first memory location with result.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$51 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Direct Indexed  |$D1 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|

Flags Affected: [-Z-N--]

**XOR** Logical XOR two memory locations, replacing the first memory location with result.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$52 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Direct Indexed  |$D2 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|

Flags Affected: [-Z-N--]

**SHL** Shifts all bits of a memory location left one position. 0 is shifted into bit-0 and the original bit-7 is shifted into the Carry.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$53 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Direct Indexed  |$D3 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|

Flags Affected: [CZ-N--]

**SHR** Shifts all bits of a memory location right one position. 0 is shifted into bit-7 and the original bit-0 is shifted into the Carry.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$54 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Direct Indexed  |$D4 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|

Flags Affected: [CZ-N--]

**ROL** Shifts all bits of a memory location left one position. The Carry is shifted into bit 0 and the original bit 7 is shifted into the Carry.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$55 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Direct Indexed  |$D5 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|

Flags Affected: [CZ-N--]

**ROR** Shifts all bits of a memory location right one position. The Carry is shifted into bit 7 and the original bit 0 is shifted into the Carry.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$56 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub>|
|Direct Indexed  |$D6 MHB<sub>1</sub> MLB<sub>1</sub> MHB<sub>2</sub> MLB<sub>2</sub> IHB ILB|

Flags Affected: [CZ-N--]

### Branch operators

OFS = Offset  (Signed byte)

**BCC** Branch on carry clear - Branch to address if the carry flag is clear. (Increments the PC by OFS).

|Addressing Mode|Instruction Format|
|---:|:---|
|Relative          |$20 OFS|

Flags Affected: [------]

**BCS** Branch on carry set - Branch if the carry flag is set.

|Addressing Mode|Instruction Format|
|---:|:---|
|Relative          |$21 OFS|

Flags Affected: [------]

**BNE** Branch on non-zero - Branch if the zero flag is clear.

|Addressing Mode|Instruction Format|
|---:|:---|
|Relative          |$22 OFS|

Flags Affected: [------]

**BEQ** Branch on zero - Branch if the zero flag is set.

|Addressing Mode|Instruction Format|
|---:|:---|
|Relative          |$23 OFS|

Flags Affected: [------]

**BPL** Branch on positive - Branch if the negative flag is clear.

|Addressing Mode|Instruction Format|
|---:|:---|
|Relative          |$24 OFS|

Flags Affected: [------]

**BMI** Branch on negative - Branch if the negative flag is set.

|Addressing Mode|Instruction Format|
|---:|:---|
|Relative          |$25 OFS|

Flags Affected: [------]

**BVC** Branch on overflow clear - Branch if the overflow flag is clear.

|Addressing Mode|Instruction Format|
|---:|:---|
|Relative          |$26 OFS|

Flags Affected: [------]

**BVS** Branch on overflow set - Branch if the overflow flag is set.

|Addressing Mode|Instruction Format|
|---:|:---|
|Relative          |$27 OFS|

Flags Affected: [------]

### Comparison operators

**CMP** Compare two memory locations. Sets zero flag if values are identical. Sets carry flag if the first memory value is equal to or greater than the second memory value.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct          |$46 MHB MLB|
|Indirect        |$66 MHB MLB|
|Direct Indexed  |$C6 MHB MLB IHB ILB|
|Indexed Indirect|$86 MHB MLB IHB ILB|
|Indirect Indexed|$A6 MHB MLB IHB ILB|

Flags Affected: [CZ-N--]

### Program Control

**JMP** Set PC to new value, altering flow of program.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct    |$5F MHB MLB|
|Indirect  |$6F MHB MLB|

Flags Affected: [------]

**JSR** Jump sub routine. Pushes the address of the next operator to the stack, then sets the PC to a new memory value.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct    |$4F MHB MLB|

Flags Affected: [------]

**RSR** Return from subroutine, pop the stack value into the PC and continues evaluating from there.

|Addressing Mode|Instruction Format|
|---:|:---|
|Implied    |$EF|

Flags Affected: [------]

### Stack operators

**PHM** Push memory to stack.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct    |$4A MHB MLB|

Flags Affected: [-Z-N--]

**PHS** Push status to stack.

|Addressing Mode|Instruction Format|
|---:|:---|
|Implied    |$FA|

Flags Affected: [------]

**PLM** Pop from stack into memory locaton.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct    |$4B MHB MLB|

Flags Affected: [-Z-N--]

**PLS** Pop from stack into status register.

|Addressing Mode|Instruction Format|
|---:|:---|
|Implied    |$FB|

Flags Affected: [CZVN--]

### Reset

**RST** Reset - Runs the interpreter's initialization routine, resets all cached references to pointers.

|Addressing Mode|Instruction Format|
|---:|:---|
|Implied|$FF|

Flags Affected: [CZVNDD]  

### GIF Index operators

IND = Index number; RVL = Red value; GVL = Green Value; BVL = Blue Value.

**IDX** Index Color Modifier - Instructs the interpreter to change the color of an index.

|Addressing Mode|Instruction Format|
|---:|:---|
|Direct    |$5E IND RVL BVL GVL|

Flags Affected: [------]

### Null operators

**NOP** No operator - Does nothing.

|Addressing Mode|Instruction Format|
|---:|:---|
|Implied    |All Other Op-Codes|

Flags Affected: [------]

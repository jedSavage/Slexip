# SLEXIP

SLEXIP is a low level programming language that uses pixels as data. It was inspired by Piet, and the 6502 microprocessor. It is not as esoteric as Piet but still offeres the ability to visually represent code as pixels.

## Data Representation

Programs for SLEXIP are written using GIF (index) images. The images must be 256 colors.  They can be any size up to the maximum limited by GIF standard. The colors chosen for each index do not matter for program execution.

SLEXIP also uses the image as its memory. As the program runs, the image changes. Altering memory locations alters the image.

Data is encoded into pixels. Each pixel can represent a value from 0 to 255 (1-byte). Larger value can be represented by using more than one pixel but must be interpreted as such by the program. Operations are 1-byte long and Memory addresses are 2-bytes (16-bits) long. Some operations do not require additional data, others may require a memory address or value which will be fetched from the pixel(s) following the operation pixel. The combination of operation and related data pixels is an Instruction. For example: $00 $42 $04 $B5 is a 4-byte instruction that copys (Operation $00 or CVM) the value of $42 into memory address $04B5. The $ character before the values indicate hexadecimal values. For decimal values, a # symbol will be used ($1F vs #31). If no symbol precedes the number, decimal can be assumed.

## Program initialization.

The interpreter initializes itself based on the values contained in specific pixel locations. These pixel locations contain pointers to various memory addresses that will be used by the interpreter. The pixels can be changed immediately upon program execution as the values are cached before any code is executed. As memory addresses are 16-bit, each pointer takes up 2 pixels. The only way to change the pointer locations after initialization is to execute a reset ($FF or RST) operation.

**Pixel 0** (The first pixel) contains a pointer to the speed-value (24-bit) that the interpreter should evaluate each pixel at. A value of 0 halts program execution (if interpreter has debugging capabilities, manual stepping can be done). A value of 1 means to evaluate 1 pixel per second (useful for debugging). A value of 60 means to step through 60 pixels per second (60hz). Max value is 16777215 or approx 16.7mhz. The value at the location pointed to can be changed during program execution to change interpreter speed. Example: If upon initialization pixel 0 and 1 contain $02 and $FF, then the interpreter will look at pixel #767 and base it's speed upon the value it finds there.  When pixel #767 gets changed, the interpreter will update it's speed to match.

**Pixel 2** contains the stack-pointer pointer. The top of the stack is pointed to at this location. The stack extends backwards from the pointed to location as it grows. The stack can be relocated during code execution by changing the value at the pointer location. When a value is pushed to the stack, the pointer value decrements by one. When a value is popped from the stack the pointer value is increased by one.

**Pixel 4** contains a pointer to the IK (Input Key) register, which contains the 7-bit ascii key code of the current or last key pressed. Bit-7 of thes register will be set if a non-modifier key is currently being pressed down.

**Pixel 6** contains a pointer to the MK (Modifier Key) register. The following bits are set when the appropriate key is being pressed down. They are cleared as soon as the key is released.
	*Bit 0 - Up Arrow
	*Bit 1 - Down Arrow
	*Bit 2 - Left Arrow
	*Bit 3 - Right Arrow
	*Bit 4 - Shift
	*Bit 5 - Control
	*Bit 6 - Alt
	*Bit 7 - Command
	
**Pixel 8** contains a pointer to a 16-bit LFSR. Setting this to 0 turns off the LFSR feature. The LFSR progresses to the next output once every pixel evaluation. Changing this value to anything other than zero seeds the LFSR with that value.

**Pixel 10** contains a pointer to the program counter. This points to the next instruction to be evaluated.

**Pixel 12** contains a pointer to the status and direction register. Bits 0-3 contain the status flags. Bit-0 is the Carry flag (C), Bit-1 is the Zero flag (Z), Bit-2 is the Overflow flag (V), bit-3 is the Negative flag (N). Bits 4-5 of this register contain the direction the PC is currently evaluating in. 00=right (default), 01=down, 10=left, 11=up. Bits 6 and 7 are unused but can be read from and written to.

**Pixel 14** contains a pointer to the width of the canvas. Changing the value at the address pointed to by this value will dynamically change the width of the canvas. New pixels will be added with default values of $FF. Clipped pixels will be lost.

**Pixel 16** contains a pointer to the height of the canvas. Changing the value at the address pointed to by this value will dynamically change the height of the canvas. New pixels will be added with default values of $FF. Clipped pixels will be lost.

Upon initialization or when a reset instruction is encountered:
1) The interpreter caches the speed-value (SV) in pixel 1 and sets its speed to match 1/SV seconds. This is the speed at which each pixel should be evaluated.
2) The interpreter caches the stack-pointer pointer (SPP). Calls to stack-based operations will use the value located at this pointer to point to the top of the stack.
3) The interpreter caches the IK-pointer. The bits of this location will be modified accoring to keyboard input.
4) The interpreter caches the MK-pointer. The bits of this location will be modified accoring to keyboard input.
5) The interpreter caches the LFSR-pointer.
6) The interpreter caches the Program Counter (PC) Pointer. This is where the program counter will reside.
7) The interpreter caches the Status / Direction register.
8) The interpreter caches the Canvas Width Pointer (CW).
9) The interpreter caches the Canvas Height Pointer (CH).
10) The interpreter begins evaluating pixels at the address pointed to by the PC.

## Instruction Set:

There are 32 operations. Only the lower 5 bits of a pixel represent the operation so they repeat after $00 to $1F: $20 to $3F, $40 to $5F, etc... Up to $FE. $FF is the reset instruction and does not repeat. This repetition allows some freedom in choosing more than one color index where an instruction sits. For example, the color indexes $0E, $2E, $4E, $6E, $8E, $AE, $CE, and $EE are all INC instructions.

**RST:** $FF - Reset - Runs the interpreter's initialization routine, resets all caches references to pointer. [] 1-Byte Instruction

**CVM:** $00 - Copy a value into memory - Set/clears the following flags: [ZN] 4-Byte Instruction
**CMM:** $01 - Copy from one memory address to another. [ZN] 5-Byte Instruction

**ADC:** $02 - Add (with carry) two memory values, replacing the second location with result. [CZVN] 5-Byte Instruction
**SBC:** $03 - Subtract (with borrow) two memory values, replacing the second location with result. The first value gets subtracted from the second. [CZVN] 5-Byte Instruction

**CLC:** $04 - Clear the carry flag. [C] 1-Byte Instruction
**SEC:** $05 - Set the carry flag. [C] 1-Byte Instruction
**CLV:** $06 - Clear overflow flag [V] 1-Byte Instruction

**AND:** $07 - Logical AND two memory locations, replacing the second location with result. [ZN] 5-Byte Instruction
**ORM:** $08 - Logical OR two memory locations, replacing the second location with result. [ZN] 5-Byte Instruction
**XOR:** $09 - Logical XOR two memory locations, replacing the second location with result. [ZN] 5-Byte Instruction

**SHL:** $0A - Shifts all bits of a memory location left one position. 0 is shifted into bit-0 and the original bit-7 is shifted into the Carry. [CZN] 1-Byte Instruction
**SHR:** $0B - Shifts all bits of a memory location right one position. 0 is shifted into bit-7 and the original bit-0 is shifted into the Carry. [CZN] 1-Byte Instruction
**ROL:** $0C - Shifts all bits of a memory location left one position. The Carry is shifted into bit 0 and the original bit 7 is shifted into the Carry. [CZN] 1-Byte Instruction
**ROR:** $0D - Shifts all bits of a memory location right one position. The Carry is shifted into bit 7 and the original bit 0 is shifted into the Carry. [CZN] 1-Byte Instruction

**INC:** $0E - Increment memory by one. [ZN] 3-Byte Instruction
**DEC:** $0F - Decrement memory by one. [ZN] 3-Byte Instruction

**BCC:** $10 - Branch on carry clear. Branch to address if the carry flag is clear. (Sets the PC to a the new memory address). [] 3-Byte Instruction
**BCS:** $11 - Branch on carry set. Branch if the carry flag is set. [] 3-Byte Instruction
**BNE:** $12 - Branch on non-zero. Branch if the zero flag is clear. [] 3-Byte Instruction
**BEQ:** $13 - Branch on zero. Branch if the zero flag is set. [] 3-Byte Instruction
**BPL:** $14 - Branch on positive. Branch if the negative flag is clear. [] 3-Byte Instruction
**BMI:** $15 - Branch on negative. Branch if the negative flag is set. [] 3-Byte Instruction
**BVC:** $16 - Branch on overflow clear. Branch if the overflow flag is clear [] 3-Byte Instruction
**BVS:** $17 - Branch on overflow set. Branch if the overflow flag is set. [] 3-Byte Instruction

**CMM:** $18 - Compare two memory locations. Sets zero flag if values are identical. Sets carry flag if the first memory value is equal to or greater than the second memory value. [CZN] 5-Byte Instruction

**JMP:** $19 - Set PC to new value, altering flow of program.[] 3-Byte Instruction
**JSR:** $1A - Jump sub routine. Pushes the address of the next operation to the stack, then sets the PC to a new memory value. [] 3-Byte Instruction
**RSR:** $1B - Return from subroutine, pop the stack value into the PC and continues evaluating from there. [] 1-Byte Instruction

**PHM:** $1C - Push memory to stack. [] 3-Byte Instruction
**PHS:** $1D - Push status to stack. [] 1-Byte Instruction
**PLM:** $1E - Pop from stack into memory locaton. [ZN] 3-Byte Instruction
**PLS:** $1F - Pop from stack into status register. [CZVN] 1-Byte Instruction

**NOP:** $20-$2F: No Operation - Does nothing. [] 3-Byte Instruction


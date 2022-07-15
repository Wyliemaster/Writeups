# Challenge

[rock_noll_vo](https://crackmes.one/crackme/6113156933c5d45db85dc125) is a linux binary from [crackme.one](https://crackmes.one/) rated `4.0` in difficulty. The challenge is to get the `Yep, you got the key!` prompt after entering a key.

md5: `1473313fd44a55aaa5eb7158d8c3e519`

## The Virtual Machine

The first thing that's clearly noticable is that `main` is a C Array filled with bytes of data.

![main](/rock_noll_vo/images/main.PNG)

Under the array are 3 functions. The first function contains a definition for a VM Struct

![struct](/rock_noll_vo/images/def_vm_struct.PNG)

The next function contained a while loop which continously called 2 functions. The first one read 3 items from the C Array and stored them in seperate atrributes

![data](/rock_noll_vo/images/read_data.PNG)

The other function contains a switch statement which takes in the first piece of data that was read, this made it apparent that the read function was storing virtual opcodes and virtual operands

![switch](/rock_noll_vo/images/switch.PNG)

Upon further inspection of this switch statement, many small functions dealing with these attributes can be found

![woah](/rock_noll_vo/images/procedures.PNG)

Using all of these functions, i was able to get a rough idea of the VM Structure

```c++
// Note: PAD refers to attributes i couldn't find being used 
struct VM_Struct
{
  uint64_t *vm_data;                // bytecode array
  uint64_t vm_data_size;            // how many bytes in the bytecode array
  uint64_t *vm_stack;               // virtual stack
  uint64_t *vm_unknown_pointer;     // unknown pointer found in the struct destructor
  uint64_t vm_registers[7];         // 7 virtual registers
  uint64_t PAD01;
  uint64_t vm_stack_ptr;            // stack pointer
  uint64_t vm_stack_ptr_max;        // Max value the stack pointer could point to
  uint64_t vm_instruction_pointer;  // which virtual instruction is being ran
  uint64_t vm_stack_size;           // the size of the virtual stack
  uint64_t PAD02;
  uint64_t vm_opcode;               // current virtual opcode being ran
  uint64_t vm_op_left;              // left operand
  uint64_t vm_op_right;             // right operand
  uint64_t vm_flag_equal_zero;      // if the 0 flag is set (for branching)
  uint64_t vm_flag_less_zero;       // if the less than 0 flag is set (for branching)
  uint64_t vm_flag_greater_zero;    // if the greater than 0 flag is set (for branching)
};
```

After learning the VM Structure, i started tackling the virtual opcodes so i could see what exactly this program did. 

```s
    ; Note: 'r' represents a list of the virtual registers
    ; 'x' and 'y' represent which register is being accessed
    ; 'z' represents a constant value

    mov r[x], r[y]          ; 0x10
    mov r[x], z             ; 0x11 && 0x12
    swap r[x], r[y]         ; 0x13
    add r[x], r[y]          ; 0x14
    add r[x], z             ; 0x15
    sub r[x], r[y]          ; 0x16
    sub r[x], z             ; 0x17
    div r[x], r[y]          ; 0x18
    div r[x], z             ; 0x19
    mul r[x], r[y]          ; 0x1A
    mul r[x], z             ; 0x1B
    mod r[x], r[y]          ; 0x1C
    mod r[x], z             ; 0x1D
    syscall r0, r1, r2, r3  ; 0x1E
    nop                     ; 0x1F
    jmp z                   ; 0x20 | Jump
    je z                    ; 0x21 | Jump if equal
    jne z                   ; 0x22 | Jump if not equal
    jbz z                   ; 0x23 | Jump if below zero
    jnbz z                  ; 0x24 | Jump if not below zero
    jgz z                   ; 0x25 | Jump if greater than zero
    jngz z                  ; 0x26 | Jump if not greater than zero
    dec r[x]                ; 0x27
    inc r[x]                ; 0x28
    reset_stack?            ; 0x29
    call z                  ; 0x2A
    ret                     ; 0x2B
    push r[x]               ; 0x2C
    push z                  ; 0x2D
    pop r[x]                ; 0x2E | pop value from stack into register
    cmp r[x], r[y]          ; 0x2F && 0x31
    cmp r[x], z             ; 0x30 && 0x32
    and r[x], r[y]          ; 0x33
    and r[x], z             ; 0x34
    or r[x], r[y]           ; 0x35
    or r[x], z              ; 0x36
    xor r[x], r[y]          ; 0x37
    xor r[x], z             ; 0x38
    shr r[x], r[y]          ; 0x39
    shr r[x], z             ; 0x3A
    shl r[x], r[y]          ; 0x3B
    shl r[x], z             ; 0x3C
    unknown r0, r1          ; 0x3D | if r0 == 0 then increase capacity on the heap by the value in r1? if r0 == 1 then free heap? if r0 == 2 then hash r1 with FNV-1 
    exit                    ; 0x3E
```

## Devirtualised code

Once devirtualised, this is the code that will be executed

```s
0x0 jmp 0x10

0x2 mov r0, 0x1
0x5 mov r1, 0x1
0x8 mov r2, &aCorrectKeyText    ; "Yep, you got the key!"
0xB mov r3, 0x16
0xE syscall r0, r1, r2, r3
0xF exit

0x10 mov r0, 0x1
0x13 mov r1, 0x1
0x16 mov r2, &aEnterAKeyText    ; "Enter: "
0x19 mov r3, 0x7
0x1C syscall r0, r1, r2, r3
0x1D call 0x32     
0x1F cmp r0, 0x6658A617BCCE24D5
0x22 je 0x2 

0x24 mov r0, 0x1
0x27 mov r1, 0x1
0x2A mov r2, &aIncorrectKeyText ; "haha ur bad."
0x2D mov r3, 0xD
0x2F syscall r0, r1, r2, r3
0x31 exit

0x32 push r9
0x34 mov r9, r8
0x37 xor r0, r0
0x3A mov r1, 0x80
0x3D unknown(r0, r1) 
0x3E mov r2, r0
0x42 xor r0, r0
0x45 xor r1, r1
0x48 mov r3, 0x7F
0x4A syscall r0, r1, r2, r3
0x4B mov r0, 0x2
0x4E mov r1, r2
0x51 unknown r0, r1
0x52 push r0
0x54 mov r0, r1
0x57 mov r1, r2
0x5A xor r2, r2
0x5D unknown r0, r1
0x5E pop r0
0x60 pop r9
0x62 ret
```

Now that we have the code for this program, we can start with finding the key. if we start from the `Enter: ` string, we can follow the assembly to find how the password validation works. on line `0x1D` we can see that a subroutine is called. Imediately, we can see that the unknown handler is used multiple times. The `xor r0, r0` on line `0x37` shows us that the first call is allocating memory and this is followed up by a syscall. by running strace, we can quickly identify that this is a `sys_read`. The size of the data written is `0x7F` bytes which gives us a clue about the max capacity of the key. 

![sys_read](/rock_noll_vo/images/sys_read.PNG)

Immediately after the syscall, the result is moved into the `r1` register and the unknown handler is called again with a type of `0x2.` Looking at that case, an algorithm is being applied of the contents of r1.

![FNV](/rock_noll_vo/images/FNV1.PNG)

By googling the constants within the algorithm, i was able to identify that it was the `Fowler-no-vo hash function.` Looking at the documentation of this hash function, i learned that this was the FNV-1 variation of the algorithm. Now that we know that our input is hashed, we are a step closer to being able to find the key. If we continue through the subroutine, we can see the hash returned from this function is pushed onto the stack. if we continue through the subroutine, we can see the hash is popped back into `r0` on line `0x5E.` Afterwards, a ret opcode is called bringing us back to line `0x1D` where we were before the jump.

On the next line, we can see that the contents of `r0` are compared with a constant value `0x6658A617BCCE24D5` if they equal flag is set then the program jumps to line `0x2.` this section of the code contains the string `Yep, you got the key!` meaning the constant value from before must be the hash. Since we know the algorithm and the hash, we can find they key via brute forcing.

## Brute Forcing

Below is the code i used to brute force the key

```c++
// String containing all characters on a standard qwerty keyboard. it is unlikely that it will use characters that noone can use
const char* AVAIL_CHARS = "\xA abcdefghijklmnopqrstuvwxyz!\"#$ % &\'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\\]^_`{|}~";
const int AVAIL_CHAR_LEN = 98;

// FNV-1 algorithm
uint64_t fnv64(const char* data)
{
    uint64_t result = 0xCBF29CE484222325LL;
    for (size_t i = 0; data[i] && data[i] != 0xA; ++i)
        result = (uint8_t)data[i] ^ (0x100000001B3LL * result);
    return result;
}

std::string brute_force(const __int64 HASH)
{

    // This is the max length the sys_read allowed
    uint8_t char_idx[0x7F];
    memset(char_idx, 0, 0x7F);

    while (true)
    {
        // Adding the available characters to the string
        std::string gen;
        for (uint8_t& idx : char_idx)
        {
            gen += AVAIL_CHARS[idx];
        }

        // Applying the FNV-1 Hash onto generated String 
        auto fnv64_hash = fnv64(gen.c_str());

        // if they match, return the string
        if (fnv64_hash == HASH) return gen;

        // otherwise try next char
        char_idx[0]++;

        for (size_t i = 0; i < 0x7F; i++)
        {
            auto ch = char_idx[i];

            if (i + 1 >= 0x7F) break;

            // if every combination for the length doesn't match then add new char
            if (AVAIL_CHAR_LEN < ch)
            {
                char_idx[i] = 0;
                char_idx[i + 1]++;
            }
        }

    }
}
```
After about 4 hours of bruteforcing, The key was finally matched

![result](/rock_noll_vo/images/result.PNG)

and with that, the challenge is complete!

![winner](/rock_noll_vo/images/key.PNG)


+++
date = '2026-06-21T12:45:49+02:00'
draft = false
title = 'Improving the Linker'
+++
## The problem
As I said, changing from `.data` to `.rodata` got rid of the warning about one segment having all permissions, but that's like a patch, not a fix. To fix it, I need to define more sections in my linker so that they end up in different segments. It's as easy as defining them the same way I defined the `.text` section.
## The solution
```ld
OUTPUT_ARCH(riscv)
ENTRY(_start)
SECTIONS
{
    . = 0x80200000;
    .text : {
        *(.text)
    }
    .rodata : {
        *(.rodata)
    }
    .data : {
        *(.data)
    }
    .bss : {
        *(.bss)
    }
}
```

It is also important to say that while I didn't explain properly, when I meant my code was being put in the correct address by luck, I didn't mean it in that way. What I meant was that the start of my program was being put in the correct address because I had defined before in the linker where it would go, but the fact that it worked without using `.global` felt like luck because had I misplaced the code on memory it wouldn't have ran.

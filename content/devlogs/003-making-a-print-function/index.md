+++
date = '2026-06-20T16:27:00+02:00'
draft = false
title = 'Writing a printing function in assembly'
+++
## Printing in assembly sucks
One of the many things that help when developing any kind of software is being able to print. Printing can make it easier to follow the values of variables, error messages or even just be like "we're in step 2 now!" so that you don't lose track of where you are during testing. However, as it was shown in my previous devlog, printing in assembly is an absolute nightmare. For each and every time I want to show a message on the screen, I have to print each character individually, which means loading the "character" register and the "function" register individually each time and then make the call so that OpenSBI knows what I want to do. Because of that, I am gonna make a function so that I can pass strings and the function takes care of printing for me.
## What do I need to know
Knowing that I can load a byte into `a0` and the `write()` function into `a7` and then do the `ecall` makes me think I can create a loop to print one character at a time, but that means I have to be able to give it different bytes each time. If a string is just a group of characters that get put in RAM in order, that means I can jump byte by byte and each time put the correct one in `a0` before it gets printed. Since I know I can have a `.data` section that holds my string because handling user input is yet far far away. I'm gonna need the `_start` label that will use the `print` label that will hold the loop, which I know already because of trying in x86 that is just a conditional jump. That means my string must contain a null terminator at the end, which allows me to make the condition be something like "if the current byte is 0, stop", which would be like "jump to this other label" and that other label would just use `wfi` to hang the CPU.
## Failed attempts
### First try
I wrote:

```assembly
.section .text
_start:
	la t0, string	// load the address of string in t0
	call print	// call the print function, which stores the address to come back in ra (return address)
	wfi	// hang the CPU
print:
	beqz t0, ret	// if t0 is 0, return
	li a0, t0	// load the address in a0
	li a7, 0x01	// load write() into a7
	ecall		// write
	addi t0, t0, 1	// add 1 byte to the address in t0: t0 = t0 + 1
	j print		// go back to the start


.section .data
string:
	.asciz "This is a string"
```

And I got an error saying `boot.S:8: Error: illegal operands 'li a0,t0'`. That's because `li` is used for hardcoded values, not just values. I thought it was used to load values at first. Also, notice how I'm using `.asciz`, that's because if I use `.ascii` I need to write `\0` at the end, but with this, it's added for me.
### Second try
I changed `li` for `mv`, which is the correct instructions to copy from one register to another, making my code look like this:

```assembly
.section .text
_start:
	la t0, string	// load the address of string in t0
	call print	// call the print function, which stores the address to come back in ra (return address)
	wfi	// hang the CPU
print:
	beqz t0, ret	// if t0 is 0, return
	li a0, t0	// load the address in a0
	li a7, 0x01	// load write() into a7
	ecall		// write
	addi t0, t0, 1	// add 1 byte to the address in t0: t0 = t0 + 1
	j print		// go back to the start


.section .data
string:
	.asciz "This is a string"
```

This time the error said that it couldn't find `_start` and that there was an undefined reference called `ret`. I thought `ret` could be used to return, but for branching you use labels, so I need a label that does the `ret` instead.
### Third try
With this code:

```assembly
.section .text
_start:
	la t0, string	// load the address of string in t0
	call print	// call the print function, which stores the address to come back in ra (return address)
	wfi	// hang the CPU
print:
	beqz t0, done	// if t0 is 0, go to "done"
	mv a0, t0	// load the address in a0
	li a7, 0x01	// load write() into a7
	ecall		// write
	addi t0, t0, 1	// add 1 byte to the address in t0: t0 = t0 + 1
	j print		// go back to the start

done:
	ret // return

.section .data
string:
	.asciz "This is a string"
```

It compiled and ran, but started printing like crazy, and not my string. It showed this in a loop:

![Screenshot of my VM going crazy](pic001-batshit-crazy-print)

That is something I said before: if you don't stop it where it's supposed to stop, it'll keep on executing code. However, it's consistently printing characters in a loop, so that means that what's wrong is in my print function.
### Fourth attempt
What I do with `t0` is load the address of the string using the `string` label. However, there's no such thing as "an address for a string. In reality, the address contains where the string begins, so what I'm loading in this specific case is where the 'T' is sitting in memory. This is something I understood but it's worth it saying it again for what's to come. Since I'm saving the address in a register, I am saving where the CPU needs to look in memory, but not what it needs to see. That means I have the address, not the value. If I use `mv`, I'm copying the address to another register. That means that it never finds the `\0` because it's not there. What I want is to load the byte, specifically in such a way it's not signed, so I need `lbu`, which will use the address that `t0` has to find the **value** and then save it into `a0`. I also have to check the `a0` now, since it contains the value, which is what I want to compare to zero. The code would look like this:

```assembly
.section .text
_start:
	la t0, string	// load the address of string in t0
	call print	// call the print function, which stores the address to come back in ra (return address)
	wfi	// hang the CPU
print:
	beqz t0, done	// if t0 is 0, go to "done"
	lbu a0, t0	// load the byte from t0 into a0
	li a7, 0x01	// load write() into a7
	ecall		// write
	addi t0, t0, 1	// add 1 byte to the address in t0: t0 = t0 + 1
	j print		// go back to the start

done:
	ret // return

.section .data
string:
	.asciz "This is a string"
```
## Successful attempt
I didn't know how `lbu` worked. It needs 3 things: the register that contains the base address, the register that will store the byte and what I was missing, the offset. In this case, I don't want any offset, I want to read exactly 1 byte from the address in `t0`, so I just put 0.

```assembly
.section .text
_start:
	la t0, string	// load the address of string in t0
	call print	// call the print function, which stores the address to come back in ra (return address)
	wfi	// hang the CPU
print:
	beqz t0, done	// if t0 is 0, go to "done"
	lbu a0, 0(t0)	// load the byte at t0 in a0
	li a7, 0x01	// load write() into a7
	ecall		// write
	addi t0, t0, 1	// add 1 byte to the address in t0: t0 = t0 + 1
	j print		// go back to the start

done:
	ret // return
```

This is the result:

![Screenshot of the VM displaying my string using the print function](pic002-print-function-working)
## Last warnings
I have a warning telling me that `_start` cannot be found, and it works because the default address is where my code lives anyway. I can fix this by using the `.global` directive with `_start` and keep the rest of the code the same.

The other warning is that there's a segment with read, write and execute permissions all at once. The reason why this happens is because my `.text` section needs read and execute permissions while my `.data` section needs read and write permissions, and my simple linker script puts both in the same segment (a chunk of binary loaded into memory), meaning I have a segment with all permissions, which is bad in theory. As for now, it doesn't matter that much, because my program does 1 thing only. 
## Links
[Post about jumps and functions](https://projectf.io/posts/riscv-jump-function/)

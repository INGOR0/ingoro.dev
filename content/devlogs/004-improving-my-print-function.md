+++
date = '2026-06-21T00:54:35+02:00'
draft = true
title = 'Improving My Print Function'
+++
## Touching something that works
While it is true that if something works you should not touch it, I read that the best function is the one that behaves like a blackbox. The caller doesn't know or need to know what's going on inside the function and the function doesn't know or need to know what will happen once it goes back. Right now, I have this code:
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

.section .data
string:
	.asciz "This is a string"
```

As you can see, I load the address of the string in `t0` before I call `print`. This isn't bad, but it's not standard. Instead, I should use `a0` and handle moving everything inside the function. Since I will need to then move the address into `t0`, I am gonna use a `print` label for the preparation step and a `print_loop` as the label for the "worker", so like this:

```assembly
.section .text
.global _start
_start:
	la a0, string	// load the address of string in t0
	call print	// call the print function, which stores the address to come back in ra (return address)
	wfi	// hang the CPU

print:
	mv t0, a0
	j print_loop

print_loop:
	lbu a0, 0(t0)	// load the byte at t0 in a0
	beqz a0, done
	li a7, 0x01	// load write() into a7
	ecall		// write
	addi t0, t0, 1	// add 1 byte to the address in t0: t0 = t0 + 1
	j print_loop	// go back to the start

done:
	ret // return

.section .rodata
string:
	.asciz "This is a string"
```

With this version, I get rid of the warning that says the segment has RWX permissions because I'm using `.rodata` instead of `.data`, since I only use that section to put the string I want to print. I also use `.global` so that `_start` is found, because compiling was showing me it was not, and it only worked because it was defaulting to an address and I was lucky that my code went there by default as well.
## Worth noting
Before, I was checking if the address that `t0` held was 0, which makes no sense, because I'm checking if it's 0 so that I know the string is finished. The fact that it worked was luck, but now it's guaranteed to work, so that's good.

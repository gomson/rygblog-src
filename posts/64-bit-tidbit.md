-title=64-bit tidbits
-time=2010-10-11 01:32:25
Two discoveries from recent months. The first one was on x86\-64 and occurs in this code fragment:

```
U32 offset = ...;
U8 *ptr = base + (offset & 0xffff)*4;
```

I expected to get one "and", with the rest folded into an address computation \(after all, x86 has a base\+index\*4 addressing mode\). What I actually got was this:

```
; eax = offset, rcx=base
movzx eax, ax ; equivalent to "and eax, 0ffffh" (but smaller)
shl   eax, 2
add   rax, rcx
```

The operand size mismatch between the instructions is the clue \- the compiler computes the "\*4" in 32 bits; the "\*4" in an address computation would be computed for 64\-bit temporaries. This is a result of the C/C\+\+ typing rules: All of `(offset & 0xffff) * 4` is computed as "unsigned int", since C does most computations at "int" size, which is 32 bits even on most 64\-bit platforms. The compiler can't use the built\-in "\*4" addressing mode since `(U32) x*4` isn't equivalent to `(U64) x*4` in the presence of overflow \(of course, in the example given, we can't ever get any overflows, but that's something that the optimizer apparently wasn't looking for\).

Easy solution: use `U8 *ptr = base + U64(offset & 0xffff)*4` \(alternatively, multiply by "4ull" or "\(size\_t\) 4"\) and everything comes out as expected. But it illustrates an interesting dilemma: In the simpler expression `U8 *ptr = base + offset*4`, would you have expect the "offset\*4" part to be computed in 32 bits? Probably not. "int" was left at 32\-bits since a lot of code relies on it, and most ints we use really don't need more than 32 bits. But the type conversion rules in C/C\+\+ implicitly assume that an "int" is the native register type, leading to this kind of problem when that's not the case. Right now, virtually nobody uses single arrays larger than 4GB, but it's only a matter of time until that changes...

Anyway, on to tidbit number two, which concerns GCC on PS3 \(PPU code\). The PPU is a 64\-bit PowerPC processor, but the PS3 has only 256MB of RAM \(plus 256MB of graphics memory\), so there's no reason to use 64\-bit pointers \- it's just a waste of memory. And again, overflow makes everything complicated. GCC assumes that \(almost\) all address computations could overflow at any time. It thus tends to perform address calculations manually and clear the top 32 bits afterwards, instead of using the built\-in addressing modes of the CPU. This produces code that's significantly larger \(and slower\) than necessary. To give an example, take this piece of code:

```
U32 *p = ...;
*++p = x;
*++p = y;
```

A good translation of this code into PPC assembly looks like this: \(with explanation in case you don't know PPC assembly\)

```
; r8=p, r9=x, r10=y
stw  r9, 4(r8)   ; *(p + 1) = x;
stwu r10, 8(r8)  ; *(p + 2) = y; p += 2;
```

but what GCC actually produces for this code sequence looks more like this:

```
addi   r11, r8, 4   ; t = p + 1;
addi   r8, r8, 8    ; p += 2;
clrldi r11, r11, 32 ; clear top 32 bits of t
clrldi r8, r8, 32   ; same for p
stw    r9, 0(r11)   ; *t = x;
stw    r10, 0(r8)   ; *p = y;
```

Note that it's not only 3x the number of instructions, but also needs extra registers \(thus increasing register pressure\). It's the same underlying problem as with the x86 example \- a lot of extra work to enforce artificial wraparound of 32\-bit quantities. A pretty dubious goal, since it's very unlikely that code actually wants that 32\-bit wraparound \(and if it does, it should probably be fixed\), but I digress... Anyway, it turns out that there's one case where GCC assumes that pointers won't wrap around: Structure accesses. So in this case:

```
struct Blah {
  U32 x, y;
};

Blah *p = ...;
p->x = x;
p->y = y;
```

You'll actually get reasonable code without an addi/clrldi combo for every access. So if you're writing out a stream of words \(something that's fairly common when building a rendering command buffer\), you can save a lot of instructions by declaring a simple struct:

```
struct CommandData {
  U32 w0, w1, w2, w3, w4, w5, w6, w7;
};
```

and then replacing this kind of code:

```
U32 *cmd = ...;
cmd[0] = foo;
cmd[1] = bar;
cmd[2] = blah;
cmd[3] = blub;
cmd += 4;
```

with the \(admittedly somewhat obscure\-looking\)

```
CommandData *cmd_out = (CommandData *) cmd;
cmd_out->w0 = foo;
cmd_out->w1 = bar;
cmd_out->w2 = blah;
cmd_out->w3 = blub;
cmd = &cmd_out->w4;
```

This is only useful to know if you're programming on PS3 and using GCC, but if you do, that kind of stuff can make a big difference. I've recently been working a lot on PS3 rendering code and got speedups in excess of 10% from this and other low\-level optimizations along the same vein. Actually reading the assembly generated by your compiler is *important*, especially when targeting in\-order CPUs. It's nearly impossible to see this kind of problem when you're thinking at the level of C/C\+\+ source code.
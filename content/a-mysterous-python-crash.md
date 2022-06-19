+++
title = "Debugging a mysterious Python crash"
date = 2022-05-28
+++

I recently wanted to prepare a Jupyter notebook with some example code and ran into an interesting problem: trying to display a Matplotlib chart made the IPython kernel crash.

## Gathering more info

Luckily, the crash was easy to reproduce outside of Jupyter:

```bash
$ python3 -c 'import matplotlib.pyplot as plt; plt.axes()'
Illegal instruction
```

This usually means one of two things:

 - some compilers / standard libraries produce an `ud2` instruction in order to abort the program execution, e.g. `core::intrinsics::abort()` in Rust.
 - the program tried to execute an instruction that is not available on the CPU it was running on.

I've encountered the former one much more often than the latter, but many popular Python libraries code call into optimized C.
To be honest, I was pretty sure this was caused by my mess of old, distro-supplied packages (from CentOS 7), in combination with some installed using `pip`.
Of course, it's better to check than to guess:

```bash
$ gdb --args python3 -c 'import matplotlib.pyplot as plt; plt.axes()'
[snip]
(gdb) r
Starting program: /usr/bin/python3 -c import\ matplotlib.pyplot\ as\ plt\;\ plt.axes\(\)
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/libthread_db.so.1".
[snip]

Thread 1 "python3" received signal SIGILL, Illegal instruction.
0x00007ffff556545c in dgemm_kernel_PILEDRIVER () from /usr/local/lib64/python3.6/site-packages/numpy/core/../../numpy.libs/libopenblasp-r0-8a0c371f.3.13.so
```

_Heads-up: for reasons I'll show below, I'm reproducing the issue on my own computer. I'm trying to doctor the outputs, but there might be some inconsistencies._

It appears to be crashing in a `dgemm` kernel in OpenBLAS, docs courtesy of LAPACK:

```
DGEMM  performs one of the matrix-matrix operations

C := alpha*op( A )*op( B ) + beta*C,

where  op( X ) is one of

op( X ) = X   or   op( X ) = X**T,

alpha and beta are scalars, and A, B and C are matrices, with op( A )
an m by k matrix,  op( B )  a  k by n matrix and  C an m by n matrix.
```

That's basically a fancy matrix multiplication.
More importantly, notice the Piledriver reference in the function name.
Piledriver is an AMD microarchitecture from around 2012&ndash;2014, which is hopefully not what I'm running on.

On the other hand, function names aren't always accurate, especially in highly-optimized code.
We can ask GDB to disassemble the current function in order to check the place where the code crashed and it will output a 4890-line monstruosity:

```asm
(gdb) disassemble
Dump of assembler code for function dgemm_kernel_PILEDRIVER:
   0x00007ffff555fe00 <+0>:    sub    $0x60,%rsp
   0x00007ffff555fe04 <+4>:    mov    %rbx,(%rsp)
   0x00007ffff555fe08 <+8>:    mov    %rbp,0x8(%rsp)
   [snip]
   0x00007ffff5565450 <+22096>:    vmovddup -0x20(%rsi,%rbp,8),%xmm1
   0x00007ffff5565456 <+22102>:    vmovups -0x80(%rdi,%rax,8),%xmm0
=> 0x00007ffff556545c <+22108>:    vfmaddpd %xmm4,%xmm1,%xmm0,%xmm4
   0x00007ffff5565462 <+22114>:    vmovddup -0x18(%rsi,%rbp,8),%xmm2
   0x00007ffff5565468 <+22120>:    vfmaddpd %xmm5,%xmm2,%xmm0,%xmm5
```

The crashing opcode was `vfmaddpd`.
I've never encountered it before &mdash; not exactly surprising &mdash; but notice how it has four operands, which is pretty rare in `x86` instructions.
That said, it's pretty easy to guess what it does if you can unpack its mnemonic:

 - `v`: vector (SIMD) instruction, meaning it operates not on scalars, but on vector registers with multiple values
 - `fma`: fused multiply-add, that is, a multiplication and addition in one step, for better performance, precision, or both
 - `p`: packed, meaning it uses all the values in the registers (as opposed to the scalar ones, which only touch a single value in a vector register)
 - `d`: double-precision, i.e. 64-bit floating-point numbers

Peeking at [the docs](https://www.amd.com/system/files/TechDocs/43479.pdf), the instruction name is `Multiply and Add Packed Double-Precision Floating-Point`:

```asm
VFMADDPD dest, src1, src2, src3 # dest = (src1 * src2) + src3
```

It doesn't matter for us, but `xmm` are 128-bit registers, so it's working on pairs `double`s.

This is an `FMA4` (four-operand FMA) instruction, which has a bit of a weird history.
It was introduced by AMD, but Intel never implemented it.
Instead, Intel added their own `FMA3` (three-operand) instructions, which look like `vfmadd213pd xmm0, xmm1, xmm2` and don't have a separate destination.
If you're wondering, `213` specifies the operand order, the one here doing `xmm0 = xmm1 * xmm0 + xmm2`.

In any case, AMD dropped FMA4 in 2017 with the Zen microarchitecture, which probably caused some confusion because the instructions still work, but give the wrong results sometimes.

By this point, we have a theory: our CPU does not support FMA4, but OpenBLAS (used by `numpy`, used by `matplotlib`) thinks it does, picks up that code path and happily crashes.

## Opteron

I don't have the `/proc/cpuinfo` output any more (see below), but the model name was `AMD Opteron 63xx class`, or something similar.
This is a virtual machine, so I'm assuming the hypervisor was reporting an older CPU model.

Software can detect CPU features in a fine-grained way (using the `cpuid` instruction), but OpenBLAS only checks for the CPU model [here](https://github.com/xianyi/OpenBLAS/blob/d33fc32cf30cf1262030c93fa44c72ca8ab27681/cpuid_x86.c#L1288-L1327) and then again [here](https://github.com/xianyi/OpenBLAS/blob/d33fc32cf30cf1262030c93fa44c72ca8ab27681/driver/others/dynamic.c#L353-L385).
There actually was an [Opteron 6300](https://en.wikipedia.org/wiki/List_of_AMD_Opteron_processors#3300-,_4300-_&_6300-series_Opterons) series, which matches the reported model name and did support `FMA4`.

I also tried [some code](https://gist.github.com/rindeal/81198b1cf8f55c356743#file-cpuid-dump2-c) that checks specifically for `FMA4`.
I can't run it again, but indeed, it reported no `FMA4` support.

## The fix and a twist

Knowing the problem, the fix was pretty simple.
Fortunately, OpenBLAS supports overriding the CPU-specific code paths through an environment variable, and setting `OPENBLAS_CORETYPE=ZEN` made it work.
I still filed [an issue](https://github.com/xianyi/OpenBLAS/issues/3638) against the OpenBLAS repo,
in case anyone else runs into this.

Unfortunately, one day later, I can't run my notebook because the `/` partition is mounted read-only, the file system might be corrupted, and pretty much everything is broken. But for once, it's not my fault 😅.

```
FileNotFoundError: [Errno 2] No usable temporary directory found in ['/tmp', '/var/tmp', '/usr/tmp', '/home/user']
```

Yes, I could mount `tmpfs` into `/tmp`, but it doesn't matter.

And the Opteron magically turned into an EPYC:

```
processor       : 0
vendor_id       : AuthenticAMD
cpu family      : 23
model           : 49
model name      : AMD EPYC 7502P 32-Core Processor
stepping        : 0
microcode       : 0x1000065
cpu MHz         : 2500.001
cache size      : 512 KB
physical id     : 0
siblings        : 1
core id         : 0
cpu cores       : 1
apicid          : 0
initial apicid  : 0
fpu             : yes
fpu_exception   : yes
cpuid level     : 16
wp              : yes
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 syscall nx mmxext fxsr_opt pdpe1gb rdtscp lm art rep_good nopl extd_apicid eagerfpu pni pclmulqdq ssse3 fma cx16 sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm cmp_legacy svm cr8_legacy abm sse4a misalignsse 3dnowprefetch osvw topoext perfctr_core retpoline_amd ssbd ibrs ibpb vmmcall fsgsbase tsc_adjust bmi1 avx2 smep bmi2 rdseed adx smap clflushopt clwb sha_ni xsaveopt xsavec xgetbv1 virt_ssbd arat npt nrip_save umip spec_ctrl
bogomips        : 5000.00
TLB size        : 1024 4K pages
clflush size    : 64
cache_alignment : 64
address sizes   : 40 bits physical, 48 bits virtual
power management:
```

## Conclusion

Just a couple of things:

 - feature detection is better than version checking not only when writing JavaScript, but even when targeting low-level code
 - it's a good idea to provide escape hatches from optimized code paths
 - the CPU is a lie

---

And by the way, if you enjoyed this short post, please consider [buying me a coffee](https://www.buymeacoffee.com/lnicolaq).

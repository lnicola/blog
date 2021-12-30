+++
title = "Rust and the Case of the Redundant Comparison"
date = 2018-08-04
+++
A couple of days ago I landed my [second pull request](https://github.com/rust-lang/rust/pull/52908) in the [Rust Programming Language](https://www.rust-lang.org/) [repository](https://github.com/rust-lang/rust/). This is the story of how that went.

This post is inspired by [other](https://llogiq.github.io/2018/08/04/improve.html) [posts](https://blog.mozilla.org/nnethercote/) about improving the Rust compiler.

**TL;DR:** I made a 20 lines PR to the Rust standard library. If you're so inclined, you should try doing the same.

## The chase after unsafe code

There's been quite a bit of [noise](https://www.reddit.com/r/rust/comments/8s7gei/unsafe_rust_in_actixweb_other_libraries/) recently about the amount of unsafe code in the [`actix-web`](https://actix.rs/) framework. I won't discuss the merits of grabbing the pitchforks as soon as someone writes code of a buggy and unidiomatic nature [^pitchforks]. But my first encounter with `unsafe` in Rust was a hard to reproduce, Windows-only, crashing bug in a crate. On that code path there was a single, seemingly innocuous, `unsafe` line. It took me and the crate's author maybe half an hour to find it, with the `unsafe` arrow pointing at it the whole time [^unsafe].

To be fair, `actix-web` now lost a large chunk of its unsafe code, although it's still not quite my cup of tea. So I downloaded the source of [`hyper`](https://github.com/hyperium/hyper), another popular, if much lower-level, HTTP library. Unfortunately (‽), it had relatively few `unsafe` blocks, many of them in test code. But one idiom caught my eye:

```rust
#[derive(Clone)]
pub struct Cursor<T> {
    bytes: T,
    pos: usize,
}

impl Cursor<Vec<u8>> {
    fn reset(&mut self) {
        self.pos = 0;
        unsafe {
            self.bytes.set_len(0);
        }
    }
}
```

The code is trying to clear a `Vec` by forcefully setting its length to `0`. That's a bad idea if the elements of the vector implement `Drop`, because they won't be destroyed. Fortunately, `hyper` only did that for `u8` values, but the whole thing seemed unnecessary.

I almost sent a PR to nuke them, but the crate author said that `set_len(0)` might be faster than `clear()`. That couldn't be true, since `clear` has nothing to do besides setting the length, so I tried to prove it:

```rust
#[no_mangle]
pub extern fn test_set_len(x: &mut Vec<u8>) {
    unsafe { x.set_len(0); }
}

#[no_mangle]
pub extern fn test_clear(x: &mut Vec<u8>) {
    x.clear();
}
```

But I was surprised to see this in the generated code:

```asm
test_set_len:
    movq    $0, 16(%rdi)
    retq

test_clear:
    cmpq    $0, 16(%rdi)
    je      .LBB4_2
    movq    $0, 16(%rdi)

.LBB4_2:
    retq
```

If you're not used to reading assembly, the `set_len` code sets the vector's length to zero (`movq`) and returns. `clear`, however, compares the current length with `0`. If it's empty, the code jumps to the end of the function (`je` stands for "jump if equal"). Otherwise, the length gets set to `0` and the function returns. That's a useless comparison, the same as:

```rust
if len != 0 {
    len = 0;
}
```

LLVM is pretty good, so it's a bit surprising to see it generate this [^peepholes]. In its favour, the debug version is much larger, so the optimizer is still doing a great job.

Do the extra two instructions matter in practice? I didn't benchmark, but my intuition says no [^intuition], and the `set_len(0)` calls were mostly in test code. However, the crate's author seemed unconvinced and I didn't want to be the one who slowed down `hyper`.

## Down the rabbit hole

Disappointed by this turn of events, I searched for the implementation of `clear`:

```rust
#[inline]
pub fn clear(&mut self) {
    self.truncate(0)
}

pub fn truncate(&mut self, len: usize) {
    unsafe {
        // drop any extra elements
        while len < self.len {
            // decrement len before the drop_in_place(), so a panic on Drop
            // doesn't re-drop the just-failed value.
            self.len -= 1;
            let len = self.len;
            ptr::drop_in_place(self.get_unchecked_mut(len));
        }
    }
}
```

The code is a bit convoluted, because of the reason described in the comment. `drop` can panic [^panic], so the function must decrement the length before dropping an element. In case of a panic, the last element of the `Vec` will be a valid one.

A quick test on the [Playground](https://play.rust-lang.org/) shows that if we set the length at the end, the compiler generates the code that we expect:

```rust
#[inline(never)]
fn truncate_wrong_dont_use<T>(x: &mut Vec<T>, len: usize) {
    unsafe {
        let mut last = x.len();
        while len < last {
            last -= 1;
            ptr::drop_in_place(x.get_unchecked_mut(last));
        }
        x.set_len(len);
    }
}
```

```asm
playground::truncate_wrong_dont_use:
    movq    $0, 16(%rdi)
    retq
```

But that won't do. At this point I filed an issue in the compiler repository, then dropped [sic!] it, waiting for someone else to come up with a fix. Unfortunately, nobody did, so I gave it a little more thought.

If we could run some code at the end, even on panics, we might be able to set the length from there. And we can, with a helper struct which applies the change when dropped. It sounds harder for the compiler, but there's only one way to check. And if we scroll around the file, we find this:

```rust
// Set the length of the vec when the `SetLenOnDrop` value goes out of scope.
//
// The idea is: The length field in SetLenOnDrop is a local variable
// that the optimizer will see does not alias with any stores through the Vec's data
// pointer. This is a workaround for alias analysis issue #32155
struct SetLenOnDrop<'a> {
    len: &'a mut usize,
    local_len: usize,
}

impl<'a> SetLenOnDrop<'a> {
    #[inline]
    fn new(len: &'a mut usize) -> Self {
        SetLenOnDrop { local_len: *len, len: len }
    }

    #[inline]
    fn increment_len(&mut self, increment: usize) {
        self.local_len += increment;
    }
}

impl<'a> Drop for SetLenOnDrop<'a> {
    #[inline]
    fn drop(&mut self) {
        *self.len = self.local_len;
    }
}
```

The comment indicates that LLVM gets confused into thinking that modifying the vector's data could change its length. And that sounds exactly like our issue.

## Compiling a compiler

I cloned the compiler repository and skimmed the [contributing guide](https://github.com/rust-lang/rust/blob/master/CONTRIBUTING.md). I copied the sample configuration file and changed a couple of lines:

```diff
diff --git 1/config.toml.example 2/config.toml
index 9907341633..477c558545 100644
--- 1/config.toml.example
+++ 2/config.toml
@@ -104,11 +104,11 @@

 # Instead of downloading the src/stage0.txt version of Cargo specified, use
 # this Cargo binary instead to build all Rust code
-#cargo = "/path/to/bin/cargo"
+cargo = "/usr/bin/cargo"

 # Instead of downloading the src/stage0.txt version of the compiler
 # specified, use this rustc binary instead as the stage0 snapshot compiler.
-#rustc = "/path/to/bin/rustc"
+rustc = "/usr/bin/rustc"

 # Flag to specify whether any documentation is built. If false, rustdoc and
 # friends will still be compiled but they will not be used to generate any
@@ -250,6 +250,7 @@
 # means "the number of cores on this machine", and 1+ is passed through to the
 # compiler.
 #codegen-units = 1
+codegen-units = 0

 # Whether or not debug assertions are enabled for the compiler and standard
 # library. Also enables compilation of debug! and trace! logging macros.
```

I didn't want to download another compiler, so I filled in the paths to mine. The `codegen-units` setting makes the compiler produce code from multiple threads at once. The trade-off involved is that the generated code is slower. Initially, I went for an unoptimized build (which the comments advise against), but ended up cancelling it.

The way compilers are usually built, there are a couple of _stages_. Stage 0 is an existing compiler, stage 1 is the compiler we are working on &mdash; built by the stage 0 one, and stage 2 is our version built by itself. If we make the compiler generate faster &mdash; or more buggy &mdash; code, stage 1 won't be affected, but stage 2 will. I wasn't planning to change anything in the compiler, so stage 1 was enough for me.

Apparently, there is a small complication: some stage 0 artefacts like `libstd` will still be rebuilt, triggering a cascade of rebuilds that I didn't want. After asking around on IRC, I found the command line I wanted:

```bash
./x.py --keep-stage 0 --stage 1
```

This is all described in the [bootstrapping documentation](https://github.com/rust-lang/rust/blob/master/src/bootstrap/README.md), which I completely missed. I also tried incremental builds (via `config.toml`, not the command line &mdash; not sure if it matters), which didn't work out too well for me: an incremental build after touching a single file was just a tad faster than a non-incremental one including LLVM. But there may have been something wrong with my build files, so it's still worth trying.

## Banishing a read

There was no need to do an initial build, but I wanted to check that everything worked. That took about 40 minutes on my laptop. I then added the missing method to the `SetLenOnDrop` struct:

```rust
#[inline]
fn decrement_len(&mut self, decrement: usize) {
    self.local_len -= decrement;
}
```

`truncate` took a couple of tries, but I ended up with the following:

```rust
pub fn truncate(&mut self, len: usize) {
    let current_len = self.len;
    unsafe {
        let mut ptr = self.as_mut_ptr().offset(self.len as isize);
        // Set the final length at the end, keeping in mind that
        // dropping an element might panic. Works around a missed
        // optimization, as seen in the following issue:
        // https://github.com/rust-lang/rust/issues/51802
        let mut local_len = SetLenOnDrop::new(&mut self.len);

        // drop any extra elements
        for _ in len..current_len {
            local_len.decrement_len(1);
            ptr = ptr.offset(-1);
            ptr::drop_in_place(ptr);
        }
    }
}
```

The code walks a pointer backwards, destroying each element. Meanwhile, the helper struct is also keeping track of the remaining length. The code does seem more complex (it's counting twice), but does it work? Let's check some assembly.

```rust
#[no_mangle]
pub extern fn foo(x: &mut Vec<u8>) {
    x.clear();
}

#[no_mangle]
pub extern fn bar(x: &mut Vec<u8>) {
    x.truncate(5);
}

#[no_mangle]
pub extern fn baz(x: &mut Vec<u8>, n: usize) {
    x.truncate(n);
}
```

The unpatched compiler version shows what we've already seen:

```asm
00000000000460a0 <foo>:
   460a0:       48 83 7f 10 00          cmpq   $0x0,0x10(%rdi)
   460a5:       74 08                   je     460af <foo+0xf>
   460a7:       48 c7 47 10 00 00 00    movq   $0x0,0x10(%rdi)
   460ae:       00
   460af:       c3                      retq

00000000000460b0 <bar>:
   460b0:       48 83 7f 10 06          cmpq   $0x6,0x10(%rdi)
   460b5:       72 08                   jb     460bf <bar+0xf>
   460b7:       48 c7 47 10 05 00 00    movq   $0x5,0x10(%rdi)
   460be:       00
   460bf:       c3                      retq

00000000000460c0 <baz>:
   460c0:       48 39 77 10             cmp    %rsi,0x10(%rdi)
   460c4:       76 04                   jbe    460ca <baz+0xa>
   460c6:       48 89 77 10             mov    %rsi,0x10(%rdi)
   460ca:       c3                      retq
   460cb:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)
```

How about the patched one?

```asm
0000000000084d10 <foo>:
   84d10:       48 c7 47 10 00 00 00    movq   $0x0,0x10(%rdi)
   84d17:       00
   84d18:       c3                      retq
   84d19:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)

0000000000084d20 <bar>:
   84d20:       48 8b 47 10             mov    0x10(%rdi),%rax
   84d24:       48 83 f8 05             cmp    $0x5,%rax
   84d28:       b9 05 00 00 00          mov    $0x5,%ecx
   84d2d:       48 0f 42 c8             cmovb  %rax,%rcx
   84d31:       48 89 4f 10             mov    %rcx,0x10(%rdi)
   84d35:       c3                      retq
   84d36:       66 2e 0f 1f 84 00 00    nopw   %cs:0x0(%rax,%rax,1)
   84d3d:       00 00 00

0000000000084d40 <baz>:
   84d40:       48 8b 47 10             mov    0x10(%rdi),%rax
   84d44:       48 39 f0                cmp    %rsi,%rax
   84d47:       48 0f 47 c6             cmova  %rsi,%rax
   84d4b:       48 89 47 10             mov    %rax,0x10(%rdi)
   84d4f:       c3                      retq
```

The `clear()` call works out nicely, it's what we wanted to see (`nopl` is a "no-operation" instruction, used to align the functions) [^zero]. `truncate(n)` also seems fine: it takes the minimum of the current and new length, then writes it back. The `truncate(5)` variant does the same thing, but loads `5` into the `ecx` register in an awkward way: `mov %rax, %rcx` might have been better. Alas, it came out pretty well.

I also tested with a `Drop` type (`String`). I didn't try to make sense of the assembly output, but the new code was similar. You can see it [on GitHub](https://github.com/rust-lang/rust/pull/52908#issue-205157661), if you're curious.

## Wrapping it up

At this point I committed the code and opened a pull request on GitHub. I got assigned a reviewer who reviewed my code in less than two hours. He asked me to add a codegen test to make sure the compiler doesn't revert to the worse code sequence in the future. I'd never written one, but after looking at the existing ones, I put this up:

```rust
// compile-flags: -O

#![crate_type = "lib"]

// CHECK-LABEL: @vec_clear
#[no_mangle]
pub fn vec_clear(x: &mut Vec<u32>) {
    // CHECK-NOT: load
    // CHECK-NOT: icmp
    x.clear()
}
```

The Rust compiler produces what's called LLVM IR as output. It's similar to, but more portable and higher-level than assembly language. The IR gets passed to LLVM, which optimizes it and outputs a binary for the platform you're targeting. `load` and `icmp` are LLVM's terms for `movq` and `cmpq`. The test checks that the generated code doesn't contain the two instructions.

To figure that out, I used the Playground to see my function's IR:

```
; Function Attrs: norecurse nounwind uwtable
define void @test_clear(%"alloc::vec::Vec<u8>"* noalias nocapture dereferenceable(24) %x) unnamed_addr #2 {
start:
  %0 = getelementptr inbounds %"alloc::vec::Vec<u8>", %"alloc::vec::Vec<u8>"* %x, i64 0, i32 3
  %1 = load i64, i64* %0, align 8, !alias.scope !5
  %2 = icmp eq i64 %1, 0
  br i1 %2, label %"_ZN33_$LT$alloc..vec..Vec$LT$T$GT$$GT$5clear17hf92f022d73112116E.exit", label %bb3.preheader.i.i

bb3.preheader.i.i:                                ; preds = %start
  store i64 0, i64* %0, align 8, !alias.scope !5
  br label %"_ZN33_$LT$alloc..vec..Vec$LT$T$GT$$GT$5clear17hf92f022d73112116E.exit"

"_ZN33_$LT$alloc..vec..Vec$LT$T$GT$$GT$5clear17hf92f022d73112116E.exit": ; preds = %start, %bb3.preheader.i.i
  ret void
}
```

That's a mouthful, but you can find the length field address computation (`getelementptr`, affectionately called `gep`), the load and comparison, the branch and the store. `%0` &hellip; `%5` are similar to variables, but are [only assigned once](https://en.wikipedia.org/wiki/Static_single_assignment_form).

I then tried to run my test:

```bash
./x.py test src/test/codegen --keep-stage 0 --stage 1
```

But that [didn't seem to work](https://github.com/rust-lang/rust/issues/52337), possibly due to me using a local compiler for stage 0 (remember those `config.toml` settings?). So I crossed my fingers and pushed the code, waited for the relatively quick (half an hour) Travis tests to pass, pinged my reviewer and then it was out of my hands. The Rust team uses [bors](https://bors.tech/) to merge one pull request at a time. This is done to avoid cases when two PRs work independently, but not together. The downside is that the whole process tends to be rather slow. Someone from the Rust team takes the pull requests that seem "harmless", merges them into a single one, then tries to land that. In my case this failed a couple of times, then finally went through.

## Aftermath

My change missed the 1.29 deadline by a day, but should be included in 1.30. After it was merged, I was finally able to send a [pull request](https://github.com/hyperium/hyper/pull/1619) to `hyper`, removing four measly (but a quarter of all!) `unsafe` blocks.

My changes didn't make `hyper` safer [^pitchforks-2], nor do I expect them to have a measurable impact on the compiled code. But in the end, every little bit helps, while you might also call it a learning experience. And if I managed, then so could you &mdash; so consider giving it a shot. And if compilers aren't your thing, there's plenty of projects that could use a hand.

[^pitchforks]: As I've been known to do myself at times.

[^unsafe]: Some detractors of Rust argue that, since the standard library and many crates use unsafe code, the whole language is unsafe. `unsafe` is unavoidable, but it's a declaration of "Here be dragons", while you can trust the compiler to have your back for the rest of the code. Compare that to C.

[^peepholes]: Compilers have "peephole optimization" phases where they clean up sequences of instructions like these. LLVM might be missing one for this specific case. It might be an interesting exercise to analyze binaries generated by LLVM to check if this pattern occurs more often.

[^intuition]: And we've already seen how that goes.

[^panic]: Rust doesn't have exceptions, except panics are totally like exceptions, but more exceptional. You're not supposed to panic without a good reason.

[^zero]: The extra `00` byte between `movq` and `retq` must be either a no-op, or part of `movq`.

[^pitchforks-2]: Except maybe from the pitchfork mobs.

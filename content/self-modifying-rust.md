+++
title = "On self-modifying executables in Rust"
date = 2022-01-23
+++
This is a short post prompted by a question on [/r/rust](https://reddit.com/r/rust/) from May 2019, asking about a way for a Rust program to modify itself.

Think of how much fun it would be to store a high score list directly inside the a executable.
When you make a copy (on floppy disk, to make things more realistic) and pass it to your friends, your high scores are persisted.

## Errata

 - the variable should probably be marked as `#[repr(C)]`.
 - as [pointed out](https://www.reddit.com/r/rust/comments/saxzs8/on_selfmodifying_executables_in_rust/htwxint/) by a reader, `unsafe { ptr::read_volatile(&RUN_COUNT) }` is a better alternative to `static mut`.
 - you probably shouldn't do this except as a party trick.

## Diving right in

As a proof of concept, we'll write a program that counts how many times it has been run.
This is hardly the only approach, but probably the simplest way is to store the counter in a separate section of the executable.

```rust
#[link_section = "count"]
#[used]
static mut RUN_COUNT: u32 = 0;
```

`#[link_section]` tells the linker to put the variable in a section called `count`.
[`#[used]`](https://rust-lang.github.io/rfcs/2386-used.html) means that the variable must be kept by the linker, even if could be optimized away.
Similarly, `mut` is required because otherwise the compiler will propagate the variable value to the code that reads it.

I'm not sure if the combination above the best way to accomplish this, or if it's guaranteed to work.
It was the only working method I found back then, and it still seems to be the case today.

Since the code is so short, I'll just paste the rest of it here:

```rust
use memmap2::MmapOptions;
use object::{File, Object, ObjectSection};
use std::env;
use std::error::Error;
use std::fs::{self, OpenOptions};

fn get_section(file: &File, name: &str) -> Option<(u64, u64)> {
    for section in file.sections() {
        match section.name() {
            Ok(n) if n == name => {
                return section.file_range();
            }
            _ => {}
        }
    }
    None
}

fn main() -> Result<(), Box<dyn Error>> {
    let run_count = unsafe { RUN_COUNT };
    println!("Previous run count: {}", run_count);
    let exe = env::current_exe()?;
    let tmp = exe.with_extension("tmp");
    fs::copy(&exe, &tmp)?;

    let file = OpenOptions::new().read(true).write(true).open(&tmp)?;
    let mut buf = unsafe { MmapOptions::new().map_mut(&file) }?;
    let file = File::parse(&*buf)?;

    if let Some(range) = get_section(&file, "count") {
        assert_eq!(range.1, 4);
        let base = range.0 as usize;
        buf[base..(base + 4)].copy_from_slice(&(run_count + 1).to_ne_bytes());

        let perms = fs::metadata(&exe)?.permissions();
        fs::set_permissions(&tmp, perms)?;
        fs::rename(&tmp, &exe)?;
    } else {
        fs::remove_file(&tmp)?;
    }

    Ok(())
}
```

We look up the current executable, and map it into memory using [`memmap2`](https://docs.rs/memmap2/).
Then we get a list of sections using the [`object`](https://docs.rs/object/) crate, search for one called `count` &mdash; beware of libraries using the same name! &mdash; then replace the variable value in the memory-mapped view of the file.

All this happens on a copy of our program.
Self-modifying programs were fine under MS-DOS, but modern operating systems won't let it fly.
Renaming or overwriting is fine, since the original file is still unchanged.

You can see a nice diagram showing the ELF (the executable format on my platform) structure [here](https://github.com/corkami/pics/blob/28cb0226093ed57b348723bc473cea0162dad366/binary/elf101/elf101-64.svg).
But we're not here to learn about executable formats.
We're here to clear out a tiny bit of backlog and see a working example, so does it work?

```bash
$ cargo run --release
   Compiling self-modify v0.1.0 (/home/grayshade/Projects/self-modify)
    Finished release [optimized] target(s) in 0.30s
     Running `target/release/self-modify`
Previous run count: 0
$ target/release/self-modify
Previous run count: 1
$ target/release/self-modify
Previous run count: 2
$ target/release/self-modify
Previous run count: 3
```

It does!

Note that I'm on Linux.
The `object` crate should work with MacOS and Windows executables too, but on Windows the `fs::rename` call will probably fail.
There you can rename the current executable to some temporary name, but I don't want to deal with that stuff [again](https://github.com/rust-analyzer/rust-analyzer/issues/6602).

## Bonus _section_

I mentioned above that declaring our variable as `mut` is necessary because the compiler will optimize it otherwise.
How did I figure that out?

First, let's simplify the code and give the variable a more noticeable value:

```rust
#[link_section = "count"]
#[used]
static mut RUN_COUNT: u32 = 0x55aa55aa;

fn main() {
    let run_count = unsafe { RUN_COUNT };
    println!("Previous run count: {}", run_count);
}
```

```bash
$ cargo build --release
$ objdump -d target/release/self-modify
```

```asm
[snip]
0000000000011160 <_ZN11self_modify4main17hd56d204cd8f62812E>:
   11160:       48 83 ec 48             sub    $0x48,%rsp
   11164:       8b 05 ee 7e 03 00       mov    0x37eee(%rip),%eax        # 49058 <__TMC_END__>
   1116a:       89 44 24 04             mov    %eax,0x4(%rsp)
   1116e:       48 8d 44 24 04          lea    0x4(%rsp),%rax
   11173:       48 89 44 24 08          mov    %rax,0x8(%rsp)
   11178:       48 8b 05 49 51 03 00    mov    0x35149(%rip),%rax        # 462c8 <_etext+0x423>
   1117f:       48 89 44 24 10          mov    %rax,0x10(%rsp)
   11184:       48 8d 05 e5 57 03 00    lea    0x357e5(%rip),%rax        # 46970 <_DYNAMIC+0x280>
   1118b:       48 89 44 24 18          mov    %rax,0x18(%rsp)
   11190:       48 c7 44 24 20 02 00    movq   $0x2,0x20(%rsp)
   11197:       00 00
   11199:       48 c7 44 24 28 00 00    movq   $0x0,0x28(%rsp)
   111a0:       00 00
   111a2:       48 8d 44 24 08          lea    0x8(%rsp),%rax
   111a7:       48 89 44 24 38          mov    %rax,0x38(%rsp)
   111ac:       48 c7 44 24 40 01 00    movq   $0x1,0x40(%rsp)
   111b3:       00 00
   111b5:       48 8d 7c 24 18          lea    0x18(%rsp),%rdi
   111ba:       ff 15 48 4f 03 00       call   *0x34f48(%rip)        # 46108 <_etext+0x263>
   111c0:       48 83 c4 48             add    $0x48,%rsp
   111c4:       c3                      ret
        ...
[snip]
```

Ookay, assembly is hard to read.
What happens without the `mut`?

```asm
0000000000011160 <_ZN11self_modify4main17hd56d204cd8f62812E>:
   11160:       48 83 ec 48             sub    $0x48,%rsp
   11164:       c7 44 24 04 aa 55 aa    movl   $0x55aa55aa,0x4(%rsp)
   1116b:       55
   1116c:       48 8d 44 24 04          lea    0x4(%rsp),%rax
   11171:       48 89 44 24 08          mov    %rax,0x8(%rsp)
   11176:       48 8b 05 4b 51 03 00    mov    0x3514b(%rip),%rax        # 462c8 <_etext+0x423>
   1117d:       48 89 44 24 10          mov    %rax,0x10(%rsp)
   11182:       48 8d 05 e7 57 03 00    lea    0x357e7(%rip),%rax        # 46970 <_DYNAMIC+0x280>
   11189:       48 89 44 24 18          mov    %rax,0x18(%rsp)
   1118e:       48 c7 44 24 20 02 00    movq   $0x2,0x20(%rsp)
   11195:       00 00
   11197:       48 c7 44 24 28 00 00    movq   $0x0,0x28(%rsp)
   1119e:       00 00
   111a0:       48 8d 44 24 08          lea    0x8(%rsp),%rax
   111a5:       48 89 44 24 38          mov    %rax,0x38(%rsp)
   111aa:       48 c7 44 24 40 01 00    movq   $0x1,0x40(%rsp)
   111b1:       00 00
   111b3:       48 8d 7c 24 18          lea    0x18(%rsp),%rdi
   111b8:       ff 15 4a 4f 03 00       call   *0x34f4a(%rip)        # 46108 <_etext+0x263>
   111be:       48 83 c4 48             add    $0x48,%rsp
   111c2:       c3                      ret
        ...
```

Did you spot the `movl   $0x55aa55aa,0x4(%rsp)` on the second line?
That's our value, inlined in the compiled code.

To be honest, we're getting a lot of extra code in there.
Instead, it's better to get rid of the `println!`, simplify the code as much as possible and paste it on godbolt.org.
Click [here](https://godbolt.org/z/d5rbj7fMG) to see how that looks.

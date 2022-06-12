+++
title = "Decoding Firefox session store data"
date = 2022-06-12
+++

If you're using Firefox (and you should), you might have wanted to read its session restore files, perhaps to recover some lost tabs, re-import an old session after a refresh, or even track your tab hoarding habit.

Unfortunately, last time I checked, there wasn't much information available about the session restore format.
I've previously used [`lz4json`](https://github.com/andikleen/lz4json) to decode them, but it's a good prompt for a post, and I'd rather not keep an extra AUR package anyway.

## Finding the files

Under `~/.mozilla/firefox` on Linux, or `%APPDATA%\Mozilla\Firefox` on Windows, you should have a `profiles.ini`, an `installs.ini` and one or more randomly-named subdirectories.
The default profile is marked as such in the two INI files.
Of course, the better way is to open `about:support` and copy the path from there.

In there, there should be a directory called `sessionstore-backups`, with a couple of files:

```bash
$ ls sessionstore-backups
previous.jsonlz4
recovery.baklz4
recovery.jsonlz4
upgrade.jsonlz4-20220518214245
upgrade.jsonlz4-20220530093943
upgrade.jsonlz4-20220606212503
```

I don't know the specifics, but these are backup versions, more or less recent, some of them saved during browser upgrades.
Looking at the last modified dates, it appears that the most recent one is called `recovery.jsonlz4`.

## The `lz4json` format

The `jsonlz4` extension is a good hint, but a good first step is to run the `file` utility, which tries to guess the type of a file:

```bash
$ file recovery.jsonlz4
recovery.jsonlz4: Mozilla lz4 compressed data, originally 145030214 bytes
```

`file` recognizes thousands of file formats, but it's relatively shallow.
This means that the uncompressed length is likely to be easily accessible.
Let's look at the file contents:

```
$ hexyl recovery.jsonlz4 | head -n7
┌────────┬─────────────────────────┬─────────────────────────┬────────┬────────┐
│00000000│ 6d 6f 7a 4c 7a 34 30 00 ┊ 46 fc a4 08 f2 21 7b 22 │mozLz400┊F××•×!{"│
│00000010│ 76 65 72 73 69 6f 6e 22 ┊ 3a 5b 22 73 65 73 73 69 │version"┊:["sessi│
│00000020│ 6f 6e 72 65 73 74 6f 72 ┊ 65 22 2c 31 5d 2c 22 77 │onrestor┊e",1],"w│
│00000030│ 69 6e 64 6f 77 73 22 3a ┊ 5b 7b 22 74 61 62 09 00 │indows":┊[{"tab_0│
│00000040│ 62 65 6e 74 72 69 65 0c ┊ 00 f3 29 75 72 6c 22 3a │bentrie_┊0×)url":│
│00000050│ 22 68 74 74 70 73 3a 2f ┊ 2f 6d 61 74 72 69 78 2e │"https:/┊/matrix.│
```

We can actually see some [JSON](en.wikipedia.org/wiki/JSON) in there.
[LZ4](https://en.wikipedia.org/wiki/LZ4_(compression_algorithm)) is designed to be as fast as possible, so it doesn't do anything fancy like [Huffman coding](https://en.wikipedia.org/wiki/Huffman_coding), yielding partially-readable text as output.

Before the JSON, though, there is a line that looks a bit strange.
The file starts with the `"mozLz40\0"` [magic number](https://en.wikipedia.org/wiki/File_format#Magic_number), followed by what appear to be six non-ASCII bytes, then the start of the JSON.
We can expect the six bytes to contain the file length, or maybe a checksum.
These are usually found either at the beginning or at the end of a file.

Luckily, `file` was nice to tell us the uncompressed data size.
`145030214`, converted to [hex](https://en.wikipedia.org/wiki/Hexadecimal) is `0x08A4FC46`, which is actually visible as `46 fc a4 08` in the [`hexyl`](https://github.com/sharkdp/hexyl) output above.

We can also [peek](https://github.com/file/file/blob/46976e05f97e4b2bc77476a16f7107ff0be12df1/magic/Magdir/mozilla#L22-L25) at the implementation of `file`.
It appears to confirm that the uncompressed size is a [little-endian](https://en.wikipedia.org/wiki/Endianness) 32-bit number, followed directly by the compressed data.
I assume that the `>8` annotation means "starting from offset 8" or something similar to that.

## Writing some code

This is a [Rust](https://www.rust-lang.org/)-related blog, so of course I'll be using that.
Fortunately, we already have libraries for everything we're going to need, so it's going to be easy.
We can use [`lz4_flex`](https://crates.io/crates/lz4_flex) for LZ4 decoding, [`serde`](https://crates.io/crates/serde) and [`serde_json`](https://crates.io/crates/serde_json) for JSON decoding, and [`anyhow`](https://crates.io/crates/anyhow) to handle errors in a nicer way.
We'll also pull in [`memmap2`](https://crates.io/crates/memmap2) to map the file in memory (which is optional, but saves a bit of RAM) and [`url`](https://crates.io/crates/serde_json) for URL parsing.

```toml
# Cargo.toml
[package]
name = "tabs"
version = "0.1.0"
edition = "2021"

[dependencies]
anyhow = "1.0"
lz4_flex = "0.9"
memmap2 = "0.5"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
url = "2.2"
```

`lz4_flex` has a nifty [helper function](https://docs.rs/lz4_flex/0.9.3/lz4_flex/fn.decompress_size_prepended.html) to decode a size-prepended block, exactly what our file uses.
So if we're lucky, it should be enough to open the file, read it, then call the decompression function.
With some imports and command line handling omitted, it simply comes to:

```rust
let file = File::open(&path)?;
let mmap = unsafe { MmapOptions::new().map(&file)? };
let buf = lz4_flex::decompress_size_prepended(&mmap[8..])?;
let buf = String::from_utf8(buf)?;

// check if it worked
println!("{}", &buf[..64]);
```

```bash
$ cargo run --release -- ~/.mozilla/firefox/ou63gnwj.default
# snip
{"version":["sessionrestore",1],"windows":[{"tabs":[{"entries":[
```

I won't show it here, but the way the JSON is structured, the session has a list of windows, each window has a list of tabs, and each tab has a list of history entries.
We only care for the last entry, which is the one currently displayed.

This is pretty easy to parse with `serde`.

```rust
#[derive(Deserialize)]
struct Entry {
    url: String,
}

#[derive(Deserialize)]
struct Tab {
    entries: Vec<Entry>,
}

#[derive(Deserialize)]
struct Window {
    tabs: Vec<Tab>,
}

#[derive(Deserialize)]
struct SessionStore {
    windows: Vec<Window>,
}

// ...

let session = serde_json::from_slice::<SessionStore>(&buf)?;
let mut domains = HashMap::<_, u32>::new();
for window in session.windows {
    for tab in window.tabs {
        if let Some(entry) = tab.entries.last() {
            let url = Url::parse(&entry.url)?;
            // skip about:blank, about:reader etc.
            if let Some(host) = url.host_str() {
                *domains.entry(host.to_string()).or_default() += 1;
            }
        }
    }
}
```

For this example, I'm grabbing every tab in every window, making sure it's not empty, taking the last entry, then sticking the domain of each URL in a `HashMap`, in order to count them.

*Note: If you know Rust, that snippet looks a bit nicer written in a functional/iterator-based style.*

Finally, we [`collect`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect) the entries into a `Vec`, sort them, and display the most common domains:

```rust
let mut domains = domains.into_iter().collect::<Vec<_>>();
domains.sort_unstable_by_key(|p| Reverse(p.1));
for (domain, count) in domains.into_iter().take(10) {
    println!("{} {}", domain, count);
}
```

```bash
# sample output, truncated
$ cargo run --release -- ~/.mozilla/firefox/ou63gnwj.default
github.com 1150
www.youtube.com 213
twitter.com 206
news.ycombinator.com 109
```

*I may or may not have a tab hoarding problem.*

Full code:

```rust
use std::cmp::Reverse;
use std::collections::HashMap;
use std::fs::File;
use std::path::PathBuf;
use std::{env, process};

use memmap2::MmapOptions;
use serde::Deserialize;
use url::Url;

#[derive(Deserialize)]
struct Entry {
    url: String,
}

#[derive(Deserialize)]
struct Tab {
    entries: Vec<Entry>,
}

#[derive(Deserialize)]
struct Window {
    tabs: Vec<Tab>,
}

#[derive(Deserialize)]
struct SessionStore {
    windows: Vec<Window>,
}

fn main() -> anyhow::Result<()> {
    let mut args = env::args_os().collect::<Vec<_>>();
    if args.len() != 2 {
        eprintln!("Usage: {} <profile>", args[0].to_string_lossy());
        process::exit(1);
    }

    let mut path = PathBuf::from(args.remove(1));
    path.push("sessionstore-backups/recovery.jsonlz4");

    let file = File::open(&path)?;
    let mmap = unsafe { MmapOptions::new().map(&file)? };
    let buf = lz4_flex::decompress_size_prepended(&mmap[8..])?;

    let session = serde_json::from_slice::<SessionStore>(&buf)?;
    let mut domains = HashMap::<_, u32>::new();
    for window in session.windows {
        for tab in window.tabs {
            if let Some(entry) = tab.entries.last() {
                let url = Url::parse(&entry.url)?;
                // println!("{url}"); // uncomment this to show all URLs
                if let Some(host) = url.host_str() {
                    *domains.entry(host.to_string()).or_default() += 1;
                }
            }
        }
    }

    let mut domains = domains.into_iter().collect::<Vec<_>>();
    domains.sort_unstable_by_key(|p| Reverse(p.1));
    for (domain, count) in domains.into_iter().take(10) {
        println!("{} {}", domain, count);
    }

    Ok(())
}
```

## Conclusion

We've uncompressed a Firefox session backup and printed the most common domains in the open tabs.

I actually think this is quite a poor choice of format for sessions like mine.
Perhaps you've noticed that my session was 142 MB uncompressed, which is not insignificant.
Worse, it's on-disk size is 42 MB, and Firefox tends to write it every couple of seconds (but only as long as it changes, I hope).
That's pretty bad, not only for performance (since the file must be rewritten every time you scroll, navigate to another page, or type something in a form), but can also reduce the lifespan of an SSD drive.

Because of this, people ended up with [workarounds](https://wiki.archlinux.org/title/profile-sync-daemon) to keep the session on a [RAM drive](https://en.wikipedia.org/wiki/RAM_drive), at the cost of durability in case of a power failure or a crash.

I don't know the reasoning behind this design, but I suspect the vast majority of users have less than four tabs.
Maybe the Firefox engineers wanted to avoid calling into SQLite while restoring the session.

---

By the way, if you've ever felt anxious about your ever-growing tab list, please consider [buying me a coffee](https://www.buymeacoffee.com/lnicolaq).

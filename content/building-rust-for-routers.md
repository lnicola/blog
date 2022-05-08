+++
title = "Building Rust code for my OpenWrt Wi-Fi router"
date = 2022-05-08
+++

I recently got interested in running Rust code on my router (stay tuned for a future blog post?).
This is supposed to be easy, but I never tried it, so let's see how it goes.

## A test project

*Hello, world*s are somewhat boring, so I'd like to build something more realistic — a DNS client.
We're not going to implement DNS here, but rather piggy-back on the [`trust-dns-resolver`](https://crates.io/crates/trust-dns-resolver) crate, which looks pretty good.

After skimming the `trust-dns` and `tokio` docs, and a `cargo new delve` (shouts to `dig` and `drill`), we have this:

```toml
[package]
name = "delve"
version = "0.1.0"
edition = "2021"

[dependencies]
anyhow = "1.0"
tokio = { version = "1.18", features = ["net", "rt-multi-thread"] }
trust-dns-resolver = { version = "0.21", features = [
    "dns-over-rustls",
    "tokio",
] }
```

```rust
use std::env;

use tokio::runtime;
use trust_dns_resolver::{
    config::{ResolverConfig, ResolverOpts},
    proto::{rr::RecordType, xfer::DnsRequestOptions},
    TokioAsyncResolver,
};

async fn run() -> Result<(), anyhow::Error> {
    let resolver =
        TokioAsyncResolver::tokio(ResolverConfig::cloudflare_tls(), ResolverOpts::default())?;
    let query = env::args().nth(1).unwrap();

    let response = resolver
        .lookup(query, RecordType::A, DnsRequestOptions::default())
        .await?;

    for address in response.iter() {
        println!("{}", address);
    }
    Ok(())
}

fn main() -> Result<(), anyhow::Error> {
    let runtime = runtime::Builder::new_multi_thread().enable_all().build()?;
    runtime.block_on(async { tokio::spawn(run()).await })?
}
```

The code is pretty simple.
It creates a `tokio` multi-threaded `Runtime`, sets up a `trust-dns` `TokioAsyncResolver`, then queries the Cloudflare DNS server for the hostname given in the command line.
`RecordType::A` means asking for an IPv4 address record.

If you're wondering about the `block_on` / `spawn` dance, apparently `tokio` prefers that you don't do a lot of work in `block_on`, but rather spawn a root future and do your stuff from there.
It doesn't really matter in this case, and `trust-dns-resolver` helpfully provides a synchronous resolver, but it will matter in the project I have in mind.

And it does appear to work fine:

```bash
$ cargo run --release -- google.com
   [snip]
    Finished release [optimized] target(s) in 9.66s
     Running `target/release/delve google.com`
142.250.179.174
```

## Inside my router

My router is an AVM FRITZ!Box 4040 running OpenWrt.
As far as embedded systems go, OpenWrt is pretty close to a Linux PC.
I've already enabled SSH and added my key, but I don't know what architecture it's running.
Fortunately, `uname -a` works just as expected:

```bash
root@OpenWrt:~# uname -a
Linux OpenWrt 4.14.171 #0 SMP Thu Feb 27 21:05:12 2020 armv7l GNU/Linux
root@OpenWrt:~# cat /proc/cpuinfo
processor	: 0
model name	: ARMv7 Processor rev 5 (v7l)
BogoMIPS	: 67.03
Features	: half thumb fastmult vfp edsp neon vfpv3 tls vfpv4 idiva idivt vfpd32 lpae evtstrm
CPU implementer	: 0x41
CPU architecture: 7
CPU variant	: 0x0
CPU part	: 0xc07
CPU revision	: 5

processor	: 1
model name	: ARMv7 Processor rev 5 (v7l)
BogoMIPS	: 67.03
Features	: half thumb fastmult vfp edsp neon vfpv3 tls vfpv4 idiva idivt vfpd32 lpae evtstrm
CPU implementer	: 0x41
CPU architecture: 7
CPU variant	: 0x0
CPU part	: 0xc07
CPU revision	: 5

processor	: 2
model name	: ARMv7 Processor rev 5 (v7l)
BogoMIPS	: 67.03
Features	: half thumb fastmult vfp edsp neon vfpv3 tls vfpv4 idiva idivt vfpd32 lpae evtstrm
CPU implementer	: 0x41
CPU architecture: 7
CPU variant	: 0x0
CPU part	: 0xc07
CPU revision	: 5

processor	: 3
model name	: ARMv7 Processor rev 5 (v7l)
BogoMIPS	: 67.03
Features	: half thumb fastmult vfp edsp neon vfpv3 tls vfpv4 idiva idivt vfpd32 lpae evtstrm
CPU implementer	: 0x41
CPU architecture: 7
CPU variant	: 0x0
CPU part	: 0xc07
CPU revision	: 5

Hardware	: Generic DT based system
Revision	: 0000
Serial		: 0000000000000000
root@OpenWrt:~# ldd --version
musl libc (armhf)
Version 1.1.24
Dynamic Program Loader
Usage: ldd [options] [--] pathname
```

So it appears a 4-core ARMv7 CPU with hardware floating-point support, running a MUSL-based distro.

## Cross-compiling

Rust supports dozens of targets, but ARMv7 is pretty common, so it's hopefully well-supported.
I'm not sure how the target is called, so `rustup target list` is handy:

```bash
$ rustup target list | rg armv7
armv7-linux-androideabi
armv7-unknown-linux-gnueabi
armv7-unknown-linux-gnueabihf
armv7-unknown-linux-musleabi
armv7-unknown-linux-musleabihf # sounds like a winner
armv7a-none-eabi
armv7r-none-eabi
armv7r-none-eabihf
$ rustup target add armv7-unknown-linux-musleabihf
info: downloading component 'rust-std' for 'armv7-unknown-linux-musleabihf'
info: installing component 'rust-std' for 'armv7-unknown-linux-musleabihf'
```

Great, let's try it!

```bash
$ cargo build --release --target armv7-unknown-linux-musleabihf
   [snip]
error: linking with `cc` failed: exit status: 1
   [snip half a screenful of errors]
```

Oh, we also need a linker.
This would normally be a version of the BFD linker (`ld`).
Unlike `clang` and `rustc`, `gcc` and `ld` only support one target at a time, so I'd have to track down or compile a compatible version.
My Linux distro actually has one, available in AUR as `muslcc-arm-linux-musleabihf-cross-bin`.

But since we're compiling pure-Rust code — did you notice the fancy `rustls` feature of `trust-dns-resolver`? — the LLVM linker, `lld`, will do the trick with less work.

The correct incantation is then:

```bash
$ CARGO_TARGET_ARMV7_UNKNOWN_LINUX_MUSLEABIHF_LINKER=rust-lld cargo build --release --target armv7-unknown-linux-musleabihf
   [snip]
    Finished release [optimized] target(s) in 9.50s
```

Is that all?
That was surpisingly easy!

And by the way, you can also set the linker in `.config/cargo.toml`, like this:

```toml
[target.armv7-unknown-linux-musleabihf]
linker = "rust-lld"
```

## Testing

We could test using QEMU, but that seems too much of a hassle, as the real hardware is already available.

```bash
$ scp target/armv7-unknown-linux-musleabihf/release/delve root@192.168.2.1:~
scp: Connection closed
```

The router is running OpenSSH 8.0p1, but `scp` is deprecated, and the version on my PC uses SFTP by default.
The `-O` flag reverts to the deprecated protocol:

```bash
$ scp -O target/armv7-unknown-linux-musleabihf/release/delve root@192.168.2.1:~
delve                                        100% 8598KB 176.1KB/s   00:48
```

That's a larger binary than I'd like, but 176 KB/s seems pretty slow.
I know my network is faster that this, but it could be the file system:

```bash
root@OpenWrt:~# mount
/dev/root on /rom type squashfs (ro,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,noatime)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,noatime)
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,noatime)
/dev/mtdblock14 on /overlay type jffs2 (rw,noatime)
overlayfs:/overlay on / type overlay (rw,noatime,lowerdir=/,upperdir=/overlay/upper,workdir=/overlay/work)
tmpfs on /dev type tmpfs (rw,nosuid,relatime,size=512k,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,mode=600,ptmxmode=000)
debugfs on /sys/kernel/debug type debugfs (rw,noatime)
```

Oh, okay, let's copy it to `/tmp` instead:

```bash
$ scp -O target/armv7-unknown-linux-musleabihf/release/delve root@192.168.2.1:/tmp
delve                                        100% 8598KB  11.6MB/s   00:00
```

More importantly, does it work?

```bash
root@OpenWrt:~# /tmp/delve news.ycombinator.com
209.216.230.240
```

Honestly, I was more suprised to see it working than you are.

## What about the binary size?

Our executable packs in quite a bit: Tokio, an async DNS client, and a TLS implementation for DNS-over-TLS.
But at 8.5 MB, it's pretty large for a Wi-Fi router with 128 MB of flash.

Let's see if we can get it to weigh less, testing with the PC version.
I know that a large part of the binary must be the symbols.
I'd normally use `strip -s`, but `cargo` can do this by itself.
This is also a good excuse to try the custom profiles feature.

Let's start by adding a new profile to `Cargo.toml`:

```toml
[profile.minsize]
inherits = "release"
```

, then build it with `cargo build --profile minsize`.

There's a lot of resources out there with tips for reducing the binary sizes (it's a common complaint), so I'll just list each thing I've tried, incrementally:

 - baseline: 8.5 MB
 - `strip = true`: 3.2 MB
 - `lto = "thin"`: 3.1 MB
 - `lto = "fat"`: 2.6 MB (didn't expect this!)
 - `opt-level = "s"`: 2.3 MB (didn't expect this either)
 - `panic = "abort"`: 2.1 MB
 - switch to the single-thread `tokio` runtime: 2.0 MB
 - `cargo build -Z build-std=panic_abort,std --profile minsize --target x86_64-unknown-linux-gnu` (you'll need nightly or `RUSTC_BOOTSTRAP=1` for this): 1.9 MB
 - disable the `system-config` feature of `trust-dns-resolver`: 1.9 MB

Rebuilding it for the router, we get a 1.4 MB binary, which is still a bit large, but workable.

## Bonus: packaging for OpenWrt

I was curious about making a binary package for OpenWrt, which appears to use a format inspired by Debian (`ipkg`).
Reading the docs isn't fun, but we can find where the package manager (`okpg`) downloads stuff from (`/etc/opkg/distfeeds.conf`), then get a random package from there.

I picked `attr_20170915-1_arm_cortex-a7_neon-vfpv4.ipk`, which appears to be a `.tar.gz` archive, containing three files:
 - `control.tar.gz`
 - `data.tar.gz`
 - `debian-binary`, a text file saying `2.0`

`control.tar.gz` has one metadata file, `control`, and some pre- and post-install scripts we can copy over or ignore:

```
Package: attr
Version: 20170915-1
Depends: libc, libattr
Source: feeds/packages/utils/attr
License: LGPL-2.1 GPL-2.0
LicenseFiles: doc/COPYING doc/COPYING.LGPL
Section: utils
Maintainer: Maxim Storchak <m.storchak@gmail.com>
Architecture: arm_cortex-a7_neon-vfpv4
Installed-Size: 11080
Description:  Extended attributes support
 This package provides xattr manipulation utilities
 - attr
 - getfattr
 - setfattr
```

Interestingly, my router uses `vfpv4` packages, even though only `vfpv3` appears in `/proc/cpuinfo`.
Well, whatever makes it happy.
The `Installed-Size` field is also strange, as it doesn't seem to match the size of the files.
It's optional optional, so it doesn't matter too much.

We can make a similar file:

```
Package: delve
Version: 0.0.1
Depends: libc
License: MIT
Section: utils
Architecture: arm_cortex-a7_neon-vfpv4
Installed-Size: 1414464
Description: DNS testing tool
```

Finally, `data.tar.gz` contains the installed tree.
Putting things together, we have:

```
.
├── control
├── control.tar.gz
├── data.tar.gz
├── debian-binary
├── delve_0.0.1_arm_cortex-a7_neon-vfpv4.ipk
└── usr
    └── bin
        └── delve

2 directories, 6 files
```

And on the router:

```bash
root@OpenWrt:/tmp# opkg install delve_0.0.1_arm_cortex-a7_neon-vfpv4.ipk
Installing delve (0.0.1) to root...
Configuring delve.
root@OpenWrt:/tmp# delve sci-hub.se
186.2.163.219
```

If you ever try this, make sure to actually compress the archives using `gzip` (or `tar czf`).
I forgot that `tar` only guesses the format when extracting, and `opkg install` greeted me with a fun `Segmentation fault` message.

## Naming apologies

I just realized that the `delve` name is already taken by a Go debugger.
Not that it matters, as nobody will be using this, but it's a common source of complaints.

## Final words

To sum up, we wrote a simple DNS client, tweaked things a little to reduce the binary size, cross-compiled it for an ARM Wi-Fi router, packaged and tested it there.
Surprisingly, it actually worked.

Thanks for staying with me, and if you found this interesting please [consider buying me a coffee](https://www.buymeacoffee.com/lnicolaq).

+++
title = "About a fun little NFS bug"
date = 2022-06-29
+++

One [cloud provider](https://blog.dend.ro/a-mysterous-python-crash/) I sometimes work with has a large repository of images, accessible using the S3 protocol or over an NFS gateway. Today I spent a bit of time debugging an issue related to the NFS gateway, so read on for more &mdash; and for a twist at the end.

## Something is broken

I have a workload which involves relatively many of those images at once, accessed through the [`s3fs`](https://github.com/s3fs-fuse/s3fs-fuse) FUSE helper (a way to make an S3 bucket look like a file system).
For some reason, sometimes requests start failing after a while, and using NFS is a way to work around that.

But something strange was happening today when I was trying to use the NFS share.
One version of [GDAL](https://github.com/OSGeo/gdal/) was able to read the files, while another, older one, complained they were invalid and wasn't able to open them.
Using `s3fs`, both GDAL versions worked fine.

This made me think that NFS is returning corrupted files (maybe with some junk at the end), and the newer GDAL was able to ignore that.
But on a closer look, both `s3fs` and NFS returned identical (up to a checksum, at least) files.
This was the cause of some head scratching, since reading a the same file from different places should yield the same results &mdash;

## My favourite tool

&mdash; unless maybe it shouldn't?
This story seemed fishy and I broke out what's probably my favourite debugging tool, `strace`.
`strace` sets breakpoints on every interaction of a program with the operating system ([system call](https://en.wikipedia.org/wiki/System_call)) and displays the request involved, including the arguments.
So, for example, if Linux a program tries to open a file, it will show up in `strace` as an `open` or `openat` call.

*Note: if that sounds interesting and you want to know more, check out [this zine](https://jvns.ca/blog/2015/04/14/strace-zine/) and [this article](https://jvns.ca/blog/2021/04/03/what-problems-do-people-solve-with-strace/).
And if you're not using Linux, you can try [Process Monitor](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon) or [DTrace](http://dtrace.org/blogs/about/).*

So I gave `strace` a try (edited for brevity):

```bash
$ strace gdalinfo /mount_nfs/path/to/file.jp2 # old GDAL
[snip]
stat("/mount_nfs/path/to/file.jp2", {st_mode=S_IFREG|0666, st_size=115495922, ...}) = 0
open("/mount_nfs/path/to/file.jp2", O_RDONLY) = 3
fstat(3, {st_mode=S_IFREG|0666, st_size=115495922, ...}) = 0
read(3, "[snip]"..., 8192) = 8192
lseek(3, 0, SEEK_SET) = 0
[snip]
open("/mount_nfs/path/to/file.jp2", O_RDONLY) = -1 EPERM (Operation not permitted)
open("/mount_nfs/path/to/file.jp2", O_RDONLY) = -1 EPERM (Operation not permitted)
[snip]
```

Most of the output above isn't too important, but the last `open` calls fail with `EPERM`, which sounds like a good reason to not work.
If you have trouble reading the log, the full sequence is:

 - `gdalinfo` checks the file length and attributes
 - `gdalinfo` opens the file
 - `gdalinfo` checks the length and attributes of the open file
 - `gdalinfo` reads 8 KB from the file, which works, then rewinds to the beginning
 - `gdalinfo` tries to open the file again and gets a permissions-related error
 - `gdalinfo` tries to open the file again and gets a permissions-related error

Sure, this is a bit redundant, but it doesn't matter that much when you're reading 115 MB.
Note how the first `open` call works, then but the next ones fail.

In contrast, the newer GDAL works because it only tries to open the file once.
So what's wrong, can we only read files once?

Of course not, otherwise a second `gdalinfo` wouldn't work.
So a better theory is that we can't open the same file twice.
And now that we have a theory, we can try to confirm it using ~~Rust~~ just kidding, let's use Python this time:

```python
# open the file once, works
>>> f1 = open("/mount_nfs/path/to/file.jp2")

# open it again, fails with EPERM
>>> f2 = open("/mount_nfs/path/to/file.jp2")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
IOError: [Errno 1] Operation not permitted: '/mount_nfs/path/to/file.jp2'

# close the first handle
>>> f1.close()

# opening it works
>>> f2 = open("/mount_nfs/path/to/file.jp2")

# opening it a second time fails again
>>> f3 = open("/mount_nfs/path/to/file.jp2")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
IOError: [Errno 1] Operation not permitted: '/mount_nfs/path/to/file.jp2'

# and so on
>>> f2.close()
>>> f3 = open("/mount_nfs/path/to/file.jp2")
>>>
```

So yeah, our theory was correct.
I don't know what kind of server we're dealing with, but it doesn't let us open any file multiple times.
It even fails &mdash; maybe not surprisingly &mdash; if we're opening it from different processes.

I wasn't able to find any reports of a similar problem, so one of my reasons for writing this is to make it available to search engines.

## The solution

I also happened to stumble upon a pretty simple workaround: add `nfsvers=3` to the mount options (`/etc/fstab`).
It appears that the problem only occurs when using NFSv4, and not NFSv3.

## A final twist

All this sounded somewhat familiar, so I searched through my inbox and found I had already reported this to the cloud provider in March 2020.
I even took a `tcpdump` capture and found out that the server was returning `NFS4ERR_PERM` on the failing call:

> This indicates that the requester is not the owner.  The operation was not allowed because the caller is neither a privileged user (root) nor the owner of the target of the operation.

But don't confuse this with `NFS4ERR_ACCESS`:

> This indicates permission denied.  The caller does not have the correct permission to perform the requested operation.  Contrast this with NFS4ERR_PERM (Section 13.1.6.2), which restricts itself to owner or privileged user permission failures.

Hear this, Google?
If you've got a weird NFS server that doesn't allow you to open files a second time and yields `NFS4ERR_PERM`, try downgrading to NFSv3.

And no, the provider never answered back.

---

By the way, if your files sometime go away or can't be reopened, please consider [buying me a coffee](https://www.buymeacoffee.com/lnicolaq).

+++
title = "Announcing the sd-notify crate"
date = 2022-01-12
+++
This is a quick post announcing the [`sd-notify`](https://crates.io/crates/sd-notify) crate.
`sd-notify` is a Rust library for interacting with `systemd` or a compatible service manager.

If you're not running Linux or you don't like `systemd`, this crate is not for you.

## systemd in 3 minutes

Assuming you are running `systemd`, at some point you might find yourself with a program you want to automatically run on start-up.
Historically, this meant doing an intricate dance which includes calling `fork()` twice and writing your PID to a file, but fortunately, with `systemd` this is no longer required and is actually discouraged.

You can take your plain console application and make a `systemd` unit for it:

```ini
[Unit]
Description=Monitors room temperature

[Service]
User=stats
ExecStart=/usr/local/bin/monitor-temperature
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

You then drop that file under `/etc/systemd/system/monitor-temperature.service`, run `systemctl daemon-reload`, `systemctl enable --now monitor-temperature` and you're done.

For the effort, you get logs (`journalctl -efu monitor-temperature`), precise child process tracking (using control groups), CPU, memory and I/O accounting, automatic restarts, and a way to manage your service across all the popular Linux distros.

## systemd start-up types

If your daemon has a long start-up sequence, `systemd` can tell you whether it is ready or not.
In order to do that, you can set the `Type` clause.

The simplest, and also the default, option is called `simple`.
It means that the daemon is marked as ready right after it staarts.
Another useful option is `forking`, which means that the service does the classic double-fork dance.
You can read more about these two (any many others) in your [`systemd.service`](https://www.freedesktop.org/software/systemd/man/systemd.service.html) documentation page.

What the `sd-notify` crate helps with is the `notify` start-up type.
This lets you do whatever initialization you want, then notify `systemd` that you are ready by sending a over a Unix socket.
Services can also say that they are reloading their settings, stopping, or even set a status message.
The protocol is described in detail [here](https://www.freedesktop.org/software/systemd/man/sd_notify.html).

## Using sd-notify

The basic usage is quite simple.
You do your initialization steps, then call:

```rust
let _ = sd_notify::notify(true, &[NotifyState::Ready]);
```

That's all and now your daemon only shows as "running" when it finished starting up.

```bash
$ systemctl status monitor-temperature
     Active: activating (start) since Wed 2022-01-12 20:36:02 EET; 9s ago
     [snip]
$ systemctl status monitor-temperature
     Active: active (running) since Wed 2022-01-12 20:36:12 EET; 2s ago
     [snip]
```

You can also include a status message:

```rust
let _ = sd_notify::notify(
    true,
    &[
        NotifyState::Ready,
        NotifyState::Status("Monitoring room temperature"),
    ],
);
```

```bash
$ systemctl status monitor-temperature
     Active: active (running) since Wed 2022-01-12 20:41:46 EET; 782ms ago
     Status: "Monitoring room temperature"
     [snip]
```

There's also one advanced feature: you can retrieve file descriptors passed by the service manager, for socket-activated daemons.
You can read more on it [here](https://www.freedesktop.org/software/systemd/man/sd_listen_fds.html).

## Final words

`sd-notify` is written in pure Rust, has no dependencies and is available under a permissive license (MIT or Apache-2.0).
My intention is that it stays lightweight, but I'm not necessarily opposed to adding more features.

If you want more functionality today, try the [`libsystemd`](https://crates.io/crates/libsystemd) or [`rust-systemd`](https://crates.io/crates/systemd) crates.

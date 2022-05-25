+++
title = "Announcing the co2mon crate"
date = 2021-12-30
+++
This is a couple of years too late, but I want to write a quick post announcing my first crate, [`co2mon`](https://crates.io/crates/co2mon). It's a small library for reading data from Holtek and similar CO₂ USB monitors.
These are relatively popular, because the protocol [was reverse-engineered](https://hackaday.io/project/5301-reverse-engineering-a-low-cost-usb-co-monitor) and it's easy to communicate with them.

## Linux prerequisites

If you're on Linux, you will need permissions to access the device node.
The following `udev` rule will give permissions to every user on the system:

```
ACTION=="add|change", SUBSYSTEMS=="usb", ATTRS{idVendor}=="04d9", ATTRS{idProduct}=="a052", MODE:="0666"
```

See the crate documentation for details on how to install it.
You can check the vendor and product id pair using the `lsusb` command.

If you get a linker error and you might need to install a `libusb-1.0-0-dev` or `libusb` package.

## Getting started

Add the `co2mon` dependency to your `Cargo.toml`, then copy-paste the following code and try to run it:

```rust
use co2mon::{Result, Sensor};
use std::thread;
use std::time::Duration;

fn main() -> Result<()> {
    let sensor = Sensor::open_default()?;
    loop {
        match sensor.read() {
            Ok(reading) => println!("{:.4} °C, {} ppm CO₂", reading.temperature(), reading.co2()),
            Err(e) => eprintln!("{}", e),
        }
        thread::sleep(Duration::from_secs(5));
    }
}
```

The sensor sends various types of messages, in an arbitrary order.
`Sensor::read` waits for a pair of temperature and CO₂ measurements and returns that.
There's also a `Sensor::read_one` method, which returns a single message.
This can be useful for versions of the sensor that also measure the relative humidity.

## Device support

Tested using a TFA Dostmann [AIRCO2NTROL MINI](https://www.tfa-dostmann.de/en/product/co2-monitor-airco2ntrol-mini-31-5006/), but thanks to [a kind contributor](https://github.com/lnicola/co2mon/pull/6), the crate should also work for newer versions like the [COACH](https://www.tfa-dostmann.de/en/product/co2-monitor-airco2ntrol-coach-31-5009/).

## What to do with it

Open the windows.
The CO₂ levels you'll find in a closed room aren't exactly dangerous, but can still cause headaches, malaise and impair thinking skills.
At one point I took mine to the office and it showed an error because it only goes up to 3000 ppm.
Reading from USB still worked, giving concentrations around 3500 ppm.

Having a constant airflow in an office will also help against Covid, possibly even more than wearing masks.

## Demo

My Grafana isn't in the best of shapes these days, but here's a screenshot:

<a href="/assets/co2.webp" target="_blank"><img src="/assets/co2.webp"></a>

---
layout: post
title:  "Modern Embedded Debugging Show-and-tell"
date:   2020-11-03 18:30:00 +1300
categories: Rust debugging embedded microcontroller disorganised
author: Ian Rees
---

Tools and techniques for debugging embedded systems are like anything else: a compromise between competing goals.  Historically, they have been some combination of proprietary (usually expensive) and hacky, and generally lag behind those for regular computers.  Debuggers are usually relatively slow, which can mean they aren't usable for debugging in the real-time environments that embedded systems are often used in; rather than a `debug_printf("got here\n");`, it might only be practical to insert a `turn_on_led();` in a suspect piece of code.  If a debug setup isn't slow and/or expensive, it has usually been costly in system resources such as program space or hardware in the target.

This "talk" was a rambly and disorganised tour through some new tooling for working on embedded systems, which can reduce each of those shortcomings.  Demos used firmware written in Rust targeting a microcontroller with an ARM M0+ core, but the Rust-based tooling is a bit more widely applicable so may find use in other contexts.  The fancy new stuff is open source, mainly developed by the [knurling-rs](https://github.com/knurling-rs) project.  Some aspects of it are a bit rough around the edges still, but progress is fast and the community is welcoming to newcomers and pull requests.

Starting from the top, the demonstrated debug stack uses:

[Cargo](https://doc.rust-lang.org/cargo/) to coordinate everything, replacing a role that was often filled by either [make](https://www.gnu.org/software/make/) or a proprietary IDE (which in some instances wrap make).  A simple `cargo run` might:
1. Build the firmware executable
2. Convert the executable from elf to a target-device-specific binary format
3. Connect to the target processor through the hardware debugger
4. Load the binary in to the target's flash memory
5. Open a console for debug output
6. Start the program on the target processor.

[defmt](https://ferrous-systems.com/blog/defmt/) moves formatting of the debug messages from the target to the host.  As an example: a call like `defmt::info!("Would send {:u32}", sample);`, by default produces a similar end result to an equivalent `printf()`, but:
* Reduces flash memory usage on the target.  The code involved in turning the u32 (unsigned 32-bit integer) in to a string is no longer needed.  Also, the string literal `"Would send {:u32}"` is stripped out of the firmware binary and replaced with an integer.
* Reduces the target processor time required to send the debug message.  Mainly, this saving comes from the relatively-expensive formatting of the u32 in to a string.  Also, only a few bytes (4? for the formatting string, and 4 for the u32) need to be moved in to the debug output queue, versus the formatting string and u32 in to the formatting code, and the result of formatting going out to the debug queue.
* Frees up bandwidth in the debug transport layer, only a couple integers are moved across it rather than the formatted message.  This is helpful for two reasons: it allows for more detailed logging, and the target processor might be interrupted less depending on the debug transport scheme.
* Fancy logging on the host side gets a lot easier.  Rather than dumping an integer in to a console, perhaps the host side uses the integer value to update a graph, log to a binary file, generate some sound, etc.

[probe-run](https://github.com/knurling-rs/probe-run), used here as a cargo runner, it uses [probe-rs](https://github.com/probe-rs/probe-rs) to interface with the underlying debug hardware.  probe-rs replaces [OpenOCD](http://openocd.org/) in the open-source world, or proprietary tooling.

[RTT](https://wiki.segger.com/RTT) is the technology used to transfer the debug information between the host and target over [SWD](https://en.wikipedia.org/wiki/JTAG#Similar_interface_standards); this uses the same hardware on the target as for flashing it, so no additional pins or peripherals need to be allocated to debugging.  RTT was developed commercially, but with the protocol made publicly available, it has become a defacto standard.

In the demo, I used an ST-Link v2.1 as the debug probe, connected via SWD to a [Feather](https://www.adafruit.com/feather) (specifically one the "M0" series) with an ATSAMD21G18AU microcontroller.  The ST-Link came from an inexpensive ST Nucleo development board, but there are several SWD capable debug probes which are compatible with probe-rs.  Both these are available from Digi-Key for instance - a Nucleo is digikey p/n [497-15096-ND](https://www.digikey.co.nz/product-detail/en/stmicroelectronics/NUCLEO-F072RB/497-15096-ND/5047984) and a Feather is [1528-1531-ND](https://www.digikey.co.nz/product-detail/en/adafruit-industries-llc/2772/1528-1531-ND/5775537) (however there are less expensive options with equivalent microcontrollers).
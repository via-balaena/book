# Raspberry Pi Pico 2

The
[Raspberry Pi Pico 2](https://www.raspberrypi.com/products/raspberry-pi-pico-2/)
is an RP2350 board, supported in Tock as
[`boards/raspberry_pi_pico_2`](https://github.com/tock/tock/tree/master/boards/raspberry_pi_pico_2).
It is cheap and easy to find, which makes it a tempting first board. It also
does not behave the way the rest of this getting started guide assumes, and each
of those differences looks like a broken setup rather than a difference.

If this is your board, read this page in place of
[Hardware Setup](./hardware.md) and [Tockloader](./tockloader.md).

## Four differences to know before you start

**The USB port is not a console.** Tock's RP2350 support has no USB device
driver, so a Pico 2 running Tock creates no serial device when you plug it into
your computer. The kernel's console is UART0, on two of the board's pins. The
boards this guide is written around all appear as a serial device when you plug
them in; this one does not, so you need a second piece of hardware to see
anything at all.

The USB port is still used before Tock runs. The RP2350's bootrom presents a USB
drive when you hold the BOOTSEL button, which is how the UF2 route below flashes
the board.

**There is no Tock bootloader, and no tockloader route.** The board provides no
tockloader target, so `tockloader listen`, `tockloader list` and
`tockloader install` are not how you drive a Pico 2. The "Test The Board"
section of [Hardware Setup](./hardware.md) does not apply here.

**Applications are part of the kernel image.** On this board an application's
[TBF](../doc/tock_binary_format.md) is copied into the kernel ELF's `.apps`
section and the combined image is flashed, so loading an application means
reflashing the kernel as well. The promise in
[Installing Applications](./apps.md), that you can install one or more apps
without having to update or re-flash the kernel, does not hold here.

**`make install` can do nothing and report success.**
[Building the Kernel](./kernel.md) recommends `make install` as the way to load
the kernel. On this board `install` is an alias for `flash`, which converts the
kernel to a UF2 and copies it to the directory named by `BOOTSEL_FOLDER`. That
variable defaults to `/run/media/$(USER)/RP2350`. On macOS, where volumes mount
under `/Volumes`, that directory is never there, and on Linux it depends on your
desktop. When it is not there the Makefile prints a suggestion and exits
_successfully_. Nothing downstream notices, and the board goes on running
whatever it held before. Set `BOOTSEL_FOLDER` yourself, as shown below.

## What you need

- A Raspberry Pi Pico 2, and a USB cable that carries data. Charge-only cables
  are common and cause failures that read as wiring faults.

- A way to read the console. Either a
  [Raspberry Pi Debug Probe](https://www.raspberrypi.com/products/debug-probe/),
  which flashes the board as well, or any 3.3V USB-to-serial adapter, which only
  reads the console and leaves you flashing over BOOTSEL.

- The tools from the [quickstart](./quickstart.md) for your platform, plus one
  of these two, depending on how you flash:

  - **picotool**, for the UF2 route. It is packaged for Homebrew
    (`brew install picotool`) and can be built from
    [source](https://github.com/raspberrypi/picotool).

  - **An OpenOCD that knows the RP2350**, for the debug probe route. This is
    usually not the OpenOCD your package manager gives you: Homebrew's
    `open-ocd` is 0.12.0 and ships `target/rp2040.cfg` but no
    `target/rp2350.cfg`, so the board's config fails with
    `Error: Can't find target/rp2350.cfg`. The
    [Raspberry Pi fork of OpenOCD](https://github.com/raspberrypi/openocd) has
    it; the board's README has build instructions.

## Build the kernel

```
$ cd tock/boards/raspberry_pi_pico_2
$ make
```

```
    Finished `release` profile [optimized + debuginfo] target(s) in 8.44s
   text	   data	    bss	    dec	    hex	filename
  69164	      0	  19100	  88264	  158c8	target/thumbv8m.main-none-eabi/release/raspberry_pi_pico_2
```

The kernel lands at
`target/thumbv8m.main-none-eabi/release/raspberry_pi_pico_2.elf`, relative to
the top of the Tock tree.

## Wire up the console first

Do this before flashing. Both flashing routes reset the board as they finish,
and the line that tells you the kernel came up goes out at that moment. If
nothing is listening you miss it, and a missed line looks exactly like a board
that never started.

The console is UART0 at 115200 baud: **GP0 transmits and GP1 receives**. Serial
connectors are labelled from the point of view of the device they are attached
to, so the two data wires cross over and the ground does not.

With a Raspberry Pi Debug Probe:

| Probe                           | Pico 2                                                                          |
| ------------------------------- | ------------------------------------------------------------------------------- |
| `D` connector, all three wires  | the header silkscreened `DEBUG`: SWCLK, GND, SWDIO, in that order, like to like |
| `U` connector, orange, probe TX | `GP1`, the board's receive                                                      |
| `U` connector, yellow, probe RX | `GP0`, the board's transmit                                                     |
| `U` connector, black            | any `GND` pin                                                                   |

Neither of the probe's connectors carries power. Power the Pico 2 separately
over its own USB port; wiring only the probe leaves you with a dead board and no
indication of why.

With a plain USB-to-serial adapter, wire the last three rows only and flash over
BOOTSEL.

Then open the port. The probe appears as a serial device on whichever machine it
is plugged into, and the surest way to learn its name is to list before and
after plugging it in:

```
$ ls /dev/tty.*      # macOS
$ ls /dev/ttyACM*    # Linux
$ picocom -b 115200 /dev/ttyACM0
```

macOS has `screen` already; on Debian and its relatives `picocom` usually needs
installing first.

## Flash the kernel

### Over BOOTSEL, with a UF2 file

This route needs no probe. Hold the BOOTSEL button down while you plug the board
into your computer, and it appears as a drive. Check where it mounted
(`ls /Volumes` on macOS, `ls /run/media/$USER` or `ls /media/$USER` on Linux)
and name that directory on the command line:

```
$ cd tock/boards/raspberry_pi_pico_2
$ make BOOTSEL_FOLDER=/Volumes/RP2350 flash
```

Naming it matters: with no `BOOTSEL_FOLDER` that finds the drive, `make flash`
converts the kernel, skips the copy, and still exits successfully.

The board reboots and the drive disappears as the copy lands, which is how you
know it worked. The same thing by hand, if you would rather see the two steps:

```
$ picotool uf2 convert \
    target/thumbv8m.main-none-eabi/release/raspberry_pi_pico_2.elf tock.uf2
$ cp tock.uf2 /Volumes/RP2350
```

### Over a debug probe

With the `D` connector wired as in the table above:

```
$ cd tock/boards/raspberry_pi_pico_2
$ make flash-openocd
```

That writes the kernel over SWD, reads it back to check it, resets the board and
exits. The read-back is **silent when it succeeds** and names the differing
addresses when it does not, so no output is the good outcome here. With no probe
found it prints `unable to find a matching CMSIS-DAP device` and exits non-zero,
which is the failure you want: loud, immediate, and about the right thing.

`flash-openocd` uses `interface/cmsis-dap.cfg` by default, which is what the
Debug Probe presents. To flash from a Raspberry Pi driving the SWD pins
directly, pass `OPENOCD_INTERFACE=swd`.

## What success looks like

With the console open, the board resets at the end of flashing and prints:

```
Initialization complete. Enter main loop
tock$
```

The first line comes from the end of the kernel's own start-up. The second is
the process console's prompt. Both of them arrive on the wire coming out of the
board, so neither says anything about the wire going in. Typing is what tests
that direction:

```
tock$ list
 PID    ShortID    Name                Quanta  Syscalls  Restarts  Grants  State
```

A header row with nothing under it is the correct answer at this point: you have
flashed a kernel and no applications. Getting any answer at all is the part that
matters, because a reply is the only thing that proves the wire into the board.

## Applications

Applications are not installed separately on this board. An application's TBF is
copied into the kernel ELF's `.apps` section and the combined image is flashed,
so every application change reflashes the kernel too. Flash for applications is
a fixed 256 kB region at `0x10040000`, declared in the board's `layout.ld`.

The board's Makefile has `program` and `program-openocd` targets that do this,
each taking `APP=<path to a .tbf>`; `program` copies a UF2 and so carries the
same `BOOTSEL_FOLDER` requirement as `flash`. See the board's
[README](https://github.com/tock/tock/tree/master/boards/raspberry_pi_pico_2)
for the details.

## If your board is a Pico 2 W

The Pico 2 W carries the same RP2350, and everything on this page works on it
unchanged: it builds, flashes, boots and takes typed commands. The difference
that matters here is the LED. On a plain Pico 2 the user LED is on GP25, and
that is the pin this board hands to the LED driver; on a Pico 2 W that pin is
the wireless chip's chip select and the LED is on the wireless chip, so the LED
driver reaches nothing. Nothing on this page uses it.

## When nothing appears

Several different faults produce the same silence on this board. These are the
ones that are worth telling apart.

| What you see                                                                  | What it means                                                                                                                                                                                        |
| ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Please edit the BOOTSEL_FOLDER variable`, and the command exits with success | The kernel built and converted; the copy is the only step that did not happen. Set `BOOTSEL_FOLDER` to where the drive actually mounted.                                                             |
| `Error: Can't find target/rp2350.cfg`                                         | Your OpenOCD predates RP2350 support. See the tools list above.                                                                                                                                      |
| `unable to find a matching CMSIS-DAP device`                                  | OpenOCD cannot see the probe. Try another USB cable first: a charge-only cable leaves the probe powered and invisible, which reads as a wiring fault.                                                |
| Flashing succeeds, and the console shows nothing at all                       | The kernel is on the chip and the debug side is fine, so the fault is in the three console wires or in which device your terminal opened. Check the device name before rewiring, because it is free. |
| The banner arrives, but typing does nothing                                   | One wire: the one carrying toward the board, probe TX to `GP1`. The banner proves everything in the other direction.                                                                                 |
| Characters arrive at about the right moments, and are mostly wrong            | The baud rate does not match, or the two sides share no ground. Check the number first.                                                                                                              |

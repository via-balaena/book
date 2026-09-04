# Raspberry Pi Pico 2

The
[Raspberry Pi Pico 2](https://www.raspberrypi.com/products/raspberry-pi-pico-2/)
is an RP2350 board, supported in Tock as
[`boards/raspberry_pi_pico_2`](https://github.com/tock/tock/tree/master/boards/raspberry_pi_pico_2).
It does not behave the way the rest of this getting started guide assumes, and
most of those differences read as a broken setup rather than as a difference.

If this is your board, read this page in place of
[Hardware Setup](./hardware.md) and [Tockloader](./tockloader.md).

## Four differences to know before you start

| Difference                                                                                                                                                              | What it means here                                                                                                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **The USB port is not a console.** Tock's RP2350 support has no USB device driver.                                                                                      | The board creates no serial device when you plug it in, so you need a second piece of hardware to see anything at all. The console is UART0, on `GP0` and `GP1`, at 115200 baud.                                                                                        |
| **There is no Tock bootloader, and no tockloader route.**                                                                                                               | `tockloader listen`, `tockloader list` and `tockloader install` are not how you drive this board, and the "Test The Board" section of [Hardware Setup](./hardware.md) does not apply.                                                                                   |
| **Applications are part of the kernel image.** An application's [TBF](../doc/tock_binary_format.md) is copied into the kernel ELF's `.apps` section.                    | Loading an application reflashes the kernel with it, so the promise in [Installing Applications](./apps.md), that you can install apps without re-flashing the kernel, does not hold. This page does not cover applications; see [Where to go next](#where-to-go-next). |
| **`make install` can do nothing and report success.** On this board `install` is an alias for `flash`, which copies a UF2 into the directory named by `BOOTSEL_FOLDER`. | When that directory is not there, the Makefile prints a suggestion and exits _successfully_, and the board goes on running whatever it held before. Set `BOOTSEL_FOLDER` yourself, as shown below.                                                                      |

## What you need

- A Raspberry Pi Pico 2, and a USB cable that carries data. Charge-only cables
  are common and cause failures that read as wiring faults.

- A way to read the console. Either a
  [Raspberry Pi Debug Probe](https://www.raspberrypi.com/products/debug-probe/),
  which flashes the board as well, or any 3.3V USB-to-serial adapter, which only
  reads the console and leaves you flashing over BOOTSEL.

- The tools from the [quickstart](./quickstart.md) for your platform, and then
  one of these, depending on how you flash: **picotool** for the UF2 route,
  packaged for Homebrew (`brew install picotool`) and buildable from
  [source](https://github.com/raspberrypi/picotool); or **an OpenOCD that knows
  the RP2350** for the debug probe route.

**The OpenOCD your package manager gives you is probably the wrong one.** Both
the [Mac](./quickstart-mac.md) and [Linux](./quickstart-linux.md) quickstarts
install OpenOCD that way. Homebrew's `open-ocd` is 0.12.0, which ships
`target/rp2040.cfg` and no `target/rp2350.cfg`, so the board's config fails with
`Error: Can't find target/rp2350.cfg`. Check for that file before assuming yours
will work. The
[Raspberry Pi fork of OpenOCD](https://github.com/raspberrypi/openocd) has it,
and the board's README has build instructions.

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

With a Raspberry Pi Debug Probe, the console is the `U` connector:

| Probe wire                      | Pico 2                      |
| ------------------------------- | --------------------------- |
| `U` connector, orange, probe TX | `GP1`, the board's receive  |
| `U` connector, yellow, probe RX | `GP0`, the board's transmit |
| `U` connector, black            | any `GND` pin               |

If you are flashing over the probe as well, its `D` connector's three wires go
to the header silkscreened `DEBUG` — SWCLK, GND, SWDIO, in that order, like to
like. With a plain USB-to-serial adapter, wire only the three `U` connector rows
and flash over BOOTSEL.

Neither of the probe's connectors carries power. Power the Pico 2 separately
over its own USB port; wiring only the probe leaves you with a dead board and no
indication of why.

Then open the port. The probe appears as a serial device on whichever machine it
is plugged into, and the surest way to learn its name is to list before and
after plugging it in:

```
$ ls /dev/tty.*      # macOS
$ ls /dev/ttyACM*    # Linux
```

Open it at 115200 baud, using the name you just found:

```
$ screen /dev/tty.usbmodemXXXX 115200   # macOS, nothing to install
$ picocom -b 115200 /dev/ttyACM0        # Linux, picocom usually needs installing
```

## Flash the kernel

### Over BOOTSEL, with a UF2 file

This route needs no probe. The USB port is not a console once Tock is running,
but the RP2350's bootrom still presents a USB drive before Tock starts, and that
is what this route writes to. Hold the BOOTSEL button down while you plug the
board into your computer, and it appears as a drive. Check where it mounted
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

With the probe's `D` connector wired to the `DEBUG` header, as described under
[Wire up the console first](#wire-up-the-console-first):

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

## Where to go next

This page does not cover applications. Loading one on this board means
reflashing the kernel with the application inside the image, which is not the
process [Installing Applications](./apps.md) describes, and it is not documented
here yet.

The [Tock Course](../course/course.md) modules assume an nRF52840DK or an imix,
so their steps will not follow directly on a Pico 2. The
[kernel documentation](../doc/index.md) applies to any board.

## If your board is a Pico 2 W

Everything on this page applies to a Pico 2 W: same RP2350, and it builds,
flashes, boots and takes typed commands the same way. The one difference is the
LED, which nothing on this page uses. On a plain Pico 2 the user LED is on GP25
and that is the pin the board hands to the LED driver; on a Pico 2 W that pin is
the wireless chip's chip select, so the LED driver reaches nothing.

## When it does not work

The failures on this board are easy to confuse: some print a message, some are
silent, and more than one of them reads as a wiring fault. These are the ones
worth telling apart.

| What you see                                                                  | What it means                                                                                                                                                                                        |
| ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Please edit the BOOTSEL_FOLDER variable`, and the command exits with success | The kernel built and converted; the copy is the only step that did not happen. Set `BOOTSEL_FOLDER` to where the drive actually mounted.                                                             |
| `Error: Can't find target/rp2350.cfg`                                         | Your OpenOCD predates RP2350 support. See [What you need](#what-you-need).                                                                                                                           |
| `unable to find a matching CMSIS-DAP device`                                  | OpenOCD cannot see the probe. Try another USB cable first: a charge-only cable leaves the probe powered and invisible, which reads as a wiring fault.                                                |
| Flashing succeeds, and the console shows nothing at all                       | The kernel is on the chip and the debug side is fine, so the fault is in the three console wires or in which device your terminal opened. Check the device name before rewiring, because it is free. |
| The banner arrives, but typing does nothing                                   | One wire: the one carrying toward the board, probe TX to `GP1`. The banner proves everything in the other direction.                                                                                 |
| Characters arrive at about the right moments, and are mostly wrong            | The baud rate does not match, or the two sides share no ground. Check the number first.                                                                                                              |

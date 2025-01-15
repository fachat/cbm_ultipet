# Ulti-PET

Note: this is a part of a larger set of repositories, with [upet_family](https://github.com/fachat/upet_family) as the main repository.

This is a re-incarnation of the Commodore PET computer(s) from the later 1970s.

It is build on a PCB that fits into a C64-II case (with extra cutouts for connectors), and has only parts that can still be obtained new in 2024.
A 3D-printed case is also available specific for the Ulti-PET PCB.

![Picture of a Ulti-PET](images/cover.jpg)

Videos on the machine are coming soon on the [YouTube 8-bit times](https://youtube.com/playlist?list=PLi1dzy7kw1iybjcUccgjCV4fhNH4IPWSx) channel.

## Features

The board is built with a number of features:

- Commodore 3032 / 4032 / 8032 / 8296 with options menu to select at boot
  - Boot-menu to select different PET versions to run (BASIC 1, 2, 4)
  - 40 and 80 col character display
  - 8296 memory map emulation
  - IEEE488 interface (card edge and 24pin flat ribbon cable)
  - Tape connector (card edge, incl. 9V for proper tape operation)
  - PET graphics keyboard, or alternatively a C64 keyboard
- Improved system design:
  - 512k video RAM, plus 512k fast RAM, accessible using banks on the W65816 CPU
  - boot from an SPI Flash ROM
  - up to 13.5 MHz mode (via configuration register)
  - Write protection for the PET ROMs once copied to RAM
  - lower 32k RAM mappable from all of the 512k fast RAM
  - single 5V power supply (<1A)
  - firmware supports SD-Card and USB keyboard or mouse
- Improved Audio output:
  - Original beeper sound
  - Audio output using a DAC with DMA
  - Dual SID output
  - beeper, DAC, and SID are mixed into a stereo line output
  - integrated amplifier can drive <1W speakers
- Improved Video output:
  - VGA colour video output (222 RGB)
  - up to 96x72 characters on screen
  - hires modes up to a 768x576 resolution
  - 16 out of 64 colour palette, Colour-PET or C128 VDC-compatible, Sprites
  - Hires graphics mode (using a configuration register)
  - modifyable character set
  - multiple video pages mappable to $8000 video mem address
- Extra I/O features:
  - Userport Joystick, software-switchable between single and dual joystick configs
  - fast serial IEC for Commodore's 1581 or 1571 disk drives (ROM support TBD)
  - 5V SPI interface for further extensions
  - 3.3V SPI interface for further extensions
  - Adafruit(tm) UEXT interface for further extensions (SPI shared with 3.3V SPI, includes I2C)
  - Dual UART serial, one RS232, one TTL (shared with UEXT)

## Overview

The system architecture is actually rather simple, as you can see in the following graphics.

![Ulti-PET System Architecture](images/ultipet-arch.png)

The main functionality is "hidden" inside the FPGA. It does:

1. clock generation and management
2. memory mapping
3. video generation.
4. SPI interface and boot
5. DAC DMA

On the CPU side of the FPGA it is actually a rather almost normal 65816 computer, 
with the exception that the bank register (that catches and stores the address lines 
A16-23 from the CPU's data bus) is in the FPGA, and that there is no ROM. The ROM has been
replaced with some code in the FPGA that copies the initial program to the CPU accessible
RAM, taking it from the Flash Boot ROM via SPI. This actually simplifies the design,
as 

1. parallel ROMs are getting harder to come by and
2. they are typically not as fast as is needed, and
3. with the SPI boot they don't occupy valuable CPU address space.

The video generation is done using time-sharing access to the video RAM.
The VGA output is 768x576 at 50Hz, or 768x480 at 60 Hz. So there is a pixel clock of 27MHz.

The system runs at 13.5MHz, so a byte of pixel output (i.e. eight pixels) has four
memory accesses to VRAM. Two of them are reserved for video access, one for fetching the
character data (e.g. at $08xxx in the PET), and the second one to fetch the "character ROM"
data, i.e. the pixel data for a character. This is also stored in VRAM, and is being loaded
there from the Flash Boot ROM by the initial boot loader.

The FPGA reads the character data, stores it to fetch the character pixel data, and streams
that out using its internal video shift register.

For more detailled descriptions of the features and how to use them, pls see the 
[FPGA repository](https://github.com/fachat/upet_fpga) that contains the code for the FPGA,
as described in the next section.

## I/O included in the Ulti-PET

The Ulti-PET not just has the Ultra-CPU, but also loads of I/O on the one board:

### Standard PET IO

The Ulti-PET includes the standard PET I/O features using the VIA and two PIAs, with some extensions. So, it has:

- IEEE488, including flat ribbon cable and card edge connector; with the additional capability of working as a device too
- Tape#1 (incl. 9V for proper tape operation), incl. card edge connector
- Tape#2 as TTL, but normally re-used for IEEE488 device handling
- PET keyboard, with the option to use a C64 keyboard
- PET userport, with the Video signal (optionally) replaced with 5V to accomodate user port extensions; incl. card edge connector
- Joystick connectors for the userport, software-switchable between single- or dual mode (compatible with 'stupid pet tricks')
- Reset and diag push buttons

### RS232

A Dual UART chip provides two serial interfaces. One interface is a real RS232 with a
DB9 connector. The other one is TTL level interface (shared with UEXT, see below).

### Fast IEC

An IEC interface is provided using a second VIA chip. Using the VIA shift register, fast mode IEC
(like in the C128) is implemented.

### Dual SID and sound mixer

Two SID chips are included for your stereo sound pleasure.

The two SID outputs are mixed with the DAC audio output from the Ultra-CPU base.
Also, the beeper sound is mixed into the overall sound output.

Two audio amps are included to, so speakers can be directly connected to the board
(switched off when the audio jack is used).

### Keyboard shift lock and reset

When the shift-lock key in the replacement keyboard is defined on a specific position in 
the keyboard matrix, the key works like a shift-lock key.

In addition, pressing the shift-lock key longer (a few seconds), resets the machine 
(as long as the keyboard scanning still works).

### CS/A expansion board

The board has a full width CS/A slot for a single expansion card, plus a short CS/A board for
the accompanying [CS/A Ultrabus]() Ultra-Bus expansion board. This allows using (compatible)
Apple-II, RC2014, and C64 cartridges with the Ulti-PET.

NOTE: In R1.2 and previous, the _short_ expansion bus had the wrong pinout - it needs DIN41612 reverse pinout! Therefore a "reverser adapter" needs to be used. This will be fixed in 1.3 forward.

### UEXT, and SPI-10 connectors

The board provides three standard I/O ports:

1. UEXT - this includes serial (see above), I2C, and SPI, all in 3.3V
2. SPI-10-3.3V - a 3.3V SPI interface for external modules (shared with UEXT)
3. SPI-10-5V - a 5V SPI interface provided by the second VIA, with selectable clock mode.

For the SPI-10 connectors see [here](http://forum.6502.org/viewtopic.php?f=4&t=4264&start=15#p48167).
For the UEXT connector see the Olimex [UEXT page](https://www.olimex.com/Products/Modules/UEXT/)

## Firmware features

Of the modern I/O, currently these are included and supported in the firmware, i.e. in the
BASIC4 models:

- USB keyboard or mouse support, if you select the boot option with a USB keyboard
- SD-Card support using the SD-Card filesystem from the Commander X16 provided by Michael Steil.
- C64 compatible kernal jump table

Note that if you select the boot option while holding the left shift key, unmodified ROMs are loaded instead.

For more details see the [ROM](https://github.com/fachat/upet_roms) repository, that contains
the current version of the firmware and accompanying documentation.
 

## Known Issues

The current revision is 1.2A

This board has a number of issues, esp. in the audio section:

- The 54MHz oscillator has the wrong footprint
- The shortbus expansion port has the wrong pinout
- The LM3900 is using the wrong bias voltage
- The LM386 is using the wrong gain and input voltage divider; also, the volume poti is wrong taper type
- The 9V generator creates quite some buzz on the audio
- The 3.3V SPI clock has too much noise
- The The power LED connector is ... not connected ...


The following sections show the fixes to be applied to the board.

Note, that even with the fixes applied, the following problems remain:

- Due to the SPI clock fixes, SPI is NOT available at the 3.3V SPI and UEXT connectors
- There is a slight noise still on the speakers as long as the 9V generator chip is inserted (9V is used for SID and Tape)
- The SRAM are actually 5V chips that are being used with 3.3V, but still seem to work even under burnin

The issues will be addressed in the next revision

Also, the following fixes are NOT included in the BOM.

### 54Mhz footprint

Remove the middle connections of the oscillator footprint JS1. Solder in the actual BOM part

![Oscillator footprint with removed pads](fixes/oscillator.png)

### Shortbus pinout

Here I managed to use the mirrored pinout due to a misunderstanding I had with the DIN41612 specs.
This can only be fixed with a pinout adapter, which is in the Board/reverseExpansionBoard directory.

![Use of the expansion port bus pinout fix adapter](fixes/expadapter.png)

### Power LED

To fix the power LED connector, connect the outer two pins of the LED connector with a bodge wire

![Fix for the power LED connector](fixes/led.png)

### 3.3V SPI

Fixing the 3.3V SPI clock requires to implement a proper termination on the line, and remove the branching 
that takes place for connecting the 3.3V SPI and UEXT connectors.

The fix is to add 100Ohm resistors to both 3.3V and GND at the 'end' of the clock line under the USB and net connectors.

![SPI clock termination](fixes/spiterm.png)

Also, cut the trace for the SPI clock that leads to the 3.3V SPI and UEXT connectors.

![SPI clock cut](fixes/spicut.png)
This is difficult to see, but it is only a single cut in a single trace.

### Linear Audio

The audio circuit is somewhat revamped to get a reasonably linear output from the DAC and SID to the speakers.
Note that in the following pictures I have in parts only fixed one of both channels.

1. replace R53, R57, R61, and R65 with 220k Ohm resistors. Only solder the joint next to the LM3900, connect the other
ends of these resistors to VCC

![LM3900 bias voltage](fixes/audio1.png)

2. Remove R68 and R69

3. Remove C156 and C157

4. Replace C129 and C136 with 100uF capacitors, with the Minus pole to the speaker connectors

![output stage](fixes/audio2.png)
Note, this shows C157 "removed" by cutting it, and the (larger black) 100uF cap on C136. This fixes one of the channels. For the other channel, C156 needs to be removed too
and  C129 should also be replaced by a 100uF.

5a. Cut the traces from the audio connector to the volume potentiometer, and add 56k Ohm resistors instead

![output poti](fixes/audio3.png)

5b. Instead of adding a 56k, use a 10k, and replace the potentiometer with a 10k linear taper type

### 9V buzz

You can either remove the 9V generator chip, which removes the buzz, but disables the Tape motor, if you don't need it.
The SID replacements usually (if not all?) do not need the 9V anyway.

To reduce the noise considerably, 

1. replace C153 with a 2200uF electrolytic capacitor

![Input bypass cap](fixes/bypass1.png)
The 2200uF is the very big black capacitor can on the edge of the board next to the expansion board (this used to be a smaller red one).

2. Add another 2200uF el. capacitor in parallel to the 100nF cap C74 next to the 9V generator chip U$24

![9V bypass cap](fixes/bypass2.png)


## Revision history

1.2a: Update
- separate 3.3V generation for the bus, USB, and network, more bypass for SD cards
- Power, Shift Lock LED
- Remove 2nd oscillator option
- Digital video out connector for further experimentation
- Fix addressing of 2nd SID
- Replace some 1N4148 with 1N4448 for lower drop voltage, fix joystick diode direction
- Dual voltage divider for TTL beeper to avoid left/right audio crosstalk
- Fixes on the UART

1.1a: Update:
- Fixed userport joystick handling, allow software switching between single- and dual mode
- Include the beeper into the sound mixer
- Move from single UART to dual UART, to provide TTL serial or serial for UEXT
- Add I2C controller for UEXT
- Add UEXT, SPI-10-3.3V, SPI-10-5V connectors
- Add SPI interface with selectable modes on the second VIA (shares shift register with fast serial IEC bus)

1.0a: Initial release

## Building

Here are the subdirectories:

- [Board](Board/) that contains the board schematics and layout

Note that a case for the Ulti-PET is in the works - the case modifications for the Micro-PET do *not* work.

In addition, two other repositories are needed - they are separate as they are shared with other variants of the upet-family:

- [FPGA](https://github.com/fachat/upet_fpga) contains the VHDL code to program the FPGA logic chip used, and describes the configuration options - including the SPI usage, the DAC usage, and the Video features.
- [ROM](https://github.com/fachat/upet_roms) ROM contents to boot

### Board

To have the board built, you can use the gerbers that are stored in the zip file in the Board/production subdirectory.

To populate the board, there is an interactive bom (bill of materials) from KiCad, as well as the KiCad BOM CSV export in the [bom](Board/bom/) folder.

Parts numbers can be found in the [BOM list](Board/bom/cbm_ultipet_v1.2a-bom.xlsx). It has parts numbers for Mouser for all parts (except the SID), 
and for JLCPCB for the SMD parts. You can have JLCPCB assemble most of the SMD parts, and order the rest from Mouser. (Note, to not order parts twice, best
remove the JLPCB assembled parts before you upload to Mouser).

The BOM contains an Ethernet breakout board that is put into the connectors "above" the USB port. As an alternative you could use the 
[Wifi breakout board](https://github.com/fachat/upet_wifi). Note that at this time, this is not tested / programmed yet.

As an audio solution I am planning to use the Visaton K40SQ speakers, if you want to order them as well.

### FPGA

The FPGA is a Xilinx Spartan 6 programmable logic chip. It runs on 3.3V and it is programmed in VHDL.

To build the FPGA content, clone the [FPGA](https://github.com/fachat/upet_fpga) repository, and look for the ShellUPet.bin.
This needs to be programmed into the SPI flash chip containing the configration for the FPGA.

### ROM

To build the boot ROM image, clone the [ROM](https://github.com/fachat/upet_roms) repository, and build the spiimg70m file. This needs to be 
programmed into the SPI flash ROM containing the 65816 boot code.

The ROM image can be built using gcc, xa65, and make. Use your favourite EPROM programmer to burn it into the SPI Flash chip.

The ROM contains images of all required ROM images for BASIC 1, 2, and 4, and corresponding editor ROMs, including
some that have been extended with wedges.

The updated editor ROMs are from [Steve's Editor ROM project](http://www.6502.org/users/sjgray/projects/editrom/index.html) and can handle C64 keyboards, has a DOS wedge included, and resets into the Micro-PET boot menu.
For more details see the description in the ROM repository.

### Case

Currently no custom case is available.

## Gallery (R1.0)

![The boot menu](images/boot.jpg)

![Demo of graphics capabilities](images/demo.jpg)

![Expansion bus](images/ultrabus.jpg)

![First showing the boot menu during the build process](images/boot_during_build.jpg)
 

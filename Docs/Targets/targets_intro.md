# Porting targets to mbed OS

To support your devices and development boards on mbed OS, you need to create one [yotta target](http://yottadocs.mbed.com/tutorial/targets.html) for each of them, which means you can easily end up with hundreds of targets.
Using target inheritance reduces configuration duplications and maintenance effort and allows you to provide first-class support for your devices and development boards. At the same time, enthusiasts and professional developers can create their own custom designs that inherit from existing targets - and therefore reuse the existing common code.

## Introduction

This guide describes conventions and best practices for partners and developers supporting a broad range of targets for mbed OS.
It discusses how to inherit yotta targets properly, starting from a generic target, through single controller targets and all the way down to board-specific configurations. Note that this guide takes a conceptual design approach first, before looking at specific examples.

In the first section of this guide we introduce the concept of target inheritance.
We will then focus on *how* you should structure your target inheritance and describe *what* you should inherit from your targets.
The last section will discuss the implementation details on a real world example.

For help on syntax and implementation of yotta targets, please see the [yotta documentation on targets](http://yottadocs.mbed.com/tutorial/targets.html).

## Target Inheritance

Targets in mbed OS should be modular and reuse common code.
You can achieve this by specialization through inheritance from generic targets:

<!--
****************************************************************
*                          .----------.                        *
*                         | Base: mbed |                       *
*                          '----------'                        *
*                               ^                              *
*            .------------------+-------------------.          *
*           | Controller family: stm32, kinetis, nrf |         *
*            '--------------------------------------'          *
*                               ^                              *
*        .----------------------+----------------------.       *
*       | Controller series: stm32f4, kinetis-k6, nrf51 |      *
*        '---------------------------------------------'       *
*                               ^                              *
*    .--------------------------+-------------------------.    *
*   | Controller chip: stm32f429zi, kinetis-k64f, nrf51822 |   *
*    '----------------------------------------------------'    *
*                               ^                              *
*               .---------------+---------------.              *
*              | Board: discovery, frdm, nrf-dk  |             *
*               '-------------------------------'              *
*                               ^                              *
*  .----------------------------+---------------------------.  *
* | Board-to-Board: Small boards soldered on breakout boards | *
*  '--------------------------------------------------------'  *
****************************************************************
-->
<!-- TODO: Figure out how to make this into an SVG instead of a PNG! -->
<center>
    <img src="/Targets/target_inheritance_concept.png" width="500"/>
</center>

The family and series targets are optional, but highly recommended!
As an mbed OS partner you MUST provide the target for the controller chip, and you MUST inherit your board target from the chip target.
This is a condition for getting approved for mbed.
This ensures a friendly ecosystem, where developers can easily inherit from your controller chip target to create their own board designs.

### Example
Vendor Unicorn produces the Rainbow family of devices which currently has two series:
the powerful and feature-rich Sparkle series using a ARM Cortex-M4F and
the energy-efficient and low-budget Dusty series using a Cortex-M0+.

Since they are both of the same family, the vendor encodes their commonalities, such as identical Flash and RAM start addresses, in a family target which inherits from the mbed base target.
Similarly, the chips in the Sparkle and Dusty series have common features, like processor type, internal clocks,
pin definitions and general mbed-hal capabilities, which are described in the series targets.

The chips targets inherit this information from the series targets and specialize it by adding memory size and
interrupt table information, peripheral counts and capabilities and most importantly, which pins are available
on this chip and how are they connected internally.

Note, that the chip targets only contain information about the chip. Everything that requires external circuitry,
especially external clock sources (crystals, oscillators), is only known to the board target.
The board target also renames the chip pins to the names on the board to make them easy to reference in code, and
provides the information how to program this chip using CMSIS-DAP.

Unicorn's products have convinced Dinodino, who want to use Unicorn's products in a new dinosaur theme park
on a tropical island far away.
So the developer starts evaluating Unicorn's Megaboard and decides to integrate it into the first prototype, by
plugging it onto a custom base PCB.
The Dinodino engineers let the Roarboard target inherit from the Megaboard target instead of copying and modifying it.

<!--      Whooshboard ->   \./  (Dinodino Dinobat)
 (Dinodino     __
  Dinosaur)   / _)  <- Roarboard
     _.----._/ /
=|==/         /=|===|===|===|===|  <- Forceboard (Dinodino Dinofield)
 __/ (  | (  |
/__.-'|_|--|_|         .  <- Squeakboard (Dinodino Dinomouse)
-->

Since the prototype was a success (they only lost _one_ dinosaur trainer this time), Dinodino gives the green light
to develop three more targets using Unicorn's chips on custom PCBs.
This is easy, since Unicorn has provided targets for its chips as well, so Dinodino's custom board targets can reuse
them as well.

Some time later, Unicorn discovers a bug in one of their target descriptions and updates and publishes a patched
version of it.
Dinodino, who inherited from Unicorn's targets now, benefit from this bug fix completely transparently.

<!--
****************************************************************
*                             .----.                           *
*                            | mbed |                          *
*                             '----'                           *
*                               ^                              *
*                       .-------+-------.                      *
*                      | Rainbow  Family |                     *
*                       '---------------'                      *
*   Vendor: Unicorn             ^                              *
*           .-------------------+------------------.           *
*  .-------+------.                          .------+-------.  *
* | Sparkle Series |                        |  Dusty Series  | *
*  '--------------'                          '--------------'  *
*        ^                                             ^       *
*        +------------.                   .------------+       *
*  .-----+----.   .----+----.       .----+----.   .----+----.  *
* | Sp128 Chip | | Sp64 Chip |     | Du32 Chip | | Du16 Chip | *
*  '----------'   '---------'       '---------'   '---------'  *
*        ^             ^               ^               ^       *
*        |             +------.        |       .-------+       *
*  .-----+----.   .----+----.  |       |      |  .-----+----.  *
* | Megaboard  | | Evalboard | |       |      | | Basicboard | *
*  '----------'   '---------'  |       |      |  '----------'  *
*  -   - ^ -    -   -   -    - | -   - | -  - | -   -   -   -  *
*        | Developer: Dinodino |       |      |                *
*        |              .-----'        |       '-----.         *
*  .-----+---.   .-----+----.   .------+----.   .-----+-----.  *
* | Roarboard | | Forceboard | | Whooshboard | | Squeakboard | *
*  '---------'   '----------'   '-----------'   '-----------'  *
****************************************************************
-->
<!-- TODO: Figure out how to make this into an SVG instead of a PNG! -->
<center>
    <img src="/Targets/target_inheritance_example.png" width="500"/>
</center>

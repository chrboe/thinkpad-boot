# Preface

TODO

# Chapter 1: Pressing the Power Button

Schematic: https://forum.thinkpads.com/viewtopic.php?t=121522

When the Power Button (PWRSW in the schematic) is pressed, it is eventually
routed to pin `VCI_IN3` on the MEC1619-GP chip

## 1.1 MEC1619

Datasheet: http://ww1.microchip.com/downloads/en/DeviceDoc/00002339A.pdf (There's
no datasheet for the MEC1619 broadly available, so this is for the similar MEC1618
processor. For our purposes the information we care about should match between
the two models though)

The MEC1619 is a standalone 32-bit ARC CPU embedded into the ThinkPad
([source](https://github.com/hamishcoleman/thinkpad-ec/blob/master/docs/chips.txt)).
It is used to handle power management (which we'll focus on) as well as some
other tasks.

`VCI` stands for `VBAT-Powered Controller Interface`. It's a subsystem within
the MEC1619 which basically binds some interrupts and wake-up events to external
pins. In our case (`VCI_IN3`) a wake-up event and an interrupt is triggered.
Looking at the `GIRQ23 Source Register` table in the datasheet, we notice that
there's a bit for the `VCI_IN3` pin. The truth table for the interrupt triggers
reveals that this bit triggers `IRQ23`, and gets set when `VCI_IN3` is pulled
low (which it is when we press the power button).

This brings us to the EC Interrupt Aggregator. It's a subsystem that basically
just handles interrupts (things such as calling the correct IRQ handler, etc).
Looking at the `EC Interrupt Structure` table reveals that the handler for
`IRQ23` is located at offset `0xB8` in the ROM. The way that the offset table
works is that there's basically just a bunch of jump instructions from `0x00`
to `0xF8` in the ROM (however `0xC0` through `0xF8` appear to be unused). Whenever
an IRQ is triggered, the CPU jumps to the corresponding offset, which in turn
jumps to the actual IRQ handler.

### 1.1.1 Disassembling the ROM

This is the fun part. I got a ROM image for the MEC1619 Chip from a
[sketchy looking russian forum](https://ascnb1.ru/forma1/viewtopic.php?f=70&t=109179)
and downloaded it. It seems to have been extracted from a Thinkpad X230, but
(like with the MEC1619), the differences between the models should be minimal.

So, as we discovered earlier, the interrupt handler of our interest resides at
offset `0xB8` in the ROM. Using radare2 to disassemble the ROM gives us the
instruction at that offset:

```shell
$ r2 -a arc rom.bin
[0x00000000]> pd 1 @ 0xb8
,=< 0x000000b8      2020800f0000.  j 0x0000278c
```

The important part here is `j 0x0000278c`. `j` is the mnemonic for the jump
instruction, which unconditionally jumps to the given address (in this case
`0x278c`.

Now let's look at the offset it's jumping to:

```shell
[0x00000000]> s 0x278c     # seek to the offset
[0x0000278c]> pd 43        # disassemble 43 instructions
       0x0000278c      bc1c08b0       st.a r0, [sp, -68]          ; irq23 handler
       0x00002790      0fd8           mov_s r0, 15
       0x00002792      041cc037       st blink, [sp, 4]
       0x00002796      42c1           st_s r1, [sp, 8]
       0x00002798      43c2           st_s r2, [sp, 12]
       0x0000279a      44c3           st_s r3, [sp, 16]
       0x0000279c      141c0031       st r4, [sp, 20]
       0x000027a0      181c4031       st r5, [sp, 24]
       0x000027a4      1c1c8031       st r6, [sp, 28]
       0x000027a8      201cc031       st r7, [sp, 32]
       0x000027ac      241c0032       st r8, [sp, 36]
       0x000027b0      281c4032       st r9, [sp, 40]
       0x000027b4      2c1c8032       st r10, [sp, 44]
       0x000027b8      301cc032       st r11, [sp, 48]
       0x000027bc      4dc4           st_s r12, [sp, 52]
       0x000027be      381c003f       st lp_count, [sp, 56]
       0x000027c2      6a248010       lr r12, [0x2]
       0x000027c6      4fc4           st_s r12, [sp, 60]
       0x000027c8      6a24c010       lr r12, [0x3]
       0x000027cc      50c4           st_s r12, [sp, 64]
       0x000027ce      fe0a4007       bl 0x000112c8
       0x000027d2      04141f30       ld blink, [sp, 4]
       0x000027d6      0ec4           ld_s r12, [sp, 56]
       0x000027d8      9f74           mov_s lp_count, r12
       0x000027da      0fc4           ld_s r12, [sp, 60]
       0x000027dc      6b248010       sr r12, [0x2]
       0x000027e0      10c4           ld_s r12, [sp, 64]
       0x000027e2      6b24c010       sr r12, [0x3]
       0x000027e6      02c1           ld_s r1, [sp, 8]
       0x000027e8      03c2           ld_s r2, [sp, 12]
       0x000027ea      04c3           ld_s r3, [sp, 16]
       0x000027ec      14140430       ld r4, [sp, 20]
       0x000027f0      18140530       ld r5, [sp, 24]
       0x000027f4      1c140630       ld r6, [sp, 28]
       0x000027f8      20140730       ld r7, [sp, 32]
       0x000027fc      24140830       ld r8, [sp, 36]
       0x00002800      28140930       ld r9, [sp, 40]
       0x00002804      2c140a30       ld r10, [sp, 44]
       0x00002808      30140b30       ld r11, [sp, 48]
       0x0000280c      0dc4           ld_s r12, [sp, 52]
       0x0000280e      44140034       ld.ab r0, [sp, 68]
       0x00002812      20204087       j.f [ilink1]
       0x00002816      2020c007       j [blink]
```

The function seems to be 43 instructions long (indicated by the jump to `[blink]`,
which essentially returns from the function), and it looks quite intimidating
at first.

(TODO: dive into how exaclty the function at `0x000112c8` works)

It's actually a pretty simple function (as simple as a disassembled function
can be); it ultimately just writes a 1 to GPIO217 (Bit 15 at SPB Offset `0x290`,
see datasheet). If we cross-reference this with our schematic, we realize that
this is wired to the `PWRSW_EC` (where `EC` stands for "Embedded Controller")
terminal, which makes a whole lot of sense. Sweet!

## 1.2 Chipset

Datasheet: https://www.intel.com/content/dam/www/public/us/en/documents/datasheets/200-series-chipset-pch-datasheet-vol-1.pdf

If we go back to the schematic and follow the `PWRSW_EC` line that the EC spews
out, we end up at the `PWRBTN#` pin on a mysterious component called
`QP8D-SFF-GP-NF`. This is actually the motherboard's chipset, an Intel C216
Series/Panther Point model in our case. It performs the next step in booting our
laptop, so let's find out what it does.

The part we're mainly interested in is the PMC (Power Management Controller),
which is covered in chapter 27 of the datasheet. It's also important to know
that PCH stands for "Platform Controller Hub".

## 1.2.1 States

The PMC actually has a state machine inside. The current state decides what is
done when the power button is pressed. Let's briefly go over the states we care
about, since that's very useful to know for what's about to come:

| State    | Description                                                                     |
|----------|---------------------------------------------------------------------------------|
| G0/S0    | Processor operating                                                             |
| G2/S5    | Soft off: All power is shut off except for the logic required to restart        |
| G3       | Mechanical off: All power is actually shut off (e.g. when removing the battery) |

The states beginning in "S" are ACPI states, which we won't be too bothered with
here.

We'll just arbitrarily assume that we're in state G2 when the power button is
pressed.

A few lines below we find a table describing the state transitions of the state
machine. The transition we care about, G2 to G0, happens on "Any Enabled Wake
Event". It just so happens that the pin we discovered earlier, `PWRBTN#`, raises
such a wake event. What a coincidence!

Anyways, we can still verify this using a table yet another few pages down, which
describes "Transitions Due to Power Button". It states that when we're in state
G2 and `PWRBTN#` goes low (inverted logic) we'll transition to state S0.

Unfortunately, thanks to Intel's very secretive nature on this matter, I can't
actually seem to obtain a firmware image for disassembling, so I can't go into
that absurd level of detail. We still have a "high-level" understanding of the
state machine though, so I guess that's sufficient for now.

## 1.2.2 In the S0 state

This part is a little unclear from the documentation, but I think this is what
happens: When the PCH transitions to state S0, all the power rails are brought
up and the CPU is signalled a `CPUPWRGD` (CPU Power Good) and a `PM_SYNC` signal
(not neccessarily in that order). The CPU Power Good signal tells the CPU that
its main power source is up, and `PM_SYNC` synchronizes the power states between
the PCH and the CPU.

Along with the `CPUPWRGD` and `PM_SYNC` signals a similar `DRAMPWRG` is also
issued to the CPU, signalling that the DRAM power is available and stable.

I guess somewhere in there a CPU reset (`CPURST` in the schematic) also happens,
but I'm not sure how. I don't think it comes from `PLTRST_FAR`, since that seems
PCI related, so I guess this is a TODO.

With all the power management stuff out of the way, this means that our CPU is
now actually up and running. Hurray!

# Chapter 2: BIOS

Now that our CPU is technically running (it doesn't do much, but still), we can
look into how the software that will run on it is actually loaded. The first
step of this journey is the BIOS.

In the past, Intel CPUs used to start at physical address `0x000FFFF0`
([source](https://en.wikipedia.org/wiki/BIOS#System_startup)). Nowadays, things
are a little different, and there's a memory region called "High BIOS". This
memory region lies "just under the 4 GB boundary" according to the
[Processor Datasheet](https://www.versalogic.com/support/Downloads/Copperhead/3rd-gen-core-family-mobile-vol-2-datasheet.pdf).
In reality this address is 4 GB minus 16 bytes, or `0xFFFFFFF0`. The datasheet
also specifies that the CPU will start execution from High BIOS, so that physical
address is where it will start.

In terms of storage for the BIOS and firmware, the X1 carbon has two SPI flash
chips, MX25L3273E (4MB) and MX25L6406E (8MB). The two chips are internally
concatenated into a single 12MB "virtual" SPI flash chip, and the BIOS resides
there (among other things).

The PCH already "knows" that it has to look for `0xFFFFFFF0` on the SPI flash
chip, because of the "Boot BIOS Strap Bit BSS". It's evaluated on startup and
sets a "Destination" for the BIOS. The default (0) instructs the Chipset to
look for it on an SPI flash chip.

So now that our CPU is running and executing our BIOS code, we are getting
closer and closer to our login prompt.

## 2.1 POST


To be continued...

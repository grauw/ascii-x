# ASCII16-X specification

## Feature set

  * FlashROM memory with commands for per-sector erasing and programming.
  * Extended addressable capacity up to 64 MB (design provided for 8 MB).
  * Two 16K mapper pages, mirrored to the full address range.
  * Two bank selection registers accessible in all pages.
  * Mostly backwards compatible with ASCII16.

## Mapper usage

### Pages

The ASCII16-X mapper has two 16K pages available at 4000H-7FFFH (page 1) and
8000H-BFFFH (page 2). For additional flexibility, their contents are also
mirrored in the other two quadrants of the address space.

  | Page | Page address  | Mirror †      |
  |------|---------------|---------------|
  | 1    | 4000H - 7FFFH | C000H - FFFFH |
  | 2    | 8000H - BFFFH | 0000H - 3FFFH |

† Extension to the ASCII16 mapper.

### Bank registers

To select a bank in a page, write the bank number to the corresponding bank
select register.

The bank register which controls the bank selection for each page is available
at address 6000H-6FFFH (page 1) and 7000H-7FFFH (page 2). For additional
flexibility, the registers are also mirrored in the other quadrants of the
address space.

  | Page | Bank register | Mirror †      | Mirror †      | Mirror †      |
  |------|---------------|---------------|---------------|---------------|
  | 1    | 6000H - 6FFFH | A000H - AFFFH | E000H - EFFFH | 2000H - 2FFFH |
  | 2    | 7000H - 7FFFH | B000H - BFFFH | F000H - FFFFH | 3000H - 3FFFH |

† Extension to the ASCII16 mapper.

### Bank numbers

To support ROM sizes greater than 4 MB, the bank register takes a 12-bit bank
number by supplementing the 8 data bits with 4 address bits. The bank number’s
lower eight bits are written in data bits D0-D7 and the high bits in
address bits A8-A11.

  | Address bits        | Data bits |
  |---------------------|-----------|
  | XX1P MMMM XXXX XXXX | LLLL LLLL |

  * `P`: Page (0: page 1, 1: page 2)
  * `L`: Bank number LSB (bits 0-7)
  * `M`: Bank number MSB (bits 8-11)
  * `X`: Don’t care

The XL 8 MB cartridge has 512 banks and the bank number is thus 9 bits wide.
The unused A9-A11 are ignored, the bank number will wrap around. It is
recommended to leave them at zero.

### Initial state

Initial bank numbers after power on and reset are 0.

However, the MSX BIOS selects a different bank in page 2 on boot-up, due to its
slot expander detection code writing to mirrored bank select registers when the
mapper is in a primary slot.

Therefore it is recommended to boot from page 1 (4000H), and to explicitly
initialise the bank registers before accessing page 2 (8000H).

### Usage example

One approach is to treat the mapper as multiple 4 MB blocks, where the address
determines which block gets selected. One could store e.g. game code and level
data in the first block, and music and demo graphics data in the second block.

    ld a,47H
    ld (6000H),a  ; select bank 047H in page 1

    ld a,47H
    ld (6100H),a  ; select bank 147H in page 1

    ld a,47H
    ld (7000H),a  ; select bank 047H in page 2

    ld a,47H
    ld (7100H),a  ; select bank 147H in page 2

Another approach is to treat the mapper as one unit and store the bank numbers
as 16 bits starting at 6000H or 7000H.

    ld hl,6047H
    ld (hl),l     ; select bank 047H in page 1

    ld hl,6147H
    ld (hl),l     ; select bank 147H in page 1

    ld hl,7047H
    ld (hl),l     ; select bank 047H in page 2

    ld hl,7147H
    ld (hl),l     ; select bank 147H in page 2

## FlashROM usage

ASCII16-X cartridges have FlashROM memory, which can be erased and reprogrammed
by issueing commands. This enables software to store save games, high scores,
user created levels, etc.

The FlashROM command set supported varies depending on the FlashROM chip in use.
The Infineon S29GL064S used by the XL 8 MB cartridge has a very large command
set, however it is recommended to limit your use of flash commands for
compatibility with a wide range of FlashROM chips. The minimum commands that can
be assumed to be supported are:

  * Autoselect
  * CFI query
  * Erase chip
  * Erase sector
  * Program byte

Below is a description of the commands to erase and program the FlashROM memory.
For more details about the FlashROM command set, refer to its respective
datasheet.

**Note:** It is important to implement the polling after issueing commands to
the FlashROM. Emulators may not implement the timings correctly, or at all.
Make sure to test on real hardware.

### Erase sector command

The erase sector command clears the contents of a block of memory to FFH.
FlashROM memory must be erased before it can be programmed. The memory is
divided into sectors, the first eight being 8K in size and the remainder 64K.
Erasing clears the contents of the entire sector, meaning that the erased memory
can span multiple mapper banks.

To erase a sector, select a bank of the sector you want to erase using the
mapper registers, and then write the erase sector command sequence to the bank.

  | Erase Sector |   1   |   2   |   3   |   4   |   5   |   6   |
  |--------------|-------|-------|-------|-------|-------|-------|
  | Address      | XAAAH | X555H | XAAAH | XAAAH | X555H | XAAAH |
  | Data         |  AAH  |  55H  |  80H  |  AAH  |  55H  |  30H  |

The erase can take up to 1 second to complete, though it typically takes around
300 ms. It varies by chip and by age. While the erasing is in progress, any
reads from the entire memory give a non-FFH value. Wait for the erasing to
complete by polling any byte in the sector until it becomes FFH, which indicates
that the erasing is complete.

**Note:** Because the erase status is given throughout the entire FlashROM
memory, the erase routine can not itself be in the FlashROM and must be placed
in RAM.

**Note:** The number of erase cycles a FlashROM is guaranteed to support is
limited but very high (typically 100,000 cycles). So you under normal usage
conditions you generally don’t need to worry about how often you erase.

### Program byte command

The program byte command writes a value to memory. FlashROM memory must be
erased before it can be programmed.

To program a byte, select the bank of the byte want to write using the mapper
registers, and then write the program byte command sequence to the bank, ending
with writing the desired value to its address.

  | Program Byte |   1   |   2   |   3   |     4     |
  |--------------|-------|-------|-------|-----------|
  | Address      | XAAAH | X555H | XAAAH | (address) |
  | Data         |  AAH  |  55H  |  A0H  |  (value)  |

The programming can take up to 1.2 ms to complete, though it is typically takes
around 0.1 ms. It varies by chip and by age. While the programming is in
progress, any reads from the entire memory give a value different from the value
being written. Wait for the programming to be complete by polling the address
you’ve written to until it becomes the correct value, which indicates
that the programming is complete.

**Note:** Because the programming status is given throughout the entire FlashROM
memory, the programming routine can not itself be in the FlashROM and must be
placed in RAM.

**Note:** Be mindful of the mapper registers area in pages 1 and 2. When writing
to flash you need to avoid writing to 6000H-6FFFH and B000H-BFFFH, since doing
so would simultaneously change the bank selection, with unexpected results. You
can write that data via A000H-AFFFH and 7000H-7FFFH instead, or alternatively,
avoid writing data to the bank selection region altogether, leaving it empty.

## Emulator support

OpenMSX supports the ASCII16-X mapper type as of version 20.0.

Note that at the time of writing openMSX’s FlashROM implementation is incomplete:

  * The timing still needs improvements to better match real hardware. When
    using FlashROM functions, be sure to correctly poll for command completion
    and test on real hardware.
  * When the contents of the FlashROM are modified the entire ROM is persisted.
    When providing a new ROM version, make sure to first delete the persistent
    data in the openMSX user directory.

### Signature

To allow auto-detection of the ASCII16-X mapper type, it is recommended to put
the string `ASCII16X` at ROM offset 00010H (at 4010H in the MSX address space).
This will ensure openMSX selects the correct ROM type automatically.

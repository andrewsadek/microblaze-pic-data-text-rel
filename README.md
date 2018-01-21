# [Microblaze] New Feature: PIC data text relative

This branch is regarding a new implemented feature in GCC Microblaze that allows Position Independent Code to run using Data Text Relative addressing instead of using Global Offset Table.

This customization was a part of Master Thesis Research with Title 'Dynamic Code Loading to a bare metal embedded target'.
Its aim was to make 'PIC' more efficient and flexible as elf size excess/ performance overhead were noticed when using GOT due to the indirect addressing.
The change was tested with the dhrystone benchmark on real Hardware (Xilinx FPGA Spartan 6) and the test report went successfully for all optimization levels.

Indeed, Microblaze does not support PC-relative addressing in Hardware like ARM. 
The idea was to store the start address of current text section in 'r20' instead of GOT, in the function prologue. Correspondingly, data references will be an offset from the original reference value to the start of text thus being added to the 'r20' base register will resolve the actual address.

Henceforth, 2 new relocations have been created:
- 'R_MICROBLAZE_TEXTPCREL_64': resolves offset of current PC to start of text in order to set r20
- 'R_MICROBLAZE_TEXTREL_64': resolves offset of mentioned data reference to start of text
Accordingly, both assembler and linker (binutils) have been adapted.




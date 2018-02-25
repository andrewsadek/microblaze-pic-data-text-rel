# [Microblaze] New Feature: PIC data text relative

This branch is regarding a new implemented feature in GCC Microblaze that allows Position Independent Code to run using Data Text Relative addressing instead of using Global Offset Table.

This customization was a part of Master Thesis Research with Title 'Dynamic Code Loading to a bare metal embedded target'.
Its aim was to make 'PIC' more efficient and flexible as elf size excess/ performance overhead were noticed when using GOT due to the indirect addressing.
The change was tested with the dhrystone benchmark on a real Hardware (Xilinx FPGA Spartan 6) and the test report went successfully for all optimization levels.

Indeed, Microblaze does not support PC-relative addressing in Hardware like ARM. 
The idea was to store the start address of current text section in 'r20' instead of GOT, in the function prologue. Correspondingly, data references will be an offset from the original reference value to the start of text thus being added to the 'r20' base register will resolve the actual address.

Henceforth, 2 new relocations have been created:
- 'R_MICROBLAZE_TEXTPCREL_64': resolves offset of current PC to start of text in order to set r20
- 'R_MICROBLAZE_TEXTREL_64': resolves offset of mentioned data reference to start of text

Accordingly, both assembler and linker (binutils) have been adapted.

Code Example
-------------
<pre>
<code>long myVar;
long myArray[] = {0x1, 0x2, 0x3};
extern void externalFunction(void);

void myFunc(unsigned int index) {
	myVar = 100;
	myArray[index%3] = 0;
	externalFunction();
}
</code></pre>

Assembly with -O1 -fPIE/-fPIC -mpic-data-text-rel
--------------------------------------------------
<pre><code>00000000 &ltmyFunc&gt:
   0:   3021ffe0        addik   r1, r1, -32
   4:   f9e10000        swi     r15, r1, 0
   8:   fa81001c        swi     r20, r1, 28
   c:   96808000        mfs     r20, rpc
  10:   b0000000        imm     0
                        10: R_MICROBLAZE_TEXTPCREL_64  *ABS*+0x8
  14:   32940000        addik   r20, r20, 0
  18:   30600064        addik   r3, r0, 100
  1c:   b0000000        imm     0
                        1c: R_MICROBLAZE_TEXTREL_64     myVar
  20:   f8740000        swi     r3, r20, 0
  24:   30600003        addik   r3, r0, 3
  28:   48632802        idivu   r3, r3, r5
  2c:   60630003        muli    r3, r3, 3
  30:   14a32800        rsubk   r5, r3, r5
  34:   64a50402        bslli   r5, r5, 2
  38:   b0000000        imm     0
                        38: R_MICROBLAZE_TEXTREL_64     myArray
  3c:   30740000        addik   r3, r20, 0
  40:   b0000000        imm     0
                        40: R_MICROBLAZE_64_PCREL       externalFunction
  44:   b9f40000        brlid   r15, 0
  48:   d8051800        sw      r0, r5, r3
  4c:   e9e10000        lwi     r15, r1, 0
  50:   ea81001c        lwi     r20, r1, 28
  54:   b60f0008        rtsd    r15, 8
  58:   30210020        addik   r1, r1, 32
</code> </pre>

As shown in code above, r20 shall hold the start address of current text section. Then, for each data reference 'R_MICROBLAZE_TEXTREL_64' shall resolve the difference between the current address and the start of text address.

All branches are relative as usual.

The diagram below describe the time perfromance analogy between -fPIE (PIC) and -mpic-data-text-rel (Enh-PIC), when running dhrystone on a Xilinx FPGA Spartan 6 (100 MHz).

![alt text](https://github.com/andrewsadek/microblaze-pic-data-text-rel/blob/pic_data_text_rel/dhrystone_time_results.png)

Summary of Added Options
=========================

GCC Microblaze
---------------
1) -mpic-data-text-rel: This allows referencing data by offset from the start of text instead of GOT. Shall be invoked with -fPIC/-fPIE.

Linker (ld)
------------
2) --adjust-insn-abs-refs: Any instruction refering to an absolute symbol reference coming from an external .elf (e.g. by invoking -R"example.elf"), will be excluded from the data-text relative approach and adjusted as follows (relative branch -> absolute ; base register r20 -> r0). [Target Dependent]
3) --disable-multiple-abs-defs: Generate error in case of multiple symbol definition in external .elf (e.g. by invoking -R"example.elf") and the current .elf

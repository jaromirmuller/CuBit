Introduction
------------
CuBitOS is a multi-processor, 64-bit, formally-verified, 
general-purpose operating system, currently for the x86-64 architecture.

Rationale
---------
The arms race today between developers, sysadmins and bad guys is at a fever
pitch. Most of the infrastructure we depend on is written in C. I know
there are a lot of talented C developers out there, but let's face it, it's
unsafe, and it's challenging even for the most experienced devs to write safe C
code. The language specification makes it difficult to statically check,
though there are some system subsets like MISRA-C or FRAMA-C that are more
amenable to static analysis and formal verification, respectively.

Advanced, persistent threats (APTs) are constantly finding vulnerabilities and
developing exploits. It's going to piss a lot of people off, but let's be
honest - C is a dead end for secure programs. It's had a good run, we've got lots of
good memories, but I wouldn't bet my life (or livelihood) on anything written
in C. Lots of people feel differently, and that's OK.

Personally, I think formal verification is the only way the good guys have any
chance, frankly. We can mathematically prove that a program satisfies its
specifications and is free from bad behavior. Today.

So, what tools exist to write formally-verified, bare-metal, production 
software today? The landscape is limited. Frama-C adds the ACSL (specification
language) that allows contracts in specially-formatted C comments.

The only other one I could find was SPARK / Ada.

Yes, *that* Ada.

It's not that bad! I was shocked how nice it was, once I started porting CuBitOS
over to it.

Why not Rust? I'm bullish on Rust, personally. Memory-safety issues are the number
one cause of CVEs, and Rust presents a great way to avoid these. However, Rust does
not (yet) have any way of mechanically proving functional correctness. I'm looking
forward to what the Rust Formal Methods working group comes up with though. From
what I can tell, it's not a priority yet. Ada's rigid, rich type system, combined
with SPARK, really gives the developer a lot of power for guaranteeing correctness.
Rust can do a lot of checks at runtime when software is built in Debug mode, so
with thorough testing, the developer can feel pretty good about functional
correctness.

SPARK lets us formally prove the absence of these types of bugs at
compile-time.

Why not ``<insert language here>?``

The original version of CuBitOS (called Fortress) was written in D, 
which is still near and dear to me. For application development, it's great,
and it has a lot of features inspired by Ada, such as contracts. Without formal
verification and compile-time checking of contracts, I had to look elsewhere.

Type-dependent languages like Coq, Idris will also have a part to play, 
but are still largely academic in nature. All the ones I've seen are purely 
functional and depend on a runtime, which makes them ill-suited to operating 
system development outside of an academic environment, in my opinion. 
It might be possible to extract bare-metal
code from Coq, but honestly, writing code in it is not the most fun thing in
the world. It should be noted that the SPARK prover can use Coq proofs in the
event that the automatic prover is unable to prove something.

I think most, if not all, of the functionality provided by dependent types can 
already be satisfied by Ada contracts, formally-proven at compile-time by
SPARK. Having said that, I think this is an exciting area of research and there
is a lot of promise for these languages going forward. I'd like to see more of
a pragmatic emphasis on using these languages for assured systems development.
From what I can tell, it's still fairly theoretical and high-level.

So, does this mean CuBitOS is a perfect system with no bugs?

HECK NO! It's still a product of human hands, with an incomplete and possibly
incorrect interpretation of the underlying hardware, which itself may have
bugs. Not everything in CuBitOS is formally-proven, either. The SPARK
subset of Ada doesn't allow certain things that an operating system just has
to do sometimes, so while the interface is checked for these functions and
procedures, the functionality isn't. Additionally, the contracts that are in 
place now don't constitute a complete specification of CuBitOS' desired 
functionality, either.

Contracts and formal verification are not and probably never will be a 
substitute for a thorough test suite, but where a test suite shows the
*presence* of bugs, formal verification can show the *absence* of them.

Contributor Notes
-----------------
Ada is case-insensitive (which is kind of nice,
honestly), but it makes all exported symbols lower-case, so anything in .asm
files that references that symbol needs to use the lower-cased version, OR
specify the External_Name for the symbol in an aspect.

Enumerated types containing "holes" cause GNAT to insert run-time checks for
valid values when assigning these. I couldn't find a pragma to disable these
checks. Eventually we'll write exception handler functions that can either
perform the checks or cause a kernel panic, but for now, fill the holes with
obvious values like BAD_1, BAD_2, etc.

Please ensure proofs are run (using `make prove`) before submitting Pull
Requests. Be very cautious about Pragma Annotate to disable warnings.
`gnatprove` will give some spurious warnings like `statement has no effect`
or `unused assignment` that are clearly incorrect, but please double-check
these.

Variant records appear to insert some extra cruft in the underlying record,
even if pragma Pack is used. Do not use these as "overlays" on top of existing
memory, for example the ACPI tables. You'll have alignment issues with some of
the fields and end up with junk data. The same is true even for non-variant
records that use discriminants.

Some things that might be confusing:
------------------------------------

The MAX_PHYS_ constants
=======================
* MAX_PHYS_ALLOC - set in config.ads, this is the max physical memory that CuBit
  supports. There only limit here is practical, since the boot mem allocator is
  set up to create bitmaps for all of this memory (128 GiB), which takes up a fair
  amount of space. (We may consider just setting aside a small linear-mapped heap 
  section for kernel use and build a better allocator so this limit is not a factor
  anymore.)

* MAX_PHYS_ADDRESS - the maximum _theoretical_ physical memory that our hardware
  supports (if we use linear mapping). In the case of x86-64, we have canonical 
  48-bit virtual addresses, so if we want all of physical memory to be mapped in
  virtual memory, we're left with about 280TB, which we need to split to use in
  the higher-half, so 128TiB. Note that Intel is looking at 56-bit addressing with
  a five-level page table, so this number may change soon for x86-64.

* MAX_PHYS_ADDRESSABLE - the top physical memory address _on our system_ that we
  detect at runtime. We linear-map memory areas up to this point
  (starting at 0xFFFF_8000_0000_0000).

* MAX_PHYS_USABLE - the top usable RAM address _on our system_. There may be
  holes inside this, so we can't blindly allocate memory from beneath this
  limit, but a physical allocator would be expected to handle addresses up to
  this point.

"Bootstrap" vs "Kernel"
=======================
Some of the structures used, like the GDT and Page Tables, have to be set up in
`boot.asm` before switching to long mode. These are referred to as the
"bootstrap" GDT and Page Tables, respectively. There's also the "bootstrap"
physical memory allocator (currently the only allocator). Later on, we re-map
the entirety of physical memory into the higher-half of the kernel along with
a new GDT in `segment.adb`. These are called the "kernel page tables" and
"kernel GDT."

The bootstrap page tables identity-map the first 1G of memory, and then map
it again into the higher-half, starting at 0xFFFF_8000_0000_0000. So the P2V
and V2P (physical-to-virtual, virtual-to-physical) work by simple addition and
subtraction because you can access the physical memory at, for example, 
0x0000_0000_DEAD_BEEF at the virtual addresses 0x0000_0000_DEAD_BEEF and also
0xFFFF_8000_DEAD_BEEF.

When we switch to the kernel map, we can no longer address memory using direct
physical addresses. Instead, a specific physical address must be accessed using
the linear-mapped address at 0xFFFF_8000_0000_0000. We linear-map all physical
memory in 0xFFFF_8000_0000_0000 to 0xFFFF_8FFF_FFFF_FFFF.

Cubit Memory Map
================
Note that due to the x86-64 ABI, the kernel must be linked in the top 2GiB of
memory when using mcmodel=kernel. Therefore, our page tables also
need a mapping for 0xFFFF_FFFF_8XXX_XXXX -> 0x0000_0000_0XXX_XXXX.

Can I Help?
-----------
Yes! I've tried to make the code very friendly to those unfamiliar with
CuBitOS, Ada or SPARK. Please try it out on a VM and let me know what you
think. Negative feedback and constructive criticism are useful too.

It's heavily-documented, and contributors are welcome!

Things that I could really use help with:
=========================================
* Adding more contracts and SPARK verification conditions
* A sensible Windowing/Graphics framework.
* Porting apps or writing new ones
* Drivers
* Testing, especially developing an automated test framework
* A proper CI/CD pipeline. As the time to prove CubitOS grows, being able to have
  this done automatically or overnight would be great.
* Articles, documentation.

Other avenues for exploration
=============================
CuBitOS may be a good starting point for academic research. SPARK has an
escape hatch to perform proofs in Coq, so for the more mathematically-minded
contributors that want a good challenge, see what you can prove about CuBitOS!

SPARK uses the Why3 framework and Z3 SMT solver. Good, fast SMT solvers
are always an area of interest, and CuBitOS might be a good way to benchmark
them for other real-world uses.

Ideas:
======
* GPU-accelerated SMT solving
* FPGA-accelerated SMT solver
* SMT proofs as part of a CI/CD pipeline

Coding Conventions
------------------
Ada keywords (type, for, loop, etc.)            - all lowercase
Types (MultibootStruct, PageTableEntry, etc.)   - Capitalized CamelCase
Package names                                   - Capitalized CamelCase
Variables, functions (kmain, toHexString, etc.) - uncapitalized camelCase
Constants (LT_GRAY, KERN_BASE, etc.)            - ALL CAPS
Filenames                                       - all lower or snake_case

NOTE: if interfacing to an external component, say Multiboot, then
variable names should use the same convention as that component's API.
For instance, the Multiboot info struct has fields like mem_lower, etc.

We'll use those names verbatim here for ease of documentation reference.
This is not a hard-and-fast rule. If the API names are overly-terse,
confusing, use a strange Hungarian notation, or are otherwise flat-out
dumb, then feel free to rename them in the Ada code.

Avoid overly-terse abbreviations. Common terms like "kern" for "kernel",
or "mem" for memory, are OK if there's no ambiguity. "VM" for virtual
memory can be confused with "virtual machine", so prefer "virtmem" or just
spell it out completely. Acronyms should only be used if they are widely
used or a convention of the underlying hardware, like ACPI, 

Please convert tabs to four spaces.

Avoid "use" clauses unless otherwise necessary for operators, and then,
limit their use to specific subprograms where needed or use the "use type"
clause. This forces the package of a type to be explicitly spelled out,
and so the package can be easily referenced and jumped to in an editor.
Exceptions to this rule are: 
* Textmode or printing, logging functions.
* The following standard library packages: System; Interfaces;
System.Storage_Elements;

Use the term "frame" when referring to physical memory, and "page" when
discussing virtual memory.

Please use descriptive names for variables, types, etc. We don't lack for
hard drive space to store the source code, so use longer
names (within reason) if they help to foster understanding of the code.

Try to keep lines less than 80 chars wide for the most part, but if
it negatively affects readability to break a line, then it's OK to bust the
80-wide limit. End of line comments can go past 80 if it hurts the flow of the
code to put them on their own line.

If SPARK Mode is disabled on a subprogram body, please add a comment why. This
may be perfectly valid, i.e. inline asm. However, try and restructure the code
to enable SPARK Mode - ESPECIALLY subprogram specifications. Sometimes this can
be a little painful, i.e. changing functions to procedures with multiple "out"
params.

Celebrate whitespace. Let the code breathe. Use two newlines after each
subprogram body. One newline after a subprogram body is appropriate if the
subprograms are minor variations of one another, i.e. overloaded arguments,
and they are closely grouped together for clarifying their relationship.

Use lots of comments, in gnatdoc format. This is obviously an area where
opinions differ, but I believe that treating the OS like a library with
thorough documentation encourages correctness and makes it friendlier to
new contributors. This also makes it easier to auto-generate documentation,
rather than maintain documentation separately. We all know a good API when
we see one.

"But the code will change and the comments will be out of date!" 

So... uh... just update the comments!

Notes, Cautions, Warnings
=========================
Borrowing a page from aircraft operator manuals:

* NOTE - Denotes information that is considered important to emphasize

* CAUTION - Denotes information that, if not considered, may result in system
  crashes or incorrect operation.

* WARNING - Denotes information that, if not considered, may result in loss
  of user data or damage to the underlying hardware.

Data Sizes:
===========
Nibble                                          - 4 bits
Byte                                            - 8 bits
Word                                            - 16 bits
Dword (double-word)                             - 32 bits
Qword (quad-word)                               - 64 bits

Generally speaking, we'll explicitly state the length of a data type using
the Ada Interfaces package, i.e. Unsigned_8, Unsigned_32, etc. The terms above
may be used in comments rather than spelling out "32-bit value", for instance.

SPARK-isms
==========
SPARK functions are not allowed to have any side-effects, so many times,
a procedure is used instead, and an out parameter for the result is required,
rather than just returning the result. It's a bit painful to assign temporaries
for all the procedure results.

Potential Pitfalls for Contributors
-----------------------------------

Duplicated constants
====================
Definitions of constants can't be (easily) shared between the Ada code and
assembly files, so some of them are duplicated. I've tried to get all the
constants used by the assembly files in cubit.inc, along with a note of where
it might be duplicated in Ada code. Please make sure that if you change any of
these values, that they are changed in both places. If you introduce your own
constant that's shared between assembly and Ada code - make sure they USE THE
SAME NAME!

Stack & Secondary Stack Implementation
======================================
CuBit divides the static stack area into per-CPU chunks, each of which
is split into the primary and secondary stacks for that CPU. The primary
stack grows down, the secondary stack grows up. The primary stack pointer is
set for the main CPU in boot.asm, and set for each additional CPU when they
boot up in boot_ap.asm.

       STACK_TOP    +-----------------------+
                    |                       |
                    | CPU 0 Primary Stack   |
                    |                       |
                    +-----------------------+
                    | CPU 0 Secondary Stack |
                    +-----------------------+
                    |                       |
                    | CPU 1 Primary Stack   |
                    |                       |
                    +-----------------------+
                    | CPU 1 Secondary Stack |
                    +-----------------------+
                    |           .           |
                    |           .           |
                    |           .           |
                    +-----------------------+
                    |                       |
                    | CPU N Primary Stack   |
                    |                       |
                    +-----------------------+
                    | CPU N Secondary Stack |
    STACK_BOTTOM    +-----------------------+

The secondary stack SS_Init call is made during each CPU boot-up. Secondary
stack overflows should be detected at runtime, however use caution. During
syscalls and interrupts, the process' kernel stack may be in use, which does
NOT have a secondary stack.

Limitations
-----------
* Only a single ATA/IDE disk controller is supported

Known or Suspected Bugs
-----------------------------
* Timer calibration and the busy time.sleep procedure are off a bit on 
  VirtualBox and QEMU, about 1s fast every 20s or so. On Bochs they are _way_
  off.

TODOs.
------
* X means finished
* - means in progress


TODO: Kernel Features
---------------------
[ ] There are a lot of potential circular dependencies for just "proof stuff",
    i.e. preconditions where we don't want to call a blockingSleep until 
    interrupts are enabled -> don't want to enable interrupts until the
    interrupt vector is loaded -> interrupt vector will update the value that
    blockingSleep depends on. It might make sense to keep a big "state"
    .ads file with nothing but Ghost variables used in SPARK proofs. It would
    not have any dependencies itself, but could be included by everything else
    to update their states. Downside is that it might grow huge and unwieldy,
    and sorta breaks encapsulation. Might make proofs take a long time too.

[X] Put the stack at a more sensible location
    [X] Per-CPU Stacks
    [X] Secondary Stacks
[X] Print out full register dump with exceptions
[-] Make type-safe more of the address/number conversions I'm doing.
[-] Error-handling. Need to formalize the mechanism, could get very messy with MP.
    [X] Exceptions (Last chance handler)
    [ ] Broadcast panic to other CPUs
[ ] Figure out a keyboard scancode -> key array scheme with a future eye towards 
    internationalization. Maybe just use SDL's keyboard handling and let them sort it out.
[X] Physical memory allocator
    [X] Boot-mem allocator using bitmaps
    [X] Boot phys memory allocator
        [X] Keep track of free space as we allocate/free
    [X] Buddy allocator
[X] Virtual memory mapper
    [X] Mark 0 page as not present
    [X] Re-map kernel pages with appropriate NXE bits, etc. depending on region.
[ ] Virtual memory allocator
    [-] Demand paging.
[-] Processes / Threads
    [ ] Kernel Tasks
    [X] Usermode
    [-] Scheduler
    [ ] Implement killing processes.
    [ ] Suspend
    [ ] Sleep / Wakeup
[-] ACPI tables
    [X] Find RSDT/XSDT
    [X] Sane code for parsing these.
    [-] APIC
    [ ] HPET
    [ ] MCFG - PCI express
    [ ] SSDT?
[-] I/O APIC
[-] Multiprocessing
    [ ] MP Tables (necessary?)
    [-] LAPIC
    [ ] X2APIC
[ ] Hardware
    [X] MSRs
    [-] Full CPUID detection
    [-] Disk drivers
        [ ] MBR/GPT Partition Table
    [-] PCI bus
        [-] Hard Drives
            [-] ATA
            [-] AHCI
    [ ] PCI express
        [ ] Enhanced Configuration Access Mechanism (ECAM) via MCFG tables
        [ ] NVMe
    [ ] Sound
    [ ] Video Drivers
        [-] VESA Modes
[-] Filesystem / VFS Layer
    [ ] Develop FS-agnostic set of VFS hooks to syscalls
    [ ] Develop Drive-agnostic set of VFS hooks to hardware
    [-] Ext2
[ ] Networking
    [ ] Interface driver
    [ ] TCP/IP Stack - RecordFlux should help here.
[ ] Security
    [ ] ASLR / KASLR
    [ ] Disable certain branch speculation behavior (see x86.MSR)
        [ ] if processor supports IBRS in ARCH_CAPABILITIES
    [-] KPTI
        [ ] Disable KPTI if ARCH_CAPABILITIES MSR indicates not susceptible to RDCL
    [ ] Sensible Kernel-Mode-Setting / Framebuffer / Compositor arrangement
[ ] Wow factor
    [ ] Sweet GRUB splash screen w/ logo
[-] Syscalls
    [X] SYSCALL/SYSRET working
[ ] Microkernel Concepts?
    [ ] User-mode drivers?
    [ ] IPC?
[-] More formal proofs of kernel correctness
    [ ] Preventing race conditions - may not be feasible outside of
        Ravenscar profile, which doesn't really apply to us.
[-] Implement more of the Ada standard library, especially for Tasks.

TODO: Usermode/Shell
--------------------
[-] Init model - should this look like UNIX? Something else?
[ ] Security Model
    [-] Codify it
    [ ] Prove it
    [ ] Implement it
[ ] IMGUI framework

TODO: engineering
-----------------
[ ] Make all package names Uppercase
[ ] Rename all setupXYZ to just setup, since package name is already there.
[X] New Makefile
[-] Use gnatdoc format in comments
    [ ] Edit gnatdoc format so NOTE, CAUTION, WARNING shows up as different
        colors.
    [ ] Edit gnatdoc format to ignore the leading and trailing horizontal rules
[ ] Work out a CI/CD pipeline
    [ ] Proof Step
    [ ] Unit Testing
    [ ] Build
    [ ] Integration/Functional Testing
    [ ] Generate Documentation
    [ ] Build installers, isos, etc.
[ ] Write unit tests
[ ] Fuzzing
[ ] Integration tests that run in the OS.

Architecture Ideas
------------------
* Use system RTC/HPET timers for real-time tasks, perhaps dedicate a CPU
  scheduler (or more than one) to exclusively run real-time events when
  they are desired?

Testing & Debugging Tips
------------------------
VirtualBox is a good tool for testing, however QEMU is nice when you need to
use GDB to track down certain issues.

Using VirtualBox, you'll probably want to use the ICH9 chipset.

Register add'l info: `rt`
Registers: `rg64`
Stack trace: `k`

To use QEMU:
qemu-system-x86_64 -s -S -m 4G -cdrom path\to\cubit_kernel.iso -serial stdio

QEMU will start in a paused state while it waits for the debugger. 

Then run GDB:
`gdb cubit_kernel` (note: no ".iso" here, we want the kernel object file itself,
which contains debugging symbols)

To connect to QEMU: `target remote localhost:1234`
Use `(gdb) c` to continue.

From here, normal gdb commands work, like `break`, `x`, etc.

Ada Wish-List
-------------
Note: I'm not an Ada expert, so some of this stuff may already be in the
language/compiler but called by a weird Ada-ish name. Please correct me
if this is the case!

1. UFCS - Uniform Function Call Syntax. It would be great to be able to do
things like COM1.init(), or COM1.send() instead of init(COM1), for instance.

2. Compile-time format strings.

Installing rflx
---------------
git clone https://github.com/Componolit/RecordFlux
install >= Python 3.6
install pip
install virtualenv if Python 3 not the default
source bin/activate
python setup.py build
python setup.py install
Now the rflx script at bin/rflx should work.

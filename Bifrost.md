= Introduction

This page aims to describes, in a hopefully understandable way, what I've discovered so far about the instruction set for ARM's Bifrost architecture. In addition to the technical details, I'll try and give a higher-level view of what's going on; that is, _why_ I think ARM did something, and the design tradeoffs involved, rather than just the instruction encoding and what it does. Some of this can be ascertained from patents, articles, etc., but some of it will necessarily be guesswork.

= Public information from ARM
ARM has publicly explained various parts of the ISA in various places. Obviously, any public information from ARM is incredibly useful in the reverse-engineering process. The two main sources of information are articles/presentations and patents. Articles and presentations are aimed at a general audience, and so they may not go into the detail required to actually understand the ISA, but they are good at giving a broad overview of ARM's intentions in designing the microarchitecture. Patents tend to be more specific, since ARM needs to describe the technical details more precisely for their patent to be accepted, but they're also full of legalese that can be confusing and tedious at times.

== Reading patents
*Note:* the usual IANAL disclaimer applies here. The information here is only meant to be used for reverse engineering, I can't vouch for its accuracy, don't use it for anything legally significant without consulting a lawyer first, etc. You've been warned!

Usually, for our purpose, it's better to skip the claims section and focus on the "embodiments" section. The claims section is important legally, since it governs when ARM can sue someone for patent infringement (in some sense, it's the "meat" of the patent), but we aren't concerned about that. The embodiments section, on the other hand, is supposed to provide the patent examiner and a potential judge/jury context about the invention it's disclosing by describing how ARM is actually using it in its own products. An "embodiment" here is just a way that the invention could actually be used in an actual, shipping product. Technically, how the patent is actually used in practice is irrelevant from a legal point of view, so you'll see language like "in one possible embodiment, ...", but it's usually obvious which "possible embodiment" is the one that ARM is actually using -- the patent will go into detail on only one of the possible embodiments, or explicitly talk about a "preferred embodiment" which is usually the one they're actually using.

== List o' Links
- http://www.anandtech.com/show/10375/arm-unveils-bifrost-and-mali-g71[Anandtech article], gives a general overview
- https://patents.google.com/patent/GB2540970A/en[UK Patent Application GB2540970A], on the Bifrost clause architecture
- https://patents.google.com/patent/US20160364209A1/en[US Patent Application US 2016-0364209 A1], describing a way to implement more efficient argument reduction for logarithms using a lookup table. It seems that the Itanium math libraries used the exact same technique, described in a paper http://www.cl.cam.ac.uk/~jrh13/papers/itj.pdf[here], specifically in the "novel reduction" section. Yes, this means the patent application is probably bogus. No, I'm not a lawyer.
- https://patents.google.com/patent/WO2017125700A1/en[WO2017125700A1]. This seems to describe encoding two different commutative operations with the same opcode, differentiating between them by whether the larger source comes first (what about when the sources are the same?). I haven't come across this.

= Overview
Bifrost is a clause-oriented architecture. Instructions are grouped into _clauses_ that consist of instructions that can be executed back-to-back without any scheduling decisions. A clause consists of a _clause header_ with information required for scheduling followed by a series of instructions and constants (immediates) referenced by the instructions. Instead of choosing instructions to run, the scheduler chooses clauses to run. As we'll see, instruction decoding is also per-clause instead of per-instruction. Instructions and immediates are packed into a clause and unpacked at run-time. That way, instructions can be packed to save space, and the unpacking cost is amortized over the entire clause.

Internally, each instruction consists of a register read/write stage, an FMA pipeline stage, and an ADD pipeline stage. The register stage is decoupled in the ISA from the functional units, which simplifies the decode hardware. The register stage bits describe which registers to read/write with various ports, while the FMA and ADD stages have only 3 bits for each source which describe which port to read for each source. In addition, the result of both stages from the previous cycle appears as another "port", allowing instructions to bypass the register file to save power and reduce register pressure, avoiding spilling. In addition, the register stage can read a given constant embedded in the clause or a uniform register, which is another "port." Finally, there's one last port, which is always 0 in the FMA unit but is the result of the FMA unit from the same instruction in the ADD unit. For example, an unfused multiply-add is done by doing a multiply in the FMA unit, using the 0 port for the sum, and then an addition in the ADD unit with the same port (here representing the result of the multiply) in the ADD unit.

Note that in addition to the execution unit, which we've been describing, there are a few other units:

- Varying interpolation unit
- Attribute fetching unit
- Load/store unit
- Texture unit

The execution unit interacts with these fixed-function blocks through special, variable-latency instructions in the FMA and ADD units. They bypass the usual, fixed-latency mechanism for reading/writing registers, and as such instructions in the same clause can't assume that registers have been read/written after the instruction is done (TODO: verify this for registers being read, and not just written). Instead, any dependent instructions must be put into a separate clause with the appropriate dependencies in the clause header set.

= Clauses
Conceptually, each clause consists of a clause header, followed by one or more 78-bit instruction words and then zero or more 60-bit constants. (Constants are actually 64 bits, but they're loaded the same port as uniform registers and share the same field in the instruction word, which includes 7 bits to choose which uniform register to load, some of which would be unused for constants, so ARM decided to be clever and stick the bottom 4 bits in each instruction where the constant is loaded, so the actual constants in the instruction stream are only 60 bits). But the instruction fetching hardware only works in 128-bit quadwords, so each clause has to be a multiple of 128 bits. To make the representation of the clauses as compact as possible which still making the decoding circuitry relatively simple, the instructions are packed so that two 128-bit quadwords can store 3 78-bit instructions, or 3 128-bit quadwords can store 4 instructions and a 60-bit constant. There were some bits left over, which seem to have been used to obviate the need to keep track of state between each word, simplifying the decoder and making it possible to decode the quadwords in parallel. Thus, the quadwords can be (almost) arbitrarily reordered while still  Each format fully describes which instruction(s) in the decoded clause the bits in the quadword represent, and whether one of those instructions is the last instruction.

== Quadword formats
The bottom 8 bits of each 128-bit quadword are a "tag" that's read by the decoding circuitry. They describe how to interpret the rest of the word, as well as possibly containing some bits from some of the instructions to decode if there were more bits to spare. Each possible tag value is described below.

[cols="8*"]
|============================
| 0            | 0            | 1             | 0            | 1          3+| `I1`
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This is the beginning of a clause with more than one instruction. The next 75 bits of the quadword are the low 75 bits of instruction 0, and `I1` gives the high 3 bits. The high 45 bits of the quadword contain the clause header.

[cols="8*"]
|============================
| 0            | `S`          | 0             | 0            | 1          3+| `I1`
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This is the beginning of a clause with only one instruction. The next 75 bits of the quadword are the low 75 bits of instruction 0, and `I1` gives the high 3 bits. The high 45 bits of the quadword contain the clause header. If `S` is one, then the clause is finished after this quadword. Otherwise, there are later quadwords (that must contain constants).

[cols="8*"]
|============================
| 0            | `S`          | 0             | 0            | 0            | 0            | 1            | 1
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This quadword contains instruction 1, which is the final instruction. As usual, the next 75 bits are the low 75 bits of instruction 1, and the high 3 bits of the quadword contain the high 3 bits of instruction 1. As before, there may be following constants, in which case `S` is set to 0.

[cols="8*"]
|============================
| 0            | 0            | 1             | 0            | 0          3+| `I1`
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This quadword contains instruction 1 and part of instruction 2. The next 75 bits are the low 75 bits of instruction 1, and `I1` contains its high 3 bits. After that, the next 45 bits are the low 45 bits of instruction 2.

[cols="8*"]
|============================
| 0            | `S`          | 0             | 0            | 0            | 1            | 0            | 0
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This quadword contains instruction 2, which is the final instruction, plus a constant. The next 60 bits are the constant, after which 15 bits are unused, and then there are 30 bits for instruction 2. The high 3 bits of the quadword are the high 3 bits of instruction 2.

[cols="8*"]
|============================
| 0            | `S`          | 0             | 0            | 0            | 1            | 0            | 1
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This quadword contains the final instruction 3 and part of instruction 2. The next 75 bits are the low bits of instruction 3, while the 30 bits after that are the next 30 bits of instruction 2. The high 3 bits of the quadword are the high 3 bits of instruction 2, while the next-highest are the high 3 bits of instruction 3.

[cols="8*"]
|============================
| 0            | 0            | 0             | 0            | 0            | 0            | 0            | 1
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

The same as the above, except that instruction 3 isn't the final instruction.

[cols="8*"]
|============================
| 1            | 0          3+| `I1`                                      3+| `I2`
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

The next 75 bits are the low 75 bits of instruction 3, and the high 3 bits of instruction 3 are given by `I1`. The next 30 bits give bits 45-74 of instruction 2, and `I2` gives the high 3 bits of instruction 2. Finally, the remaining high 15 bits are the low 15 bits of the first constant.

[cols="8*"]
|============================
| 0            | `S`          | 0             | 1            | 0          3+| `I1`
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This must come after a word with the previous format, and gives instruction 4, which must be the final instruction, and the rest of the constant. The next 75 bits are the low 75 bits of instruction 4, and `I1` contains the high 3 bits. The remaining 45 bits give the high 45 bits of the first constant.

[cols="8*"]
|============================
| 0            | 1            | 1             | 0            | 0          3+| `I1`
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This quadword contains instruction 4 and part of instruction 5. The next 75 bits are the low 75 bits of instruction 4, and `I1` contains its high 3 bits. After that, the next 45 bits are the low 45 bits of instruction 5.

[cols="8*"]
|============================
| 0            | `S`          | 0             | 0            | 0            | 1            | 1            | 1
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This quadword contains the final instruction 6 and part of instruction 5. The next 75 bits are the low bits of instruction 6, while the 30 bits after that are the next 30 bits of instruction 5. The high 3 bits of the quadword are the high 3 bits of instruction 5, while the next-highest are the high 3 bits of instruction 6.

[cols="8*"]
|============================
| 0            | `S`          | 0             | 0            | 0            | 1            | 1            | 0
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This quadword contains part of instruction 5, which is the final instruction, plus a constant. The next 60 bits are the constant, after which 15 bits are unused, and then there are 30 bits for instruction 5. The high 3 bits of the quadword are the high 3 bits of instruction 5.

[cols="8*"]
|============================
| 1            | 1          3+| `I1`                                      3+| `I2`
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

The next 75 bits are the low 75 bits of instruction 6, and the high 3 bits of instruction 6 are given by `I1`. The next 30 bits give bits 45-74 of instruction 5, and `I2` gives the high 3 bits of instruction 5. Finally, the remaining high 15 bits are the low 15 bits of the first constant.

[cols="8*"]
|============================
| 0            | `S`          | 0             | 1            | 1          3+| `I1`
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This must come after a word with the previous format, and gives instruction 7, which must be the final instruction, and the rest of the constant. The next 75 bits are the low 75 bits of instruction 7, and `I1` contains the high 3 bits. The remaining 45 bits give the high 45 bits of the first constant.

[cols="8*"]
|============================
| 0            | `S`          | 1             | 1          4+| `pos`
| {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}  | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp} | {nbsp}{nbsp}
|============================

This format contains two 60-bit constants in the rest of the 120 bits of the quadword. The `pos` bits describe where in the instruction stream the constants are. In particular, it encodes the total number of instructions in the clause and the number of constants before the current two. Since the packing algorithm below can only produce some of these combinations, not every possible pair of instructions and constants is representable. The table below lists the ones that are currently known.

[options="header"]
|============================
| `pos` | Instructions | Constants
| 0     | 1            | 0
| 1     | 2            | 0
| 2     | 4            | 0
| 3     | 3            | 1
| 4     | 5            | 1
| 5     | 4            | 2
| 6     | 7            | 0
| 7     | 6            | 1
| 8     | 5            | 3
| 9     | 8            | 1
| a     | 7            | 2
| b     | 6            | 3
| c     | 8            | 3
| d     | 7            | 4
|============================

There is a trivial limit for the number of constants per clause: since each instruction can only load one (64-bit) constant, there can be at most as many constants as instructions. It seems that the blob compiler currently refuses to add more than 5 constants to an 8-instruction clause, although it's happy to use at least 6 constants for a 7-quadword clause. Beyond that, the limit isn't known. However, there are only two remaining values for `pos` that haven't been observed (e and f), so there isn't much space left for more constants.

== Algorithm for packing clauses

This section describes how the blob compiler seems to use these formats to pack as many instructions and constants into as few words as possible. There may be other equivalent ways to do it, but given how complicated all the different formats are, and that the hardware decoder and encoding algorithm were developed in tandem, it's probably best to stick with what the blob does.

First, we assign instructions to quadwords. We may assign an entire instruction to a quadword, or we may split an instruction across two quadwords.

- If only one instruction, assign it to the first quadword.
- Assign the second instruction to the second quadword.
- Split the third instruction across the second and third quadwords.
- Assign the fifth instruction to the third quadword.
- Assign the sixth instruction to the fourth quadword.
- Split the seventh instruction acrosss the fourth and fifth quadword.
- Assign the eighth instruction to the fifth quadword.

Simply go down the list until there are no more instructions left.

Now, we assign constants to quadwords if we have any. We do this by looking at the last quadword, and do the following:

- If it only contains an instruction that was split across two quadwords, then there are 75 bits free. Put the constant where the next instruction would have gone, and use the appropriate format to indicate that.
- If it only contains one instruction, and the previous quadword has an instruction and a split instruction, then we can split the constant across the last two instructions.

For any remaining constants, we simply add quadwords with two constants each. Note that in some cases, we need to add a "dummy" constant, even when the clause doesn't use any constants, because there's no format that does what we want. For example, say that we have a clause with 5 instructions and no constants. The fourth quadword is supposed to contain only instruction 4, which is the final instruction, but there is no format for that. Instead, we add a constant split across the third and fourth quadwords, since there is a format with a final instruction 4 and part of a constant. From a design point of view, this reduces the number of possible formats, which reduces the complexity of the decoder and means less bits are needed to describe the format.

== Clause Header

The clause header mainly contains information about "variable-latency" instructions like SSBO loads/stores/atomics, texture sampling, etc. that use a separate functional units. There can be at most one variable-latency instruction per clause. It also indicates when execution should stop, and has some information about branching. The format of the header is as follows:

[options="header"]
|============================
| Field                        | Bits
| unknown                      | 18
| Register                     | 6
| Scoreboard dependencies      | 6
| unknown                      | 2
| Scoreboard entry             | 3
| Instruction type             | 4
| unknown                      | 1
| Next clause instruction type | 4
| unknown                      | 1
|============================

=== Register field

A lot of variable-latency instructions have to interact with the register file in ways that would be awkward to express in the usual manner, i.e. with the per-instruction register field. For example, the STORE instruction has to read up to 4 32-bit registers, which the usual pathways for reading a register can't handle -- they're designed for reading up to three 32-bit or 64-bit registers each cycle, and it also needs to load a 64-bit address from registers. The LOAD instruction can't write to the register until the operation has finished, possibly well after the instruction executes. For cases like these, there's a "register" field in the clause header that lets the variable-latency instruction read/write one, or a sequence of, registers, in a manner different than the usual one. Since there can only be one variable-latency instruction per clause, this field isn't ambiguous about which instruction it applies to. If more than one register is being read from or written to, it must be a power of two, and the register field must be aligned to that power of two. For example, a two-register source could be R0-R1 (if the register field is 0), R2-R3 (register field is 2), R4-R5, etc. Or a four-register source could be R0-R3, R4-R7, etc.

=== Dependency tracking

No instructions depend upon a high-latency instruction in the same clause and so all the intra-clause scheduling can be done by the compiler. On the other hand, instructions in one clause might depend on a variable-latency instruction in another clause, and the latency obviously can't be known beforehand, so some kind of inter-clause dependency tracking mechanism must exist. Bifrost uses a six-entry https://en.wikipedia.org/wiki/Scoreboarding[scoreboard], with the scoreboard entries and dependencies manually described by the compiler. Each clause has a scoreboard entry, which corresponds to a given bit in the scoreboard. When the clause is dispatched, the bit is set, and when the variable-latency instruction in the clause completes, the bit is cleared. Any clauses afterwards can set that clause's scoreboard entry in its "scoreboard dependencies" bitfield to halt execution until the variable-latency instruction completes.

As a concrete example, consider this program:

[source,glsl]
----
layout(std430, binding = 0) buffer myBuffer {
    int inVal1, inVal2, outVal;
}

void main() {
    outVal = inVal1 + inVal2;
}
----

It might get translated into something like this, in assembly-like pseudocode (assuming for a second that the loads don't get combined):

[source]
----
{
LOAD.i32 R0, ptr + 0x0
}
{
LOAD.i32 R1, ptr + 0x4
}
{
ADD.i32 R0, R0, R1
STORE.i32 R0, ptr + 0x8
}
----

The third clause must depend on the first two, although the first two are independent and can be executed in any order. The dependency bits to express this might be:

[options="header"]
|============================
| Clause | Scoreboard entry | Scoreboard dependencies
| 1 | 0 | 000000
| 2 | 1 | 000000
| 3 | 2 | 000011
|============================

Since the first two clauses have no dependencies, they will be executed in-order, one immediately after the other. They will queue up two requests to the load/store unit, with scoreboard tags of 0 and 1, and set bits 0 and 1 of the scoreboard. The first load will clear bit 0 of the scoreboard (based on the tag that was sent with the load) when it is finished, and the second load will clear bit 1. The third clause has bits 0 and 1 set in the dependencies, so it will will wait for bits 0 and 1 to clear before executing. Therefore, it won't run until both of the loads have been completed.

The final wrinkle in all of this is that the scoreboard dependencies encoded in the clause are actually the dependencies before the _next_ clause is ready to execute. So in the above example, the actual encoding for the clauses would look like:

[options="header"]
|============================
| Clause | Scoreboard entry | Scoreboard dependencies
| 1 | 0 | 000000
| 2 | 1 | 000011
| 3 | 2 | 000000
|============================

The first clause in a program implicitly has no dependencies. This scheme makes it possible to determine whether the next clause can be run before actually fetching it, presumably simplifying the hardware scheduler a little.

=== Instruction type

The "instruction type" and "next clause instruction type" fields tell whether the clause has a variable-latency instruction, and if it does, which kind. Unsurprisingly, the "next clause instruction type" field applies to the next clause to be executed. If the clause doesn't have any variable-latency instructions, then the whole scoreboarding mechanism is skipped -- the clause is always executed immediately and it never sets or clears any scoreboard bits.

[options="header"]
|============================
| Value | Instruction type
| 0     | no variable-latency instruction
| 5     | SSBO store
| 6     | SSBO load
|============================

TODO: fill out this table

= Instructions

Now that we know how instructions and constants are packed into clauses, let's take a look at the instructions themselves. Each instruction is executed 3 stages, the register read/write stage, the FMA stage, and the ADD stage, each of which corresponds to a part of the 78 bit instruction word. We'll describe what they do, and the format of their part of the instruction word, in the next sections. We'll go by the order they're executed, as well as the order of the bits in the instruction word.

== Register read/write (35 bits)

As the name suggests, this stage reads from and writes to the register file. The current instruction reads from the register file at the same time that the previous instruction writes to the register file. Thus, the field contains both reads from the current instruction and writes from the previous instruction. Presumably, the scheduler makes this happen by interleaving the execution of multiple clauses from different quads. It only executes one instruction from a given quad every 3 cycles, so that the register write phase of one instruction happens at the same time as the register read phase of the next. Of course, it's possible that the FMA and ADD stages take more than 1 cycle, and more threads are interleaved as a consequence; this is a microarchitectural decision that's not visible to us. The result is that a write to a register that's immediately read by the next instruction won't work, but that's never necessary anyways thanks to the passthrough sources detailed later.

The register file has four ports, two read ports, a read/write port, and a write port. Thus, up to 3 registers can be read during an instruction. These ports are represented directly in the instruction word, with a field for telling each port the register address to use. There are three outputs, corresponding to the three read ports, and two inputs, corresponding to the FMA and ADD results from the previous stage. The ports are controlled through what the ARM patent calls the "register access descriptor," which is a 4-bit entry that says what each of the ports should do. Finally, there is the uniform/const port, which is responsible for loading uniform registers and constants embedded in the clause. Note that the uniforms and constants share the same port, which means that only one uniform or one constant (but not both) can be loaded for an instruction. This port supplies 64 bits of data, though, so two 32-bit parts of the same 64-bit value can be accessed in the same instruction.

The format of the register part of the instruction word is as follows:

[options="header"]
|============================
| Field               | Bits
| Uniform/const       | 8
| Port 2 (read/write) | 6
| Port 3 (write)      | 6
| Port 0 (read)       | 5
| Port 1 (read)       | 6
| Control             | 4
|============================

Control is what ARM calls the "register access descriptor." To save bits, if the Control field is 0, then Port 1 is disabled, and the field for Port 1 instead contains the "real" Control field in the upper 4 bits. Bit 1 is set to 1 if Port 0 is disabled, and bit 0 is reused as the high bit of Port 0, allowing you to still access all 64 registers. If the Control field isn't 0, then both Port 0 and Port 1 are always enabled. In this way, the Control field only needs to describe how Port 2 and Port 3 are configured, except for the magic 0 value, reducing the number of bits required.

Before we get to the actual format of the Control field, though, we need to describe one more subtlety. Each instruction's register field contains the writes for the previous instruction, but what about the writes of the last instruction in the clause? Clauses should be entirely self-contained, so we can't look at the first instruction in the next clause. The answer turns out to be that the first instruction in the clause contains the writes for the last instruction. There are a few extra values for the control field, marked "first instruction," which are only used for the first instruction of a clause. The reads are processed normally, but the writes are delayed until the very end of the clause, after the last instruction. The list of values for the control field is below:

[options="header"]
|============================
| Value | Meaning
| 1     | Write FMA with Port 2
| 3     | Write FMA with Port 2, read with Port 3
| 4     | read with Port 3
| 5     | Write ADD with Port 2
| 6     | Write ADD with Port 2, read with Port 3
| 8     | Nothing, first instruction
| 9     | Write FMA, first instruction
| 11    | Nothing
| 12    | read with Port 3, first instruction
| 15    | Write FMA with Port 2, write ADD with Port 3
|============================

Unlike the other ports, the uniform/const port always loads 64 bits at a time. If an FMA or ADD instruction only needs 32 bits of data, the high 32 bits or low 32 bits are selected later in the source field, described below.

The uniform/const bits describe what the uniform/const port should load. If the high bit is set, then the low 7 bits describe which pair of 32-bit uniform registers to load. For example, 10000001 would load from uniform registers 2 and 3. If the high bit isn't set, then the next-highest 3 bits indicate what 64-bit constant to load, while the low 4 bits contain the low 4 bits of the constant. The mapping from from bits to constants is a little strange:

[options="header"]
|============================
| Field value | Constant loaded
| 4           | 0
| 5           | 1
| 6           | 2
| 7           | 3
| 2           | 4
| 3           | 5
| 0           | disable?
| 1           | unused?
|============================

== Source fields

When the FMA and ADD stages want to use the result of the register stage, they do so through a 3-bit source field in the instruction word. There are as many source fields are there are sources for each operation. The following table shows the meaning of this field:

[options="header"]
|============================
| Field value | Meaning
| 0           | Port 0
| 1           | Port 1
| 2           | Port 3
.2+| 3        | FMA: always 0
| ADD: result of FMA unit from same instruction
| 4           | Low 32 bits of uniform/const
| 5           | High 32 bits of uniform/const
| 6           | Result of FMA from previous instruction (FMA passthrough)
| 7           | Result of ADD from previous instruction (ADD passthrough)
|============================

== FMA (23 bits)

Both the FMA and ADD units have various instruction formats. The high bits are always an opcode, of varying length. They must have the property that no opcode from one format is a prefix for another opcode in a different format. This guarantees that no instruction is ambiguous. Since there's no format tag, it would seem that decoding which format each instruction has is complicated, although it's possible that some trick is used to speed it up. In the disassmbler, we just try matching each opcode with the actual one, masking off irrelevant bits. I'm only going to list the categories here, and not the actual opcodes; I'll leave the former to the disassembler source code, simply because there are a lot of them and it's tedious to type them all up, and error-prone too. The disassembler is at least easy to test, so the chances of making a mistake are lower.

=== One Source (FMAOneSrc)

[options="header"]
|============================
| Field   | Bits
| Src0    | 3
| Opcode  | 20
|============================

=== Two Source (FMATwoSrc)

[options="header"]
|============================
| Field   | Bits
| Src0    | 3
| Src1    | 3
| Opcode  | 17
|============================

=== Floating-point Comparisons (FMAFcmp)

[options="header"]
|============================
| Field               | Bits
| Src0                | 3
| Src1                | 3
| Src1 absolute value | 1
| unknown             | 1
| Src1 negate         | 1
| unknown             | 3
| Src0 absolute value | 1
| Comparison op       | 3
| Opcode              | 7
|============================

Where the comparison ops are given by:

[options="header"]
|============================
Value   | Meaning
| 0 | Ordered Equal
| 1 | Ordered Greater Than
| 2 | Ordered Greater Than or Equal
| 3 | Unordered Not-Equal
| 4 | Ordered Less Than
| 5 | Ordered Less Than or Equal
|============================

=== Two Source with Floating-point Modifiers (FMATwoSrcFmod)

[options="header"]
|============================
| Field               | Bits
| Src0                | 3
| Src1                | 3
| Src1 absolute value | 1
| Src0 negate         | 1
| Src1 negate         | 1
| unknown             | 3
| Src0 absolute value | 1
| unknown             | 2
| Outmod              | 2
| Opcode              | 6
|============================

The output modifier (Outmod) is given by:

[options="header"]
|============================
Value   | Meaning
| 0 | Nothing
| 1 | max(output, 0)
| 2 | clamp(output, -1, 1)
| 3 | saturate - clamp(output, 0, 1)
|============================

=== Three Source (FMAThreeSrc)

[options="header"]
|============================
| Field   | Bits
| Src0    | 3
| Src1    | 3
| Src2    | 3
| Opcode  | 14
|============================

=== Three Source with Floating Point Modifiers (FMAThreeSrcFmod)

[options="header"]
|============================
| Field               | Bits
| Src0                | 3
| Src1                | 3
| Src2                | 3
| unknown             | 3
| Src0 absolute value | 1
| unknown             | 2
| Outmod              | 2
| Src0 negate         | 1
| Src1 negate         | 1
| Src1 absolute value | 1
| Src2 absolute value | 1
| Opcode              | 2
|============================

=== Four Source (FMAFourSrc)

[options="header"]
|============================
| Field   | Bits
| Src0    | 3
| Src1    | 3
| Src2    | 3
| Src3    | 3
| Opcode  | 11
|============================

== ADD (20 bits)

The instruction formats for ADD are similar, except it can only have up to 2 sources instead of FMA's four sources.


=== One Source (ADDOneSrc)

[options="header"]
|============================
| Field   | Bits
| Src0    | 3
| Opcode  | 17
|============================

=== Two Source (ADDTwoSrc)

[options="header"]
|============================
| Field   | Bits
| Src0    | 3
| Src1    | 3
| Opcode  | 14
|============================

=== Two Source with Floating-point Modifiers (ADDTwoSrcFmod)

[options="header"]
|============================
| Field               | Bits
| Src0                | 3
| Src1                | 3
| Src1 absolute value | 1
| Src0 negate         | 1
| Src1 negate         | 1
| unknown             | 2
| Outmod              | 2
| unknown             | 2
| Src0 absolute value | 1
| Opcode              | 4
|============================

=== Floating-point Comparisons (ADDFcmp)

[options="header"]
|============================
| Field               | Bits
| Src0                | 3
| Src1                | 3
| Comparison op       | 3
| unknown             | 2
| Src0 absolute value | 1
| Src1 absolute value | 1
| Src0 negate         | 1
| Opcode              | 6
|============================

= Special instructions

This section describes some of the instructions that interact with fixed-function units like the texture sampler, varying interpolation unit, etc. They oftentimes use a special format, since they have to pass extra information to the fixed-function block.

== Texture instructions

A single instruction encoding is used for every texture operation. What operation to do, and what texture/sampler to use, is encoded in a 32-bit control word which is loaded from the uniform/immediate port. The intention is that it's an immediate, basically used to extend the instruction. Trying to encode everything in the instruction wouldn't go too well, simply because there are too many possible combinations.

The texture instruction is encoded similar to a normal two-source instruction, with the bottom 6 bits devoted to sources, except that bit 6 is also used to encode whether the control word comes from the low 32 bits or high 32 bits of the port. The instruction can read up to 6 registers, with two coming from normal sources and four coming from the special per-clause register (hence they must be adjacent, starting on a multiple of 4). The texture unit then writes the result to the same per-clause register. Hence, those 4 registers are (sometimes) read and then later written.

The format of the control word is as follows:

[options="header"]
|============================
| Field                        | Bits
| Sampler Index/Indirects      | 4
| Texture Index                | 7
| Separate Sampler and Texture | 1
| Filter                       | 1
| unknown                      | 2
| Texel Offset                 | 1
| Shadow                       | 1
| Array                        | 1
| Texture Type                 | 2
| Compute LOD                  | 1
| No LOD Bias                  | 1
| Calculate Gradients          | 1
| unknown                      | 1
| Result Type                  | 4
| unknown                      | 4
|============================

Unlike other architectures, where samplers and textures are just handles passed around, Bifrost seems to use an older binding-table-based approach. There is are two tables, one for samplers and one for textures, containing all the state for each, and the texture instruction chooses an index into each table. If the "Separate Sampler and Texture" bit is set, then the "Texture Index" supplies the index into the texture table, and "Sampler Index/Indirect" provides the index into the sampler table. If the bit is unset, then by default, both indices come from from the "Texture Index" field. However, the "Sampler Index/Indirects" field then provides a bitmask of indirect sources. If the low bit is set, the texture index is "indirect," i.e. it is obtained by a register source. Similarly, if the second-lowest bit is set, the sampler index is indirect. The next two bits are unknown, and always observed as 11. Note that indirect indices are only possible if "Separate Sampler and Texture" is unset -- if it is set, then both indices are always taken directly from their respective fields in the control word.

The "filter" bit is unset for `texelFetch` and `textureGather` operations, where the normal bilinear filtering pipeline is bypassed. The "texel offset" is used for `textureOffset`, and specifies an extra register source for the offset. "Shadow" is used for shadow textures, and implies an extra source for the depth reference. "Array" is similarly used for array textures, and implies another source for the array index. Texture Type indicates the type of texture, as in the following table:

[options="header"]
|============================
| Field Value | Type
| 0 | Cube
| 1 | Buffer
| 2 | 2D
| 3 | 3D
|============================

The next three bits select between `texture`, `texture` with an LOD bias passed, `textureLod`, and `textureGrad`. All of these provide different ways of changing the computed Level of Detail before sampling the texture. The usage is as follows:

[options="header"]
|============================
| GLSL operation          | Compute LOD | No LOD bias | Calculate Gradients
| plain `texture`         | 1 | 1 | 1
| `texture` with LOD bias | 1 | 0 | 1
| `textureLod`            | 0 | 0 | 1
| `textureGrad`           | 1 | 1 | 0
|============================

The actual gradients for `textureGrad` would take up too many registers, so they get stored in a separate earlier texture instruction with some unknown fields changed.

Finally, the result type is interpreted based on the following table:

[options="header"]
|============================
| Value | Result Type
| 4 | 32-bit floating point
| 14 | 32-bit signed integer
|15 | 32-bit unsigned integer
|============================

TODO: compact texture instructions, dual texture instructions

== Varying Interpolation

The varying interpolation unit is responsible for, well, interpolating varyings. This gets more complicated with support for per-sample shading and centroid interpolation, which is where most of the complexity comes from. In per-sample shading, the fragment shader is run for each sample, compared to traditional multisampling where the fragment shader is run once per pixel, and samples have different colors only if they are covered by different triangles (or with alpha-to-coverage). In that case, the coordinates used for sampling have to come from the current sample. If the fragment shader is run per-pixel as usual, then the coordinates are normally taken from the center of the pixel. But this doesn't work so well for multisampling, where the sample might not actually lie on the triangle if the triangle doesn't go through the center of the pixel. So, there's an alternate "centroid" sampling method that chooses the coordinates based on the covered samples. Finally, the shader can specify an offset for the coordinates through `interpolateAtOffset` in GLSL. After all this, the result is the coordinates of the point to be sampled. These coordinates are converted to barycentric coordinates, and then the usual barycentric interpolation is applied to the varyings at all three corners of the triangle to get the final result.

On Bifrost, this is all handled through a single instruction. Varyings are always vec4-aligned, i.e. the address is always expressed in 128-bit units, even though there are instructions for interpolating single floats, vec2's, vec3's and vec4's. The instruction always has at least one source, which is always R61. Before the shader starts, R61 is filled out as follows: the low 16 bits contain the sample mask (`gl_SampleMaskIn`), the next 4 bits contain the sample ID if doing per-sample shading (`gl_SampleID`), and the rest of the bits are unknown (always 0?). The sample mask is needed for doing centroid sampling, while the sample ID is needed for doing per-sample shading.

The format of the instruction as follows:

[options="header"]
|============================
| Field                        | Bits
| Src0                         | 3
| Address                      | 5
| Components                   | 2
| Interpolation Type           | 2
| Reuse previous coordinates   | 1
| Flat shading                 | 1
| Opcode (=10100)              | 5
|============================

The address field requires some explanation. Only addresses under 20 can be encoded directly. This table shows exactly how the field is decoded.

[options="header"]
|============================
| Bit pattern | Meaning
| `0xxxx`       | Interpolate at address `0xxxx`.
| `100xx`       | Interpolate at address `100xx`.
| `101xx`       | Unknown.
| `11xxx`       | The address is indirect, given by an additional source indicated by `xxx`.
|============================

To get the number of components, add one to the components field.

The interpolation field has the following meaning:

[options="header"]
|============================
| Value | Meaning
| 0 | Force per-fragment sampling (when per-sample shading is enabled).
| 1 | Centroid sampling.
| 2 | Normal -- per-sample shading if enabled through GL state, otherwise pixel center.
| 3 | Use explicit coordinates loaded through a previous instruction.
|============================

If the coordinates computed would be the same as the previous interpolation instruction, the "reuse previous coordinates" bit can be set. Finally, the "flat shading" bit enables flat shading, where the varying is chosen from one of the triangle corners based on the GL provoking vertex rules.
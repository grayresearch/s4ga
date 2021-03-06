# S4GA
Copyright (C) 2021, Gray Research LLC.
Licensed under the Apache License, Version 2.0.

S4GA is a small simple slow serial FPGA core,
targeting ~0.1mm2 of the 130nm Skywater ASIC PDK,
using the efabless Caravel harness and the
Zero-to-ASIC multiproject framework.

It configurably implements an FPGA with x inputs, y outputs, and n
k-LUTs, for a total of w=n+x global nets. The first y global nets are
the outputs. The FPGA is organized as n/m LUT-clusters of m k-LUTs. For
each LUT, 0<=g<=k of its inputs may specify global nets, while k-g of
its inputs must be local nets from the m k-LUTs of its cluster.

![Block diagram](doc/s4ga.png)

Within a LUT-cluster, LUTs are evaluated one after another, serially.
Therefore it takes m clock cycles to evaluate all the LUTs once, a "full
cycle". Technology mapping is sensitive to order of LUT evalation within
a full cycle and can use this to implement reg -> comb* -> reg. For example,
`a <= mux(eq(neg(a, ...)))` is implemented by

	; a=[2]
	0: LUT{neg}(2,...)
	1: LUT{eq}(0,...)
	2: LUT{mux}(1,...)

An evaluated LUT output net may not be used as the input to a LUT *in
a different LUT cluster* in the next clock cycle.

During each k-LUT evaluation, the k-1-LUT output (i.e. the output of
the lower half of the LUT truth table) is registered as "q", which may
be used as an input of the next LUT evaluation in that cluster.

	; k=4
	; i:   LUT{lut4}(in3,in2,in1,in0)
	; i,q: LUT{lut3,lut3}(=1,in2,in1,in0)

When k>=4 this enables relatively efficient ripple-carry adders:

	; Example 1:
	; n=8 x=8 y=8 m=8 k=4 g=1
	; a[7:0]={7..0}
	; b[7:0]={15..8}
	; a += b:
	0: LUT{sum0,carry0}(=1,,0,8)
	1: LUT{sum,carry}(=1,=q,1,9)
	2: LUT{sum,carry}(=1,=q,2,10)
	3: LUT{sum,carry}(=1,=q,3,11)
	4: LUT{sum,carry}(=1,=q,4,12)
	5: LUT{sum,carry}(=1,=q,5,13)
	6: LUT{sum,carry}(=1,=q,6,14)
	7: LUT{sum,carry}(=1,=q,7,15)
	----
	8b adder, 8 minor cycles = 1 full cycle

	; Example 2:
	; n=8 x=15 y=8 m=8 k=4 g=2
	; sel = 8
	; a[6:0]={15..9}
	; b[6:0]={22..16}
	; o[6:0]={7..1}
	; o = sel ? a : b:
	0: LUT{,in0}(=1,,,8) ; q=sel
	1: LUT{mux,q}(=1,=q,9,16)
	2: LUT{mux,q}(=1,=q,10,17)
	3: LUT{mux,q}(=1,=q,11,18)
	4: LUT{mux,q}(=1,=q,12,19)
	5: LUT{mux,q}(=1,=q,13,20)
	6: LUT{mux,q}(=1,=q,14,21)
	7: LUT{mux,}(=1,=q,15,22)
	----
	7b 2:1 mux, 8 minor cycles = 1 full cycle

	; Example 3:
	; n=16 x=28 y=16 m=8 k=4 g=2
	; sum[13:0]={15..9,7,5..0}
	; a[13:0]={29..16}
	; b[13:0]={43..30}
	; sum = a + b:
	0: LUT{sum0,carry0}(=1,,16,30)
	1: LUT{sum,carry}(=1,=q,17,31)
	2: LUT{sum,carry}(=1,=q,18,32)
	3: LUT{sum,carry}(=1,=q,19,33)
	4: LUT{sum,carry}(=1,=q,20,34)
	5: LUT{sum,carry}(=1,=q,21,35)
	6: LUT{carry,q}(=1,=q,22,36)	// 6=co[6], pass q
	7: LUT{sum,}(=1,=q,22,36)
	---- full cycle
	8: LUT{,copy}(,,23,)			// help 9, since g<3
	9: LUT{sum,carry}(=1,=q,6,37)	// 0.6 => 1.1, OK
	10: LUT{sum,carry}(=1,=q,24,38)
	11: LUT{sum,carry}(=1,=q,25,39)
	12: LUT{sum,carry}(=1,=q,26,40)
	13: LUT{sum,carry}(=1,=q,27,41)
	14: LUT{sum,carry}(=1,=q,28,42)
	15: LUT{sum,}(=1,=q,29,45)
	----
	14b adder, II=1 full, latency=2 full

"q" also helps with global input fan-in. For example if k=4 g=2 and a
LUT has four global inputs (i.e. LUT outputs from other clusters), form:

	5: LUT{in1,in0}(=1,,g1,g2)	// 5=g1,q=g2
	6: LUT{lut4}(5,=q,g3,g4)	// LUT(g1,g2,g3,g4)

A k-LUT is represented by a 2<sup>k</sup> truth table and a vector of k input nets
(net indices). A global net index designates the output of any LUT,
or any input, and is lg(w=n+x) bits. A local net index designates the output
of a LUT in the cluster and is lg(m) bits.
Example:

	// Example: n=32 m=8 k=4 g=2 
	packed struct LUT_n32_m8_k4_g2 {
		bit[16] tt;
		bit[5] in0; // global
		bit[5] in1; // global
		bit[3] in2; // local
		bit[3] in3; // local
	};

	// Example: n=64 m=4 k=3 g=1 
	packed struct LUT_n64_m4_k3_g1 {
		bit[8] tt;
		bit[6] in0; // global
		bit[2] in1; // local
		bit[2] in2; // local
	};

If the last or second last net index is all 1's it designates a special
input net, depending upon LUT input position:

* k-1: =1: cial: constant "1";
* k-2: =q: the preceding half-LUT output;

## Inputs and Outputs

A S4GA core has x inputs and y outputs. The caravel_s4ga interface is
used to route Wishbone data or input pads (or some of both) to the inputs.

Output nets may be read back over Wishbone and may drive output pads.

## Caravel Interface

The module caravel_s4ga implements S4GA in the context of the
efabless Caravel harness. This includes a Wishbone slave interface
and a logic analyzer interface.

The interface to the S4GA core is memory mapped.

	Addr	Name		Bits	Description

	xx00	s4ga_csr			S4GA control/status (RW)
			.reset		0		1 => fully reset S4GA (WO)
			.cfgd		1		0 => not configured; 1 => configured
			.step		2		0 => run; 1 => single step on WB R/W
	
	xx04	s4ga_config			S4GA configuration (WO)
	xx08	s4ga_input_mask		S4GA input mask register (WO)
	xx10	s4ga_input			S4GA input register (WO)
	xx20	s4ga_output			S4GA output register (RO)

Using the interface, the Caravel MCU can

	* reset S4GA
	* load its configuration bitstream
	* read back its status
	* configure its input mask
	* reset S4GA global nets
	* configure S4GA to be free running
	* configure S4GA to run one full cycle 
	* configure S4GA to run until LUT [n-1] is 1

The MCU writes each 32b frame of the configuration data into S4GA.

...


## Implementation in Skywater 130nm PDK

This design, an assignment in Matthew Venn's Zero-to-ASIC class, targets a
300x300um extent of the user-project area of the efabless Caravel harness.

![S4GA implementation in Skywater 130nm PDK](doc/s4ga-sky130.png)

### LUT Configuration Memory

The LUT configuration memory uses the vast majority of the area of the FPGA.
For example, a n=64 m=8 k=4 g=2 FPGA requires 64*(16+6+6+3+3) bits = 2176b.
The simplest implementation is eight 8x34b DFF-SRAMs but this requires write
port address decoders and write enables and read port address decoders
and mux trees.

Even if the bits themselves are stored in the smallest DFFs,
i.e. sky130_fd_sc_hddfxtp_1's @ 20um2, this uses almost half
(43520um2/90000um2) of the subproject area budget.

However we do not require random access to LUTs, just serial access.
Therefore if we keep LUT configs in a serial shift register, we can
do away with everything except for the DFFs themselves and their
clock drivers.

By using shift registers sans clock enables, shifting each cycle,
we can also avoid using a feedback clock enable gate for every DFF,
also a huge area savings, enabling a larger FPGA.

The downside is the shift registers are always shifting, always
burning power. To mitigate this somewhat, and to be a good neighbor on
a multiproject Caravel project, on reset or project-deactivate the LUT
configs shift regs are loaded with all 0s.

## Ideas

Idea: reduce area of each LUT input result forwarding mux input by
replacing lg(w)-bit comparison against variable lilut_1 with comparison
with constant ~0.

Idea: add an output bit mask per cluster and only shift those outputs
into the global nets.

Idea: as globals are fetched during serial LUT evaluation,
cache them in a shift register, accessible as more "local nets",
so that if some globals are reused they can be re-referenced
as local nets.

Idea: reviewing the two adder examples above, we see each adder LUT TTs
are redundant from LUT to LUT and there is a constant stride of two of
the LUT inputs (,,+1,+1). It may be possible to to prefix the LUT with
a code to RLE that. For example:

* prefix: meaning
* 2'd0: 1x LUT(,,,,)
* 2'd1: 4x LUT(,,,+1)
* 2'd2: 4x LUT(,,+1,+1))

This requires either variously 1b/4b wide LUTs and input buses or 4 cycles
of existing 1b wide LUTs. For the latter the cluster's LUT shift register
storage bits would have to be (significantly larger area) clock enabled
D-FFs which could counterintuitively reduce the LUT capacity of the FPGA.

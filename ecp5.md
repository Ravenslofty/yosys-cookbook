# `synth_ecp5`: Synthesis for Lattice ECP5 chips

## TL;DR

You'll need Yosys master, and nextpnr master; keep these in sync as changes can break the netlist format between the two.

High speed: Use `yosys -p "synth_ecp5 -abc9 -json filename.json" [files]` to produce a netlist for nextpnr, and then use nextpnr's `--json filename.json` argument to load it.

Reduced area: Use `yosys -p "synth_ecp5 -abc9 -nowidelut -json filename.json" [files]` to produce a netlist for nextpnr, and then use nextpnr's `--json filename.json` argument to load it.

The "high speed" option muxes together ECP5 LUT4s to produce larger LUTs. This reduces critical path delay but uses up significant design area.
If your design does not fit in your chip with the high speed option, consider retrying with `-nowidelut` for reduced area at the cost of performance.

If you're willing to spend a bit more time, you can generally get a more efficient mapping with a command like `yosys -p "scratchpad -copy abc9.script.flow3 abc9.script; synth_ecp5 -abc9 -json filename.json" [files]`.
This runs FPGA mapping several times, producing a generally-better mapping at the cost of increased runtime.

## Yosys options

### Useful

- `-top` - generally Yosys can autodetect the top-level module of your code; if it gets it wrong or you want to select a different top-level module, you can use `-top modulename` to set it.
- `-noflatten` - Yosys generally inlines modules for efficiency, but `-noflatten` tells it not to: this gives you per-module synthesis statistics that can be used to discover expensive modules that can be optimised.
- `-abc9` - ABC9 can better optimise carry chains and produces a faster, smaller, netlist than the default ABC mapping used when `-abc9` is not specified. If you are struggling to meet a timing target, use `-abc9`; it can be just that bit more that you need.

### Not recommended

- `-dff` - this makes LUT mapping DFF-aware, and can sometimes remove flops. However, the optimisations are generally not worth it, and debugging is trickier when registers are moved.
- `-retime` - like `-dff`, but it can move flops too to balance logic. Cannot be used in combination with `-abc9`, and ECP5 really benefits from that option.
- `-abc2` - this runs further logic optimisation before LUT mapping for modest savings in area. Cannot be used in combination with `-abc9`, and ECP5 really benefits from that option.
- `-asyncprld` - ECP5 flops support the asynchronous load of a value into a flop, and this can be used for latches or set/reset flops. However, Lattice don't use it in synthesis, so it's unclear if it's safe to rely on.

### Not as useful as you think: `-nowidelut`

By default, synthesis will use ECP5's hard multiplexers to connect LUT4s together to produce larger LUTs to reduce delay; disabling this reduces LUT duplication and improves routability, but increases netlist delay.

ABC9 often finds a very low delay mapping and so does not have much opportunity to reduce area without sacrificing delay.
I see a lot of people thinking that `-nowidelut` solves the area problem while not being aware of the delay problem.

I think a better solution is to more closely inform ABC9 of the actual delay target involved here:
- take your target frequency in megahertz (for this example I will be using 50 MHz)
- double it to account for delay in routing (so, 100 MHz in my example).
- convert this to picoseconds, rounding up if necessary (There are 1e12 picoseconds in a second, so `1e12 / 100e6 = 10000` picoseconds for a 100MHz target)
- add `scratchpad -set abc9.D <picoseconds>` before your call to `synth_ecp5 -abc9` (in my example this is `scratchpad -set abc9.D 10000`)

I've found doubling the target frequency to be a good starting point between giving ABC9 room to recover area while not missing the initial delay target.
If this results in a network substantially above your target frequency, feel free to increase `abc9.D`, and conversely if it fails the timing target, decrease `abc9.D`.

### Alternative netlist formats

- `-blif` - this produces a [BLIF] representation of the netlist. Nothing actually uses this, though.
- `-edif` - this produces an [EDIF] representation of the netlist.
- `-json` - this produces a JSON representation of the netlist that nextpnr uses.
- `-vpr` - this produces a [BLIF] representation of the netlist, but it's "experimental and incomplete" and you should use nextpnr instead.

### Debug options

- `-run [start label]:[stop label]` - `synth_ecp5` is made up of smaller passes; with this command you can stop and explore the netlist at a certain point.
- `-nocc2` - this disables the use of carry chains, using Brent-Kung addition in logic instead. This generally slows down the design and increases area, but can be used to check for bugs in carry chain mapping.
- `-nodffe` - this disables the use of flops with clock enables, which can result in less routing congestion and expose optimisations. You can get finer control over this with `-dffe_min_ce_use` though.
- `-nobram` - this disables the use of block RAM for memories, forcing Yosys to build memories out of LUTRAM and flops. This majorly increases design area and reduces performance, but can be used to check for bugs in memory mapping.
- `-nolutram` - this disables the use of LUT RAM for memories, forcing Yosys to build memories out of flops. This increases design area and reduces performance, but can be used to check for bugs in memory mapping.
- `-nodsp` - this disables the use of DSP slices for multipliers, and instead multipliers are built as shift/add cascades. This can be used to check for bugs in multiplier mapping.

[BLIF]: https://www.cse.iitb.ac.in/~supratik/courses/cs226/spr16/blif.pdf
[EDIF]: https://www.iue.tuwien.ac.at/phd/minixhofer/node53.html
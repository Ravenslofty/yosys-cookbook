# `synth_ice40`: Synthesis for Lattice iCE40 chips

## TL;DR

You'll need Yosys master, and nextpnr master; keep these in sync as changes can break the netlist format between the two.

iCE40HX and iCE40LP: Use `yosys -p "synth_ice40 -json filename.json" [files]` to produce a netlist for nextpnr, and then use nextpnr's `--json filename.json` argument to load it.

iCE40UP (UltraPlus): Use `yosys -p "synth_ice40 -dsp -json filename.json" [files]` to produce a netlist for nextpnr, and then use nextpnr's `--json filename.json` argument to load it. Since the UltraPlus chips have hard multipliers, it makes sense to use them with `-dsp`.

## Yosys options

### Useful

- `-top` - generally Yosys can autodetect the top-level module of your code; if it gets it wrong or you want to select a different top-level module, you can use `-top modulename` to set it.
- `-noflatten` - Yosys generally inlines modules for efficiency, but `-noflatten` tells it not to: this gives you per-module synthesis statistics that can be used to discover expensive modules that can be optimised.
- `-abc2` - this runs further logic optimisation before LUT mapping for modest savings in area. Cannot be used in combination with `-abc9`.

### Sometimes useful

- `-dffe_min_ce_use N` - if you're having routing congestion problems, one cause could be lots of small flop clock-enables. This option tells Yosys not to use a DFFE if the clock enable line goes to less than *N* flops.
- `-abc9 -device < hx | lp | u >` - ABC9 can better optimise carry chains and *can* produce a faster, if slightly larger, netlist. The `-device` argument optimises for a specific subfamily. For iCE40 it seems to behave worse than the default ABC, so try it and see.

### Not recommended

- `-dff` - this makes LUT mapping DFF-aware, and can sometimes remove flops. However, the optimisations are generally not worth it, and debugging is trickier when registers are moved.
- `-retime` - like `-dff`, but it can move flops too to balance logic. Cannot be used in combination with `-abc9`.
- `-noabc` - this disables using ABC for LUT mapping, falling back to blind LUT mapping. The resulting netlist is high area and slow. If you really can't use ABC, use `-flowmap`.
- `-flowmap` - this disables using ABC for LUT mapping, falling back to using the `flowmap` pass in Yosys. This is much better than `-noabc` in terms of speed, but still high area.

### Alternative netlist formats

- `-blif` - this produces a [BLIF] representation of the netlist that arachne-pnr uses, but you should use nextpnr instead.
- `-edif` - this produces an [EDIF] representation of the netlist. While this could be used for Yosys synthesis to iCECube, the latter crashes on Yosys input.
- `-json` - this produces a JSON representation of the netlist that nextpnr uses.
- `-vpr` - this produces a [BLIF] representation of the netlist, but it's "experimental and incomplete" and you should use nextpnr instead.

### Debug options

- `-run [start label]:[stop label]` - `synth_ice40` is made up of smaller passes; with this command you can stop and explore the netlist at a certain point.
- `-nocarry` - this disables the use of carry chains, using Brent-Kung addition in logic. This generally slows down the design and bloats it, but can be used to check for bugs in carry chain mapping.
- `-nodffe` - this disables the use of flops with clock enables, which can result in less routing congestion and expose optimisations. You can get finer control over this with `-dffe_min_ce_use` though.
- `-nobram` - this disables the use of block RAM for memories, forcing Yosys to build memories out of flops. This generally bloats the design, but can be used to check for bugs in memory mapping.

[BLIF]: https://www.cse.iitb.ac.in/~supratik/courses/cs226/spr16/blif.pdf
[EDIF]: https://www.iue.tuwien.ac.at/phd/minixhofer/node53.html

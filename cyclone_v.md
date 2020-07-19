# `synth_intel_alm` - Synthesis for Intel Cyclone V chips

## TL;DR

You'll need Yosys master and Quartus 18.1 or later.

First, create a skeleton Quartus Settings File as below - I'm going to call it `example.qsf`, but whatever you name it, Quartus will refer to it by the base name of the file, which is `example` here.

```tcl
set_global_assignment -name FAMILY "Cyclone V"
set_global_assignment -name VQM_FILE filename.vqm
set_global_assignment -name DEVICE <YOUR_PART_NAME_HERE>
set_global_assignment -name TOP_LEVEL_ENTITY <YOUR_TOP_LEVEL_MODULE_NAME>
```

`<YOUR_PART_NAME_HERE>` is the part name of your chip, e.g. the DE10-Nano has a `5CSEBA6U23I7`. `<YOUR_TOP_LEVEL_MODULE_NAME>` is the module name of your top-level module.

Use `yosys -p "synth_intel_alm -family cyclonev -vqm filename.vqm" [files]` to produce a netlist for Quartus.
Then, run `quartus_map -c example example` to get Quartus to read the netlist, and `quartus_fit -c example example` to place and route it.

## Yosys options

### Useful

- `-top` - generally Yosys can autodetect the top-level module of your code; if it gets it wrong or you want to select a different top-level module, you can use `-top modulename` to set it.
- `-noflatten` - Yosys generally inlines modules for efficiency, but `-noflatten` tells it not to: this gives you per-module synthesis statistics that can be used to discover expensive modules that can be optimised.

### Sometimes useful

- `-dff` - this makes LUT mapping DFF-aware, and can remove flops. However, debugging is trickier when registers are moved. I'm considering making this the default.

### Alternative netlist formats

- `-vqm` - this produces a Verilog Quartus Mapping representation of the netlist, needed for Quartus.

### Alternative device families

- `-family < cyclonev | cyclone10gx >` - `synth_intel_alm` can be used to synthesize for Cyclone V, or experimentally Cyclone 10 GX; to select the latter, you would use `-family cyclone10gx`.

### Debug options

- `-quartus` - this outputs a netlist using Quartus cells, rather than the internal MISTRAL_* cells. See the note on cell formats.
- `-run [start label]:[stop label]` - `synth_intel_alm` is made up of smaller passes; with this command you can stop and explore the netlist at a certain point.
- `-nobram` - this disables the use of block RAM for memories, forcing Yosys to build memories out of LUTRAM and flops. This majorly increases design area and reduces performance, but can be used to check for bugs in memory mapping.
- `-nolutram` - this disables the use of LUT RAM for memories, forcing Yosys to build memories out of flops. This increases design area and reduces performance, but can be used to check for bugs in memory mapping.
- `-nodsp` - this disables the use of DSP slices for multipliers, and instead multipliers are built as shift/add cascades. This can be used to check for bugs in multiplier mapping.

## Quartus options

While I won't make this a comprehensive list of Quartus settings, there are a few options you can add to the Quartus Settings File to improve the quality of result.

- `set_global_assignment -name OPTIMIZATION_MODE "AGGRESSIVE PERFORMANCE"` - this tells Quartus to look harder for the best place and route solution, and is generally an easy performance gain if you have the time.
- `set_global_assignment -name ADV_NETLIST_OPT_SYNTH_WYSIWYG_REMAP ON` - this essentially runs Quartus synthesis on the netlist that Yosys produced; it can increase performance but equally make it much worse.

## Note on cell formats

The `synth_intel_alm` pass uses internal `MISTRAL_*` cells, rather than Quartus cells, because the Quartus cells are family-dependent and don't reflect actual hardware capabilities. The `MISTRAL` cells are easier for Yosys to manipulate.
Here's a list of them and what they represent.

|  Yosys cell                     |  Corresponding Quartus cell       |                           Description                              |
|---------------------------------|-----------------------------------|--------------------------------------------------------------------|
| `MISTRAL_ALUT[23456]`           | `cyclonev_lcell_comb`             | A LUT; the MISTRAL cell describes the size of the LUT.             |
| `MISTRAL_NOT`                   | `cyclonev_lcell_comb`/`NOT`       | An inverter; Yosys doesn't always push them into LUTs.             |
| `MISTRAL_ALUT_ARITH`            | `cyclonev_lcell_comb`             | A LUT connected to carry logic; often produced by addition.        |
| `MISTRAL_FF`                    | `cyclonev_ff`/`dffeas`            | A flip-flop, with enable, asynchronous clear and synchronous load. |
| `MISTRAL_MLAB`                  | `cyclonev_mlab_cell`              | A 32-address by 1-bit LUT RAM cell.                                |
| `MISTRAL_M10K`                  | `cyclonev_ram_block`/`altsyncram` | An M10K 10-kilobit block RAM.                                      |
| `MISTRAL_MUL_{9X9,18X18,27X27}` | `cyclonev_mac`                    | A 9x9/18x18/27x27 DSP slice, used for multiplication.              |

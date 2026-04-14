---
layout: post
title: "Getting Started with xviv: FPGA Development Without the GUI"
date: 2025-01-01
categories: [fpga, vivado, tutorial]
---

# Getting Started with xviv

`xviv` is a command-line controller that drives Vivado and Vitis in non-project (batch) mode from a single `project.toml` file. No `.xpr` files, no stale GUI state — just reproducible, scriptable FPGA builds.

This tutorial walks through a complete project from scratch: packaging a custom IP, building a Block Design, synthesising for hardware, and loading an embedded application onto a Pynq Z2.

---

## Prerequisites

- Vivado 2024.1 installed at `/opt/Xilinx/Vivado/2024.1`
- Vitis 2024.1 (required for embedded sections)
- Python 3.11+
- A Pynq Z2 board (or substitute your own part throughout)

```bash
pip install xviv pyslang argcomplete
eval "$(register-python-argcomplete xviv)"
```

---

## 1. Project Structure

Create a new project directory:

```
my_project/
├── project.toml
└── srcs/
    ├── rtl/
    │   └── counter.sv
    ├── xdc/
    │   └── pynqz2.xdc
    └── sw/
        └── firmware/
            └── main.c
```

Write a minimal RTL module to use throughout this tutorial:

```systemverilog
// srcs/rtl/counter.sv
module counter #(
    parameter int WIDTH = 8
)(
    input  logic        aclk,
    input  logic        aresetn,

    // AXI-Stream output
    output logic [WIDTH-1:0] m_axis_tdata,
    output logic             m_axis_tvalid,
    input  logic             m_axis_tready
);
    always_ff @(posedge aclk) begin
        if (!aresetn) begin
            m_axis_tdata  <= '0;
            m_axis_tvalid <= 1'b0;
        end else begin
            m_axis_tvalid <= 1'b1;
            if (m_axis_tready)
                m_axis_tdata <= m_axis_tdata + 1;
        end
    end
endmodule
```

---

## 2. project.toml

```toml
# project.toml

[fpga]
part       = "xc7z020clg400-1"
board_part = "tul.com.tw:pynq-z2:part0:1.0"
board_repo = "/opt/board_files/pynq-z2"

[vivado]
path        = "/opt/Xilinx/Vivado/2024.1"
mode        = "batch"
max_threads = 8

[vitis]
path = "/opt/Xilinx/Vitis/2024.1"

[build]
dir         = "build"
ip_repo     = "build/ip"
bd_dir      = "build/bd"
wrapper_dir = "build/wrapper"

[sources]
rtl = ["srcs/rtl/**/*.sv"]
xdc = ["srcs/xdc/*.xdc"]
sim = ["srcs/sim/**/*.sv"]

[[ip]]
name    = "counter"
vendor  = "myorg.com"
library = "user"
version = "1.0"
top     = "counter"
rtl     = ["srcs/rtl/counter.sv"]

[[bd]]
name       = "system"
xdc        = ["srcs/xdc/pynqz2.xdc"]
export_tcl = "scripts/bd/system.tcl"

[[synthesis]]
top          = "system_wrapper"
xdc          = ["srcs/xdc/pynqz2.xdc"]
report_synth = true
report_route = true

[[platform]]
name      = "pynqz2_bsp"
cpu       = "ps7_cortexa9_0"
os        = "standalone"
synth_top = "system_wrapper"

[[app]]
name     = "firmware"
platform = "pynqz2_bsp"
template = "empty_application"
src_dir  = "srcs/sw/firmware"
```

---

## 3. Creating a Custom IP

### 3a. Generate the Hooks File

Before packaging the IP, generate a hooks file. This is where you customise bus interface inference, parameter layout, and memory maps.

```bash
xviv config --ip counter
# Edit: scripts/ip/counter_1.0.tcl
```

The generated file at `scripts/ip/counter_1.0.tcl` contains empty proc stubs. For the counter IP, the default AXI-Stream inference is automatic — no changes needed yet. But here is what the hooks look like with a customisation added:

```tcl
# scripts/ip/counter_1.0.tcl

# Called after default AXI-Stream / AXI-MM inference.
# Add inferences for any additional bus standards your IP uses.
proc ipx_infer_bus_interfaces {} {
    # Clock and reset are inferred automatically from port names
    # (aclk -> clock signal, aresetn -> reset signal).
    # Nothing extra needed for this IP.
}

# Called after HDL parameters are added to the IP GUI.
# Use to reorder, group, or rename parameters.
proc ipx_add_params {} {
    # Move WIDTH to the top of Page 0
    ipgui::move_param -component [ipx::current_core] \
        -order 0 \
        [ipgui::get_guiparamspec -name "WIDTH" \
            -component [ipx::current_core]] \
        -parent [ipgui::get_pagespec -name "Page 0" \
            -component [ipx::current_core]]
}

proc synth_pre   {} {}
proc synth_post  {} {}
proc place_post  {} {}
proc route_post  {} {}
proc bitstream_post {} {}
```

### 3b. Package the IP

```bash
xviv create --ip counter
```

Vivado runs in batch mode. You will see output like:

```
INFO: [+0m00s] Scaffolding IP skeleton - counter_1_0
INFO: [+0m01s] Stripping default AXI-Lite scaffold
INFO: [+0m02s] Adding RTL sources
INFO: [+0m04s] Inferring bus interfaces
INFO: Inferring AXI-Stream interfaces
INFO: Inferring AXI-MM interfaces
INFO: Sourced hooks - scripts/ip/counter_1.0.tcl
INFO: [+0m06s] Exposing HDL parameters
Name: WIDTH | Value: 8
INFO: [+0m07s] Wiring AXI-Lite memory maps
INFO: [+0m08s] Finalising and saving IP
INFO: IP saved - counter_1_0
INFO: IP creation complete - 0m09s
```

The packaged IP lives at `build/ip/counter_1_0/component.xml`.

### 3c. Editing an Existing IP

If you change your RTL or want to add ports, reopen the IP in the Vivado IP Packager GUI:

```bash
xviv edit --ip counter
```

Vivado opens the GUI with the IP edit project loaded. The IP Packager sidebar will appear on the left.

**In the GUI — typical editing workflow:**

1. **Identification** tab — confirm vendor, library, name, version.
2. **Compatibility** — add supported Vivado versions if needed.
3. **File Groups** — drag additional source files in, or re-scan.
4. **Customization Parameters** — reorder or rename parameters.
5. **Ports and Interfaces** — verify that `M_AXIS`, `aclk`, and `aresetn` were correctly inferred. If a port is miscategorised, right-click → *Add Bus Interface* → assign logical port mappings manually.
6. **Review and Package** → click **Re-Package IP**.

After the GUI closes, `component.xml` is updated in place. Commit the entire `build/ip/counter_1_0/` directory to version control to share the packaged IP.

> **Tip:** Run `xviv create --ip counter` (not `edit`) when you have changed the RTL port list. The `edit` command assumes the port list is stable and focuses on packaging metadata.

---

## 4. Creating a Block Design

### 4a. Generate the BD Hooks File

```bash
xviv config --bd system
# Edit: scripts/bd/system_hooks.tcl
```

The hooks file controls what happens when `create --bd` runs. The default stub checks for an exported TCL script and, if found, sources it automatically to recreate the BD. If no exported TCL exists, it opens the Vivado GUI.

```tcl
# scripts/bd/system_hooks.tcl  (auto-generated, typically unchanged)
set ::_bd_design_tcl [file join [file dirname [info script]] "system.tcl"]

proc bd_design_config { parentCell } {
    global _bd_design_tcl

    if {[file exists $_bd_design_tcl]} {
        puts "INFO: Sourcing exported BD TCL - $_bd_design_tcl"
        source $_bd_design_tcl
        xviv_refresh_bd_addresses
        validate_bd_design
        save_bd_design
        exit 0
    } else {
        puts "INFO: No exported BD TCL found — opening GUI."
        start_gui
    }
}
```

### 4b. Create the BD (Interactive)

On first creation there is no exported TCL, so Vivado opens the GUI:

```bash
xviv create --bd system
```

Vivado opens with an empty Block Design canvas named `system`.

**In the GUI — building the Block Design:**

#### Step 1: Add the Processing System (Zynq PS)

1. Click **+** (Add IP) in the toolbar, or press `Ctrl+I`.
2. Search for **ZYNQ7 Processing System** → double-click to add.
3. Click **Run Block Automation** (the green banner at the top of the canvas) → **OK**. This applies your board's preset, connecting DDR and fixed IO automatically.

#### Step 2: Add Your Custom IP

1. Press `Ctrl+I` → search for **counter** → double-click.
2. The IP appears on the canvas. Double-click it to open the customisation dialog.
3. Set **WIDTH** to `32` → **OK**.

#### Step 3: Add AXI DMA (to move AXI-Stream data to PS memory)

1. Press `Ctrl+I` → search **AXI Direct Memory Access** → double-click.
2. In the customisation dialog, disable **Write Channel** (we only need read from PS perspective) → **OK**.
3. Click **Run Connection Automation** → enable `S_AXI_LITE` and `M_AXI_MM2S` → **OK**.

#### Step 4: Connect the AXI-Stream Path

Connect the counter AXI-Stream output to the DMA stream input manually:

1. Hover over the `M_AXIS` port on the counter IP. The cursor becomes a pencil.
2. Click and drag to the `S_AXIS_S2MM` port on the AXI DMA.

> **Tip:** Use **Run Connection Automation** aggressively — it handles clock, reset, and AXI Interconnect wiring automatically. You only need manual connections for domain-specific paths like AXI-Stream data flows.

#### Step 5: Validate and Save

1. **Tools → Validate Design** (or `F6`). Resolve any critical warnings.
2. **File → Save Block Design** (`Ctrl+S`).

#### Step 6: Export the BD as TCL

Close Vivado (or keep it open), then run:

```bash
xviv export --bd system
# Exported : scripts/bd/system_abc1234.tcl
# Symlink  : scripts/bd/system.tcl -> system_abc1234.tcl
```

This creates a git-tagged, fully self-contained TCL that recreates the exact BD. Commit `scripts/bd/system.tcl` (the symlink) and `scripts/bd/system_abc1234.tcl` to version control.

From now on, `xviv create --bd system` will recreate the BD automatically from the exported TCL — no GUI needed.

### 4c. Editing an Existing Block Design

To modify the BD later:

```bash
xviv edit --bd system
```

Vivado opens the GUI with `system.bd` loaded. Make your changes, then re-export:

```bash
xviv export --bd system
```

The previous versioned TCL file is kept. The `system.tcl` symlink is updated to point at the new export. Old exports are safe to delete once you have confirmed the new one works.

### 4d. Connecting BD Components — Reference

| Task | How |
|---|---|
| Add an IP | `Ctrl+I`, search, double-click |
| Auto-connect | **Run Connection Automation** banner → select ports → OK |
| Manual wire | Hover port → pencil cursor → click-drag to target |
| Delete a net | Click the wire → `Delete` |
| Resize/navigate | Scroll to zoom, middle-drag to pan |
| Make external | Right-click port → **Make External** (creates a top-level BD port) |
| Validate | `F6` or **Tools → Validate Design** |
| Re-customise an IP | Double-click the IP block |
| Regenerate addresses | **Tools → Reassign All Segment Addresses** |
| Hierarchy | Right-click selection → **Create Hierarchy** |

---

## 5. Generating BD Output Products

After creating or editing the BD, generate the synthesis and implementation output products and the Verilog wrapper:

```bash
xviv generate --bd system
```

Output:

```
INFO: Generating Block Design output products - system
INFO: BD generation complete - 0m45s
```

The BD wrapper Verilog is copied to `build/wrapper/system_wrapper.v`. This is the top-level module for synthesis.

---

## 6. Synthesis

Run the full synthesis → placement → routing → bitstream flow:

```bash
xviv synth --top system_wrapper
```

Vivado runs in batch mode. Progress is printed with elapsed timestamps:

```
INFO: [+0m00s] Synthesis - system_wrapper  (sha: abc1234)
INFO: [+4m32s] Placement
INFO: [+7m18s] Routing
INFO: [+9m01s] Generating bitstream
INFO: Symlink updated: system_wrapper.bit -> system_wrapper_abc1234.bit
INFO: Build manifest written - build/synth/system_wrapper/build.json
INFO: Build complete - 9m22s
```

Output files in `build/synth/system_wrapper/`:

```
post_synth.dcp
post_place.dcp
post_route.dcp
system_wrapper_abc1234.bit
system_wrapper_abc1234.xsa
system_wrapper.bit          → symlink
system_wrapper.xsa          → symlink
build.json
reports/
    post_synth_timing_summary.rpt
    post_route_timing_summary.rpt
    post_route_drc.rpt
```

### Inspecting a Checkpoint

Open any checkpoint in the Vivado GUI for timing analysis, resource utilisation, or routing visualisation:

```bash
xviv open --dcp post_route --top system_wrapper
```

Vivado opens with the design fully loaded. Use **Reports → Timing Summary**, **Reports → Utilization**, or the **Device** view to inspect routing.

---

## 7. OOC Synthesis for Large BDs

For BDs with many IPs, out-of-context synthesis pre-compiles each IP separately. Only IPs whose `component.xml` has changed are re-synthesised on subsequent runs.

Enable `report_synth = true` in `project.toml` for a particular synthesis entry, then:

```bash
xviv synth --bd system --ooc-run
```

Phase 1 synthesises each leaf IP out-of-context and writes a DCP:

```
INFO: [+0m00s] OOC synthesis: counter_0  (top: counter  xilinx: false)
INFO: OOC checkpoint written - build/synth/system_wrapper/ooc/counter_0/post_synth.dcp
INFO: [+0m48s] OOC synthesis: axi_dma_0  (top: axi_dma_0  xilinx: true)
INFO: OOC DCP up to date, skipping - ps7_0_axi_periph
...
INFO: [+2m10s] BD wrapper synthesis: system_wrapper
```

Phase 2 synthesises the BD wrapper with black-box stubs for each IP, then loads the DCPs.

---

## 8. Creating the BSP Platform

Once you have a bitstream and XSA, generate the Board Support Package:

```bash
xviv create --platform pynqz2_bsp
```

```
INFO: Generating BSP platform
INFO:   XSA    : build/synth/system_wrapper/system_wrapper.xsa
INFO:   CPU    : ps7_cortexa9_0
INFO:   OS     : standalone
INFO:   BSP dir: build/bsp/pynqz2_bsp
INFO: BSP generated at build/bsp/pynqz2_bsp
```

Build the BSP libraries:

```bash
xviv build --platform pynqz2_bsp
```

---

## 9. Creating and Building the Application

### 9a. Scaffold the Application

```bash
xviv create --app firmware
```

This generates the Makefile and linker script in `build/app/firmware/` using the `empty_application` template.

### 9b. Write Your Firmware

Put your C sources in `srcs/sw/firmware/`:

```c
// srcs/sw/firmware/main.c
#include "xaxidma.h"
#include "xparameters.h"
#include "xil_printf.h"

#define DMA_DEV_ID  XPAR_AXIDMA_0_DEVICE_ID
#define RX_BUFFER_BASE  0x01000000
#define TRANSFER_LEN    256

int main(void) {
    XAxiDma dma;
    XAxiDma_Config *cfg;

    xil_printf("Counter DMA demo\r\n");

    cfg = XAxiDma_LookupConfig(DMA_DEV_ID);
    XAxiDma_CfgInitialize(&dma, cfg);

    // Start an S2MM (stream to memory) transfer
    u32 *rx_buf = (u32 *)RX_BUFFER_BASE;
    XAxiDma_SimpleTransfer(&dma, (UINTPTR)rx_buf,
                           TRANSFER_LEN * sizeof(u32),
                           XAXIDMA_DEVICE_TO_DMA);

    // Wait for completion
    while (XAxiDma_Busy(&dma, XAXIDMA_DEVICE_TO_DMA));

    xil_printf("Received %d words. First: 0x%08X\r\n",
               TRANSFER_LEN, rx_buf[0]);
    return 0;
}
```

### 9c. Build the Application

```bash
xviv build --app firmware --info
```

```
=== ELF size: firmware.elf ===
   text    data     bss     dec
  14320     512    8192   23024

=== ELF sections: firmware.elf ===
firmware.elf:     file format elf32-microblaze
Idx Name          Size      VMA
  0 .text         0x37f0    0x00000000
  1 .data         0x0200    0x00040000
  2 .bss          0x2000    0x00040200
```

---

## 10. Programming the FPGA

Connect the board via USB/JTAG, start `hw_server` if not already running:

```bash
/opt/Xilinx/Vivado/2024.1/bin/hw_server &
```

Program bitstream and ELF in a single command:

```bash
xviv program --platform pynqz2_bsp --app firmware
```

```
INFO: Programming FPGA
INFO:   Bitstream : build/synth/system_wrapper/system_wrapper.bit
INFO:   ELF       : build/app/firmware/Debug/firmware.elf
INFO:   hw_server : localhost:3121
INFO: JTAG targets:
  1  APU
     2  ARM Cortex-A9 MPCore #0 (Running)
  3  xc7z020
INFO: Bitstream programmed successfully
INFO: ELF loaded: firmware.elf
INFO: Processor running
```

Program only the bitstream (no ELF):

```bash
xviv program --bitstream build/synth/system_wrapper/system_wrapper.bit
```

Reload ELF without re-programming the FPGA:

```bash
# After rebuilding only the app:
xviv program --platform pynqz2_bsp --app firmware
```

---

## 11. Full Workflow Summary

```bash
# 1. Package the custom IP
xviv config --ip counter          # generate hooks file
# (edit scripts/ip/counter_1.0.tcl as needed)
xviv create --ip counter           # package the IP

# 2. Build the Block Design
xviv config --bd system            # generate BD hooks
xviv create --bd system            # open GUI → build → close GUI
xviv export --bd system            # export BD as versioned TCL
xviv generate --bd system          # generate output products + wrapper

# 3. Synthesise
xviv synth --top system_wrapper    # synth → place → route → bitstream

# 4. BSP + App
xviv create --platform pynqz2_bsp
xviv build --platform pynqz2_bsp
xviv create --app firmware
xviv build --app firmware

# 5. Program
xviv program --platform pynqz2_bsp --app firmware

# ── On subsequent runs (BD already exported) ──────────────────
# git pull (system.tcl updated)
xviv create --bd system            # fully automated — no GUI
xviv generate --bd system
xviv synth --top system_wrapper
xviv build --app firmware
xviv program --platform pynqz2_bsp --app firmware
```

---

## Appendix: Common Patterns

### Selecting a Named FPGA Target

```toml
[fpga.pynqz2]
part       = "xc7z020clg400-1"
board_part = "tul.com.tw:pynq-z2:part0:1.0"

[fpga.artix7]
part = "xc7a35tcpg236-1"

[[synthesis]]
top  = "my_top"
fpga = "artix7"    # overrides default FPGA for this entry
```

### OOC-Only Module Synthesis

```toml
[[synthesis]]
top            = "my_filter"
rtl            = ["srcs/ip/my_filter/**/*.sv"]
out_of_context = true
xdc_ooc        = ["srcs/xdc/my_filter_ooc.xdc"]
```

### Custom IP with Auto-Generated Wrapper

For IPs that use SystemVerilog interface ports, `create_wrapper = true` generates a flat wrapper automatically:

```toml
[[ip]]
name           = "dsp_core"
top            = "dsp_core"
rtl            = ["srcs/ip/dsp_core/**/*.sv"]
create_wrapper = true
```

```bash
xviv create --ip dsp_core
# Parses dsp_core.sv with pyslang
# Generates build/wrapper/dsp_core_wrapper.sv
# Packages dsp_core_wrapper as the IP top
```

### Checking the Build Manifest

Each successful synthesis writes `build/synth/<top>/build.json`:

```json
{
  "vivado_version": "2024.1",
  "part": "xc7z020clg400-1",
  "top": "system_wrapper",
  "sha_tag": "abc1234",
  "sha_short": "abc1234",
  "dirty": "false",
  "mode": "global",
  "bitstream": "system_wrapper_abc1234.bit",
  "xsa": "system_wrapper_abc1234.xsa",
  "elapsed": "9m22s",
  "timestamp": "2025-01-01T12:00:00Z"
}
```

### Incremental Waveform Reload

After changing RTL and re-running elaboration, reload the waveform without closing xsim:

```bash
xviv elab --top tb_counter
xviv open --snapshot --top tb_counter
# (in another terminal, after changing RTL and re-elabbing:)
xviv reload --snapshot --top tb_counter
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `FPGA part 'xc7...' is not in the installed catalog` | Wrong part string or Vivado not sourced | Check `[fpga] part` in `project.toml`; source Vivado settings |
| `BD file not found` | `create --bd` or `generate --bd` not run yet | Run `xviv create --bd <n>` then `xviv generate --bd <n>` |
| `No RTL source files were added` | Glob pattern matched nothing | Check `rtl = [...]` paths relative to `project.toml` |
| `Hooks file configured but does not exist` | `hooks` path wrong or file deleted | Run `xviv config --ip <n>` to regenerate, or fix the path |
| OOC DCP always rebuilt | `component.xml` timestamp newer than DCP | Touch the DCP or delete it to force a clean rebuild |
| `No FPGA target found on JTAG` | `hw_server` not running or board not connected | Start `hw_server`; check USB cable and power |
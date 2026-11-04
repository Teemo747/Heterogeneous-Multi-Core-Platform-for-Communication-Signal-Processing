# 6-node FLX/DLX mixed platform

This is a derived, parameterized RTL workspace for the legacy 2026FPGA sources. The source generator reads only `旧有平台/code` and `dlx_cnn/dlx`; it does not alter either source tree. `generated/` is reproducible output and is ignored by Git. Run the generator before opening the Vivado Tcl scripts.

## Topology

| Node | Endpoint | PM word | Role |
|---:|---|---:|---|
| 0 | `MC_HYBRID` | n/a | Host-facing old main controller |
| 1 | `PE_HYBRID`, FLX | 128 bits | Legacy vector core |
| 2 | `PE_HYBRID`, FLX | 128 bits | Legacy vector core |
| 3 | Legacy `mem_node` | n/a | DDR endpoint |
| 4 | `PE_HYBRID`, DLX CNN | 192 bits | CNN core |
| 5 | `PE_HYBRID`, DLX CNN | 192 bits | CNN core |

The six NoC node identifiers and mesh coordinates remain those of the selected old 3x2 network. Nodes 0 and 3 are boundary endpoints; `HYBRID_COMPUTE_FABRIC` exposes simple 100-bit NI-side ports for them and integrates the four real PE wrappers with the actual old NI/router RTL. It is useful as a focused platform boundary and simulation top. `TOP_HYBRID_6NODES` is the separate board-oriented top derived from the old platform, retaining its DDR4, Zynq 32-bit link and host controls.

## RTL layout

- `rtl/PE_HYBRID.v`: old PE arbitration and CoP wiring, generated FLX/DLX selection, parameterized PM and inferred PM/CM/DM RAMs, corrected 16-entry NI receive FIFO.
- `rtl/CoP_HYBRID.v`: derived old 100-bit NI protocol engine, local PM width, stall-safe DM transmitter and accepted-tail receive accounting.
- `rtl/dlx_task_adapter.v`: supplies DLX metadata not present on its generated core interface; validates and holds task descriptors.
- `rtl/MainCtrl_HYBRID.v`, `rtl/MC_HYBRID.v`: the old main-controller flow with an algorithm-3 four-core option and two DLX configuration packets before START.
- `rtl/HYBRID_COMPUTE_FABRIC.v`: nodes 0-5, old NoC/NI and four mixed compute PEs, without DDR/host physical interface assumptions.
- `rtl/TOP_HYBRID_6NODES.v`: old board top with PE1/2=FLX and PE4/5=DLX. It preserves the legacy external pin list; protocol errors are available as an internal four-bit vector.
- `generated/cores/{flx,dlx}`: namespaced copies of the complete generated cores. Same-named modules are prefixed `HFLX_` and `HDLX_`; singleton-only simulation logging regions are removed, as synthesis already excludes them.
- `generated/noc`: selected 3x2 legacy network snapshot; unconnected mesh-edge input ports are tied idle.
- `sim`: self-checking protocol, adapter, RAM/FIFO, controller, PE, and actual six-node network simulations.
- `tools/prepare_sources.ps1`: fail-closed deterministic derivation. It aborts if expected source structure changes.
- `tools/add_sources.tcl`: adds only the generated mixed compute fabric sources to the current Vivado project.
- `tools/check_rtl.tcl`: Vivado RTL elaboration/synthesis check with deliberately small test memories.

## Protocol contract

The existing NI packet format is unchanged: `[99:85]` is flit number, `[84:70]` packet length, `[69:64]` source/task ID, and `[63:0]` payload. A data packet's flit 1 holds the destination mesh coordinate in payload `[63:58]`. Flit 2 holds `{destination_address[31:0], byte_count[19:0], memory_type[1:0], 10'b0}`. For PM, destination address is a native instruction-word index. For DM/CM, it is a byte address, converted by the legacy controller to a 64-byte-row address. Payload starts at flit 3, low 64-bit beat first. PM is memory type 3, DM is 1, and CM is 2. Preserve the legacy NI route and source/task fields.

PM now packs complete native instructions rather than assuming every core has the old 128-bit word. For a 128-bit FLX PM, two ordered 64-bit payload beats form one word. For a 192-bit DLX PM, three ordered beats form one word. A word may not be split between independently addressed downloads; begin each download on a native word boundary. Host-side firmware/compiler tools must use the selected node's width. The NoC/NI fragments a longer packet internally; the payload flit numbering and order remain continuous.

DLX has no FLX `StartAddr`, `TargetAddr`, `TargetNode`, `TaskNum`, `MemType`, or `RunFlag` outputs. This adapter contract maps DLX `Fetchaddr` to the source DM byte address, `Nextaddr` to the destination DM byte address, and `Datanum`/`Vartype` to the legacy CoP count fields. The existing generated DLX logic does not itself document these as a host-visible ABI; its compiler/program must follow this v1 mapping. The adapter always declares DM (type 1). It converts the core count to bytes for validation, requires both addresses to be 64-byte aligned, rejects zero/oversize transfers, and holds the descriptor stable through CoP use.

Before either DLX can be started, node 0 sends a two-flit configuration command (opcode `8'h80`) to that core while reset. The command header is the ordinary flit-1 route; the flit-2 payload is:

| Bits | Meaning |
|---|---|
| `[63:58]` | target node, 0-5 |
| `[57:52]` | number of input tasks (must be nonzero for receive modes 2/5) |
| `[51:50]` | memory type, must be DM=1 |
| `[49:47]` | legacy run mode 1-5 |
| `[46:40]` | reserved, zero |
| `[39:32]` | configuration version, 1 |
| `[31:16]` | magic `16'hd1a5` |
| `[15:8]` | opcode `8'h80` |
| `[7:0]` | reserved, zero |

Defaults select DM, run mode 1, zero receive tasks. Override `DLX4_CFG`/`DLX5_CFG` at the board top or main-controller instance for the actual application. Algorithm select 3 targets nodes 1, 2, 4, 5; algorithm 1 remains nodes 1, 2 and algorithm 2 remains nodes 4, 5. The controller configures nodes 4 and 5 before sending starts.

## Build and tests

From PowerShell at `2026FPGA`:

```powershell
.\hybrid_6nodes\tools\prepare_sources.ps1
.\hybrid_6nodes\sim\run_tests.ps1
```

Run a subset, for example `.\hybrid_6nodes\sim\run_tests.ps1 -Tests tb_fabric,tb_pe_hybrid` (Icarus Verilog at `C:\iverilog\bin` is expected). In Vivado 2024.1, from `hybrid_6nodes/build`, run:

```powershell
& 'D:\xilinx\Vivado\2024.1\bin\vivado.bat' -mode batch -source ..\tools\check_rtl.tcl
```

The Vivado script elaborates/synthesizes the compute fabric using 16-entry test RAMs, verifies exactly two FLX and two DLX core instances, and rejects unresolved black boxes. It is not a full-size resource estimate or a bitstream build.

## Verified and not yet verified

The default regression also runs `tb_paper_flow`, based on section 3.2 (PDF pages 27-30) of the supplied legacy platform paper. It tests two algorithms with two DM groups each, ACK/START ordering, peer dependency transfers, result returns, completion and algorithm switching through the real controller and six-node fabric. CPU result descriptors are driven by the testbench, and MEM is an NI-side behavioral model. This is a transport/control test, not CDMA numerical verification or a DDR/host test. Run it alone with `sim/run_tests.ps1 -Tests tb_paper_flow`. Waveforms and event/result CSVs are written to `build/paper_flow.*` and `build/paper_flow_*.csv`. See [the simulation report](sim/PAPER_SIMULATION_REPORT.md) for scope, evidence and remaining work.

The self-checking tests exercise 128/192-bit PM packing and byte masks, synchronous-DM transmit under NI stalls, blocked tail accounting, stable DLX descriptor/config validation, controller packet order, coexistence of actual generated FLX and DLX RTL, inferred memory behavior, and a cross-clock transfer through the actual six-node NoC/NI. The long network test crosses the NI's 10-flit framing boundary and checks PM readback. Vivado RTL synthesis checks the complete mixed compute fabric hierarchy.

This does not prove CNN numerical correctness or application/compiler compatibility, DDR4 read/write behavior, host/Zynq data packaging, timing closure, BRAM utilization with production memory depths, or a board bitstream. The board top still depends on the legacy DDR MIG and host-side IP and constraints. Validate actual PM/DM ranges and DLX address/count conventions against the selected CNN image before hardware use. The inferred RAMs truncate addresses to their configured depth, so software must not address outside those ranges. There is no cache-coherence protocol: simultaneous computation is supported only with software-assigned disjoint DM/CM regions or explicit network synchronization.

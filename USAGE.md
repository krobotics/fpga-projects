# Prophesee FPGA Projects - Usage Guide

This guide provides comprehensive instructions for building, simulating, and using the Prophesee FPGA projects.

## Table of Contents

- [Quick Start](#quick-start)
- [Building FPGA Projects](#building-fpga-projects)
- [IP Core Simulation](#ip-core-simulation)
- [Running Tests](#running-tests)
- [Synthesis and Implementation](#synthesis-and-implementation)
- [Project Structure](#project-structure)
- [Available IPs](#available-ips)
- [Advanced Usage](#advanced-usage)

## Quick Start

### Prerequisites

Ensure you have completed the setup steps in [SETUP.md](SETUP.md):

1. Vivado 2022.2 installed and sourced
2. Repository cloned
3. (Optional) Python virtual environment activated for simulations

### Basic Workflow

```bash
# 1. Source Vivado settings
source /tools/Xilinx/Vivado/2022.2/settings64.sh

# 2. Create a project
./scripts/create_project.tcl -tclargs kv260

# 3. Open the project in Vivado GUI
vivado build/projects/kv260/kv260.xpr
```

## Building FPGA Projects

### Creating a Project

There are two methods to create an FPGA project:

#### Method 1: Using the Global Script (Recommended)

```bash
./scripts/create_project.tcl -tclargs <project_name>
```

Example:
```bash
./scripts/create_project.tcl -tclargs kv260
```

#### Method 2: Using the Project-Specific Script

```bash
./projects/<project_name>/scripts/<project_name>.tcl
```

Example:
```bash
./projects/kv260/scripts/kv260.tcl
```

### Project Output Location

The Vivado project is generated in:

```
build/projects/<project_name>/
```

For the kv260 project:

```
build/projects/kv260/kv260.xpr
```

### Available Projects

Currently, the repository includes:

- **kv260**: AMD Kria KV260 Vision AI Starter Kit project
  - Target device: Zynq UltraScale+ MPSoC (xck26-sfvc784-2lv-c)
  - Features: Event-based vision processing pipeline
  - See [projects/kv260/README.md](projects/kv260/README.md) for detailed information

## IP Core Simulation

The repository contains standalone IP cores that can be simulated independently.

### Available IP Cores

1. **axis_tkeep_handler_2_0**
   - Manages AXI4-Stream TKEEP signals
   - Ensures 64-bit valid data for Event Format 2.1

2. **event_stream_smart_tracker_2_0**
   - Handles back-pressure from output pipeline
   - Event dropping and regeneration capabilities

3. **ps_host_if_3_0**
   - DMA buffer management
   - PS interface with configurable packet size and timeout

### Creating an IP Simulation Project

```bash
./scripts/create_ip_sim_project.tcl -tclargs --project_name <ip_name_X_Y>
```

Examples:

```bash
# Create simulation project for axis_tkeep_handler
./scripts/create_ip_sim_project.tcl -tclargs --project_name axis_tkeep_handler_2_0

# Create simulation project for event_stream_smart_tracker
./scripts/create_ip_sim_project.tcl -tclargs --project_name event_stream_smart_tracker_2_0

# Create simulation project for ps_host_if
./scripts/create_ip_sim_project.tcl -tclargs --project_name ps_host_if_3_0
```

### IP Simulation Project Options

#### Force Overwrite

If the build directory already exists:

```bash
./scripts/create_ip_sim_project.tcl -tclargs --project_name <ip_name_X_Y> --force
```

#### Create and Run Simulations

To create the project and automatically run all testcases:

```bash
./scripts/create_ip_sim_project.tcl -tclargs --project_name <ip_name_X_Y> --run
```

### IP Simulation Output

IP simulation projects are generated in:

```
build/ip/<ip_name_X_Y>/
```

## Running Tests

### IP Core Testcases

Each IP core includes multiple testcases located in:

```
ip/<ip_name_X_Y>/tb/tc_*/
```

#### Testcase Structure

Each testcase directory contains:

- **simulation.properties**: Testcase configuration
  - Pattern file paths
  - Simulation generics
- **pattern/**: Input and reference data files
- **wave/**: (Optional) Waveform configuration files (*.wcfg)

#### Running Testcases

##### Option 1: During Project Creation

```bash
./scripts/create_ip_sim_project.tcl -tclargs --project_name <ip_name> --run
```

##### Option 2: In Vivado GUI

1. Open the IP simulation project:
   ```bash
   vivado build/ip/<ip_name_X_Y>/<ip_name>_sim.xpr
   ```

2. In Vivado:
   - Select the desired testcase from the simulation sets
   - Click "Run Simulation" → "Run Behavioral Simulation"

##### Option 3: Command Line (after project creation)

```bash
vivado -mode batch -source run_sim.tcl build/ip/<ip_name_X_Y>/<ip_name>_sim.xpr
```

### Project Testcases

FPGA projects (like kv260) include testbenches:

```
projects/<project_name>/tb/tc_*/
```

To simulate:

1. Open the project in Vivado GUI
2. Select the simulation set
3. Run behavioral simulation

### Python Pattern Generation

Some testcases use Python scripts to generate pattern files:

```bash
# Activate Python virtual environment
source venv/bin/activate

# The pattern generation happens automatically during project creation
# if a virtual environment is active
```

## Synthesis and Implementation

### Opening a Project in Vivado GUI

```bash
vivado build/projects/<project_name>/<project_name>.xpr
```

Example:
```bash
vivado build/projects/kv260/kv260.xpr
```

### Synthesis

In Vivado GUI:

1. **Flow Navigator** → **Synthesis** → **Run Synthesis**
2. Wait for synthesis to complete
3. Review synthesis reports in the **Reports** tab

Or via Tcl console:

```tcl
launch_runs synth_1 -jobs 8
wait_on_run synth_1
```

### Implementation

After synthesis completes:

1. **Flow Navigator** → **Implementation** → **Run Implementation**
2. Wait for implementation to complete
3. Review timing and utilization reports

Or via Tcl console:

```tcl
launch_runs impl_1 -jobs 8
wait_on_run impl_1
```

### Bitstream Generation

After successful implementation:

1. **Flow Navigator** → **Program and Debug** → **Generate Bitstream**
2. Wait for bitstream generation
3. Find the bitstream in:
   ```
   build/projects/<project_name>/<project_name>.runs/impl_1/<design_name>.bit
   ```

Or via Tcl console:

```tcl
launch_runs impl_1 -to_step write_bitstream -jobs 8
wait_on_run impl_1
```

## Project Structure

### Repository Organization

```
fpga-projects/
├── build/                     # Generated build artifacts (not in git)
│   ├── ip/                    # IP simulation projects
│   └── projects/              # FPGA projects
├── doc/                       # Documentation and images
├── ip/                        # IP core source files
│   ├── axis_tkeep_handler_2_0/
│   │   ├── component.xml      # IP-XACT component definition
│   │   ├── doc/               # IP documentation and changelogs
│   │   ├── hdl/               # HDL source files
│   │   ├── tb/                # Testbench files
│   │   │   ├── src/           # Testbench source
│   │   │   └── tc_*/          # Test cases
│   │   └── xgui/              # IP customization GUI
│   ├── event_stream_smart_tracker_2_0/
│   └── ps_host_if_3_0/
├── projects/                  # Complete FPGA projects
│   └── kv260/
│       ├── README.md          # Project-specific documentation
│       ├── coe/               # Coefficient files
│       ├── constr/            # Constraint files (XDC)
│       ├── doc/               # Project documentation
│       ├── hdl/               # HDL top-level files
│       ├── scripts/           # Project build scripts
│       └── tb/                # Project testbenches
├── scripts/                   # Global build scripts
│   ├── create_ip_sim_project.tcl
│   └── create_project.tcl
├── .gitignore
├── CHANGELOG.md
├── README.md
├── SETUP.md                   # Setup guide
└── USAGE.md                   # This file
```

### Build Artifacts

All build artifacts are generated in the `build/` directory, which is excluded from version control (.gitignore).

## Available IPs

### 1. axis_tkeep_handler (v2.0)

**Purpose:** Manages AXI4-Stream TKEEP signals to ensure full 64-bit valid data.

**Key Features:**
- Handles TKEEP signal validation
- Event Format 2.1 compliance
- Bypass mode support

**Register Interface:** AXI4-Lite slave
**Data Interface:** AXI4-Stream (64-bit)

**Testcases:**
- tc_001: Basic functionality test

### 2. event_stream_smart_tracker (v2.0)

**Purpose:** Back-pressure management and event flow control.

**Key Features:**
- Event dropping on back-pressure
- Event regeneration (time-related events)
- Status counters and flags
- Configurable thresholds

**Register Interface:** AXI4-Lite slave
**Data Interface:** AXI4-Stream (64-bit)

**Testcases:**
- tc_001 through tc_005: Various back-pressure scenarios

### 3. ps_host_if (v3.0)

**Purpose:** Interface between FPGA processing logic and PS DMA.

**Key Features:**
- DMA buffer management
- Configurable packet size
- Timeout-based TLAST generation
- Event insertion on timeout
- Pipeline flush control

**Register Interface:** AXI4-Lite slave
**Data Interfaces:**
- AXI4-Stream slave (input)
- AXI4-Stream master (to DMA)

**Testcases:**
- tc_001 through tc_005: Various timing and control scenarios

## Advanced Usage

### Customizing IP Parameters

When using IPs in block designs, parameters can be customized through:

1. **Vivado GUI:** Double-click the IP in the block design
2. **Tcl:** Use `set_property` commands

### Custom Testcases

To add custom testcases:

1. Create a new directory under `ip/<ip_name>/tb/tc_<number>/`
2. Add required files:
   - `simulation.properties`: Configuration
   - `pattern/`: Pattern files
   - (Optional) `*.py`: Pattern generation script
   - (Optional) `wave/*.wcfg`: Waveform configuration

3. Re-run the IP simulation project creation

### Working with Block Designs

The kv260 project uses Vivado Block Design:

1. Open project in Vivado
2. **Flow Navigator** → **IP Integrator** → **Open Block Design**
3. Modify the design as needed
4. **Validate Design** (F6)
5. **Generate Output Products**

### Exporting Hardware

After bitstream generation, export hardware for software development:

```tcl
# In Vivado Tcl console
write_hw_platform -fixed -include_bit \
  -file design_wrapper.xsa
```

### Command-Line Automation

For automated builds without GUI:

```bash
vivado -mode batch -source build_script.tcl
```

Example build_script.tcl:

```tcl
open_project build/projects/kv260/kv260.xpr
launch_runs synth_1 -jobs 8
wait_on_run synth_1
launch_runs impl_1 -to_step write_bitstream -jobs 8
wait_on_run impl_1
close_project
```

## Tips and Best Practices

### Performance Optimization

- Use multiple jobs for synthesis and implementation: `-jobs 8`
- Enable incremental compilation for faster iterations
- Use out-of-context synthesis for IP cores

### Simulation Best Practices

- Always run testcases after modifying IP cores
- Use waveform configurations (*.wcfg) for easier debugging
- Check simulation log files for "End of Test with Success" messages

### Version Control

- Never commit `build/` directory contents
- Keep scripts executable with `chmod +x`
- Document any custom modifications in CHANGELOG.md

### Debugging

- Use Vivado ILA (Integrated Logic Analyzer) for hardware debugging
- Enable verbose logging during simulation
- Check timing reports after implementation

## Troubleshooting

### Build Fails

1. Check Vivado version (must be 2022.2)
2. Verify all required components are installed
3. Clean build directory and retry:
   ```bash
   rm -rf build/
   ```

### Simulation Fails

1. Verify pattern files exist in `tc_*/pattern/`
2. Check simulation.properties for correct paths
3. Enable Python virtual environment for pattern generation

### Timing Not Met

1. Review timing reports in Vivado
2. Adjust clock constraints in XDC files
3. Consider design optimizations or pipelining

## Additional Resources

- **Prophesee Knowledge Center:** https://support.prophesee.ai/
- **KV260 Project Details:** [projects/kv260/README.md](projects/kv260/README.md)
- **AMD Vivado Documentation:** https://docs.xilinx.com/
- **IP Changelogs:**
  - [axis_tkeep_handler](ip/axis_tkeep_handler_2_0/doc/axis_tkeep_handler_v2_0_changelog.txt)
  - [event_stream_smart_tracker](ip/event_stream_smart_tracker_2_0/doc/changelog.txt)
  - [ps_host_if](ip/ps_host_if_3_0/doc/ps_host_if_v3_0_changelog.txt)

## Support

For issues, questions, or contributions:

- **Issues:** https://github.com/krobotics/fpga-projects/issues
- **Support:** https://support.prophesee.ai/

---

**Last Updated:** 2024-12-30

For setup instructions, see [SETUP.md](SETUP.md)

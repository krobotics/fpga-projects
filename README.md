# Prophesee FPGA Projects

## Overview

This repository contains the source files and scripts necessary to build Prophesee FPGA projects for event-based vision applications. It includes complete FPGA projects for AMD platforms and reusable IP cores for event stream processing.

## Quick Start

```bash
# 1. Source Vivado settings
source /tools/Xilinx/Vivado/2022.2/settings64.sh

# 2. Create a project
./scripts/create_project.tcl -tclargs kv260

# 3. Open in Vivado GUI
vivado build/projects/kv260/kv260.xpr
```

For detailed instructions, see [SETUP.md](SETUP.md) and [USAGE.md](USAGE.md).

## Documentation

- **[SETUP.md](SETUP.md)** - Complete setup guide with prerequisites and installation steps
- **[USAGE.md](USAGE.md)** - Comprehensive usage guide for building, simulating, and testing
- **[CHANGELOG.md](CHANGELOG.md)** - Version history and changes
- **[projects/kv260/README.md](projects/kv260/README.md)** - KV260 project-specific documentation

## Requirements

The projects in this repository have been tested and validated on the following setup:

- **Operating System:** Ubuntu 20.04.6 LTS
- **FPGA Tools:** AMD Vivado 2022.2 (64-bit)
  - Required support packages:
    - Kria SOMs and Starter Kits
    - Zynq UltraScale+ MPSoC Production Devices
- **Python:** 3.8+ (optional, for simulation pattern generation)
- **Target Hardware:** AMD Kria KV260 Vision AI Starter Kit

![Vivado Install Extra Content Window](doc/img/vivado-install-extra-content.png "Vivado Install Extra Content Window")

### Environment Setup

Before running any Tcl script or launching Vivado GUI, source the Vivado settings:

```bash
source /tools/Xilinx/Vivado/2022.2/settings64.sh
```

See [SETUP.md](SETUP.md) for detailed installation and configuration instructions.

## Content

This repository is organized into the following main sections:

### FPGA Projects

The **projects** directory contains complete FPGA projects for AMD Vivado platforms.

**Available Projects:**
- **kv260**: AMD Kria KV260 Vision AI Starter Kit project
  - Event-based vision processing pipeline
  - MIPI CSI-2 interface support
  - See [projects/kv260/README.md](projects/kv260/README.md) for details

**Building Projects:**

Projects can be built using the project-specific script:

```bash
./projects/kv260/scripts/kv260.tcl
```

Or using the global creation script in the **scripts** directory:

```bash
./scripts/create_project.tcl -tclargs kv260
```

The Vivado project will be generated in **build/projects/<project_name>/** and can be opened with Vivado GUI to run synthesis, implementation, and bitstream generation:

```bash
vivado build/projects/kv260/kv260.xpr
```

Each project includes testcases that can be simulated with Vivado.

### Event Processing IP Cores

The **ip** directory contains Prophesee Event Processing IPs used in FPGA projects. These IPs can be simulated independently.

**Available IP Cores:**

1. **axis_tkeep_handler (v2.0)**
   - Manages AXI4-Stream TKEEP signals
   - Ensures 64-bit valid data alignment
   - Required for Event Format 2.1 compliance

2. **event_stream_smart_tracker (v2.0)**
   - Back-pressure management
   - Event dropping and regeneration
   - Status monitoring and counters

3. **ps_host_if (v3.0)**
   - DMA buffer management
   - PS interface with packet size control
   - Configurable timeout for TLAST generation

**Simulating IP Cores:**

Create an IP simulation project using:

```bash
./scripts/create_ip_sim_project.tcl -tclargs --project_name <ip_name_X_Y>
```

Examples:

```bash
# Simulate axis_tkeep_handler IP
./scripts/create_ip_sim_project.tcl -tclargs --project_name axis_tkeep_handler_2_0

# Simulate event_stream_smart_tracker IP
./scripts/create_ip_sim_project.tcl -tclargs --project_name event_stream_smart_tracker_2_0

# Simulate ps_host_if IP
./scripts/create_ip_sim_project.tcl -tclargs --project_name ps_host_if_3_0
```

The Vivado project will be generated in **build/ip/<ip_name_X_Y>/** directory.

**Running All Testcases:**

Add the `--run` option to automatically execute all testcases during project creation:

```bash
./scripts/create_ip_sim_project.tcl -tclargs --project_name <ip_name_X_Y> --run
```

**Overwriting Existing Projects:**

Use the `--force` option to overwrite an existing build directory:

```bash
./scripts/create_ip_sim_project.tcl -tclargs --project_name <ip_name_X_Y> --force
```

## Repository Structure

```
fpga-projects/
├── doc/                       # Documentation and images
├── ip/                        # IP core source files
│   ├── axis_tkeep_handler_2_0/
│   ├── event_stream_smart_tracker_2_0/
│   └── ps_host_if_3_0/
├── projects/                  # Complete FPGA projects
│   └── kv260/
├── scripts/                   # Build and simulation scripts
│   ├── create_ip_sim_project.tcl
│   └── create_project.tcl
├── build/                     # Generated projects (not in git)
├── CHANGELOG.md               # Version history
├── README.md                  # This file
├── SETUP.md                   # Detailed setup guide
└── USAGE.md                   # Comprehensive usage guide
```

## Features

- **Complete FPGA Projects:** Ready-to-build projects for supported platforms
- **Reusable IP Cores:** Event processing IPs with standalone testbenches
- **Automated Testing:** Multiple testcases per IP with pattern generation
- **Comprehensive Documentation:** Setup and usage guides with examples
- **Block Design Integration:** IP cores compatible with Vivado IP Integrator

## Main Features

### Event-Based Vision Processing

- **MIPI CSI-2 Support:** 2-lane MIPI interface with 1500 Mbps line rate
- **Event Format 2.1:** 64-bit event vector format support
- **High-Throughput Pipeline:** Optimized for high event rate processing
- **Back-Pressure Handling:** Intelligent event management under system load

### Configurable IP Cores

- **AXI4-Lite Register Interface:** Runtime configuration support
- **AXI4-Stream Data Path:** Standard interface for data flow
- **DMA Integration:** Efficient data transfer to processing system
- **Status Monitoring:** Counters and flags for system diagnostics

## Getting Started

1. **Setup Environment:** Follow [SETUP.md](SETUP.md) for installation and configuration
2. **Build a Project:** See [USAGE.md](USAGE.md) for detailed build instructions
3. **Run Simulations:** Test IP cores or projects with included testcases
4. **Synthesize Design:** Generate bitstreams for target hardware

## Testing

All IP cores include comprehensive testbenches with multiple test scenarios:

- **axis_tkeep_handler:** 1 testcase covering TKEEP signal handling
- **event_stream_smart_tracker:** 5 testcases covering various back-pressure scenarios
- **ps_host_if:** 5 testcases covering packet management and timeout scenarios

Run all tests for an IP core:

```bash
./scripts/create_ip_sim_project.tcl -tclargs --project_name <ip_name> --run
```

## License

This project is licensed under Apache License 2.0 (as of version 0.2.3, May 2024).

## Support

For additional information or support:

- **Prophesee Knowledge Center:** https://support.prophesee.ai/
- **Repository Issues:** https://github.com/krobotics/fpga-projects/issues
- **AMD Vivado Documentation:** https://docs.xilinx.com/

## Contributing

When contributing to this repository:

1. Follow the existing code structure and naming conventions
2. Update documentation for any new features
3. Add testcases for new IP cores
4. Update CHANGELOG.md with your changes

## Version History

See [CHANGELOG.md](CHANGELOG.md) for detailed version history and changes.

**Current Version:** 1.0.0 (Released November 8, 2024)

### Recent Changes

- PL clock frequency set to 125MHz
- Updated IP versions:
  - axis_tkeep_handler v2.0
  - event_stream_smart_tracker v2.0
  - ps_host_if v3.0
- New IP simulation project generation script
- Improved directory structure

## Acknowledgments

- Prophesee for the event-based vision IP cores
- AMD Xilinx for Vivado tools and Kria platform support

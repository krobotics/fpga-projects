# Prophesee FPGA Projects - Setup Guide

This guide provides detailed instructions for setting up your development environment to work with Prophesee FPGA projects.

## Table of Contents

- [System Requirements](#system-requirements)
- [Prerequisites](#prerequisites)
- [Installation Steps](#installation-steps)
- [Environment Configuration](#environment-configuration)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## System Requirements

### Supported Operating Systems

- **Ubuntu 20.04.6 LTS** (Recommended and tested)
- Other Linux distributions may work but are not officially tested

### Hardware Requirements

- Minimum 16 GB RAM (32 GB recommended for synthesis and implementation)
- At least 100 GB of free disk space for Vivado and project files
- Multi-core processor (4+ cores recommended)

### Target Hardware

This repository supports the following FPGA platforms:

- **AMD Kria KV260 Vision AI Starter Kit**
  - Zynq UltraScale+ MPSoC (xck26-sfvc784-2lv-c)
  - MIPI CSI-2 interface support
  - Event-based vision sensor compatibility

## Prerequisites

### 1. AMD Vivado Design Suite

**Version Required:** AMD Vivado 2022.2 (64-bit)

#### Installation Components

When installing Vivado, ensure you include the following components:

- **Vivado Design Edition** or **Vivado ML Edition**
- **Kria SOMs and Starter Kits** support
- **Zynq UltraScale+ MPSoC Production Devices** support

![Vivado Install Extra Content Window](doc/img/vivado-install-extra-content.png)

#### Download and Installation

1. Download Vivado 2022.2 from the [AMD Xilinx Downloads Page](https://www.xilinx.com/support/download.html)
2. Run the installer and select the required components mentioned above
3. Default installation path: `/tools/Xilinx/Vivado/2022.2/`

### 2. Python (Optional for Simulation)

Python is required for generating simulation pattern files.

**Recommended:** Python 3.8 or later

```bash
# Install Python if not already available
sudo apt-get update
sudo apt-get install python3 python3-pip python3-venv
```

## Installation Steps

### Step 1: Clone the Repository

```bash
git clone https://github.com/krobotics/fpga-projects.git
cd fpga-projects
```

### Step 2: Verify Repository Structure

Ensure the following directories exist:

```
fpga-projects/
├── doc/                    # Documentation and images
├── ip/                     # Prophesee Event Processing IPs
│   ├── axis_tkeep_handler_2_0/
│   ├── event_stream_smart_tracker_2_0/
│   └── ps_host_if_3_0/
├── projects/               # Complete FPGA projects
│   └── kv260/
├── scripts/                # Build and simulation scripts
├── README.md
└── CHANGELOG.md
```

### Step 3: Set Up Python Virtual Environment (Optional)

For simulation pattern generation:

```bash
# Create a virtual environment
python3 -m venv venv

# Activate the virtual environment
source venv/bin/activate

# Install any required Python packages (if a requirements.txt exists)
# pip install -r requirements.txt
```

## Environment Configuration

### Source Vivado Settings

Before running any Tcl script or launching Vivado GUI, you must source the Vivado settings script:

```bash
# Adjust the path according to your Vivado installation
source /tools/Xilinx/Vivado/2022.2/settings64.sh
```

**Recommended:** Add this to your shell profile for automatic setup:

```bash
# Add to ~/.bashrc or ~/.zshrc
echo 'source /tools/Xilinx/Vivado/2022.2/settings64.sh' >> ~/.bashrc
source ~/.bashrc
```

### Verify Environment Variables

After sourcing the settings script, verify the environment:

```bash
# Check Vivado is in PATH
which vivado

# Verify Vivado version
vivado -version
```

Expected output should show:

```
Vivado v2022.2 (64-bit)
```

## Verification

### Test Script Execution

Verify that scripts are executable:

```bash
# Check script permissions
ls -l scripts/

# Make scripts executable if needed
chmod +x scripts/*.tcl
chmod +x projects/kv260/scripts/*.tcl
```

### Quick Test Build

Test the environment with a simple IP simulation project:

```bash
# Create an IP simulation project (without running simulations)
./scripts/create_ip_sim_project.tcl -tclargs --project_name axis_tkeep_handler_2_0

# Check if project was created successfully
ls -l build/ip/axis_tkeep_handler_2_0/
```

If successful, you should see a Vivado project created in `build/ip/axis_tkeep_handler_2_0/`.

## Troubleshooting

### Common Issues

#### Issue: "vivado: command not found"

**Solution:** Source the Vivado settings script:

```bash
source /tools/Xilinx/Vivado/2022.2/settings64.sh
```

#### Issue: Script execution permission denied

**Solution:** Make the script executable:

```bash
chmod +x scripts/create_project.tcl
```

#### Issue: Missing Python virtual environment warning

If you see a warning about missing virtual environment when running simulations:

**Solution:** Create and activate a Python virtual environment as described in Step 3.

#### Issue: "ERROR: Project directory already exists"

When creating IP simulation projects, if the build directory exists:

**Solution:** Use the `--force` option to overwrite:

```bash
./scripts/create_ip_sim_project.tcl -tclargs --project_name <ip_name> --force
```

#### Issue: Missing Kria SOMs or Zynq UltraScale+ support

**Solution:** Re-run the Vivado installer and add the missing components, or use the Xilinx Update tool:

```bash
# Launch Xilinx Update tool
xsetup
```

### Getting Help

For additional support and documentation:

- **Prophesee Knowledge Center:** https://support.prophesee.ai/
- **AMD Vivado Documentation:** https://docs.xilinx.com/
- **Repository Issues:** https://github.com/krobotics/fpga-projects/issues

## Next Steps

Once your environment is set up, proceed to the [USAGE.md](USAGE.md) guide to learn how to:

- Build FPGA projects
- Run simulations
- Synthesize and implement designs
- Generate bitstreams

---

**Note:** This setup has been tested and validated on Ubuntu 20.04.6 LTS with Vivado 2022.2. Other configurations may work but are not officially supported.

# VHDL Files Analysis and Extension Guide

## Executive Summary

This document provides a comprehensive analysis of the VHDL (VHD) files in the fpga-projects repository, covering 101 VHDL files across three main IP cores designed for event-based vision processing. The analysis includes operational descriptions, architecture details, and recommendations for extending functionality.

## Table of Contents

1. [Repository Overview](#repository-overview)
2. [IP Core Analysis](#ip-core-analysis)
   - [axis_tkeep_handler_2_0](#1-axis_tkeep_handler_20)
   - [event_stream_smart_tracker_2_0](#2-event_stream_smart_tracker_20)
   - [ps_host_if_3_0](#3-ps_host_if_30)
3. [Common Architecture Patterns](#common-architecture-patterns)
4. [Extension Recommendations](#extension-recommendations)
5. [Development Guidelines](#development-guidelines)

---

## Repository Overview

### Statistics
- **Total VHDL Files**: 101
- **Main IP Cores**: 3
- **Supporting Modules**: Multiple utility and testbench components
- **Target Platform**: AMD Xilinx Vivado 2022.2, Kria KV260
- **Primary Application**: Event-based vision processing with MIPI CSI-2 interface

### File Organization
```
ip/
├── axis_tkeep_handler_2_0/
│   ├── hdl/                    # Implementation files
│   ├── tb/src/                 # Testbench files
│   └── doc/                    # Documentation
├── event_stream_smart_tracker_2_0/
│   ├── hdl/                    # Implementation files
│   ├── tb/src/                 # Testbench files
│   └── doc/                    # Documentation
└── ps_host_if_3_0/
    ├── hdl/                    # Implementation files
    ├── tb/src/                 # Testbench files
    └── doc/                    # Documentation
```

---

## IP Core Analysis

## 1. axis_tkeep_handler_2.0

### Purpose
Manages AXI4-Stream TKEEP signals to ensure 64-bit valid data alignment, required for Event Format 2.1 compliance.

### Key Files
- **axis_tkeep_handler.vhd** - Top-level wrapper with AXI4-Lite register interface
- **axis_tkeep_handler_core.vhd** - Core processing logic
- **axis_tkeep_handler_reg_bank.vhd** - Register bank for configuration

### Operational Description

#### Architecture
The TKEEP handler addresses a common challenge in AXI4-Stream processing where partial data words (indicated by TKEEP signals) need to be realigned to full 64-bit boundaries.

**Key Components:**
1. **Input Buffer System**: Three half-word buffers (3 × 32 bits) provide flexible data alignment
2. **Buffer Pointer Management**: Tracks which buffer positions contain valid data
3. **Word Order Control**: Allows swapping MSB/LSB halves for endianness handling
4. **Bypass Mode**: Direct pass-through when alignment is not needed

**Data Flow:**
```
Input Stream → Buffer Management → Alignment Logic → Output Stream
     ↓              ↓                    ↓               ↓
  TKEEP         3x Half-Word        Combine into      Full 64-bit
  Analysis        Buffers           Full Words         Words
```

#### Control Registers
| Register | Function |
|----------|----------|
| CONTROL.enable | Enable/disable the core |
| CONTROL.global_reset | Reset all internal state |
| CONTROL.clear | Clear internal buffers |
| CONFIG.bypass | Bypass alignment processing |
| CONFIG.word_order | Swap MSB/LSB word order (endianness) |

#### Algorithm
1. **Detect partial words** via TKEEP signal analysis
2. **Buffer partial data** in internal registers
3. **Combine buffered data** with new incoming data to form complete 64-bit words
4. **Maintain TLAST** signal integrity across realignment
5. **Preserve TUSER** metadata throughout processing

### Extension Possibilities

#### 1. Multi-Width Support
**Current Limitation**: Fixed 64-bit data width
**Extension**: Generic parameterization for 128-bit, 256-bit, or wider buses
```vhdl
generic (
  AXIS_TDATA_WIDTH_G : positive := 64;
  ALIGNMENT_WIDTH_G  : positive := 64  -- New: configurable alignment boundary
);
```

#### 2. Configurable Buffer Depth
**Current Limitation**: Fixed 3-buffer depth
**Extension**: Runtime configurable buffer count for different latency/throughput tradeoffs
```vhdl
generic (
  BUFFER_DEPTH_G : positive range 2 to 8 := 3
);
```

#### 3. Performance Monitoring
**Extension**: Add counters for alignment events and buffer usage
```vhdl
-- Status registers to add:
stat_partial_word_count : out std_logic_vector(31 downto 0);
stat_buffer_full_count  : out std_logic_vector(31 downto 0);
stat_max_buffer_usage   : out std_logic_vector(7 downto 0);
```

#### 4. Advanced TKEEP Patterns
**Extension**: Support for non-contiguous TKEEP patterns (currently assumes contiguous valid bytes)
```vhdl
-- Configuration for non-contiguous byte enables
cfg_allow_sparse_tkeep : in std_logic;
```

---

## 2. event_stream_smart_tracker_2.0

### Purpose
Comprehensive back-pressure management system for event streams with intelligent event dropping, timestamp checking, and time-high event recovery.

### Key Files
- **event_stream_smart_tracker.vhd** - Top-level integration
- **evt21_smart_drop.vhd** - Smart event dropper with priority-based dropping
- **evt21_ts_checker.vhd** - Timestamp validation and correction
- **evt21_th_recovery.vhd** - Time-high event recovery and generation
- **evt_smart_fifo.vhd** - Two-stage threshold FIFO
- **axi4s_demux_1_2_keep.vhd** - 1-to-2 demultiplexer for bypass path
- **axi4s_mux_2_1_keep.vhd** - 2-to-1 multiplexer for output selection
- **axi4s_pipeline_stage.vhd** - Configurable pipeline stages for timing closure

### Operational Description

#### Architecture
This is the most complex IP core, implementing a sophisticated event stream management pipeline with three optional processing stages that can be enabled/disabled via generics.

**Processing Pipeline:**
```
Input → Demux → Smart Drop → Smart FIFO → TS Checker → TH Recovery → Mux → Output
         ↓                                                                ↑
      Bypass Path (4 pipeline stages) ──────────────────────────────────┘
```

**Configurable Features (Generics):**
- `ENABLE_SMART_DROP_G`: Enable/disable smart event dropper
- `ENABLE_TS_CHECKER_G`: Enable/disable timestamp checker
- `ENABLE_TH_RECOVERY_G`: Enable/disable time-high recovery
- `TIME_HIGH_PERIOD_US_G`: Expected time-high event period (default: 16 µs)
- `BYPASS_PIPELINE_STAGES_G`: Number of pipeline stages in bypass path (default: 4)

#### Module Descriptions

##### evt21_smart_drop
**Purpose**: Intelligently drop events under back-pressure conditions while preserving critical timing events.

**Two-Stage Dropping Strategy:**
1. **Stage 1 (Reduce Flow)**: Drop regular events, preserve Time-High (TH) events
2. **Stage 2 (Drop All)**: Drop all events including TH when system is critically full

**Event Type Handling:**
- **Always Preserved**: Time-High events (until Stage 2)
- **First to Drop**: Regular CD/EM events
- **Configurable**: Can generate "OTHER" event type to signal drops

**Counters:**
- `stat_th_drop_cnt`: Count of dropped Time-High events
- `stat_evt_drop_cnt`: Count of dropped regular events
- `stat_evt_drop_flag`: Flag indicating drops occurred

##### evt21_ts_checker
**Purpose**: Validate timestamp consistency and detect timestamp errors or corruption.

**Key Features:**
1. **Timestamp Validation**: Checks for timestamp monotonicity
2. **Error Detection**: Identifies corrupt or invalid timestamps
3. **Configurable Threshold**: Define acceptable timestamp deviation
4. **Automatic Correction**: Can drop or regenerate events with bad timestamps

**Configuration:**
- `cfg_bypass`: Disable timestamp checking
- `cfg_ts_threshold`: Maximum allowed timestamp difference
- `cfg_enable_drop_evt`: Drop events with invalid timestamps
- `cfg_gen_other_evt`: Generate OTHER event to signal timestamp issues
- `cfg_gen_tlast_on_other`: Generate TLAST with OTHER events

**Statistics:**
- `stat_th_detect_cnt`: Time-High events detected
- `stat_th_corrupt_cnt`: Corrupted Time-High events
- `stat_th_error_cnt`: Timestamp errors detected

##### evt21_th_recovery
**Purpose**: Monitor Time-High events and regenerate missing ones to maintain timestamp integrity.

**Functionality:**
1. **Expected TH Tracking**: Monitors TH events based on configured period
2. **Missing TH Detection**: Identifies when expected TH event doesn't arrive
3. **TH Generation**: Creates synthetic TH events to fill gaps
4. **Event Dropping**: Optionally drops regular events when TH is missing

**Configuration:**
- `TIME_HIGH_PERIOD_US`: Expected period between TH events (default: 16 µs)
- `cfg_gen_missing_th`: Enable automatic TH generation
- `cfg_enable_drop_evt`: Drop events when TH is missing
- `cfg_gen_other_evt`: Generate OTHER event for TH recovery

**Status:**
- `stat_gen_th_flag`: Flag indicating TH was generated
- `stat_evt_drop_flag`: Flag indicating events were dropped

##### evt_smart_fifo
**Purpose**: Two-threshold FIFO that signals different back-pressure levels.

**Threshold System:**
1. **Threshold 1 (Reduce Flow)**: FIFO is filling, start reducing event flow
   - Default: 21 entries (configurable via `STEP1_ALMOST_FULL_THRESH_G`)
2. **Threshold 2 (Drop All)**: FIFO critical, drop all events
   - Default: 5 entries (configurable via `STEP2_ALMOST_FULL_THRESH_G`)

**Configuration:**
- `DEPTH_G`: Total FIFO depth (32 to 4,194,304 entries)
- `MEMORY_TYPE_G`: "auto", "block", "distributed", or "ultra"

#### Control Registers

**Global Control:**
| Register | Field | Function |
|----------|-------|----------|
| CONTROL | enable | Enable entire IP core |
| CONTROL | global_reset | Reset all state machines |
| CONTROL | clear | Clear FIFOs and counters |
| CONFIG | bypass | Global bypass (skip all processing) |

**Smart Dropper Control:**
| Register | Field | Function |
|----------|-------|----------|
| SMART_DROPPER_CONTROL | bypass | Bypass smart dropper |
| SMART_DROPPER_CONTROL | gen_other_evt | Generate OTHER event on drops |

**Timestamp Checker Control:**
| Register | Field | Function |
|----------|-------|----------|
| TS_CHECKER_CONTROL | bypass | Bypass timestamp checker |
| TS_CHECKER_CONTROL | enable_drop_evt | Drop events with bad timestamps |
| TS_CHECKER_CONTROL | gen_other_evt | Generate OTHER event for errors |
| TS_CHECKER_CONTROL | threshold | Timestamp deviation threshold |

**TH Recovery Control:**
| Register | Field | Function |
|----------|-------|----------|
| TH_RECOVERY_CONTROL | bypass | Bypass TH recovery |
| TH_RECOVERY_CONTROL | gen_missing_th | Auto-generate missing TH events |
| TH_RECOVERY_CONTROL | enable_drop_evt | Drop events on TH miss |
| TH_RECOVERY_CONTROL | gen_other_evt | Generate OTHER event for recovery |

### Extension Possibilities

#### 1. Adaptive Threshold Control
**Extension**: Dynamic FIFO thresholds based on system load
```vhdl
-- Runtime configurable thresholds
cfg_smart_drop_threshold_1 : in std_logic_vector(31 downto 0);
cfg_smart_drop_threshold_2 : in std_logic_vector(31 downto 0);
-- Automatic threshold adjustment
cfg_adaptive_threshold_enable : in std_logic;
```

#### 2. Priority-Based Event Filtering
**Extension**: Configure per-event-type dropping priorities
```vhdl
-- Priority configuration per event type
cfg_cd_event_priority    : in std_logic_vector(3 downto 0);
cfg_em_event_priority    : in std_logic_vector(3 downto 0);
cfg_other_event_priority : in std_logic_vector(3 downto 0);
```

#### 3. Enhanced Statistics
**Extension**: Per-event-type statistics and histograms
```vhdl
-- Detailed statistics
stat_cd_event_count      : out std_logic_vector(31 downto 0);
stat_em_event_count      : out std_logic_vector(31 downto 0);
stat_th_event_count      : out std_logic_vector(31 downto 0);
stat_fifo_max_fill_level : out std_logic_vector(15 downto 0);
-- Latency measurement
stat_min_latency_cycles  : out std_logic_vector(15 downto 0);
stat_max_latency_cycles  : out std_logic_vector(15 downto 0);
stat_avg_latency_cycles  : out std_logic_vector(15 downto 0);
```

#### 4. Burst Mode Support
**Extension**: Optimize for bursty event traffic
```vhdl
generic (
  ENABLE_BURST_MODE_G : boolean := false;
  BURST_THRESHOLD_G   : positive := 100
);
-- Burst detection
stat_burst_detected : out std_logic;
stat_burst_count    : out std_logic_vector(31 downto 0);
```

#### 5. Multi-Stream Support
**Extension**: Handle multiple independent event streams
```vhdl
generic (
  NUM_STREAMS_G : positive range 1 to 4 := 1
);
-- Per-stream control
cfg_stream_enable : in std_logic_vector(NUM_STREAMS_G-1 downto 0);
```

#### 6. Event Filtering and Transformation
**Extension**: Add spatial/temporal filtering capabilities
```vhdl
-- ROI filtering
cfg_roi_enable    : in std_logic;
cfg_roi_x_min     : in std_logic_vector(15 downto 0);
cfg_roi_x_max     : in std_logic_vector(15 downto 0);
cfg_roi_y_min     : in std_logic_vector(15 downto 0);
cfg_roi_y_max     : in std_logic_vector(15 downto 0);

-- Temporal filtering
cfg_refractory_enable : in std_logic;
cfg_refractory_period : in std_logic_vector(15 downto 0);
```

---

## 3. ps_host_if_3.0

### Purpose
Interface between FPGA processing logic and Zynq Processing System (PS), managing DMA transfers with configurable packet sizes and timeout handling.

### Key Files
- **ps_host_if.vhd** - Top-level integration
- **axi4s_packetizer.vhd** - Packet size management and TLAST generation
- **axi4s_packet_timeout.vhd** - Timeout-based TLAST insertion
- **ps_host_if_reg_bank.vhd** - Register bank for configuration

### Operational Description

#### Architecture
Manages event data transfer from FPGA fabric to PS via AXI DMA, with intelligent packet formation and timeout handling.

**Processing Flow:**
```
Input Stream → Packetizer → Packet Timeout → Output to DMA
                  ↓              ↓
            Count Words    Insert Timeout
            Insert TLAST    Event if idle
```

**Key Features:**
1. **Configurable Packet Size**: Control DMA transfer granularity
2. **Timeout Handling**: Generate TLAST if no data for specified time
3. **Test Pattern Generation**: Built-in pattern generator for testing
4. **Skid Buffer**: Prevents back-pressure propagation

#### Module Descriptions

##### axi4s_packetizer
**Purpose**: Control packet boundaries by inserting TLAST signals.

**Functionality:**
1. **Word Counting**: Tracks number of 64-bit words in current packet
2. **TLAST Generation**: Inserts TLAST when packet_length reached
3. **Timeout Support**: Can also assert TLAST on timeout signal
4. **Pattern Mode**: Generates test patterns instead of forwarding input

**Key Logic:**
- Counts valid data transfers (tvalid && tready && tkeep != 0)
- Compares count against configured packet_length
- Asserts TLAST when count matches packet_length
- Resets counter after each packet

**Skid Buffer:**
Implements a single-entry buffer to handle timing when slave is not ready:
```
If master not ready && data available:
  → Store data in buffer
  → Accept new data when buffer empties
```

##### axi4s_packet_timeout
**Purpose**: Insert timeout events when data stream becomes idle.

**Functionality:**
1. **Idle Detection**: Monitors data stream activity
2. **Timeout Counter**: Counts clock cycles without valid data
3. **Event Insertion**: Injects configurable timeout event data
4. **TLAST Assertion**: Forces packet termination on timeout

**Configuration:**
- `timeout_enable`: Enable timeout detection
- `timeout_value`: Number of clock cycles before timeout (in clock cycles)
- `insert_tdata`: Data to insert on timeout (typically a special event marker)

**Use Cases:**
- Prevent DMA buffer indefinite waiting
- Ensure regular PS polling
- Signal end of event burst

#### Control Registers

| Register | Field | Function |
|----------|-------|----------|
| CONTROL | enable | Enable IP core |
| CONTROL | global_reset | Reset all state |
| CONTROL | clear | Clear packet counter |
| CONFIG | test_pattern | Enable test pattern generation |
| CONFIG | timeout_enable | Enable timeout mechanism |
| PACKET_LENGTH | value | Words per packet (default: 4096) |
| TIMEOUT | value | Clock cycles before timeout |
| TIMEOUT_EVENT_MSB | value | Upper 32 bits of timeout event |
| TIMEOUT_EVENT_LSB | value | Lower 32 bits of timeout event |

### Extension Possibilities

#### 1. Multiple DMA Channels
**Extension**: Support multiple independent DMA streams
```vhdl
generic (
  NUM_DMA_CHANNELS_G : positive range 1 to 4 := 1
);
port (
  -- Per-channel interfaces
  s_axis_tready : out std_logic_vector(NUM_DMA_CHANNELS_G-1 downto 0);
  s_axis_tvalid : in  std_logic_vector(NUM_DMA_CHANNELS_G-1 downto 0);
  -- ... (replicate all AXI4-Stream signals)
);
```

#### 2. Dynamic Packet Sizing
**Extension**: Adjust packet size based on system load
```vhdl
-- Runtime packet size adjustment
cfg_packet_size_mode    : in std_logic_vector(1 downto 0);
-- 00: Fixed size
-- 01: Adaptive (increase under high load)
-- 10: Burst-based (one packet per burst)
-- 11: Event-based (packet per event type)

stat_current_packet_size : out std_logic_vector(31 downto 0);
```

#### 3. Advanced Timeout Policies
**Extension**: Multiple timeout levels and actions
```vhdl
-- Tiered timeout system
cfg_timeout_warning_cycles : in std_logic_vector(31 downto 0);
cfg_timeout_error_cycles   : in std_logic_vector(31 downto 0);
-- Actions
cfg_timeout_warning_action : in std_logic_vector(1 downto 0);
-- 00: None, 01: Insert event, 10: Assert TLAST, 11: Both

stat_timeout_warning_count : out std_logic_vector(31 downto 0);
stat_timeout_error_count   : out std_logic_vector(31 downto 0);
```

#### 4. DMA Descriptor Integration
**Extension**: Generate DMA descriptors in hardware
```vhdl
-- Descriptor generation
cfg_enable_desc_gen     : in std_logic;
cfg_desc_base_addr      : in std_logic_vector(63 downto 0);
m_desc_tvalid           : out std_logic;
m_desc_tready           : in std_logic;
m_desc_tdata            : out std_logic_vector(127 downto 0);
```

#### 5. Performance Monitoring
**Extension**: Detailed transfer statistics
```vhdl
-- Throughput monitoring
stat_bytes_transferred    : out std_logic_vector(63 downto 0);
stat_packets_transferred  : out std_logic_vector(31 downto 0);
stat_avg_packet_size      : out std_logic_vector(15 downto 0);
-- Timing
stat_min_packet_interval  : out std_logic_vector(31 downto 0);
stat_max_packet_interval  : out std_logic_vector(31 downto 0);
-- Back-pressure
stat_backpressure_cycles  : out std_logic_vector(31 downto 0);
stat_backpressure_percent : out std_logic_vector(7 downto 0);
```

#### 6. Compression Support
**Extension**: Optional data compression before DMA
```vhdl
generic (
  ENABLE_COMPRESSION_G : boolean := false
);
-- Compression control
cfg_compression_enable : in std_logic;
cfg_compression_level  : in std_logic_vector(1 downto 0);
stat_compression_ratio : out std_logic_vector(15 downto 0);
```

---

## Common Architecture Patterns

### 1. AXI4-Lite Register Interface
**All IP cores use a common register interface pattern:**

```vhdl
-- Standard AXI4-Lite slave ports
s_axi_awaddr  : in  std_logic_vector(ADDR_WIDTH-1 downto 0);
s_axi_awvalid : in  std_logic;
s_axi_awready : out std_logic;
s_axi_wdata   : in  std_logic_vector(DATA_WIDTH-1 downto 0);
s_axi_wvalid  : in  std_logic;
s_axi_wready  : out std_logic;
-- ... (full AXI4-Lite protocol)
```

**Benefits:**
- Standard interface compatible with Xilinx IP Integrator
- Runtime configuration via software
- Status readback for debugging

### 2. AXI4-Stream Data Path
**Standardized streaming interface:**

```vhdl
-- Input stream
s_axis_tvalid : in  std_logic;
s_axis_tready : out std_logic;
s_axis_tdata  : in  std_logic_vector(TDATA_WIDTH-1 downto 0);
s_axis_tkeep  : in  std_logic_vector(TDATA_WIDTH/8-1 downto 0);
s_axis_tuser  : in  std_logic_vector(TUSER_WIDTH-1 downto 0);
s_axis_tlast  : in  std_logic;
```

**Standard Handshake:**
- Valid/Ready protocol prevents data loss
- TKEEP indicates valid bytes
- TUSER carries metadata
- TLAST marks packet boundaries

### 3. Reset Architecture
**Three-level reset system:**

```vhdl
-- 1. Asynchronous reset (aresetn)
if aresetn = '0' then
  -- Hardware reset

-- 2. Global reset (from register)
cfg_control_global_reset

-- 3. Soft clear (buffer/counter clear)
cfg_control_clear
```

### 4. Enable/Bypass Pattern
**Common control pattern in all cores:**

```vhdl
if enable_i = '0' then
  -- Disabled: create back-pressure
  ready <= '0';
elsif bypass_i = '1' then
  -- Bypass: direct connection input→output
  output <= input;
else
  -- Normal processing
  output <= processed_data;
end if;
```

### 5. Pipeline Stages
**Used for timing closure:**

```vhdl
component axi4s_pipeline_stage is
  generic (
    PIPELINE_STAGES : integer := 1
  );
  -- Adds specified number of register stages
  -- Improves timing at cost of latency
end component;
```

---

## Extension Recommendations

### Priority 1: High-Value Extensions

#### 1. Unified Statistics and Debug Interface
**Problem**: Each core has independent statistics registers
**Solution**: Common statistics aggregation and streaming interface

```vhdl
-- Unified statistics streaming
m_stat_tvalid : out std_logic;
m_stat_tdata  : out std_logic_vector(63 downto 0);
m_stat_tid    : out std_logic_vector(7 downto 0);  -- Statistic ID
-- Standard IDs:
-- 0x00-0x0F: axis_tkeep_handler
-- 0x10-0x1F: event_stream_smart_tracker
-- 0x20-0x2F: ps_host_if
```

**Benefits:**
- Centralized monitoring
- Easier debug data collection
- Support for software analytics

#### 2. Power Management
**Extension**: Clock gating and power domains

```vhdl
-- Power control
cfg_clock_gate_enable : in std_logic;
cfg_power_mode        : in std_logic_vector(1 downto 0);
-- 00: Full power
-- 01: Reduced power (lower clock)
-- 10: Standby (clock gated)
-- 11: Off

stat_power_state : out std_logic_vector(1 downto 0);
```

#### 3. Error Detection and Reporting
**Extension**: Comprehensive error handling

```vhdl
-- Error detection
error_o             : out std_logic;
error_code_o        : out std_logic_vector(7 downto 0);
error_timestamp_o   : out std_logic_vector(63 downto 0);

-- Error codes:
-- 0x01: FIFO overflow
-- 0x02: Protocol violation
-- 0x03: Timestamp error
-- 0x04: Configuration error
-- ...
```

### Priority 2: Performance Enhancements

#### 1. Parallel Processing
**Extension**: Multiple parallel processing lanes

```vhdl
generic (
  NUM_LANES_G : positive range 1 to 4 := 1
);
-- Distribute events across lanes for higher throughput
```

#### 2. Hardware Acceleration
**Extension**: Dedicated accelerators for common operations

```vhdl
-- Histogram accelerator
cfg_enable_histogram : in std_logic;
stat_histogram_bin_0 : out std_logic_vector(31 downto 0);
stat_histogram_bin_1 : out std_logic_vector(31 downto 0);
-- ...

-- Event rate calculator
stat_event_rate_hz   : out std_logic_vector(31 downto 0);
```

#### 3. Prefetching and Caching
**Extension**: Predictive data movement

```vhdl
-- Prefetch configuration
cfg_prefetch_enable : in std_logic;
cfg_prefetch_depth  : in std_logic_vector(7 downto 0);
```

### Priority 3: Interface Extensions

#### 1. PCIe DMA Support
**Extension**: Direct PCIe DMA interface (alternative to PS DMA)

```vhdl
-- PCIe streaming interface
m_pcie_tvalid : out std_logic;
m_pcie_tready : in std_logic;
m_pcie_tdata  : out std_logic_vector(511 downto 0);  -- 512-bit PCIe
m_pcie_tkeep  : out std_logic_vector(63 downto 0);
```

#### 2. Ethernet Streaming
**Extension**: Direct Ethernet UDP streaming

```vhdl
-- UDP packet generation
cfg_udp_enable     : in std_logic;
cfg_udp_dest_ip    : in std_logic_vector(31 downto 0);
cfg_udp_dest_port  : in std_logic_vector(15 downto 0);
```

### Priority 4: Advanced Features

#### 1. Machine Learning Integration
**Extension**: Pre-processing for ML inference

```vhdl
-- Feature extraction
cfg_enable_features : in std_logic;
m_features_tvalid   : out std_logic;
m_features_tdata    : out std_logic_vector(127 downto 0);
```

#### 2. Multi-Sensor Fusion
**Extension**: Combine multiple event cameras

```vhdl
generic (
  NUM_SENSORS_G : positive range 1 to 4 := 1
);
-- Timestamp synchronization across sensors
-- Spatial alignment
-- Event correlation
```

---

## Development Guidelines

### Adding New Functionality

#### Step 1: Generic Parameters
Add compile-time configuration:
```vhdl
generic (
  ENABLE_NEW_FEATURE_G : boolean := false;
  FEATURE_PARAMETER_G  : positive := 16
);
```

#### Step 2: Register Interface
Add runtime control registers:
```vhdl
-- In register bank package:
cfg_feature_control_o : out std_logic_vector(31 downto 0);
stat_feature_status_i : in  std_logic_vector(31 downto 0);
```

#### Step 3: Conditional Generation
Use VHDL generate statements:
```vhdl
feature_gen: if (ENABLE_NEW_FEATURE_G = true) generate
  -- Feature implementation
  feature_inst : feature_module
    port map (...);
end generate feature_gen;
```

#### Step 4: Testbench
Create comprehensive testbench:
```vhdl
-- In tb/src/ directory
-- Test all modes: enabled, disabled, bypass
-- Test corner cases and error conditions
```

### Best Practices

#### 1. Maintain AXI Compatibility
- Always use standard AXI4-Stream interfaces
- Follow AXI protocol specifications
- Use Xilinx AXI Protocol Checker IP for verification

#### 2. Parameterization
- Make widths and depths generic
- Allow compile-time feature selection
- Provide sensible defaults

#### 3. Documentation
- Comment all generics and ports
- Document register map
- Provide timing diagrams for complex operations

#### 4. Simulation
- Create testbenches for new modules
- Use assertions for protocol checking
- Test with realistic event data patterns

#### 5. Timing Closure
- Add pipeline stages as needed
- Use registered outputs
- Avoid long combinatorial paths

### Integration Checklist

When adding a new module:

- [ ] Define clear generic parameters
- [ ] Implement AXI4-Stream interfaces
- [ ] Add AXI4-Lite registers if needed
- [ ] Support enable/disable/bypass
- [ ] Implement proper reset handling
- [ ] Add debug statistics
- [ ] Create testbench
- [ ] Document operation
- [ ] Verify timing
- [ ] Test with hardware

---

## Conclusion

The fpga-projects repository contains a well-architected event processing pipeline with three main IP cores:

1. **axis_tkeep_handler**: Ensures data alignment for Event Format 2.1
2. **event_stream_smart_tracker**: Manages back-pressure with intelligent event handling
3. **ps_host_if**: Interfaces with PS for efficient DMA transfers

**Key Strengths:**
- Modular architecture with clear separation of concerns
- Extensive configurability via generics and registers
- Standards-compliant AXI interfaces
- Comprehensive testbenches

**Extension Opportunities:**
- Enhanced statistics and monitoring
- Performance optimizations (parallel processing, caching)
- Additional interfaces (PCIe, Ethernet)
- Advanced features (ML integration, multi-sensor fusion)

**Recommended Next Steps:**
1. Implement unified statistics interface
2. Add power management capabilities
3. Enhance error detection and reporting
4. Develop advanced filtering and transformation features

The modular design and extensive use of VHDL generics make the codebase highly extensible while maintaining backward compatibility.

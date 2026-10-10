# SPI Wrapper UVM Verification

A complete **UVM (SystemVerilog) verification environment** for an SPI slave, a single-port RAM, and the **SPI wrapper** that connects them. The project is split into three parts: the SPI slave and the RAM are each verified on their own, then the integrated SPI wrapper is verified with a third environment that reuses the same ideas and binds the block-level assertions into the wrapper.

Each part includes a verification plan, a reference (golden) model, a full UVM testbench, SystemVerilog Assertions (SVA), and functional, code, and assertion coverage collected in QuestaSim.

**Authors**
- Rawan Khaled Yahia
- Waad Rabie Mahmoud

Group 2, 2025

---

## Table of Contents

- [Project Structure](#project-structure)
- [Design Under Test](#design-under-test)
- [UVM Architecture](#uvm-architecture)
- [Verification Plan](#verification-plan)
- [Bugs Found and Fixed](#bugs-found-and-fixed)
- [Assertions](#assertions)
- [Sequences](#sequences)
- [Coverage Results](#coverage-results)
- [How to Run](#how-to-run)
- [Tools](#tools)
- [Authors](#authors)

---

## Project Structure

| Part | Design under test | What is verified |
|------|-------------------|------------------|
| **1. SPI Slave** | `spi_slave` | FSM behavior, MOSI deserialization, MISO serialization, `rx_valid` / `tx_valid` timing |
| **2. RAM** | `spr_sync` (single-port RAM) | Reset, write/read address and data operations, `tx_valid` behavior |
| **3. SPI Wrapper** | `spi_wrapper` (slave + RAM) | End-to-end write and read transactions through the SPI pins only |

A suggested repository layout:

```
.
├── part1_slave/
│   ├── design/            # spi_slave.v (with the bug fixes below)
│   ├── golden_model/
│   ├── uvm/               # interface, config, package, seq item, driver, monitor,
│   │                      # sequencer, agent, scoreboard, coverage, env, sequences, test
│   ├── sva/
│   ├── top.sv
│   ├── src_files.list
│   └── run.do
├── part2_ram/             # same structure as part 1
├── part3_wrapper/         # same structure, plus bound slave and RAM assertions
├── docs/                  # UVM diagrams, verification plans, QuestaSim snippets
└── README.md
```

## Design Under Test

### SPI Slave

A five-state FSM (`IDLE`, `CHK_CMD`, `WRITE`, `READ_ADD`, `READ_DATA`) that deserializes `MOSI` into a 10-bit `rx_data` word, asserts `rx_valid`, and in the read-data state serializes the RAM output (`tx_data`, `tx_valid`) onto `MISO`.

The first 3 bits on `MOSI` after `SS_n` falls select the command:

| MOSI bits | Operation     |
|-----------|---------------|
| `000`     | Write address |
| `001`     | Write data    |
| `110`     | Read address  |
| `111`     | Read data     |

FSM transitions:

```
IDLE ─► CHK_CMD ─► WRITE ─► IDLE
                ├► READ_ADD ─► IDLE
                └► READ_DATA ─► IDLE
```

### Single-Port RAM

Parameterized by `MEM_DEPTH` and `ADDR_SIZE`. It takes a 10-bit `din` plus `rx_valid`. The top two bits `din[9:8]` select the operation:

| `din[9:8]` | Operation     |
|------------|---------------|
| `00`       | Write address |
| `01`       | Write data    |
| `10`       | Read address  |
| `11`       | Read data     |

`tx_valid` is asserted while the read data is being returned.

### SPI Wrapper

Instantiates the slave and the RAM and exposes only `MOSI`, `MISO`, `SS_n`, `clk`, and `rst_n`.

## UVM Architecture

All three environments share the same structure:

```
top
└── uvm_test
    └── env
        ├── agent
        │   ├── sequencer ──► driver ──► (virtual interface) ──► DUT
        │   └── monitor   ◄── (virtual interface) ◄──────────────┘
        ├── scoreboard          (compares DUT outputs with the reference model)
        └── coverage collector  (functional coverage)
```

| Component                | Role                                                                                   |
|--------------------------|----------------------------------------------------------------------------------------|
| **Interface**            | Connects the DUT to the testbench                                                      |
| **Configuration object** | Holds the virtual interface and agent settings, shared through the `uvm_config_db`     |
| **Shared package**       | Common types, parameters, and includes                                                 |
| **Sequence item**        | Randomized transaction with constraints on `rst_n`, `SS_n`, `MOSI`, and valid commands |
| **Driver / Sequencer**   | Turns sequence items into pin-level activity                                           |
| **Monitor**              | Samples the interface and sends transactions to the scoreboard and coverage collector  |
| **Scoreboard**           | Checks results against the golden / reference model                                    |
| **Coverage collector**   | Covergroups for the plan items below                                                   |
| **SVA**                  | Concurrent assertions bound to the DUT                                                 |

The configuration database passes the virtual interface and the configuration object from `top` to the test and the agent (for the slave: `SPI_SLAVE_IF` and `SPI_SLAVE_CFG`).

## Verification Plan

The plan maps each requirement to its stimulus, functional coverage, and checking method. A sample from each part:

**SPI Slave**

| Label        | Requirement                                                                                    | Check                                                    |
|--------------|------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| SPI_SLAVE_1  | On reset, all outputs are low                                                                  | Concurrent assertion + reference model                   |
| SPI_SLAVE_2  | `SS_n` low throughout a transaction; `rx_valid` asserts after deserializing 10 bits            | Concurrent assertions + reference model                  |
| SPI_SLAVE_3  | First 3 MOSI bits are a valid command (`000`, `001`, `110`, `111`)                             | Concurrent assertions + reference model                  |
| SPI_SLAVE_4  | `tx_valid` is high while serializing read data on MISO                                         | Reference model                                          |
| SPI_SLAVE_5  | Correct FSM transitions                                                                        | Concurrent assertions in the design                      |

**RAM**

| Label | Requirement                                                                           | Check                                  |
|-------|---------------------------------------------------------------------------------------|----------------------------------------|
| RAM_1 | On reset, all outputs are low                                                         | Concurrent assertion + reference model |
| RAM_2 | When `rx_valid` is high, `din[9:8]` selects write/read address or write data          | Reference model                        |
| RAM_3 | `tx_valid` is high while reading data                                                 | Concurrent assertions + reference model|
| RAM_4 | A write-address operation is always followed by write address or write data           | Concurrent assertion + reference model |

**SPI Wrapper**: the same requirements checked end to end, plus:

| Label      | Requirement                                                                             |
|------------|-----------------------------------------------------------------------------------------|
| WRAPPER_1  | On reset, `MISO` is low                                                                 |
| WRAPPER_2/3| Valid command sequences and `rx_valid` timing (bound slave assertions)                  |
| WRAPPER_4  | `000` selects write address and `001` selects write data (bound RAM assertions)         |
| WRAPPER_5  | `110` selects read address and `111` selects read data (bound RAM assertions)           |
| WRAPPER_6  | Full command sequencing across write and read operations                                |

**Stimulus highlights**
- Reset is directed at the start of simulation, then randomized with a constraint that keeps it off most of the time.
- `SS_n` goes high for one cycle every 13 cycles in normal operations, and every 23 cycles for read data (which needs 10 extra cycles to serialize `MISO`).
- `MOSI` is driven bit by bit from a randomized array; the first 3 bits are constrained to valid commands.

## Bugs Found and Fixed

Verification exposed the following bugs in the original RTL.

**SPI Slave**
- `received_address` check was inverted; it should be `!received_address`.
- `counter` was not initialized to zero in reset.
- In `READ_DATA`, the counter should be `9`, not `8`, so the FSM waits for `rx_valid` to be asserted.

**RAM**
- Added the `MEM_DEPTH` and `ADDR_SIZE` parameters.
- In the read-data case (`11`), the memory is read with `MEM[Rd_Addr]` instead of `MEM[Wr_Addr]`.
- `tx_valid` is now set inside the case branches so that it stays asserted while the data is read out on `MISO`, instead of pulsing once.
- Added the missing `begin`/`end` on the `else` branch of `if (~rst_n)`.

## Assertions

| Block   | Properties checked                                                                                                                                  |
|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| Slave   | Reset behavior; FSM transitions (`IDLE`, `CHK_CMD`, `WRITE`, `READ_ADD`, `READ_DATA`); for each valid command sequence, `rx_valid` asserts after 10 cycles and `SS_n` eventually closes the communication |
| RAM     | Reset behavior; `tx_valid` stays low during address/data phases; `tx_valid` rises after a read-data operation and eventually falls; write address is followed by write address/data; read address is followed by read data |
| Wrapper | After reset `MISO` goes low; `MISO` stays stable when not in `READ_DATA` (bound together with the slave and RAM assertions)                          |

Example (wrapper reset check):

```systemverilog
(@(posedge spi_wrapperif.clk) (!spi_wrapperif.rst_n) |-> ##1 (spi_wrapperif.MISO == 0))
```

## Sequences

| Sequence             | Used in       | Description                                                                                  |
|----------------------|---------------|----------------------------------------------------------------------------------------------|
| Reset sequence       | All parts     | Applies reset at the start of simulation                                                     |
| Main sequence        | Slave         | Randomized commands with the constraints from the verification plan                          |
| Write sequence       | RAM, Wrapper  | Write address followed by write address or write data                                        |
| Read sequence        | RAM, Wrapper  | Read address followed by read data                                                           |
| Write-read sequence  | RAM, Wrapper  | Mixed traffic; after write data the next operation is read address (60%) or write address (40%) |

## Coverage Results

Collected in QuestaSim. Code coverage (toggle) shows every DUT port toggled in the wrapper run.

### Functional coverage

| Environment | Covergroup        | Coverage | Notes                                                                       |
|-------------|-------------------|---------:|-----------------------------------------------------------------------------|
| Slave       | `spi_slave_coverage` | 96.25% | `SS_n`, `MOSI`, and `SS_n x MOSI` cross at 100%; `rx_data` at 85%           |
| RAM         | `ram_coverage`    | 100%     | `din`, `rx_valid`, `tx_valid`, and both crosses at 100%                     |
| Wrapper     | `spi_slave_coverage` | 96.25% | Same breakdown as the slave environment                                     |
| Wrapper     | `ram_coverage`    | 97.14%   | `din` at 85.71%; `rx_valid`, `tx_valid`, and both crosses at 100%           |

### Assertion coverage

In all three environments there were **0 assertion failures**, and every cover directive was hit (100%). Examples from the wrapper run include the reset cover, the stable-`MISO` cover, and the FSM state covers (`IDLE`, `CHK_CMD`, `WRITE`, `READ_ADD`, `READ_DATA`).

## How to Run

Each part has its own source list and `.do` file. From inside a part's folder, in QuestaSim:

```tcl
do run.do
```

A typical run script compiles the DUT and the UVM testbench, runs the test, and saves coverage:

```tcl
vlib work
vlog -f src_files.list +cover -covercells
vsim -voptargs=+acc work.top -cover
add wave /top/*
run -all
coverage save top.ucdb
```

> Adjust the script names and paths to match your folders.

## Tools

- **SystemVerilog** and **UVM** for the testbench
- **SVA** for concurrent assertions
- **QuestaSim** for simulation, waveform debugging, and coverage

## Authors

| Name                  | Group |
|-----------------------|-------|
| Rawan Khaled Yahia    | G2    |
| Waad Rabie Mahmoud    | G2    |

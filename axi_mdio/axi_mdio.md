# Overview

The MDIO Controller (axi_mdio) bridges an AXI4-Lite register interface to the MDIO bus, enabling software to read and write registers on a 10G Ethernet PHY using the IEEE 802.3 Clause 45 two-frame protocol.

A complete Clause 45 transaction consists of two 64-bit MDIO frames: an address frame that sets the target register, followed by a read or write data frame. The module handles preamble generation, frame serialization/deserialization, MDC clock generation, and tri-state control.

# Ports

## Clocks

| Name | Direction | Description       |
|------|-----------|-------------------|
| clk  | input     | Main module clock |

## Resets

| Name | Active-level | Direction | Description                   |
|------|--------------|-----------|-------------------------------|
| rst  | high         | input     | Synchronous active-high reset |

## Parameters

| Name        | Default       | Description                          |
|-------------|---------------|--------------------------------------|
| CLK_FREQ_HZ | 100,000,000  | System clock frequency in Hz         |
| MDC_FREQ_HZ | 2,500,000    | Desired MDC clock frequency in Hz    |

## MDIO Physical Interface

The MDIO bus uses separate out/oe/in signals (FPGA tri-state convention). The top-level IOB handles tri-state: `assign mdio = mdio_oe ? mdio_out : 1'bz;`

| Name     | Width | Direction | Description                        |
|----------|-------|-----------|------------------------------------|
| mdc      | 1     | output    | MDIO clock (gated, only during TX) |
| mdio_out | 1     | output    | MDIO data output                   |
| mdio_oe  | 1     | output    | MDIO output enable (active high)   |
| mdio_in  | 1     | input     | MDIO data input from PHY           |

## Interrupt

| Name | Width | Direction | Description                                          |
|------|-------|-----------|------------------------------------------------------|
| irq  | 1     | output    | Level-sensitive interrupt, masked by IRQ_EN register |

## Register Interface

The register interface is an AXI4-Lite subordinate interface (`reg_axi_*` prefix) that provides software access to control and status registers. It uses 32-bit data with a 5-bit address bus (32 bytes of address space).

### Register Interface Ports

| Name              | Width | Direction | Description              |
|-------------------|-------|-----------|--------------------------|
| reg_axi_awready   | 1     | output    | Write address ready      |
| reg_axi_awvalid   | 1     | input     | Write address valid      |
| reg_axi_awaddr    | 5     | input     | Write address            |
| reg_axi_awprot    | 3     | input     | Write protection type    |
| reg_axi_wready    | 1     | output    | Write data ready         |
| reg_axi_wvalid    | 1     | input     | Write data valid         |
| reg_axi_wdata     | 32    | input     | Write data               |
| reg_axi_wstrb     | 4     | input     | Write byte strobes       |
| reg_axi_bready    | 1     | input     | Write response ready     |
| reg_axi_bvalid    | 1     | output    | Write response valid     |
| reg_axi_bresp     | 2     | output    | Write response           |
| reg_axi_arready   | 1     | output    | Read address ready       |
| reg_axi_arvalid   | 1     | input     | Read address valid       |
| reg_axi_araddr    | 5     | input     | Read address             |
| reg_axi_arprot    | 3     | input     | Read protection type     |
| reg_axi_rready    | 1     | input     | Read data ready          |
| reg_axi_rvalid    | 1     | output    | Read data valid          |
| reg_axi_rdata     | 32    | output    | Read data                |
| reg_axi_rresp     | 2     | output    | Read response            |

# Generated Register Block

The register interface logic is generated from `axi_mdio_regs.rdl` using [PeakRDL-regblock](https://github.com/SystemRDL/PeakRDL-regblock). The generated files (`axi_mdio_regs/axi_mdio_regs.sv` and `axi_mdio_regs/axi_mdio_regs_pkg.sv`) should not be edited by hand. To regenerate after modifying the RDL source:

```bash
peakrdl regblock axi_mdio/axi_mdio_regs.rdl \
    -o axi_mdio/axi_mdio_regs \
    --cpuif axi4-lite-flat \
    --default-reset rst \
    --err-if-bad-addr
```

# Register Map

All registers are 32-bit, byte-addressed, 4-byte aligned.

| Offset | Name     | R/W | Bits                              | Description                                                                                      |
|--------|----------|-----|-----------------------------------|--------------------------------------------------------------------------------------------------|
| 0x00   | CTRL     | R/W | [0] go, [1] wr_nrd, [2] pre_dis   | Bit 0: trigger transaction (self-clears). Bit 1: direction (1=write, 0=read). Bit 2: suppress 32-bit preamble on both frames. |
| 0x04   | PHY_ADDR | R/W | [4:0] prtad, [12:8] devad         | PHY port address (PRTAD) and device address (DEVAD).                                             |
| 0x08   | REG_ADDR | R/W | [15:0] addr                       | 16-bit register address for the Clause 45 address frame.                                         |
| 0x0C   | WRDATA   | R/W | [15:0] data                       | Write data to send during MDIO write operations.                                                 |
| 0x10   | RDDATA   | RO  | [15:0] data                       | Read data captured from PHY during MDIO read operations.                                         |
| 0x14   | STATUS   | RO  | [0] busy, [1] done, [2] error     | Bit 0: transaction in progress. Bit 1: done (latched, cleared on read). Bit 2: TA error (latched, cleared on read). |
| 0x18   | IRQ_STATUS | R/W | [0] done, [1] error              | Latched interrupt status. Write 1 to a bit to clear it (W1C).                                                        |
| 0x1C   | IRQ_EN   | R/W | [0] done_en, [1] error_en         | Interrupt enable mask. `irq` output = `|(IRQ_STATUS & IRQ_EN)`. Level-sensitive; clears when SW writes 1 to IRQ_STATUS. |

Accesses to undefined register offsets return DECERR.

# Clause 45 Protocol

A complete transaction consists of two 64-bit MDIO frames:

**Frame 1 (Address):** `PRE(32x1) | ST(00) | OP(00) | PRTAD(5) | DEVAD(5) | TA(10) | REG_ADDR(16)`

**Frame 2 (Read):** `PRE(32x1) | ST(00) | OP(11) | PRTAD(5) | DEVAD(5) | TA(Z0) | DATA_IN(16)`

**Frame 2 (Write):** `PRE(32x1) | ST(00) | OP(01) | PRTAD(5) | DEVAD(5) | TA(10) | DATA_OUT(16)`

MDIO output is driven on MDC falling edge; MDIO input is sampled on MDC rising edge. During read turnaround (TA), the controller releases the bus and checks that the PHY drives bit 1 low. A TA violation sets the error status bit.

# Usage Example

```c
// Enable done interrupt
IRQ_EN = 0x01;  // done_en=1

// Write 0x1234 to PHY 0x01, device 0x03, register 0x0000
REG_ADDR = 0x0000;
PHY_ADDR = (0x03 << 8) | 0x01;
WRDATA   = 0x1234;
CTRL     = 0x03;  // go=1, wr_nrd=1
while (STATUS & 0x01);  // wait for busy to clear

// Read from PHY 0x01, device 0x03, register 0x0000
REG_ADDR = 0x0000;
PHY_ADDR = (0x03 << 8) | 0x01;
CTRL     = 0x01;  // go=1, wr_nrd=0
while (STATUS & 0x01);  // wait for busy to clear
uint16_t data = RDDATA;
```

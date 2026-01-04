# Synchronous FIFO (Single-Clock FIFO) --- RTL Explanation  
### Overview  
This module implements a parameterized synchronous FIFO (First-In-First-Out) buffer using a single clock domain.  
It supports independent read/write enables, full/empty detection, and configurable data width and depth.  

The design uses binary read/write pointers with an extra MSB to correctly distinguish between empty and full conditions.  
  
##
  
### Interface Description
#### Parameters
``` Verilog
parameter FIFO_DEPTH = 8;
parameter DATA_WIDTH = 32;
```
* `FIFO_DEPTH`: Number of entries in the FIFO
* `DATA_WIDTH`: Bit-width of each data entry
  
The address width is automatically derived using `$clog2(FIFO_DEPTH)`.
  
##
  
### Ports  
|  Direction  |      Signal      | Desciption             |
| ----------- | ---------------- | -----------------------|
| Input  |      clk       | System clock                  |
| Input  |     rst_n      | Active-low asynchronous reset |
| Input  |       cs       | Chip select (FIFO enable)     |
| Input  |     wr_en      | Write enable                  |
| Input  |     rd_en      | Read enable                   |
| Input  | [DATA_WIDTH-1:0] data_in  | Data input         |
| Output | [DATA_WIDTH-1:0] data_out | Data output        |
| Output |     empty      | FIFO empty flag               |
| Output |     full       | FIFO full flag                |          
  
##
  
### Internal Architecture
### 1. FIFO Memory Array
``` Verilog
reg [DATA_WIDTH-1:0] fifo [0:FIFO_DEPTH-1];
```
* `FIFO_DEPTH`: Number of entries in the FIFO
* `DATA_WIDTH`: Bit-width of each data entry
### 2. Read/Write Pointers with Extra MSB
``` Verilog
reg [FIFO_DEPTH_LOG:0] write_pointer;
reg [FIFO_DEPTH_LOG:0] read_pointer;
```
Each pointer is one bit wider than the address width:  
* Lower bits → index into the FIFO memory
* MSB → wrap-around (phase) bit

This extra bit allows the design to distinguish:  
* Empty: pointers equal
* Full: pointers equal except MSB inverted
  
##
  
### Write Operation
``` Verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        write_pointer <= 0;
    end
    else if (cs && wr_en && !full) begin
        fifo[write_pointer[FIFO_DEPTH_LOG-1:0]] <= data_in;
        write_pointer <= write_pointer + 1'b1;
    end
end
```
#### Behavior
* On reset, the write pointer is cleared.
* When `cs` and `wr_en` are asserted and FIFO is not full:
  * Input data is written to memory.
  * Write pointer increments.
##
  
### Read Operation
``` Verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        read_pointer <= 0;
    end
    else if (cs && rd_en && !empty) begin
        data_out <= fifo[read_pointer[FIFO_DEPTH_LOG-1:0]];
        read_pointer <= read_pointer + 1'b1;
    end
end
```
#### Behavior
* On reset, the read pointer is cleared.
* When `cs` and `rd_en` are asserted and FIFO is not empty:
  * Data is read from memory.
  * Read pointer increments.  
##
  
### Empty and Full Detection
### Empty Condition
``` Verilog
assign empty = (read_pointer == write_pointer);
```
* FIFO is empty when read and write pointers are identical.

### Full Condition
``` Verilog
assign full = (read_pointer == {~write_pointer[FIFO_DEPTH_LOG], write_pointer[FIFO_DEPTH_LOG-1:0]});
```
* FIFO is full when:
  * Lower address bits match
  * Lower address bits match
This is a standard and efficient full-detection technique for circular buffers.
  
##
  
### Design Characteristics

* :ballot_box_with_check: Single-clock synchronous
* :ballot_box_with_check: Parameterized depth and data width
* :ballot_box_with_check: No counters required (pointer-based full/empty detection)
* :ballot_box_with_check: Synthesizable and scalable
* :ballot_box_with_check: Suitable for FPGA or ASIC front-end design
---
# 🧪 FIFO Synchronous Testbench Verification
### Overview  
This testbench is designed to systematically verify the functionality and boundary behavior of the `fifo_sync` module.  
It focuses on functional correctness, full/empty handling, and read/write sequencing, rather than random stress testing.  

The verification environment is intentionally kept simple, readable, and deterministic, making it suitable for:  
* RTL functional validation
* Interview / portfolio demonstration
* Learning reference for synchronous FIFO behavior 
  
##
  
### Testbench Architecture
#### Clock Generation
``` Verilog
always begin
    #5 clk = ~clk;
end
```
* Generates a 10 ns clock period
* All FIFO operations are synchronized to the positive edge of `clk`
  
##
  
### DUT Instantiation  
``` Verilog
fifo_sync #(
    .FIFO_DEPTH(FIFO_DEPTH),
    .DATA_WIDTH(DATA_WIDTH)
) dut ( ... );
```
* FIFO depth and data width are parameterized
* Allows easy scalability and reuse
  
##
  
### Transaction-Level Tasks  
To improve readability and modularity, FIFO operations are abstracted into tasks.  
### Write Task  
``` Verilog
task write_data (input [DATA_WIDTH-1:0] d_in);
    begin
        @(posedge clk);
        cs = 1; wr_en = 1;
        data_in = d_in;
        $display($time, " write_data data_in = %0d", data_in);
        @(posedge clk);
        cs = 1; wr_en = 0;
    end
endtask
```
#### Behavior
* Generates a one-cycle write pulse
* Data is written only when `wr_en` is asserted
* Time-stamped logging improves waveform correlation
  
##

### Read Task
``` Verilog
task read_data ();
    begin
        @(posedge clk);
        cs = 1; rd_en = 1;
        @(posedge clk);
        $display($time, " read_data data_out = %0d", data_out);
        cs = 1; rd_en = 0;
    end
endtask
```
#### Behavior
* Generates a one-cycle read pulse
* aptures and prints FIFO output data
* Ensures reads are aligned to clock edges
  
##

### Verification Scenarios
#### 🔹 Scenario 1 — Basic FIFO Operation
``` Verilog
write_data(1);
write_data(10);
write_data(100);
read_data();
read_data();
read_data();
```
#### Purpose
* Verifies basic FIFO ordering (FIFO property)
* Confirms correct data sequence after multiple writes
* Ensures empty condition is reached correctly
  
#### 🔹 Scenario 2 — Alternating Write / Read
``` Verilog
for (i = 0; i < FIFO_DEPTH; i = i + 1) begin
    write_data(2**i);
    read_data();
end
```
#### Purpose
* Tests FIFO behavior under continuous producer/consumer interaction
* Verifies correct pointer update without accumulation
* Confirms no false full/empty assertions
  
#### 🔹 Scenario 3 — Full and Empty Boundary Test
``` Verilog
for (i = 0; i <= FIFO_DEPTH; i = i + 1) begin
    write_data(2**i);
end

for (i = 0; i < FIFO_DEPTH; i = i + 1) begin
    read_data();
end
```
#### Purpose
* Intentionally pushes FIFO to full condition
* Confirms no false full/empty assertions
  * Writes are blocked when `full` is asserted
  * Reads correctly drain all stored data
  * FIFO returns to empty state cleanly

##
   
### Design Philosophy

* :ballot_box_with_check: Deterministic test cases (easy to debug and review)

* :ballot_box_with_check: Task-based abstraction for clean stimulus generation

* :ballot_box_with_check: Covers normal, boundary, and stress-adjacent behaviors

* :ballot_box_with_check: Designed for clarity and maintainability rather than randomness

This testbench is intended to demonstrate verification thinking, not just simulation execution.

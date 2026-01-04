# -Justin-.github.io-fifo_sync

  ---------

## Synchronous FIFO (Single-Clock FIFO) --- RTL Explanation  
### Overview  
This module implements a parameterized synchronous FIFO (First-In-First-Out) buffer using a single clock domain.  
It supports independent read/write enables, full/empty detection, and configurable data width and depth.  

The design uses binary read/write pointers with an extra MSB to correctly distinguish between empty and full conditions.  

  ---

## Interface Description
Parameters

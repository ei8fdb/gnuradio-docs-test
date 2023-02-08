# Types of Blocks

## Introduction

To take advantage of the gnuradio framework, users will create various blocks to implement the desired data processing. There are several types of blocks to choose from:

- Synchronous Blocks (1:1)
- Decimation Blocks (N:1)
- Interpolation Blocks (1:M)
- Basic (a.k.a. General) Blocks (N:M)

## Synchronous Block

The sync block allows users to write blocks that consume and produce an equal number of items per port. A sync block may have any number of inputs or outputs. When a sync block has zero inputs, its called a source. When a sync block has zero outputs, its called a sink.

An example sync block in C++: 

```
some code
```

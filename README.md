# CFG-Tools
A simple, header-only C++ tool utilizing the Boost Graph Library (BGL) to modularize and linearize Control Flow Graphs (CFGs).

## Overview

CFG-Tools is a lightweight, header-only C++ library built on BGL for analyzing and transforming CFGs. 
It provides reusable tools that simplify common compiler and program analysis tasks without requiring a custom graph representation.

The library is intended for projects involving compiler infrastructure, decompilers, binary analysis, program transformations, and intermediate representations. 
By using standardized ```boost::adjacency_list``` it can be easily integrated into existing pipelines.

Current functionality includes:
* **Linearization** - Produce a deterministic linear ordering of CFG basic blocks suitable for emitting source code, assembly, or intermediate representations.
* **Modularization** - Partition a CFG into reusable modules for higher-level analysis and transformation.

The library is implemented as a collection of header-only templates, requiring no additional builds beyond including the headers.


## Importing
Importing all library functions just requires
```
#import "cfg_tools/common.hpp"
```

## Documentation and Examples
All documentation and examples for each can be found:

- [Linearization](docs/linearization.md) - Linearize a CFG into sequential text
- [Modularization](docs/modularization.md) - Build a modular map from a CFG

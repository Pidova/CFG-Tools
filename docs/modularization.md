# Modularization
A lightweight, header-only C++ tool designed to identify and extract Single-Entry Multiple-Exit (SEME) regions from a Control Flow Graph (CFG).

## What is CFG Modularization?

In compiler infrastructure, functions are analyzed and optimized using a multi-directional graph representation called a **CFG**. 
Loops and structured sub-regions within that graph are a primary target for transformations such as loop unrolling, invariant hoisting, or abstraction into callable blocks.

**Modularization** identifies every loop region in the graph and maps each region's entry block (the loop header) to the set of internal 
body blocks that have edges leaving the region, its exit sources.

## Idea
Given input CFG
```
        init
          |
      loop_cond <---
      /       \    |
   body        exit|
   /  \            |
break  inc---------|
```
Its modularized SEME regions would be:
```
entry: loop_cond
{
    exit-source: body  ->  break (outside region)
}
```
**loop_cond** is the single entry. **body** is the interior block that branches to **break**, which lives outside the loop region. 
Back-edges from **inc** back to **loop_cond** are internal re-entries and do not qualify as exit sources.

## Algorithm

### Back-edge detection via iterative DFS

Vertices are colored in three states:
* White - Unvisited 
* Gray  - On the DFS stack 
* Black - Fully processed 

The DFS maintains an explicit stack of **(vertex, out_edge_iterator)** pairs. 
When the iterator for a vertex is exhausted the vertex is colored black and popped. 
When a gray target is encountered on an out-edge, a **back-edge** has been found and a loop region is identified with that gray target as the header.

* Time: O(V + E)
* Memory: O(V)

### Flood-fill the loop body (flood_loop_body)

Starting from the back-edge source, a reverse BFS walks predecessor edges to collect every block that lies between the source and the header. 
The header acts as the boundary and is never added to the body set.

* Time: O(V  E) per back-edge (reverse-edge lookup without an in-edge index)
* Memory: O(B) where B is the number of body blocks

### Register exit sources (extract_and_register_exits)

Each body block's out-edges are examined. A block is recorded as an exit source when it has at least one edge that leads to a block that is:
* outside the body set, **and**
* not the header itself (edges back to the header are internal re-entries, recursive or loop-back, and do not count)

Results are merged into the SEME registry keyed on the header block.

* Time: O(B  d) where d is average out-degree
* Memory: O(R) where R is the number of registered exit sources

## Determinism

Results are deterministic provided the underlying Boost graph exposes deterministic vertex and edge iteration (vecS).

Graphs using unordered or implementation-defined containers may produce different but semantically equivalent region sets.

## Example
C Code below was compiled to X86 Assembly then generated with boost graphs.

```c
int cond;
for (int i = 0; i < 10; i++) {
    if (condition_var == 5) {
        break;
    }
}
return 0;
```

Example code to modularize:
```cpp
#include "cfg_tools/common.hpp"
#include <iostream>

struct node {
      std::string str = "";
};
struct edge {
      std::string str = "";
};
using Graph = boost::adjacency_list<boost::vecS, boost::vecS, boost::directedS, node, edge>;
Graph g;

/* int i = 0; */
auto v_init = boost::add_vertex(node{
                                    ".L_init:\n"
                                    "    mov dword ptr [rbp-4], 0\n"}, g);
/* for i < 10 */
auto v_cond = boost::add_vertex(node{
                                    ".L_loop_cond:\n"
                                    "    cmp dword ptr [rbp-4], 10\n"
                                    "    jge .L_loop_exit\n"}, g);
/*  cond == 5 */
auto v_body = boost::add_vertex(node{
                                    ".L_loop_body:\n"
                                    "    cmp dword ptr [rbp-8], 5\n"
                                    "    je .L_loop_break\n"}, g);
/* then break */
auto v_if_break = boost::add_vertex(node{
                                        ".L_loop_break:\n"
                                        "    jmp .L_loop_exit\n"}, g);
/* else ++i */
auto v_incr = boost::add_vertex(node{
                                    ".L_loop_inc:\n"
                                    "    add dword ptr [rbp-4], 1\n"
                                    "    jmp .L_loop_cond\n"}, g);
/* return 0; */
auto v_exit = boost::add_vertex(node{
                                    ".L_loop_exit:\n"
                                    "    xor eax, eax\n"}, g);
/* Structure loop */
boost::add_edge(v_init, v_cond, edge{}, g);
boost::add_edge(v_cond, v_body, edge{}, g);
boost::add_edge(v_cond, v_exit, edge{}, g);
boost::add_edge(v_body, v_if_break, edge{}, g);
boost::add_edge(v_body, v_incr, edge{}, g);
boost::add_edge(v_if_break, v_exit, edge{}, g);
boost::add_edge(v_incr, v_cond, edge{}, g);

/* Execute */
for (const auto &[b, v] : cfg_tools::modularize(g)) {
      std::cout << "Block: " << g[b].str << "{\n";
      for (const auto &i : v) {
            std::cout << "B: " << g[i].str << std::endl;
      }
      std::cout << "}\n";
}
```

Output:
```
Block: .L_loop_cond:
    cmp dword ptr [rbp-4], 10
    jge .L_loop_exit
{
B: .L_loop_body:
    cmp dword ptr [rbp-8], 5
    je .L_loop_break
}
```

**loop_cond** is identified as the SEME entry. 
**loop_body** is its sole exit source, it holds the early-break branch that jumps to **loop_break**, a block outside the loop region. 
**loop_inc** back-edge to **loop_cond** is an internal re-entry and is correctly omitted.

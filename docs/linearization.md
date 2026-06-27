# Linearization
A lightweight, header-only C++ tool designed to flatten structured or arbitrary Control Flow Graphs (CFGs) into a linear sequence of string representation.

## What is CFG Linearization?

In compiler infrastructure, functions are analyzed and optimized using a multi-directional graph representation called a **CFG**.
While a graph structure is useful for optimizations (e.g., dead-code elimination, loop analysis, or liveness tracking), assembly files are linear (sequential).

**Linearization** flattens this graph of basic blocks into a straight-line sequence of text to preserve the logic of the original graph layout.

## Idea
Given input CFG
```
        init
          |
      loop_cond
      /       \
 body         exit
 /  \
break inc
  |     |
  +-----+
```
Its Linearized output would be:
```
init
loop_cond
body
inc
jmp loop_cond
break
jmp exit
exit
```

## Algorithm

Given on what type of graph was inputed several algorithms would be used

### Directed Acyclic Graph (DAG)
```
boost::topological_sort(...)
```
This produces a valid topological ordering.

* Time: O(V + E)
* Memory: O(V) (plus output)

## Cyclic graph

If topological_sort throws, it uses an iterative depth-first search post-order traversal.
Visits every vertex once, examines every edge once, records postorder, reverses it.

* Time: O(V + E)
* Memory: O(V)

## Determinism

The ordering is deterministic provided the underlying
Boost graph exposes deterministic vertex and edge iteration (vecS).

Graphs using unordered or implementation-defined containers may
produce different but semantically equivalent output.

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

Example code to linearize:
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
                                    ".L_loop_incr:\n"
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
cfg_tools::cb_node_string<Graph> format_asm_cb = [](const Graph &graph, const cfg_tools::basic_block<Graph> &node) -> std::string {
      return graph[node].str;
};
cfg_tools::linearize(g, format_asm_cb, std::cout);
```

Output:
```
.L_init:
    mov dword ptr [rbp-4], 0
.L_loop_cond:
    cmp dword ptr [rbp-4], 10
    jge .L_loop_exit
.L_loop_body:
    cmp dword ptr [rbp-8], 5
    je .L_loop_break
.L_loop_incr:
    add dword ptr [rbp-4], 1
    jmp .L_loop_cond
.L_loop_break:
    jmp .L_loop_exit
.L_loop_exit:
    xor eax, eax
```
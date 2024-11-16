+++
title = "LeetCode Graph Question Summary"
date = 2024-11-09
draft = false
[taxonomies]
  tags = ["LeetCode", "C++", "Algorithm", "Graph"]
[extra]
  toc = true
	keywords = "LeetCode, C++, Algorithm, Graph"
+++

## Data Structure

Graph has multiple representations, overall there are 3 ways:

### Adjacency Matrix

A two-dimensional matrix, in which the rows represent source vertices and columns represent destination vertices. 

Data on edges and vertices must be stored externally.

Only the cost for one edge can be stored between each pair of vertices.

### Adjacency List

Vertices are stored as records or objects, and every vertex stores a list of adjacent vertices.

Sometimes the adjacency list can be flatten into a 1-D array of pairs. Each pair indicates an edge between 2 nodes.

### Incidence matrix

A two-dimensional matrix, in which the rows represent the vertices and columns represent the edges.

The entries indicate the incidence relation between the vertex at a row and edge at a column.

This is barely seen on LeetCode so I am not really familiar with it.

## Graph Traversal

Traversing a graph is trivial for both adjacency matrix and adjacency list data structures.

If the graph is not a DAG, then we need to maintain a `visited` set to prevent entering a cycle.

We can traverse a graph using DFS or BFS algorithm. If we just need to visit every node once, we can use a global `visited` set to prevent visiting a node twice.

This is typical useful for updating a chess board, etc. questions.

### Topology Sorting

This only applies to DAG. 

In computer science, a topological sort or topological ordering of a directed graph is a linear ordering of its vertices such that for every directed edge `(u,v)` from vertex `u` to vertex `v`, `u` comes before `v` in the ordering.

#### Kahn's algorithm

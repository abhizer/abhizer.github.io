---
layout: default
permalink: /
---

I am a first year PhD student at the **University of Texas at Austin**, co-advised by [**Professor Dixin Tang**](https://dx-tang.github.io/)
and [**Professor Vijay Chidambaram**](https://www.cs.utexas.edu/~vijay/).
I am broadly interested in databases and data systems.

Previously, I was an engineer at [Feldera](https://feldera.com) working on incremental computation.

## Publications

*Mihai Budiu, Leonid Ryzhyk, Gerd Zellweger, Ben Pfaff, Lalith Suresh, Simon Kassing, **Abhinav Gyawali**, Matei Budiu,
Tej Chajed, Frank McSherry, Val Tannen,* **“[DBSP : Automatic Incremental View Maintenance for Rich Query Language](https://rdcu.be/elhs5),”**  published in
**The International Journal on Very Large Data Bases (VLDB)**, Vol. 34, no 39, **2025**, 25 pages.

## Projects

### [DBSP (Present)](https://github.com/feldera/feldera/tree/main/crates/dbsp)
DBSP is a computational engine for continuous analysis of changing data. With DBSP, a programmer writes code in terms of
computations on a complete data set, but DBSP implements it incrementally, meaning that changes to the data set run in
time proportional to the size of the change rather than the size of the data set. This is a major advantage for applications that
work with large data sets that change frequently in small ways.

### [SocketDB (2024)](https://github.com/abhizer/socketdb)
SocketDB is a lightweight SQL database that enables real-time updates through WebSockets. Clients can subscribe to query results and receive updates whenever the underlying data changes. This project allowed me to explore the intersection of database design and real-time communication, providing valuable insights into building efficient and responsive systems.  

### [Nyx-lang (2023)](https://github.com/abhizer/nyx-lang)
Nyx-lang is a statically typed, tree-walking interpreted language with a type-checking mechanism to ensure safety and correctness. Working on this project deepened my understanding of compiler design and type systems, while encouraging me to approach language implementation from a structured and reliable perspective.  

### [Loogle-rs (2023)](https://github.com/abhizer/loogle-rs)
Loogle is a "Local Google" like search engine based on Term Frequency - Inverse Document Frequency (TF-IDF). Users can search for text in their search space and Loogle returns a sorted list of the files based on the page rank.

### [Monkey-rs (2022)](https://github.com/abhizer/monkey-rs)
Monkey-rs is my take on the concepts from the [Writing an Interpreter in Go](https://interpreterbook.com/) book. This project was a pivotal learning experience, helping me become more comfortable with Rust and introducing me to the challenges and rewards of interpreter development. It marked an important step in my growth as a developer.  

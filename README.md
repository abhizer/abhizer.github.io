I am currently working as a Compiler Engineer at [Feldera](https://feldera.com) and have recently completed my Bachelor of Science in Computer Science and Information Technology from Tribhuvan University, Nepal.

As a prospective PhD student, my research interests span **programming languages, compilers,** and **databases**. In databases, I am particularly intrigued by **Incremental View Maintenance** and the structured testing of database systems through query synthesis. In programming languages, while I am still exploring the specific problems I want to focus on, I deeply enjoy the field and have a strong appreciation for languages like Rust. I admire the functional flavor it brings, along with its robust build system, which I believe significantly enhances developer productivity. Beyond these areas, I am also open to exploring research opportunities in Operating Systems and Networking, as I find these fields offer compelling challenges and significant potential for impactful contributions.

## Publications

*Mihai Budiu, Leonid Ryzhyk, Gerd Zellweger, Ben Pfaff, Lalith Suresh, Simon Kassing, **Abhinav Gyawali**, Matei Budiu,
Tej Chajed, Frank McSherry, Val Tannen,* **“DBSP : Automatic Incremental View Maintenance for Rich Query Language,”** submitted to **The VLDB Journal**.
  
> In this paper, we redefine Incremental View Maintenance and propose a solution using DBSP. DBSP is a simple but expressive language for describing computations over data streams. We give an algorithm for converting a DBSP program into an incremental program. Finally we demonstrate how to build upon DBSP to support rich query languages like SQL. A practical implementation of this lies in: <https://github.com/feldera/feldera/tree/main/sql-to-dbsp-compiler>.

***Abhinav Gyawali,*** **“SocketDB: DBMS with Data Streaming via WebSockets,”** to be published in
**Deerwalk Journal of Computer Science and Information Technology**, 2025.

> In this paper, I describe my final year project, SocketDB, a lightweight SQL database that allows clients to subscribe to real-time updates through WebSockets. This paper argues that, in certain cases, this approach would reduce the load on the application server from constant requerying of the data by the client.


## Projects

### [SocketDB (2024)](https://github.com/abhizer/socketdb)
SocketDB is a lightweight SQL database that enables real-time updates through WebSockets. Clients can subscribe to query results and receive updates whenever the underlying data changes. This project allowed me to explore the intersection of database design and real-time communication, providing valuable insights into building efficient and responsive systems.  

### [Nyx-lang (2023)](https://github.com/abhizer/nyx-lang)
Nyx-lang is a statically typed, tree-walking interpreted language with a type-checking mechanism to ensure safety and correctness. Working on this project deepened my understanding of compiler design and type systems, while encouraging me to approach language implementation from a structured and reliable perspective.  

### [Loogle-rs (2023)](https://github.com/abhizer/loogle-rs)
Loogle is a "Local Google" like search engine based on Term Frequency - Inverse Document Frequency (TF-IDF). Users can search for text in their search space and Loogle returns a sorted list of the files based on the page rank.

### [Monkey-rs (2022)](https://github.com/abhizer/monkey-rs)
Monkey-rs is my take on the concepts from the [Writing an Interpreter in Go](https://interpreterbook.com/) book. This project was a pivotal learning experience, helping me become more comfortable with Rust and introducing me to the challenges and rewards of interpreter development. It marked an important step in my growth as a developer.  

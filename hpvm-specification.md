# HPVM Abstraction
An HPVM program is a combination of host code plus a set of one or more distinct dataflow graphs. Each dataflow graph (DFG) is a hierarchical graph with side effects. Nodes represent units of execution, and edges between nodes describe the explicit data transfer requirements. A node can begin execution once a data item becomes available on every one of its input edges. Repeated transfer of data items between nodes (if more inputs are provided) yields a pipelined execution of different nodes in the graph. The execution of a DFG is initiated and terminated by host code that launches the graph. Nodes may access globally shared memory through load and store instructions (side-effects).

## Dataflow Node
A dataflow node represents unit of computation in the DFG. A node can begin execution once a data item becomes available on every one of its input edges.

A single static dataflow node represents multiple dynamic instances of the node, each executing the same computation with different index values. The dynamic instances of a node are required to be independent of each other except on HPVM synchronization operations.

Each dataflow node in a DFG can either be a leaf node or an internal node. An internal node contains a complete DFG, called a child graph, and the child graph itself can have internal nodes and/or leaf nodes.

Internal nodes only create the structure of the child graph, and cannot include actual computation. The DFG cannot be modified at runtime except for the number of dynamic instances, which can be data dependent.

Leaf nodes contain code expressing actual computations. Leaf nodes may contain instructions to query the structure of the underlying DFG, and any non host side HPVM operation for synchronization and memory allocation.

## Dataflow Edge
A dataflow edge from the output out of a source dataflow node ```Src``` to the input in of a sink dataflow node ```Dst``` describes the explicit data transfer requirements. ```Src``` and ```Dst``` node must belong to the same child graph, i.e. must be children of the same internal node.

An edge from source to sink has the semantics of copying the specified data from the source to the sink after the source node has completed execution. The pairs ```(Src, out)```, and ```(Dst, in)``` must be unique, i.e. no two dataflow edges in the same graph can have the same source or destination.

A static edge also represents multiple dynamic instances of that edge between the dynamic instances of the source and the sink nodes.

An edge can be instantiated at runtime using one of two replication mechanisms: all-to-all, where all dynamic instances of the source node are connected to all dynamic instances of the sink node, thus expressing a synchronization barrier between the two groups of nodes, or one-to-one, where each dynamic instance of the source node is connected to a single corresponding instance of the sink node. One-to-one replication requires that the grid structure (number of dimensions and the extents in each dimension) of the source and sink nodes be identical.

## Input and Output Bind
An internal node is responsible for mapping its inputs, provided by incoming dataflow edges, to the inputs to one or more nodes of the child graph.

An internal node binds its input ```ip``` to input ```ic``` of its child node ```Dst``` using an input bind.
The pair ```(Dst, ic)``` must be unique, i.e. no two input binds in the same graph can have the same destination, as that would create a conflict. Semantically, these represent name bindings of input values and not data movement.

Conversely, an internal node binds output ```oc``` of its child node ```Src``` to its output ```op``` using an output bind. The pair ```(Src, oc)``` and destination ```op``` must be unique, i.e. no two output binds in the same graph can have the same source destination, as that would create a conflict.

A bind is always all-to-all.

## Host Code
In an HPVM program, the host code is responsible for setting up, initiating the execution and blocking for completion of a DFG. The host can interact with the DFG to sustain a streaming computation by sending all data required for, and receiving all data produced by, one execution of the DFG. The list of actions that can be performed by the host is described below:

- **Initialization and Cleanup**
All HPVM operations must be enclosed by the HPVM initialization and cleanup. These operations perform initialization and cleanup of runtime constructs that provide the runtime support for HPVM.
- **Track Memory**
Memory objects that are passed to dataflow graphs need to be managed by the HPVM runtime. The HPVM runtime includes a memory tracker for recording the location of HPVM-managed memory objects. Track memory starts tracking specified memory object.
- **Untrack Memory**
Stop tracking specified memory object.
- **Request Memory**
If the specified memory object is not present in host memory, copy it to host memory.
- **Launch**
The host code initiates execution of specified DFG, either streaming or non streaming, and passes initial data. All data for one graph execution must be provided.
- **Wait**
The host code blocks for completion of specified DFG.
- **Push**
Push a set of data required for one graph execution to the specified DFG. The DFG must have been launched using a streaming launch operation. This is a blocking operation.
- **Pop**
Read data produced from one execution of the specified DFG. The DFG must have been launched using a streaming launch operation. This is a blocking operation.

# HPVM Implementation

This section describes the implementation of HPVM on top of LLVM IR.

We use intrinsic functions to implement the HPVM IR. iN is the N-bit integer type in LLVM.

The code for each dataflow node is given as a separate LLVM function, called the node function. The node function may call additional, auxiliary functions. However, the auxiliary functions are not allowed to include any HPVM intrinsics, as they are not considered to be part of the HPVM node hierarchy.

The incoming dataflow edges and their data types are denoted by the parameters to the node function. The outgoing dataflow edges are represented by the return type of the node function, which must be an LLVM struct type with zero or more fields (one per outgoing edge).

We represent nodes with opaque handles (pointers of LLVM type i8*). We represent input edges of a node as integer indices into the list of function arguments, and output edges of a node as integer indices into the return struct type.

Pointer arguments of node functions are required to be annotated with attributes in, and/or out, depending on their expected use (read only, write only, read write).

## Intrinsics for Describing Graphs

The intrinsics for describing graphs can only be used by internal nodes. Also, internal nodes are only allowed to have these intrinsics as part of their node function, with the exception of a return statement of the appropriate type, in order to return the result of the outgoing dataflow edges.


```i8* llvm.hpvm.createNode(i8* F)```  
Create a static dataflow node with no dynamic instances executing node function ```F```. Return a handle to the created node.

```i8* llvm.hpvm.createNode1D(i8* F, i64 n1)```  
Create a static dataflow node with ```n1``` dynamic instances executing node function ```F```. Return a handle to the created node.

```i8* llvm.hpvm.createNode2D(i8* F, i64 n1, i64 n2)```  
Create a static dataflow node replicated in two dimensions, with ```n1``` and ```n2``` dynamic instances in each dimension respectively, executing node function ```F```. Return a handle to the created node.

```i8* llvm.hpvm.createNode3D(i8* F, i64 n1, i64 n2, i64 n3)```  
Create a static dataflow node replicated in three dimensions, with ```n1```, ```n2``` and ```n3``` dynamic instances in each dimension respectively, executing node function ```F```. Return a handle to the created node.

```i8* llvm.hpvm.createEdge(i8* Src, i8* Dst, i1 ReplType, i32 sp, i32 dp, i1 isStream)```  
Create edge from output ```sp``` of node ```Src``` to input ```dp``` of node ```Dst```. ```ReplType``` chooses between a 1-1 or all-to-all edge. ```isStream``` chooses a streaming (1) or non streaming (0) edge. Return a handle to the created edge.

```void llvm.hpvm.bind.input(i8* N, i32 ip, i32 ic, i1 isStream)```  
Bind input ```ip``` of current node to input ```ic``` of child node ```N```. ```isStream``` chooses a streaming (1) or non streaming (0) bind.

```void llvm.hpvm.bind.output(i8* N, i32 oc, i32 op, i1 isStream)```  
Bind output ```oc``` of child node ```N``` to output ```op``` of current node. ```isStream``` chooses a streaming (1) or non streaming (0) bind.

## Intrinsics for Querying Graphs

The following intrinsics are used to query the structure of the DFG. They can only be used by leaf nodes.

```i8* llvm.hpvm.getNode()```  
Return a handle to the current dataflow node.

```i8* llvm.hpvm.getParentNode(i8* N)```  
Return a handle to the parent in the hierarchy of node ```N```.

```i32 llvm.visc.getNumDims(i8* N)```  
Get the number of dimensions of node ```N```.

```i64 llvm.hpvm.getNodeInstanceID.{x,y,z}(i8* N)```  
Get index of current dynamic node instance of node ```N``` in dimension x, y or z respectively. The dimension must be one of the dimensions in which the node is replicated.

```i64 llvm.hpvm.getNumNodeInstances.{x,y,z}(i8* N)```  
Get number of dynamic instances of node ```N``` in dimension x, y or z respectively. The dimension must be one of the dimensions in which the node is replicated.

## Intrinsics for Memory Allocation and Synchronization

The following intrinsics are used for memory allocation and synchronization. They can only be used by leaf nodes.
```i8* llvm.hpvm.malloc(i64 nBytes)```  
Allocate a block of memory of size ```nBytes``` and return pointer to it. The allocated object can be shared by all nodes, although the pointer returned must somehow be communicated explicitly for use by other nodes.

```i32 llvm.hpvm.atomic.add(i8* m, i32 v)```  
Atomically computes the bitwise ADD of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```i32 llvm.hpvm.atomic.sub(i8* m, i32 v)```  
Atomically computes the bitwise SUB of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```i32 llvm.hpvm.atomic.min(i8* m, i32 v)```  
Atomically computes the bitwise MIN of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```i32 llvm.hpvm.atomic.max(i8* m, i32 v)```  
Atomically computes the bitwise MAX of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```i32 llvm.hpvm.atomic.xchg(i8* m, i32 v)```  
Atomically computes the bitwise XCHG of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```i32 llvm.hpvm.atomic.and(i8* m, i32 v)```  
Atomically computes the bitwise AND of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```i32 llvm.hpvm.atomic.or(i8* m, i32 v)```  
Atomically computes the bitwise XOR of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```i32 llvm.hpvm.atomic.xor(i8* m, i32 v)```  
Atomically computes the bitwise XOR of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```void llvm.hpvm.barrier()```  
Local synchronization barrier across dynamic instances of current leaf node.

## Intrinsics for Graph Interaction

The following intrinsics are for graph initialization/termination and interaction with the host code, and can be used only by the host code.

```void llvm.hpvm.init()```  
Initialization of HPVM runtime.

```void llvm.hpvm.cleanup()```  
Cleanup of HPVM runtime created objects.

```void llvm.hpvm.trackMemory(i8* ptr, i64 sz)```  
Insert memory starting at ```ptr``` of size ```sz``` in the memory tracker. ```ptr``` becomes the key for identifying this memory object. As soon as a memory object is inserted in the memory tracker it starts being tracked, and can be passed as a data item to a DFG.

```void llvm.hpvm.untrackMemory(i8* ptr)```  
Stop tracking memory object with key ```ptr```, and remove it from memory tracker.

```void llvm.hpvm.requestMemory(i8* ptr, i64 sz)```  
If memory object with key ```ptr``` is not located in host memory, copy it to host memory.

```i8* llvm.hpvm.launch(i8* RootGraph, i8* Args, i1 isStream)```  
Launch the execution of DFG with node function ```RootGraph```. ```Args``` is a pointer to packed struct, containing one field per argument of the ```RootGraph``` function, consecutively. For non-streaming DFGs with a non empty result type, ```Args``` must contain an additional field of the type ```RootGraph.returnTy```, where the result of the graph will be returned. ```isStream``` chooses between a non streaming (0) or streaming (1) graph execution. Return a handle to the invoked DFG.

```void llvm.hpvm.wait(i8* GraphID)```  
Wait for completion of execution of DFG with handle ```GraphID```.

```void llvm.hpvm.push(i8* GraphID, i8* args)```  
Push set of input data ```args``` (same as type included in launch) to streaming DFG with handle ```GraphID```.

```i8* llvm.hpvm.pop(i8* GraphID)```  
Pop  and return data from streaming DFG with handle ```GraphID```.

## Implementation Limitations
Due to limitations of our current prototype implementation, the following restrictions are imposed:
- In HPVM, a memory object is represented as a combination of a pointer to a memory location and the size of the pointed-to object. Therefore, when an edge/bind carries a pointer to an input location, then the edge/bind incoming to the next input location is required to transfer a value of type i64 containing the size, in bytes, of the transferred memory object represented by the pointer-size pair.
- Pointers cannot be transferred between nodes using dataflow edges. Instead, they may be passed using the bind operation from the (common) parent of the source and sink nodes.

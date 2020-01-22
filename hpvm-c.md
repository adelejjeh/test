# HPVM-C Language

## Host API

    void __visc__init()
Used before all other HPVM calls to initialize the HPVM runtime.

```void __visc__cleanup()```  
    - Used at the end of HPVM program to clean up all remaining runtime-created HPVM objects.

```void llvm_visc_track_mem(void* ptr, size_t sz)```  
Insert memory starting at ```ptr``` of size ```sz``` in the memory tracker of HPVM runtime.

```void llvm_visc_untrack_mem(void* ptr)```  
Stop tracking the memory object identified by ```ptr```.

```void llvm_visc_request_mem(void* ptr, size_t sz)```  
If the memory object identified by ```ptr``` is not in host memory, copy it to host memory.

```void* __visc__launch(unsigned isStream, void* rootGraph, void* args)```  
Launches the execution of the dataflow graph with node function ```rootGraph```. ```args``` is a pointer to a packed struct, containing one field per argument of the RootGraph function, consecutively. For non-streaming DFGs with a non empty result type, ```args``` must contain an additional field of the type ```RootGraph.returnTy```, where the result of the graph will be returned. ```isStream``` chooses between a non streaming (0) or streaming (1) graph execution. Returns a handle to the executing graph.

```void __visc__wait(void* G)```  
Waits for completion of execution of the dataflow graph with handle ```G```.

```void __visc__push(void* G, void* args)```  
Push set of input data items, ```args```, (same as type included in launch) to streaming DFG with handle ```G```.

```void* __visc__pop(void* G)```  
Pop and return data produced from one execution of streaming DFG with handle ```G```. 

| |
| - |
|```void __visc__init()```<br>Used before all other HPVM calls to initialize the HPVM runtime.|
|```void __visc__cleanup()```|
|Used at the end of HPVM program to clean up all remaining runtime-created HPVM objects.|
|```void llvm_visc_track_mem(void* ptr, size_t sz)```|
|Insert memory starting at ```ptr``` of size ```sz``` in the memory tracker of HPVM runtime.|
|```void llvm_visc_untrack_mem(void* ptr)```|
|Stop tracking the memory object identified by ```ptr```.|
|```void llvm_visc_request_mem(void* ptr, size_t sz)```|
|If the memory object identified by ```ptr``` is not in host memory, copy it to host memory.|
|```void* __visc__launch(unsigned isStream, void* rootGraph, void* args)```|
|Launches the execution of the dataflow graph with node function ```rootGraph```. ```args``` is a pointer to a packed struct, containing one field per argument of the RootGraph function, consecutively. For non-streaming DFGs with a non empty result type, ```args``` must contain an additional field of the type ```RootGraph.returnTy```, where the result of the graph will be returned. ```isStream``` chooses between a non streaming (0) or streaming (1) graph execution. Returns a handle to the executing graph.|
|```void __visc__wait(void* G)```|
|Waits for completion of execution of the dataflow graph with handle ```G```.|
|```void __visc__push(void* G, void* args)```|
|Push set of input data items, ```args```, (same as type included in launch) to streaming DFG with handle ```G```.|
|```void* __visc__pop(void* G)```|
|Pop and return data produced from one execution of streaming DFG with handle ```G```.|

## Internal Node API

```void* __visc__createNodeND(unsigned dims, void* F, ...)```  
Creates a static dataflow node replicated in ```dims``` dimensions (0 to 3), each executing node function ```F```. The arguments following ```F``` are the size of each dimension, respectively, passed in as a ```size_t```. Returns a handle to the created dataflow node.

```void* __visc__edge(void* src, void* dst, unsigned replType, unsigned sp, unsigned dp, unsigned stream)```  
Creates an edge from output ```sp``` of node ```src``` to input ```dp``` of node ```dst```. If ```replType``` is 0, the edge is a one-to-one edge, otherwise it is an all-to-all edge. ```isStream``` defines whether or not the edge is streaming. Returns a handle to the created edge.

```void __visc__bindIn(void* N, unsigned ip, unsigned ic, unsigned isStream)```  
Binds the input ```ip``` of the current node to input ```ic``` of child node function ```N```. ```isStream``` defines whether or not the input bind is streaming.

```void __visc__bindOut(void* N, unsigned op, unsigned oc, unsigned isStream)```  
Binds the output ```op``` of the current node to output ```oc``` of child node function ```N```. ```isStream``` defines whether or not the output bind is streaming.

```void __visc__hint(enum Target target)``` (C\) / ```void __visc__hint(visc::Target target)``` (C++)  
Must be called once in each node function. Indicates which hardware target the current function should run in

```void __visc__attributes(unsigned ni, …, unsigned no, …)```  
Must be called once at the beginning of each node function. Defines the properties of the pointer arguments to the current function. ```ni``` represents the number of input arguments, and ```no``` the number of output arguments. The arguments following ```ni``` are the input arguments, and the arguments following ```no``` are the output arguments. Arguments can be marked as both input and output. All pointer arguments must be included.

## Leaf Node API
```void __visc__hint(enum Target target)``` (C\) / ```void __visc__hint(visc::Target target)``` (C++)  
As described in internal node API.

```void __visc__attributes(unsigned ni, …, unsigned no, …)```  
As described in internal node API.

```void __visc__return(unsigned n, ...)```  
Returns ```n``` values from a leaf node function. The remaining arguments are the values to be returned. All ```__visc__return``` statements within the same function must return the same number of values.

```void* __visc__getNode()```  
Returns a handle to the current leaf node.

```void* __visc__getParentNode(void* N)```  
Returns a handle to the parent node of node ```N```.

```long __visc__getNodeInstanceID_{x,y,z}(void* N)```  
Returns the dynamic ID of the current instance of node ```N``` in the x, y, or z dimension respectively. The dimension must be one of the dimensions in which the node is replicated.

```long __visc__getNumNodeInstances_{x,y,z}(void* N)```  
Returns the number of dynamic instances of node ```N``` in the x, y, or z dimension respectively. The dimension must be one of the dimensions in which the node is replicated.

```void __visc__barrier()```  
Local synchronization barrier across dynamic instances of current leaf node.

```void* __visc__malloc(long nBytes)```  
Allocate a block of memory of size ```nBytes``` and returns a pointer to it. The allocated object can be shared by all nodes, although the pointer returned must somehow be communicated explicitly for use by other nodes.

```int __visc__atomic_add(int* m, int v)```  
Atomically adds ```v``` to the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```int __visc__atomic_sub(int* m, int v)```  
Atomically subtracts ```v``` from the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```int __visc__atomic_xchg(int* m, int v)```  
Atomically swaps ```v``` with the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```int __visc__atomic_inc(int* m)```  
Atomically increments the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```int __visc__atomic_dec(int* m)```  
Atomically decrements the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```int __visc__atomic_min(int* m, int v)```  
Atomically computes the min of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```int __visc__atomic_max(int* m, int v)```  
Atomically computes the max of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```int __visc__atomic_and(int* m, int v)```  
Atomically computes the bitwise AND of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```int __visc__atomic_or(int* m, int v)```  
Atomically computes the bitwise OR of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

```int __visc__atomic_xor(int* m, int v)```  
Atomically computes the bitwise XOR of ```v``` and the value stored at memory location ```[m]```. Returns the value previously stored at ```[m]```.

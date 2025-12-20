---
title:  "PMDK Tutorial Notes Part 1"
---

Book: S. Scargall, Programming Persistent Memory: A Comprehensive Guide for Developers, 2020

# Chapter 5: Introducing PMDK
## Summary
PMDK consists of bunch of libraries and tools.

## Details
PMDK offers two catergories of libraries: 
1. Volatile libraries, for applications that do not require persistency (use PM for capacity). 
Ex. libmemkind (heap manager), libvmemcache (in-memory cache).
2. Persistent libraries. Ex. libpmem (low level C lib), libpmemobj (txnal object store), libpmemobj-cpp,
libpmemkv (key-value store), libpmemblk (managing fix sized array blocks).

PMDK also includes various command utilitizes. Ex. pmempool can be used to create memory pools and checking 
pool statistics. pmemcheck is a Valgrind based runtime analysis tool for checking pm errors such as 
missing flush or incorrect usage of txns. pmemorder records and replay application persistent states, which helps
debugging.

I should consider posting specific PMDK related questions to https://groups.google.com/forum/#!forum/pmem.

## Questions
1. What is in-memory cache?
2. What is key-value store?

# Chapter 6: libpmem
## Summary
libpmem is a very low level C library, only providing raw access to the PM. Libraries such as libpmemobj build
upon libpmem to raise the level of abstraction.

## Details
To memory map a persistent memory file:
```c
80 /* create a pmem file and memory map it */
81 if ((pmemaddr = pmem_map_file(argv[2], BUF_LEN,
82          PMEM_FILE_CREATE|PMEM_FILE_EXCL,
83          0666, &mapped_len, &is_pmem)) == NULL) {
84      perror("pmem_map_file");
85      exit(1);
86 }
```
pmem_map_file returns is_pmem, indicating if the given address range is actually PM or traditional hard drive.
The programmer can only use the optimized pmem_persist() ONLY if is_pmem is set. If not, user should call pemem_msync()
instead, which is a small wrapper around msync (using pmem_persist is not safe).
```c
74 /* Flush above strcpy to persistence */
75 if (is_pmem)
76      pmem_persist(pmemaddr, mapped_len);
77 else
78      pmem_msync(pmemaddr, mapped_len);
```
pmem_persist() forces changes to persist in persistent memory. This may be more optimal than msync as it will avoid calling
into the kernel if possible. There are no restrictions on the alignment of pmemaddr. Note that pmem_persist is not atomic 
nor transactional. Some stores may have already been persisted via cache eviction by the time pmem_persist is called.

Furthermore, we can perform pmem_persist at a finer granularity. Persisting changes involves two steps: flushing the cache line
and wait for the write buffers to drain. Users can use pmem_flush() and pmem_drain() to perform persist in two steps.


## Questions
1. What is an allocator?
2. What do they mean when they say libpmem "does not track what changed for you?"
    - I think they mean that you need keep track of when each data need to be flushed/persisted. 

## Sources
https://pmem.io/pmdk/libpmem/
> Make sure you are looking at the manpages of the latest pmdk. Docs from different versions may contradict each other.

# Chapter 7: libpmemobj
## Summary
libpmemobj builds upon libpmem to provide APIs for atomic operations, transactions, and reserve/publish.
The purpose of POP is to locate the root of the pmem pool.

## Details
Provides transactional object store. One key feature of libpmemobj is that it provides atomicity and durability via atomic 
transactions and redo/undo logging (not provided by libpmem). libpmemobj uses memory pools at the core. 
```c
#define LAYOUT_NAME "rweg"
#define MAX_BUF_LEN 31

struct my_root {
	size_t len;
	char buf[MAX_BUF_LEN];
};

int main(int argc, char *argv[]) {
    if (argc != 2) {
    	fprintf(stderr, "usage: %s file-name\n", argv[0]);
    	exit(1);
    }

    // create pool with permission 0666. The pool file name and extension can be anything.
    // PMEMOBJ_MIN_POOL is the smallest pool size we can create.
    // POP stands for pool object pointer. More on this later.
    PMEMobjpool *pop = pmemobj_create(argv[1], 
    	LAYOUT_NAME, PMEMOBJ_MIN_POOL, 0666);
    
    if (pop == NULL) {
    	perror("pmemobj_create");
    	exit(1);
    }
    
    // locate the root object (not root pointer yet) from POP pointer.
    PMEMoid root = pmemobj_root(pop, 
    	sizeof(struct my_root));
    
    // get root pointer from root object
    struct my_root *rootp = pmemobj_direct(root); //save the root ptr of the pmem pool
    
    char buf[MAX_BUF_LEN] = "Hello PMEM World";
    
    rootp->len = strlen(buf);
    // note that pmemobj_persist() is still not atomic nor transactional, just like pmem_persist() in libpmem.
    pmemobj_persist(pop, &rootp->len,  //persist length
    	sizeof(rootp->len));
    
    pmemobj_memcpy_persist(pop, rootp->buf, buf,  //persist string
    	rootp->len);
    
    pmemobj_close(pop);
    
    exit(0);
}
```
Most OS use address space layout randomization (ASLR), which causes the location of the pool to vary between executions and 
reboots. libpmemobj provides mechanism to locate data within the pool. Every pool has a root object, which is used as an 
entry point to find all other objects created in the pool. Applications need to use a special object called pool object pointer (POP)
to locate the root object. Note that there is a distinction between the root object and root pointer. The sequence is POP -> root object
-> root pointer. Here is an example.
```c
// Hmm, so let's say we want to store an integer array in the mem pool.
// Then I think we would add an extra int array member in the my_root struct
struct my_root {
    size_t len;
    char buf[MAX_BUF_LEN];
};

int main(int argc, char *argv[]) {
    if (argc != 2) {
    	fprintf(stderr, "usage: %s file-name\n", argv[0]);
    	exit(1);
    }

    // open existing pool. returns pool object pointer
    PMEMobjpool *pop = pmemobj_open(argv[1], LAYOUT_NAME);
    
    if (pop == NULL) {
    	perror("pmemobj_open");
    	exit(1);
    }

    // use pop to get root object
    PMEMoid root = pmemobj_root(pop, sizeof(struct my_root)); 
    // use root object to get root pointer
    struct my_root *rootp = pmemobj_direct(root);
    
    // access pool objects through the root pointer. 
    if (rootp->len == strlen(rootp->buf))
    	printf("%s\n", rootp->buf);
    
    pmemobj_close(pop);
    
    exit(0);
}
```
We also have the option to combine multiple pools, perhaps from different pm devices, into a pool set.

Skipping TOIDs...

pmemobj_alloc() allows users to allocate new objects from the pool. This should be used outside of transactions.
This is volatile memory allocation. I am not entirely sure when would one want to use this. Perhaps when you 
want to use a chunk of the pool as volatile memory?

To **save data persistently**, users need to use one of three APIs (I guess you can technically use libpmem too,
but I think these are easier to use):
- Atomic allocation
- Reserver/publish
- Transactional

**Atomic allocation:** pmemobj_alloc() can also be used to save data persistently. It does this by reserving the object
in a temporary state, calling the constructor provided in the argument, and in one atomic action make the allocation 
persistent. I think one use case could be if you already created a memory pool and wants to add more persistent objects to it.
However, its use is only limited to allocating and initializing.

**Reserver/publish:** skipping for now. 

**Transactional:** use TX_BEGIN and TX_END to encapsulate a transaction. This sounds pretty straightforward.

Skipping debugging...

## Questions
1. What the heck is object store?
   - A storage architecture that stores data as objects. This contrasts file store, which stores data as files.
   Obj store sounds like a file system with only the root folder to me...
2. Why memory pools?
3. What is ASLR?

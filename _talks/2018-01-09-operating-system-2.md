---
layout: post
title: "Designing an Operating System - II"
date: 2018-01-09
---

We (I along with two other super-talented blokes) designed an Operating System in three phases across the Fall of 2017. The three phases were:
* Threads and Scheduling
* Pseudo-virtual memory
* Simple File System

This writeup details out our second phase of implementation.

## Pseudo-virtual memory

We implemented a pseudo-virtual memory for our thread library by further dividing this phase into 4 smaller sub-phases (I guess, it was just a phase :P).

### Phase A: Direct-Mapped Memory

To emulate physical memory, we created a static char array of size 8MB. We maintained all our bookkeeping state in this array and allocated memory from it as well. Our memory manager managed memory for our thread library and presumed its existence. Its code is part of our thread library. We added a macro definition to transform all calls to malloc() and free() calls to calls of our own functions:

`void* myallocate(int size, char* file_num, int line_num, int alloc_flag)`

We have kept an alloc_flag which tells us whether this thread has been allocated a page before or not.
Inside this are two conditions:
1. If the head of the block is null, meaning no allocation has yet happened inside the page.

If the requested size is less than the remaining size on the page belonging to that thread, we create a node_ptr nodule which points to the various memory allocations inside each page. Here mem_iter points to the first free memory location inside the thread. 


We created nodules which will store the meta-data of each of the memory allocations inside the page.
The various components of Node structure is as follows:
`typedef struct Node{
    int size;       // 4
    char* data;     // 8
    struct Node* next; // 8
    bool valid;     // 1
}node,*node_ptr;`

`size`: size of the memory allocation for this node
`data`: pointer to the actual data location (explained below)
`next`: pointer to the next node
`valid`: if this memory allocation is still valid or not. Invalid means it can be overwritten.

It is important to note inside myallocate we do: 
`char_iter += sizeof(struct Node);
nodule->data = char_iter;`

The actual data is actually pointed by `nodule->data` which we set as the size of the first free memory location plus the size of this Node.

2. If there are allocated memory blocks inside the page.

In this case we need to iterate over to the next free memory block to allocate this new memory request. We do this using the help of the “node_ptr iter”. Please note at every iteration we check whether it is a valid memory allocation or not.
`#define free(x) mydeallocate(x, __FILE__, __LINE__, THREADREQ)
void mydeallocate(void* ptr, char* file_num, int line_num, int dealloc_flag)`

Deallocate is pretty straight forward. We iterate till we find that memory location (which is “data” in this case) and then iter->valid = 0 marks the block as invalid. 
`int my_pthread_create(my_pthread_t * thread, pthread_attr_t * attr, void *(*function)(void*), void * arg)`

Whenever a new thread is created, a 4KB page automatically needs to be assigned to it. We do this in my_pthread_create where we iterate over the complete memory block using meme_iter which finds the new blank space to insert the new page. There is one detailing here which we would like to now explain.
Just like the metadata node for every memory allocation inside a page, we have a page node with this structure:
`typedef struct PageNode{
	int counter;
	int page_id;
    struct Node* next;
}page_node,*page_ptr;`

This PageNode acts as a metadata for every page and acts as a linked list connecting all the pages belonging to the same thread. Hence it contains counter, which is the index of this particular page in the linked list of pages for one particular thread. Next is the page id which stores the page number of that particular page. Finally we have the next pointer which points to the next page node.
As you can see we had to make changes to the thread control block to change the singular page id to an array of page ids for all the pages of that thread.

### Phase B: Virtual Memory

One thread can have multiple pages and this is taken care of by using an array of page ids in Thread Control Block instead of just 1 page id as before. Whenever the current thread tries to access any other thread’s page, it causes a segmentation fault(due to mprotect implementation in scheduler). On `SIGSEGV`, the scheduler is called again but with `signum=11`. This section handles the segmentation fault. Here, we look if the other page exists or not. If it exists, it can be found using next pointers like a linked list. If it does not exist, a new page is created and linked to the existing linked list of the current thread.

### Phase C: Shared Region

We allocated a space of 16KB as shared memory location at the end of the complete allocation of individual pages. During every context switch, the scheduler calls 3 mprotects. One is to protect the pages not belonging to that thread. The second is to provide read write access to the pages belonging to that thread. And the third one is to grant read write access to the shared memory allocation using the the `shared_head”` as a global pointer to the 16KB shared page.
`mprotect(shared_head, PAGE_SIZE * 4, PROT_READ | PROT_WRITE);`


### Phase D: Page Replacement Algorithm

- Naive Replacement
	We have associated a counter with every `TCB`. This counter is incremented every time that page is accessed in the memory. Hence, the natural choice would be to evict the page with the least counter value. This is a Naive Replacement strategy because history or recency is not taken into account. In our code the mem-iter is incremented and the page with minimum counter as of then is recorded. When the loop finishes, the `page_to_evict` will be returned.
- 2nd Chance Replacement
	In this replacement strategy we go about the list of pages in a circular loop. The first page we find with the used bit as zero, we return that as the page to be replaced. If while looping around the pages, if the `used-bit` is 1, we reset it to 0 and then move on to the next page. The `mem_iter = (mem_iter+4096)%(4*1024*1024)` helps loop around after the completion of one cycle if none of the pages have `used-bit` as 0.

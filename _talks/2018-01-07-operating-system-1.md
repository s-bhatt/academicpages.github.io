---
layout: post
title: "Designing an Operating System - I"
date: 2018-01-07
---

# Designing an Operating System - I

We (I along with two other super-talented blokes) designed an Operating System in three phases across the Fall of 2017. The three phases were:
* Threads and Scheduling
* Pseudo-virtual memory
* Simple File System

This writeup details out our first phase of implementation.

## Threads and Scheduling

In this phase we implemented user threads using the Linux pthread API. This required us to generate new execution contexts (threads) on pthread_create and to write a scheduler to switch between our various threads. Since we provided a multi-threaded environment, we also need to implemented pthread mutexes, mutual exclusion devices that keep a thread locked if it is waiting for a particular mutex. This assignment also helped us explore the mechanics and difficulties of scheduling tasks within an operating system.

We built a library named "my_pthread_t.h" that contains implementations of the prototypes below:

* `int my_pthread_create( my_pthread_t * thread, pthread_attr_t * attr, void *(*function)(void*), void * arg);`

Creates a pthread that executes function. Attributes are ignored, arg is not.


* `void my_pthread_yield();`

Explicit call to the my_pthread_t scheduler requesting that the current context can be swapped out andanother can be scheduled if one is waiting.

 

* `void pthread_exit(void *value_ptr);`

Explicit call to the my_pthread_t library to end the pthread that called it. If the value_ptr isn't NULL,any return value from the thread will be saved.

 

* `int my_pthread_join(my_pthread_t thread, void **value_ptr);`

Call to the my_pthread_t library ensuring that the calling thread will not continue execution until the one it references exits. If value_ptr is not null, the return value of the exiting thread will be passed back.

* `int my_pthread_mutex_init(my_pthread_mutex_t *mutex, const pthread_mutexattr_t *mutexattr);`

Initializes a my_pthread_mutex_t created by the calling thread. Attributes are ignored.

 

* `int my_pthread_mutex_lock(my_pthread_mutex_t *mutex);`

Locks a given mutex, other threads attempting to access this mutex will not run until it is unlocked.


* `int my_pthread_mutex_unlock(my_pthread_mutex_t *mutex);`

Unlocks a given mutex.


Further, we used the ucontext.h system library to help us with this purpose. It has a series of commands to make, swap and get the currently running context. When a context is running, it will continue running until it completes. In order to interrupt the current context, we set an interrupt time (setitimer) in 25ms quanta. Next, we had to implement and register a signal handler to run any time this interrupt fires. In your signal handler we scheduled and swapped to the next context.

 
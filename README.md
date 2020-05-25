# linux-learning
### 2020.05.25 Mon 12:00  

learning linux by reading the kernel src code, step by step  


## 1. Lock and IPC | 锁与进程间通信  
##### 1.1 原子操作  

##### 1.2 自旋锁 spinlock  
- 结构体  spinlock_t
  ```c
  typedef struct {
    raw_spinlock_t raw_lock;
  #if defined(CONFIG_PREEMPT) && defined(CONFIG_SMP)
    unsigned int break_lock;
  #endif
  #ifdef CONFIG_DEBUG_SPINLOCK
    unsigned int magic, owner_cpu;
    void *owner;
  #endif
  #ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
  #endif
  } spinlock_t;

  //接口：
  spin_lock(spinlock_t* lp)
  spin_unlock(spinlock_t* lp)
  ```
- 获取自旋锁   
  ```c
  //include\linux\spinlock.h
  #define spin_lock(lock)			_spin_lock(lock)
  ------------------------------------------------------
  //kernel\spinlock.c
  void __lockfunc _spin_lock(spinlock_t *lock)
  {
    preempt_disable();
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
    LOCK_CONTENDED(lock, _raw_spin_trylock, _raw_spin_lock);
  }
  ------------------------------------------------------
  //include\linux\preempt.h
  #define preempt_disable() \
  do { \
    inc_preempt_count(); \
    barrier(); \
  } while (0)


  #define inc_preempt_count() add_preempt_count(1)

  # define add_preempt_count(val)	do { preempt_count() += (val); } while (0)

  #define preempt_count()	(current_thread_info()->preempt_count)
  ------------------------------------------------------
  //include\asm-ia64\thread_info.h
  #define current_thread_info()	((struct thread_info *) ((char *) current + IA64_TASK_SIZE))
  ------------------------------------------------------
  //include\asm-ia64\thread_info.h
  /*
  * On IA-64, we want to keep the task structure and kernel stack together, so they can be
  * mapped by a single TLB entry and so they can be addressed by the "current" pointer
  * without having to do pointer masking.
  */
  struct thread_info {
    struct task_struct *task;	/* XXX not really needed, except for dup_task_struct() */
    struct exec_domain *exec_domain;/* execution domain */
    __u32 flags;			/* thread_info flags (see TIF_*) */
    __u32 cpu;			/* current CPU */
    __u32 last_cpu;			/* Last CPU thread ran on */
    __u32 status;			/* Thread synchronous flags */
    mm_segment_t addr_limit;	/* user-level address space limit */
    int preempt_count;		/* 0=premptable, <0=BUG; will also serve as bh-counter */
    struct restart_block restart_block;
  };

  //struct task_struct 见: include\linux\sched.h
  ```
  >从代码中可以看出，spin_lock() 通过 preempt_disable()，将 thread_info.preempt_count 的值增加1，preempt_count = 0 表示允许抢占，> 0 表示不允许抢占; 这样就保证了 spin_lock() 调用过程中线程不会被抢占。  
  释放自旋锁 spin_unlock() 会调用 preempt_enable(), 该接口调用栈如下：  

  ```c
  //include\linux\preempt.h
  #define preempt_enable() \
  do { \
    preempt_enable_no_resched(); \
    barrier(); \
    preempt_check_resched(); \
  } while (0)
  ------------------------------------------------------
  #define preempt_enable_no_resched() \
  do { \
    barrier(); \
    dec_preempt_count(); \
  } while (0)
  ------------------------------------------------------
  #define dec_preempt_count() sub_preempt_count(1)
  ------------------------------------------------------
  # define sub_preempt_count(val)	do { preempt_count() -= (val); } while (0)
  ```  
  >如上可看出，preempt_enable() 会将 thread_info.preempt_count 的值减1，当 preempt_count 为0时，表示允许进行抢占  
- 优化屏障 barrier()  
  preempt_disable() 和 preempt_enable() 中都调用了 barrier()，该函数会插入一个优化屏障。该指令告知编译器，保存在CPU寄存器中、在屏障之前有效的所有内存地址，在屏障之后都将失效。本质上，这意味着编译器在屏障之前发出的读写请求完成之前，不会处理屏障之后的任何读写请求。  
  barrier() 源码：
  ```c
  /* Optimization barrier */
  /* The "volatile" is due to gcc bugs */
  #define barrier() __asm__ __volatile__("": : :"memory")
  ```


##### 1.3 信号量 semaphore  
- 结构体  semaphore
  ```c
  struct semaphore {
    atomic_t count; //可以同时进入临界区的线程的数目
    int sleepers;  //等待进入临界区的线程的数目
    wait_queue_head_t wait;  //队列，保存所有在该信号量上水面的线程的task_struct
  };
  ```
- 获取信号量  
  ```c
  //arch\sparc64\kernel\semaphore.c
  void __sched down(struct semaphore *sem)
  {
    might_sleep();
    /* This atomically does:
    * 	old_val = sem->count;
    *	new_val = sem->count - 1;
    *	sem->count = new_val;
    *	if (old_val < 1)
    *		__down(sem);
    *
    * The (old_val < 1) test is equivalent to
    * the more straightforward (new_val < 0),
    * but it is easier to test the former because
    * of how the CAS instruction works.
    */

    __asm__ __volatile__("\n"
  "	! down sem(%0)\n"
  "1:	lduw	[%0], %%g1\n"
  "	sub	%%g1, 1, %%g7\n"
  "	cas	[%0], %%g1, %%g7\n"
  "	cmp	%%g1, %%g7\n"
  "	bne,pn	%%icc, 1b\n"
  "	 cmp	%%g7, 1\n"
  "	membar	#StoreLoad | #StoreStore\n"
  "	bl,pn	%%icc, 3f\n"
  "	 nop\n"
  "2:\n"
  "	.subsection 2\n"
  "3:	mov	%0, %%g1\n"
  "	save	%%sp, -160, %%sp\n"
  "	call	%1\n"
  "	 mov	%%g1, %%o0\n"
  "	ba,pt	%%xcc, 2b\n"
  "	 restore\n"
  "	.previous\n"
    : : "r" (sem), "i" (__down)
    : "g1", "g2", "g3", "g7", "memory", "cc");
  }
  ```  

- 释放信号量  
  ```c
  //arch\sparc64\kernel\semaphore.c
  void up(struct semaphore *sem)
  {
    /* This atomically does:
    * 	old_val = sem->count;
    *	new_val = sem->count + 1;
    *	sem->count = new_val;
    *	if (old_val < 0)
    *		__up(sem);
    *
    * The (old_val < 0) test is equivalent to
    * the more straightforward (new_val <= 0),
    * but it is easier to test the former because
    * of how the CAS instruction works.
    */

    __asm__ __volatile__("\n"
  "	! up sem(%0)\n"
  "	membar	#StoreLoad | #LoadLoad\n"
  "1:	lduw	[%0], %%g1\n"
  "	add	%%g1, 1, %%g7\n"
  "	cas	[%0], %%g1, %%g7\n"
  "	cmp	%%g1, %%g7\n"
  "	bne,pn	%%icc, 1b\n"
  "	 addcc	%%g7, 1, %%g0\n"
  "	membar	#StoreLoad | #StoreStore\n"
  "	ble,pn	%%icc, 3f\n"
  "	 nop\n"
  "2:\n"
  "	.subsection 2\n"
  "3:	mov	%0, %%g1\n"
  "	save	%%sp, -160, %%sp\n"
  "	call	%1\n"
  "	 mov	%%g1, %%o0\n"
  "	ba,pt	%%xcc, 2b\n"
  "	 restore\n"
  "	.previous\n"
    : : "r" (sem), "i" (__up)
    : "g1", "g2", "g3", "g7", "memory", "cc");
  }
  ```


##### 1.4 读者/写者锁
- 读者/写者锁 结构体  
```c
typedef struct {
	spinlock_t lock;
	volatile int counter;
#ifdef CONFIG_PREEMPT
	unsigned int break_lock;
#endif
} rwlock_t;
```

  接口:  
```c
//include\linux\spinlock.h
#define write_lock(lock)		_write_lock(lock)

#define write_unlock(lock)		_write_unlock(lock)

#define read_lock(lock)			_read_lock(lock)

#define read_unlock(lock)		_read_unlock(lock)


//include\linux\spinlock_api_up.h
#define _read_lock(lock)			__LOCK(lock)

#define _read_unlock(lock)			__UNLOCK(lock)

#define _write_lock(lock)			__LOCK(lock)

#define _write_unlock(lock)			__UNLOCK(lock)

#define __LOCK(lock) \
  do { preempt_disable(); __acquire(lock); (void)(lock); } while (0)

#define __UNLOCK(lock) \
  do { preempt_enable(); __release(lock); (void)(lock); } while (0)

```

>读写锁与互斥量类似, 不过读写锁允许更高的并行性. 读写锁用于读称为共享锁, 读写锁用于写称为排它锁;  
读写锁规则:  
&nbsp;&nbsp;只要没有线程持有给定的读写锁用于写, 那么任意数目的线程可以持有读写锁用于读;  
&nbsp;&nbsp;仅当没有线程持有某个给定的读写锁用于读或用于写时, 才能分配读写锁用于写;






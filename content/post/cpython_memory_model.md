---
title: "CPython内存模型"
date: 2021-11-29T02:00:00+08:00
draft: false
ShowToc: true
---

# Python内存模型

本文源码基于CPython 3.10

Python实现了自己的内存管理，用以**加快内存操作**和**减少内存碎片化**。Python定义了一个阈值，小于等于这个阈值的小内存请求，由Python实现的这套内存管理来分配，大于则直接交给`malloc()`。

```c
// Objects/obmalloc.c
#define SMALL_REQUEST_THRESHOLD 512
```




# 内存管理模型

Python的内存分配器分成如下层级

```纯文本
    Object-specific allocators
    _____   ______   ______       ________
   [ int ] [ dict ] [ list ] ... [ string ]       Python core         |
+3 | <----- Object-specific memory -----> | <-- Non-object memory --> |
    _______________________________       |                           |
   [   Python's object allocator   ]      |                           |
+2 | ####### Object memory ####### | <------ Internal buffers ------> |
    ______________________________________________________________    |
   [          Python's raw memory allocator (PyMem_ API)          ]   |
+1 | <----- Python memory (under PyMem manager's control) ------> |   |
    __________________________________________________________________
   [    Underlying general-purpose allocator (ex: C library malloc)   ]
 0 | <------ Virtual memory allocated for the python process -------> |

   =========================================================================
    _______________________________________________________________________
   [                OS-specific Virtual Memory Manager (VMM)               ]
-1 | <--- Kernel dynamic storage allocation & management (page-based) ---> |
    __________________________________   __________________________________
   [                                  ] [                                  ]
-2 | <-- Physical memory: ROM/RAM --> | | <-- Secondary storage (swap) --> |

```




按Python内存模型，一个典型的调用过程如下：

```纯文本
PyDict_New()          # 3层
  PyObject_GC_New()   # 2层
    PyObject_Malloc() # 2层
      new_arena()     # 1层
      malloc()        # 0层

```


第0层往下是操作系统和硬件的内存实现，就不在Python的讨论范畴了。



 以下内容的前提是，python使用它自己的small-block内存分配器，当然，大多数情况下还是启用了的。

```c
// pyconfig.h
/* Use Python's own small-block memory-allocator. */
#define WITH_PYMALLOC 1    // 默认是打开的
```


此时`PYOBJ_ALLOC`指向`_PyObject_*`APIs，而不是和`PYRAW_ALLOC`一样指向`_PyMem_*`APIs

```c
// obmalloc.c
#define MALLOC_ALLOC {NULL, _PyMem_RawMalloc, _PyMem_RawCalloc, _PyMem_RawRealloc, _PyMem_RawFree}
#ifdef WITH_PYMALLOC
#  define PYMALLOC_ALLOC {NULL, _PyObject_Malloc, _PyObject_Calloc, _PyObject_Realloc, _PyObject_Free}
#endif

#define PYRAW_ALLOC MALLOC_ALLOC
#ifdef WITH_PYMALLOC
#  define PYOBJ_ALLOC PYMALLOC_ALLOC
#else
#  define PYOBJ_ALLOC MALLOC_ALLOC
#endif
#define PYMEM_ALLOC PYOBJ_ALLOC
```




## 几种内存单位

Python抽象出**arena**，**pool**，**block**这三种内存单位，对应关系如下。在使用本文这种内存管理机制的情况下，每次内存请求的目标是获取到一个足够大小的block。

![](/cpython_memory_model/image/image.png "")

通过数组和多条链表进行管理，使得内存分配开销从O(N)到O(1)。

- **pool**通过freeblock和nextoffset字段以数组+单链表形式对**block**进行管理
- **arena**通过freepools和address字段以数组+单链表形式对**pool**进行管理
- `usedpools[]`全局变量通过存储多条双向链表维护当前已使用且还能继续分配**block**的**pool**
- `unused_arena_objects`全局变量单链表管理已创建但未与（`_PyObject_Arena.alloc()`分配的）实际堆内存关联的`arena_object`
- `usable_arena`全局变量双向链表按拥有空闲**pool**数量升序维护可用**arena**
- `arenas`全局变量维护所有已创建的`arena_object`（不一定已经被`_PyObject_Arena.alloc()`分配了实际堆内存）

以上的总结性描述暂时还不好理解，我们可以在阅读下面内容时，再来回顾。



## block在pool中的组织方式

block这个词语的指代稍微有些混乱，先做一点解释。

```c
/* When you say memory, my mind reasons in terms of (pointers to) blocks */
typedef uint8_t block;
```


代码中的block就是`uint8_t`，但是代码写起来都是用`block*`指针，来指向前面所提到内存单元中的，不同类型的pool中，大小不一，但是都对齐到`ALIGNMENT`宏的整数倍个字节大小的内存块。以下的block一词，都指内存块，而不是单个`uint8_t`。

### pool的结构

每个pool都维护相同大小的block，在较早的python中，block大小为8的倍数

```c
 * For small requests we have the following table:
 *
 * Request in bytes     Size of allocated block      Size class idx
 * ----------------------------------------------------------------
 *        1-8                     8                       0
 *        9-16                   16                       1
 *       17-24                   24                       2
 *       25-32                   32                       3
 *       33-40                   40                       4
 *       41-48                   48                       5
 *       49-56                   56                       6
 *       57-64                   64                       7
 *       65-72                   72                       8
 *        ...                   ...                     ...
 *      497-504                 504                      62
 *      505-512                 512                      63
 *
 *      0, SMALL_REQUEST_THRESHOLD + 1 and up: routed to the underlying
 *      allocator.

```


但是较新版的源码中，在x86_64机器上是16的倍数了。

```c
#if SIZEOF_VOID_P > 4
#define ALIGNMENT              16               /* must be 2^N */
#define ALIGNMENT_SHIFT         4
#else
#define ALIGNMENT               8               /* must be 2^N */
#define ALIGNMENT_SHIFT         3
#endif
```


比如`ALIGNMENT`为16的情况下，size class为0的pool长这样：



![](/cpython_memory_model/image/image_1.png "")

头部储存在pool里，保存一些重要信息。

- obmalloc.c中pool_header的定义

  ```c
  struct pool_header {
      union { block *_padding;
              uint count; } ref;          // pool中已分配block的数量
      block *freeblock;                   // 指向下一个空闲block
      struct pool_header *nextpool;       // 下一个同size class的pool
      struct pool_header *prevpool;       
      uint arenaindex;                    // 该pool所属的arena
      uint szidx;                         // 当前的pool分配哪一类的空间, 等于usepools中的idx
      uint nextoffset;                    // 从未分配过的block偏移
      uint maxnextoffset;                 // nextoffset最大值，超过说明block都分配过，且如果没有block被释放，说明pool已满
  };
  
  typedef struct pool_header *poolp;
  ```



了解完pool_header结构体，再来看一个pool详细的样子，单个pool大小是4Kb，为大多数系统的页大小，但**在新的CPython中有变化，**`USE_LARGE_POOLS`**定义下是16Kb**。

我们先关注下面这个pool怎么以数组+单链表形式来维护block

![](/cpython_memory_model/image/image_2.png "")



### pool三种状态

- **used**：其中部分block被分配了，且至少有一个block没被分配。这也意味着一个pool至少有两个block。因为pool只有在需要内存的时候才被分配，所以`used`是一个pool的初始状态，而不是`empty`
- **full**: 所有block都被分配了，pool会被从`usedpools[]`上取下，此时pool的`prevpool`和`nextpool`没有实际意义
- **empty**：所有block都可用，pool会被从`usedpools[]`上取下，放回对应arena_object的单链表`freepools`里面，此时pool的prevpool没有实际意义



在pool中申请空间有几种情况，但是有一个**原则**很简单就是**每次去找**`freeblock`**指向的内存块**，再根据情况更新一些数据。

下面的内容涉及到`usedpools[]`对pool的管理，但我们主要关注pool怎么管理block

### 在未释放过block的pool中申请新的空间

![](/cpython_memory_model/image/image_3.png "")

每次`freeblock`都指向一个内容为`NULL`的block，表明目前 pool 中没有其他之前被申请过但是当前已经被释放了的 block 存在，新申请空间的话，需要从 pool 尾部寻找之前没有用到过的新的空间。

这时候的管理更像是数组形式，`nextoffset`和 `maxnextoffset`在这种情况下会被用来查看 pool 尾部是否还有剩余的未使用过的空间。

然后就是更新`ref.count`，`nextoffset`，`freeblock`了。每次`freeblock`更新为`nextoffset`指向的地址，所以这里的图中，freeblock指向倒数第3个块，而pool+nextoffset在倒数第2个block

- 这一部分的代码

  ```c
  static void
  pymalloc_pool_extend(poolp pool, uint size)
  {
      // freeblock为NULL时，扩展pool尾部
      if (UNLIKELY(pool->nextoffset <= pool->maxnextoffset)) {
          /* There is room for another block. */
          pool->freeblock = (block*)pool + pool->nextoffset;
          pool->nextoffset += INDEX2SIZE(size);
          *(block **)(pool->freeblock) = NULL;
          return;
      }
  
      /* Pool is full, unlink from used pools. */
      poolp next;
      next = pool->nextpool;
      pool = pool->prevpool;
      next->prevpool = pool;
      pool->nextpool = next;
  }
  
  
  static inline void*
  pymalloc_alloc(void *ctx, size_t nbytes)
  {
      // ...
      if (UNLIKELY(nbytes == 0)) {
          return NULL;
      }
      if (UNLIKELY(nbytes > SMALL_REQUEST_THRESHOLD)) {
          return NULL;
      }
  
      uint size = (uint)(nbytes - 1) >> ALIGNMENT_SHIFT;
      poolp pool = usedpools[size + size];
      block *bp;
  
      if (LIKELY(pool != pool->nextpool)) {
          /*
           * There is a used pool for this size class.
           * Pick up the head block of its free list.
           */
          ++pool->ref.count;
          bp = pool->freeblock;
          assert(bp != NULL);
  
          // 这里的freeblock赋值还是比较巧妙的，可以思考下
          if (UNLIKELY((pool->freeblock = *(block **)bp) == NULL)) {
              // Reached the end of the free list, try to extend it.
              pymalloc_pool_extend(pool, size);
          }
      }
      // ...
  }
  ```

申请完毕后

![](/cpython_memory_model/image/image_4.png "")

再申请一个

![](/cpython_memory_model/image/image_5.png "")

此时`nextoffset`大于`maxnextoffset`了。

再申请

![](/cpython_memory_model/image/image_6.png "")

`nextoffset` 的值比 `maxnextoffset` 的大, 表示尾部已经没有多余的新的未使用过的空间可以使用了, 并且目前情况下当前的pool是满的了, 所以`freeblock`变成了一个空指针

因为pool已经满了, 我们需要把pool从`usedpools`全局变量管理的双向链表中移除。本例中这个pool的block都是8字节，所以这个pool本来位于`usedpools[0]`和`usedpools[1]`维护的一条双链表中。

![](/cpython_memory_model/image/image_7.png "")

在pool中释放block的核心代码如下：

```c
    block *lastfree = pool->freeblock;
    *(block **)p = lastfree;
    pool->freeblock = (block *)p;
    pool->ref.count--;
```


其实就是把要释放的内存，插入到`freeblock`指向的单链表中的**头部**。链表每个节点都是没有用到的block，正好用来指向下一个空闲block的地址。

### 在满的 pool 中进行释放

释放如图位置的block。

![](/cpython_memory_model/image/image_8.png "")

**step1**，把内存块里第一个`block`的值设置为和 `freeblock`相同的值，图中因为是满的，所以设置为了NULL

![](/cpython_memory_model/image/image_9.png "")

**step2**, 让`freeblock`指向当前正在释放的block

![](/cpython_memory_model/image/image_10.png "")

减小`ref.count`的值

**step3**，检查pool**释放内存之前**是否是满的(通过检查当前正在释放的block是否空指针即可确定)，如果是, 把pool重新链接到`usedpools`中并返回, 如果不是则跳转到 step4

![](/cpython_memory_model/image/image_11.png "")

**step4**

- 检查当前arena中的所有的pools是否都为空的, 如果是的话, 释放整个arena
- 如果这是arena中唯一的有空余空间的pool，把arena加回到 `usable_arenas` 列表中
- 给`usable_arenas`进行排序, 确保更多空的`pool`的arena排在更后面



- 释放block的`pymalloc_free(void *ctx, void *p)`源码

  ```c
  // obmalloc.c
  
  /* Free a memory block allocated by pymalloc_alloc().
     Return 1 if it was freed.
     Return 0 if the block was not allocated by pymalloc_alloc(). */
  static inline int
  pymalloc_free(void *ctx, void *p)
  {
      assert(p != NULL);
  
  #ifdef WITH_VALGRIND
      if (UNLIKELY(running_on_valgrind > 0)) {
          return 0;
      }
  #endif
  
      poolp pool = POOL_ADDR(p);
      if (UNLIKELY(!address_in_range(p, pool))) {
          return 0;
      }
      /* We allocated this address. */
  
      /* Link p to the start of the pool's freeblock list.  Since         // 用过但释放了的block，在此收入freeblock单链表进行管理
       * the pool had at least the p block outstanding, the pool
       * wasn't empty (so it's already in a usedpools[] list, or
       * was full and is in no list -- it's not in the freeblocks
       * list in any case).
       */
      assert(pool->ref.count > 0);            /* else it was empty */
      block *lastfree = pool->freeblock;
      *(block **)p = lastfree;
      pool->freeblock = (block *)p;
      pool->ref.count--;
  
      if (UNLIKELY(lastfree == NULL)) {
          /* Pool was full, so doesn't currently live in any list:
           * link it to the front of the appropriate usedpools[] list.
           * This mimics LRU pool usage for new allocations and
           * targets optimal filling when several pools contain
           * blocks of the same size class.
           */
          insert_to_usedpool(pool);
          return 1;
      }
  
      /* freeblock wasn't NULL, so the pool wasn't full,
       * and the pool is in a usedpools[] list.
       */
      if (LIKELY(pool->ref.count != 0)) {
          /* pool isn't empty:  leave it in usedpools */
          return 1;
      }
  
      /* Pool is now empty:  unlink from usedpools, and
       * link to the front of freepools.  This ensures that
       * previously freed pools will be allocated later
       * (being not referenced, they are perhaps paged out).
       */
      insert_to_freepool(pool);
      return 1;
  }
  
  ```


  `POOL_ADDR(p)`的设计：
  因为pool保证对齐到4KiB（定义了`USE_LARGE_POOL`时对齐到16KiB）

  ```c
  #ifdef USE_LARGE_POOLS
  #define POOL_BITS               14                  /* 16 KiB */
  #else
  #define POOL_BITS               12                  /* 4 KiB */
  #endif
  #define POOL_SIZE               (1 << POOL_BITS)
  
  #define _Py_ALIGN_DOWN(p, a) ((void *)((uintptr_t)(p) & ~(uintptr_t)((a) - 1)))
  
  #define POOL_ADDR(P) ((poolp)_Py_ALIGN_DOWN((P), POOL_SIZE))
  
  ```

  简便起见，假如`POOL_SIZE`是`0b00001000`，一个block的地址是`0b00011010`，那么这里的计算方式就是

  ```Python
    0b00011010 & ~(0b00001000 - 1)
  = 0b00011010 &  ~0b00000111
  = 0b00011010 &   0b11111000
  = 0b00011000
  ```

  



### 在有空余空间的 pool 中进行释放

其实步骤和上面一样，比如释放最后一个内存块，位于`0x10ae3dff8`

**Step1** 设置当前释放的内存块的值

![](/cpython_memory_model/image/image_12.png "")

**Step2**

更新freeblock指针

![](/cpython_memory_model/image/image_13.png "")

减小`ref.count`

**Step3** 显然pool之前不是满的，直接跳到Step4

**Step4** 返回

![](/cpython_memory_model/image/image_14.png "")



再来释放倒数第二个，也是同理

Step1

![](/cpython_memory_model/image/image_15.png "")

**Step2**

![](/cpython_memory_model/image/image_16.png "")

**Step3** pool之前不满，跳到step4

**Step4**返回

![](/cpython_memory_model/image/image_17.png "")



### 在释放过block的pool中申请新的空间

和在未释放过block的pool中申请新的空间是一样的思路。从代码上来看

```c
// pymalloc_alloc in  obmalloc.c

    ++pool->ref.count;
    bp = pool->freeblock;
    assert(bp != NULL);

    if (UNLIKELY((pool->freeblock = *(block **)bp) == NULL)) {
        // Reached the end of the free list, try to extend it.
        pymalloc_pool_extend(pool, size);
    }

```


和之前的区别在于这次从`freeblock`获取到的不是NULL了，就不用进入`pymalloc_pool_extend()`



## pool在arena和usedpools中的组织方式

### 内存请求的过程

- 一个arena中存储了许多大小相同的pool，不同arena的pool大小不一定相同。
- `arenas[]`保存了所有`arena_object`
- `usedpools[]`是一个全局数组，其中的元素是个双向链表，**两个元素为一组**，将**相同大小**的pool串起来。一个size没有分配pool时，`usedpools[size+size]`里面的节点指向的pool的`prevpool`和`nextpool`都指向自己。

先来看看`usedpools[]`，内存请求到达`pymalloc_alloc`时，小于等于`SMALL_REQUEST_THRESHOLD`的请求，就是在`usedpools`里来找合适大小的pool。寻找的方式是

- 计算请求内存大小所属size class：`uint size = (uint)(nbytes - 1) >> ALIGNMENT_SHIFT`
- 在`usedpools[size + size]`上寻找pool

如果没有合适大小的pool，就会从`arenas`里请求新的pool，添加到对应位置的双向链表上，从这个新pool中申请block。

![](/cpython_memory_model/image/image_18.png "")

- usedpools的初始定义

  ```c
  // usedpools初始化
  #define PTA(x)  ((poolp )((uint8_t *)&(usedpools[2*(x)]) - 2*sizeof(block *)))
  #define PT(x)   PTA(x), PTA(x)
  
  static poolp usedpools[2 * ((NB_SMALL_SIZE_CLASSES + 7) / 8) * 8] = {
      PT(0), PT(1), PT(2), PT(3), PT(4), PT(5), PT(6), PT(7)
  #if NB_SMALL_SIZE_CLASSES > 8
      , PT(8), PT(9), PT(10), PT(11), PT(12), PT(13), PT(14), PT(15)
  #if NB_SMALL_SIZE_CLASSES > 16
      , PT(16), PT(17), PT(18), PT(19), PT(20), PT(21), PT(22), PT(23)
  #if NB_SMALL_SIZE_CLASSES > 24
      , PT(24), PT(25), PT(26), PT(27), PT(28), PT(29), PT(30), PT(31)
  #if NB_SMALL_SIZE_CLASSES > 32
      , PT(32), PT(33), PT(34), PT(35), PT(36), PT(37), PT(38), PT(39)
  #if NB_SMALL_SIZE_CLASSES > 40
      , PT(40), PT(41), PT(42), PT(43), PT(44), PT(45), PT(46), PT(47)
  #if NB_SMALL_SIZE_CLASSES > 48
      , PT(48), PT(49), PT(50), PT(51), PT(52), PT(53), PT(54), PT(55)
  #if NB_SMALL_SIZE_CLASSES > 56
      , PT(56), PT(57), PT(58), PT(59), PT(60), PT(61), PT(62), PT(63)
  #if NB_SMALL_SIZE_CLASSES > 64
  #error "NB_SMALL_SIZE_CLASSES should be less than 64"
  #endif /* NB_SMALL_SIZE_CLASSES > 64 */
  #endif /* NB_SMALL_SIZE_CLASSES > 56 */
  #endif /* NB_SMALL_SIZE_CLASSES > 48 */
  #endif /* NB_SMALL_SIZE_CLASSES > 40 */
  #endif /* NB_SMALL_SIZE_CLASSES > 32 */
  #endif /* NB_SMALL_SIZE_CLASSES > 24 */
  #endif /* NB_SMALL_SIZE_CLASSES > 16 */
  #endif /* NB_SMALL_SIZE_CLASSES >  8 */
  };
  
  ```

上面这个代码有点巧妙，我们看下`usedpools@0x00007FFED35FF410`刚初始化后里面长啥样

![](/cpython_memory_model/image/image_19.png "")

![](/cpython_memory_model/image/image_20.png "")

回顾一下`pool_header`定义就知道是怎么回事了。

比如`0+0`位置上维护的双链表，`&usedpools[0+0]`在`0x00007FFED35FF410`，而`usedpools[0+0]`为`0x00007FFED35FF400`，`(pool_header*)0x00007FFED35FF400->nextpool` 跨过两个指针大小的内存，刚好就指向`&usedpools[0+0]`了，也就是`usedpools[size+size]->nextpool==&usedpools[size+size]`

总之初始化的值使得正好所有的`usedpools[size+size]`的pool的`prevpool`和`nextpool`字段都指向自己了。



现在假设我们一开始请求一个10字节的内存块。

初始时没有能用的arena和pool，于是在`new_arena()`中：第一次创建的是16个新的arena，分配了16个`arena_object`结构体的空间，保存到全局变量`arenas[]`。但并没有立即分配16个`ARENA_SIZE`的空间作为arena。然后初始化`unused_arena_objects`指向的`arena_object`，此时也就是`arenas[0]`。

![](/cpython_memory_model/image/image_21.png "")

第一个可用的arena的第一个空闲pool被插入`used_pools`里。sentinel就是哨兵的意思，idx1指向的sentinel其实存的是`used_pools[0]`的地址（没理解的话查看上面内容）。

![](/cpython_memory_model/image/image_22.png "")

然后按照前面讲到的[在未释放过block的pool中申请新的空间](https://www.wolai.com/87Y1jakYGgTFnNSbKPJ9G9)获得请求所需要的内存块。



### arena中对pool的管理

arena管理pool的方式和pool管理block的方式非常相似。也是数组+单链表。

`arena_object`结构体：

- 源码如下，注释说得比较清楚

  ```c
  /* Record keeping for arenas. */
  struct arena_object {
      /* The address of the arena, as returned by malloc.  Note that 0
       * will never be returned by a successful malloc, and is used
       * here to mark an arena_object that doesn't correspond to an
       * allocated arena.
       */
      uintptr_t address;
  
      /* Pool-aligned pointer to the next pool to be carved off. */
      block* pool_address;
  
      /* The number of available pools in the arena:  free pools + never-
       * allocated pools.
       */
      uint nfreepools;
  
      /* The total number of pools in the arena, whether or not available. */
      uint ntotalpools;
  
      /* Singly-linked list of available pools. */
      struct pool_header* freepools;
  
      /* Whenever this arena_object is not associated with an allocated
       * arena, the nextarena member is used to link all unassociated
       * arena_objects in the singly-linked `unused_arena_objects` list.
       * The prevarena member is unused in this case.
       *
       * When this arena_object is associated with an allocated arena
       * with at least one available pool, both members are used in the
       * doubly-linked `usable_arenas` list, which is maintained in
       * increasing order of `nfreepools` values.
       *
       * Else this arena_object is associated with an allocated arena
       * all of whose pools are in use.  `nextarena` and `prevarena`
       * are both meaningless in this case.
       */
      struct arena_object* nextarena;
      struct arena_object* prevarena;
  };
  ```

当然，我这可以再翻译一下和做点解释

`address`

用来保存分配给arena地址。

`pool_address`

pool的地址，指向下一个要取出的pool的地址



因为`malloc()`返回给arena的地址不一定对齐到`POOL_SIZE`，所以如果真的没有对齐，那么arena会舍弃前后加起来共`POOL_SIZE`大小的空间（图中`POOL_SIZE`为4K），保证最开始时，`pool_address`指向已经对齐好的pool，随时可以分配。

arena的对齐实现：

![](/cpython_memory_model/image/image_23.png "")

#### 在未释放过pool的arena中申请新的空间

按顺序，前面的所有pool都是used，`pool_address`负责指向空闲内存

![](/cpython_memory_model/image/image_24.png "")

申请一个新的pool，从`pool_address`上取

![](/cpython_memory_model/image/image_25.png "")

`nfreepools`减小，`pool_address`更新为下一个空余的pool

再申请一个，满了。

![](/cpython_memory_model/image/image_26.png "")



#### 在arena中释放空间

pool变为empty时，pool从`usedpools`中移除，插入到与之对应的arena中，`freepools`所管理的单链表中。

比如我们释放pool3

`pool3->nextpool`设置为`arena->freepools`相同值。因为是满了之后第一个释放的，所以这个时候是0

![](/cpython_memory_model/image/image_27.png "")

freepools更新为pool3的地址，nfreepools增加

![](/cpython_memory_model/image/image_28.png "")

比如再释放一个pool63

![](/cpython_memory_model/image/image_29.png "")



#### 在释放过pool的arena中申请新的空间

在arena中申请新的空间，先判断`freepools`是不是`NULL`。这里的情况下，`nfreepools`大于0，`freepools`不为空，从这个单链表中取pool。

![](/cpython_memory_model/image/image_30.png "")

更新`freepools`指针，和`nfreepools`

![](/cpython_memory_model/image/image_31.png "")



## arena的管理

arena由`arena_object`结构体数组`arenas[]`和链表`unused_arena_objects`和`usable_arenas`进行管理。



**单链表****`unused_arena_objects`**

维护`arenas`中所有pool都没被使用或都被释放了的`arena_object`，凡是放入这个链表的`arena_object`，其arena占用的pool都会被释放，但`arena_object`结构体本身不会被释放。

在 Python 2.5 之前, arena申请过的空间是从来不会被释放的, 这个策略是在 Python 2.5 之后引入的。



**双向链表****`usable_arenas`**
维护有空闲pool的arena的`arena_object`



第一次内存请求前没有分配`arena_object`，`usable_arenas == NULL`，第一次分配时，分配16个`arena_object`存入`arenas[]`，之后每次扩容arenas，总共分配的`arena_object`数量翻倍。

由于是第一次分配，所以先构建`unused_arena_objects`单链表，留在这个链表中的`arena_object`一定是没分配实际内存的，也就是`address`字段是0

（白色为不关联实际内存，绿色表示有可用pool，红色表示所有pool已满）

![](/cpython_memory_model/image/image_32.png "")

`new_arena()`内从`unused_arena_objects`处获取`arena_object`并为之真正分配`ARENA_SIZE`的内存，做好初始化：对齐arena中的pool到POOL_SIZE，设置一些字段。

`usable_arenas`是个双向链表（下图没体现出来），维护包含空闲pool的arena，管理的`arena_object`按其`nfreepools`升序排列，获取pool时，优先从重度使用的arena中分配，这样可以让比较空的arena有更多释放内存的可能性。

![](/cpython_memory_model/image/image_33.png "")



16个arena都用完了的时候，`arena_objects`数量翻倍，然后做前面同样的初始化。

扩容用的realloc，不用管指针了？是的。因为既然前面的`arena_objects`对应的arena都满了，`usable_arenas`和`unused_arena_objects`就都为`NULL`了，自然也没有`nextarena`，`prevarena`指针指向原来的内存，realloc后，构建新的`unused_arena_objects`单链表

![](/cpython_memory_model/image/image_34.png "")

返回新的arena给`usable_arenas`

![](/cpython_memory_model/image/image_35.png "")



- `new_arena()`源码

  ```c
  // obmalloc.c
  
  /* Allocate a new arena.  If we run out of memory, return NULL.  Else
   * allocate a new arena, and return the address of an arena_object
   * describing the new arena.  It's expected that the caller will set
   * `usable_arenas` to the return value.
   */
  static struct arena_object*
  new_arena(void)
  {
      struct arena_object* arenaobj;
      uint excess;        /* number of bytes above pool alignment */
      void *address;
      static int debug_stats = -1;
  
      if (debug_stats == -1) {
          const char *opt = Py_GETENV("PYTHONMALLOCSTATS");
          debug_stats = (opt != NULL && *opt != '\0');
      }
      if (debug_stats)
          _PyObject_DebugMallocStats(stderr);
  
      if (unused_arena_objects == NULL) {
          uint i;
          uint numarenas;
          size_t nbytes;
  
          /* Double the number of arena objects on each allocation.
           * Note that it's possible for `numarenas` to overflow.
           */
          numarenas = maxarenas ? maxarenas << 1 : INITIAL_ARENA_OBJECTS;     // 将最大arenas数量翻倍
          if (numarenas <= maxarenas)
              return NULL;                /* overflow */
  #if SIZEOF_SIZE_T <= SIZEOF_INT
          if (numarenas > SIZE_MAX / sizeof(*arenas))
              return NULL;                /* overflow */
  #endif
          nbytes = numarenas * sizeof(*arenas);
          arenaobj = (struct arena_object *)PyMem_RawRealloc(arenas, nbytes); // realloc内存
          if (arenaobj == NULL)
              return NULL;
          arenas = arenaobj;
  
          /* We might need to fix pointers that were copied.  However,        // Realloc后的指针无需处理
           * new_arena only gets called when all the pages in the             // 因为前面的arena满了，才需要realloc，
           * previous arenas are full.  Thus, there are *no* pointers         // 既然满了，就没有指针(unused_arena_objects 和 usable_arenas链表中的指针)
           * into the old array. Thus, we don't have to worry about           // 指向原来的arenas数组里。
           * invalid pointers.  Just to be sure, some asserts:
           */
          assert(usable_arenas == NULL);
          assert(unused_arena_objects == NULL);
  
          /* Put the new arenas on the unused_arena_objects list. */
          for (i = maxarenas; i < numarenas; ++i) {
              arenas[i].address = 0;              /* mark as unassociated */
              arenas[i].nextarena = i < numarenas - 1 ?
                                     &arenas[i+1] : NULL;
          }
  
          /* Update globals. */
          unused_arena_objects = &arenas[maxarenas];                          // unused_arena_objects指向arenas新扩容出来的部分头部
          maxarenas = numarenas;
      }
  
      /* Take the next available arena object off the head of the list. */
      assert(unused_arena_objects != NULL);
      arenaobj = unused_arena_objects;
      unused_arena_objects = arenaobj->nextarena;
      assert(arenaobj->address == 0);
      address = _PyObject_Arena.alloc(_PyObject_Arena.ctx, ARENA_SIZE);
  #if WITH_PYMALLOC_RADIX_TREE
      if (address != NULL) {
          if (!arena_map_mark_used((uintptr_t)address, 1)) {
              /* marking arena in radix tree failed, abort */
              _PyObject_Arena.free(_PyObject_Arena.ctx, address, ARENA_SIZE);
              address = NULL;
          }
      }
  #endif
      if (address == NULL) {
          /* The allocation failed: return NULL after putting the
           * arenaobj back.
           */
          arenaobj->nextarena = unused_arena_objects;
          unused_arena_objects = arenaobj;
          return NULL;
      }
      arenaobj->address = (uintptr_t)address;
  
      ++narenas_currently_allocated;
      ++ntimes_arena_allocated;
      if (narenas_currently_allocated > narenas_highwater)
          narenas_highwater = narenas_currently_allocated;
      arenaobj->freepools = NULL;
      /* pool_address <- first pool-aligned address in the arena
         nfreepools <- number of whole pools that fit after alignment */
      arenaobj->pool_address = (block*)arenaobj->address;
      arenaobj->nfreepools = MAX_POOLS_IN_ARENA;
      excess = (uint)(arenaobj->address & POOL_SIZE_MASK); // 当前arena地址没对齐到pool大小
      if (excess != 0) {
          --arenaobj->nfreepools;                          // 为了对齐pool，可用pool会减少一个
          arenaobj->pool_address += POOL_SIZE - excess;    // 进行对齐操作
      }
      arenaobj->ntotalpools = arenaobj->nfreepools;
  
      return arenaobj;
  }
  
  
  ```



# 一些API

## obmalloc中的调用链

`_PyObject_Malloc`和`_PyObject_Calloc`调用`pymalloc_alloc`，申请的内存大于`SMALL_REQUEST_THRESHOLD`的内存时，`pymalloc_alloc`返回NULL，于是前者又分别会调用`PyMem_RawMalloc`和`PyMem_RawCalloc`

参考源码`_PyObject_Malloc`和`_PyObject_Calloc`的实现

small request的调用链

```C
PyObject_Malloc(size_t size)
    -> _PyObject.malloc  == (PYOBJ_ALLOC == PYMALLOC_ALLOC).malloc == _PyObject_Malloc
    _PyObject_Malloc(void *ctx, size_t nbytes)
        -> pymalloc_alloc(void *ctx, size_t nbytes) get memory from pool
```


bigger request的调用链

```C
PyObject_Malloc(size_t size)
    -> _PyObject.malloc  == (PYOBJ_ALLOC == PYMALLOC_ALLOC).malloc == _PyObject_Malloc
    _PyObject_Malloc(void *ctx, size_t nbytes)
        get NULL from pymalloc_alloc
        -> PyMem_RawMalloc(size_t size)
            -> malloc(size_t size) of CRT
```




`PyMem_`系列



`obmalloc.c`中：

```c
#define MALLOC_ALLOC {NULL, _PyMem_RawMalloc, _PyMem_RawCalloc, _PyMem_RawRealloc, _PyMem_RawFree}
#ifdef WITH_PYMALLOC
#  define PYMALLOC_ALLOC {NULL, _PyObject_Malloc, _PyObject_Calloc, _PyObject_Realloc, _PyObject_Free}
#endif

#define PYRAW_ALLOC MALLOC_ALLOC
#ifdef WITH_PYMALLOC
#  define PYOBJ_ALLOC PYMALLOC_ALLOC
#else
#  define PYOBJ_ALLOC MALLOC_ALLOC
#endif
#define PYMEM_ALLOC PYOBJ_ALLOC
```


默认是定义了`WITH_PYMALLOC`的，也就是说默认选用`_PyObject_*`函数族而不是`_PyMem_Raw*`



WITH_PYMALLOC

PYMEM_ALLOC→PYOBJ_ALLOC→PYMALLOC_ALLOC→`_PyObject_*`函数族

没有WITH_PYMALLOC

PYMEM_ALLOC→PYOBJ_ALLOC→MALLOC_ALLOC→`_PyMem_Raw*`函数族



# TODO

- [ ] [bpo-37029: keep usable_arenas in sorted order without searching (#13… · python/cpython@1c263e3 (github.com)](https://github.com/python/cpython/commit/1c263e39c4ed28225a7dc8ca1f24953225ac48ca#diff-399a22135f328b4e42b0722ef216587945eedf2d8c103a584a3dca5b30650329)


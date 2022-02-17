# iOS Teach Team iOS开发过程中常用的锁

### 引言
在 iOS 开发过程中我们通过异步和多线程来提高程序的运行性能，于此同时多线程安全也就成为了一个我们必须要面对的问题，从安全上来说应该尽量避免资源在线程之间共享，以减少线程间的相互作用，因此线程锁就应运而生。在使用锁的过程中一定要小心，避免造成死锁而引起程序无法正常运行。

---
* ### 常用的几种锁
> 1. @synchronized        // 互斥锁（互斥递归锁）
> 2. NSLock               // 互斥锁
> 3. dispatch_semaphore   // 信号量
> 4. pthread_mutex        // 互斥锁
> 5. NSCondition          // 条件锁
> 6. NSRecursiveLock      // 递归锁
> 7. NSConditionLock      // 条件锁
> 8. OSSpinLock           // 自旋锁


**上面8种类型锁，按照功能来说可以划分为2两种：**

**1. 互斥锁：**

> 是一种用于多线程编程中，防止两条线程同时对同一公共资源（比如全局变量）进行读写的机制。该目的通过将代码切片成一个一个的临界区域（critical section）达成。临界区域指的是一块对公共资源进行存取的代码，并非一种机制或是算法。一个程序、进程、线程可以拥有多个临界区域，但是并不一定会应用互斥锁。
>
> **如果调用线程在想要获得锁资源的时候发现锁已经被其他线程持有，那么该调用线程将会进入休眠状态，CPU 进而去执行其他线程任务，直到被锁资源释放锁，此时会唤醒休眠线程，这样抢占式策略不会占用 CPU 资源。但是因为关系到 CPU 上下文切换，因此会有时间的消耗**。

**互斥锁:** `@synchronized`、 `NSLock`、 `pthread_mutex`、 `NSConditionLock`、 `NSCondition`、 `NSRecursiveLock`

**2. 自旋锁**

> 自旋锁是多线程同步的一种锁，线程反复检查锁变量是否可用。由于线程在这一过程中保持执行，因此是一种忙等待。一旦获取了自旋锁，线程会一直保持该锁，直至显式释放自旋锁。**自旋锁避免了进程上下文的调度开销，因此对于线程只会阻塞很短时间的场合是有效的**。

**自旋锁:** `OSSpinLock`

---
* ### 具体使用介绍

1. @synchronized 是一个互斥递归锁：

递归锁也称为可重入锁。**互斥锁**可以分为**非递归锁/递归锁**两种，**主要区别在于:** 同一个线程可以重复获取递归锁，不会死锁; 同一个线程重复获取非递归锁，则会产生死锁。

通常使用方式如下：

```
@synchronized (obj) {
    // Add operation
} 
```
上面代码实际执行过程如下：

> 1. objc_sync_enter(id obj) // 具体执行源码在下面
> 2. 要加锁的操作
> 3. objc_sync_exit(id obj) // 具体执行源码在下面

`@synchronized` 是一个互斥锁->递归锁，内部搭配 `nil` 防止死锁，通过表的结构存要锁的对象,表内部的对象又是通过哈希存储的

```
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        SyncData* data = id2data(obj, ACQUIRE); // 从表中取出需要锁的数据
        ASSERT(data);
        data->mutex.lock();
    } else {
        // @synchronized(nil) does nothing
        if (DebugNilSync) {
            _objc_inform("NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug");
        }
        objc_sync_nil();
    }

    return result;
}
```

```
int objc_sync_exit(id obj)
{
    int result = OBJC_SYNC_SUCCESS;
    
    if (obj) {
        SyncData* data = id2data(obj, RELEASE); 
        if (!data) {
            result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
        } else {
            bool okay = data->mutex.tryUnlock();
            if (!okay) {
                result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
            }
        }
    } else {
        // @synchronized(nil) does nothing
    }
	
    return result;
}
```

**注意点：** 

1. 在多线程异步同时操作同一个对象时，因为递归锁会不停的 `objc_sync_enter(id obj)` ，有些特殊场景下 `obj` 对象可能会为 `nil` ，而此时 `objc_sync_enter(id obj)` 内部会进行判断，如果 `obj==nil` ，就不会再加锁，进而导致线程访问冲突。
2. 如果多线程中 `@synchronized (obj)` 传入的 `obj` 不相同，即一个新的 `obj` ，那么线程每次都将会拥有它的锁，并持续处理，中间不会被其他线程阻塞。

下面着重介绍一下 `SyncData` 的存取过程，即 `id2data(obj, ACQUIRE)` ，在介绍代码调用流程之前，先看一下几个结构体：

```
typedef struct alignas(CacheLineSize) SyncData {
    struct SyncData* nextData; // 下一个SyncData
    DisguisedPtr<objc_object> object; // 锁的对象
    int32_t threadCount;  // number of THREADS using this block 等待的线程数量
    recursive_mutex_t mutex; // 互斥递归锁
} SyncData;

typedef struct {
    SyncData *data;
    unsigned int lockCount;  // number of times THIS THREAD locked this block
} SyncCacheItem;

typedef struct SyncCache {
    unsigned int allocated;
    unsigned int used;
    SyncCacheItem list[0];
} SyncCache;

struct SyncList {
    SyncData *data;
    spinlock_t lock;

    constexpr SyncList() : data(nil), lock(fork_unsafe_lock) { }
};
```

```
using recursive_mutex_t = recursive_mutex_tt<LOCKDEBUG>;
template <bool Debug>
class recursive_mutex_tt : nocopy_t {
    os_unfair_recursive_lock mLock;

  public:
    constexpr recursive_mutex_tt() : mLock(OS_UNFAIR_RECURSIVE_LOCK_INIT) {
        lockdebug_remember_recursive_mutex(this);
    }

    constexpr recursive_mutex_tt(__unused const fork_unsafe_lock_t unsafe)
        : mLock(OS_UNFAIR_RECURSIVE_LOCK_INIT)
    { }
};
```

`recursive_mutex_t` 是一个互斥递归锁，它是基于 `os_unfair_recursive_lock` 互斥锁的封装（再早些版本是对 `pthread_mutex_t` 的封装），而 `os_unfair_recursive_lock` 底层是对 `os_unfair_lock` 的封装，进一步跟进代码会发现

```
#define OS_UNFAIR_RECURSIVE_LOCK_AVAILABILITY \
        __OSX_AVAILABLE(10.14) __IOS_AVAILABLE(12.0) \
        __TVOS_AVAILABLE(12.0) __WATCHOS_AVAILABLE(5.0)
```

可以知道，`os_unfair_recursive_lock` 是 `iOS 12.0` 之后才可用的，即 `iOS 12.0` 之后 `@synchronized` 是一个封装了 `os_unfair_lock` 的互斥递归锁。

接下来我们回到 `id2data` 的调用上，因为源码内容较多，此处仅对核心部分进行介绍，大致分为四步：

```
static SyncData* id2data(id object, enum usage why)
{
    // 第一步：
    // 检查每个线程的单项快速缓存以查找匹配的对象
    // 从快速缓存中获取 SyncData
    SyncData *data = (SyncData *)tls_get_direct(SYNC_DATA_DIRECT_KEY);
    if (data) {
        if (data->object == object) {
            //...省略
            return result;
        }
    }
    
    // 第二步：
    // 检查已拥有锁的每个线程缓存以查找匹配对象
    // 从当前线程缓存中获取 SyncCache
    SyncCache *cache = fetch_cache(NO);
    if (cache) {
            //...省略
            return result;
        }
    }
    
    // 第三步：
    // 如果上面缓存都找不到，遍历使用中的列表，寻找匹配的对象，再进行相应操作
    lockp->lock();
    {
       //...省略
    }

    posix_memalign((void **)&result, alignof(SyncData), sizeof(SyncData));
    result->object = (objc_object *)object;
    result->threadCount = 1;
    new (&result->mutex) recursive_mutex_t(fork_unsafe_lock);
    result->nextData = *listp;
    *listp = result;
    
    // 第四步：
    // 如果一切正常，支持快速缓存则存入快速缓存，否则存入线程缓存中，便于下次快速查找。如果有错误，则抛出异常；
 done:
    lockp->unlock();
    if (result) {
        //...省略
    }

    return result;
}
```

下面具体介绍一下每一步都做了什么。

1. 第一步：快速缓存

如果支持快速缓存，则优先从快速缓存中查找匹配对象，如果匹配到对象，根据 `usage` 枚举值进行相应操作，然后返回 `result` 。

```
#if SUPPORT_DIRECT_THREAD_KEYS
    // 检查每个线程的单项快速缓存以查找匹配的对象
    // Check per-thread single-entry fast cache for matching object
    bool fastCacheOccupied = NO;
    // 从快速缓存中获取 SyncData
    SyncData *data = (SyncData *)tls_get_direct(SYNC_DATA_DIRECT_KEY);
    if (data) {
        fastCacheOccupied = YES;

        if (data->object == object) {
            // Found a match in fast cache.
            uintptr_t lockCount;

            result = data;
            // 获取当前线程 tls 缓存里的 SyncData 加锁次数
            lockCount = (uintptr_t)tls_get_direct (SYNC_COUNT_DIRECT_KEY);
            if (result->threadCount <= 0  ||  lockCount <= 0) {
                _objc_fatal("id2data fastcache is buggy");
            }
            // 根据枚举 enum usage { ACQUIRE, RELEASE, CHECK } 更新当前线程 tsl 缓存里的加锁次数
            switch(why) {
            case ACQUIRE: {
                // 获取加锁一次
                lockCount++;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);
                break;
            }
            case RELEASE:
                // 释放减锁一次
                lockCount--;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);
                if (lockCount == 0) {
                    // 如果当前线程 tls 加锁次数为0，则从当前 tls 移除
                    // remove from fast cache
                    tls_set_direct(SYNC_DATA_DIRECT_KEY, NULL);
                    // atomic because may collide with concurrent ACQUIRE
                    OSAtomicDecrement32Barrier(&result->threadCount);
                }
                break;
            case CHECK:
                // 检查锁时，不做操作
                // do nothing
                break;
            }
            // 此处返回
            return result;
        }
    }
#endif
```

2. 第二步：线程缓存

快速缓存没找到，则从线程缓存中中查找，如果匹配到对象，根据 `usage` 枚举值进行相应操作，然后返回 `result` 。

```
// 检查已拥有锁的每个线程缓存以查找匹配对象
// Check per-thread cache of already-owned locks for matching object
// 从当前线程缓存中获取 SyncCache
SyncCache *cache = fetch_cache(NO);
if (cache) {
    unsigned int i;
    for (i = 0; i < cache->used; i++) {
        SyncCacheItem *item = &cache->list[i];
        if (item->data->object != object) continue;

        // Found a match.
        result = item->data;
        if (result->threadCount <= 0  ||  item->lockCount <= 0) {
            _objc_fatal("id2data cache is buggy");
        }
            
        switch(why) {
        case ACQUIRE:
            item->lockCount++; // 加锁次数+1
            break;
        case RELEASE:
            item->lockCount--; // 加锁次数-1
            if (item->lockCount == 0) {
                // remove from per-thread cache
                // 加锁次数为0，则移除缓存
                cache->list[i] = cache->list[--cache->used];
                // atomic because may collide with concurrent ACQUIRE
                OSAtomicDecrement32Barrier(&result->threadCount);
            }
            break;
        case CHECK:
            // do nothing
            break;
        }

        return result;
    }
}
```

3. 第三步：无缓存

如果当前 `object` 无缓存，遍历使用中的列表，寻找匹配的对象，再进行相应操作。

```
lockp->lock(); // 加锁

{
    SyncData* p;
    SyncData* firstUnused = NULL;
    // 如果没有缓存，使用 listp 进行操作，使用空闲节点或创建新节点
    for (p = *listp; p != NULL; p = p->nextData) {
        if ( p->object == object ) {
            result = p;
            // atomic because may collide with concurrent RELEASE
            OSAtomicIncrement32Barrier(&result->threadCount);
            goto done;
        }
        if ( (firstUnused == NULL) && (p->threadCount == 0) )
            firstUnused = p;
    }
    // 没有当前与 object 关联的 SyncData 用于 RELEASE 或 CHECK 时直接 goto done 
    // no SyncData currently associated with object
    if ( (why == RELEASE) || (why == CHECK) )
        goto done;

    // listp 中有空闲节点，则使用这个节点把它和当前 object 关联起来
    // an unused one was found, use it
    if ( firstUnused != NULL ) {
        result = firstUnused;
        result->object = (objc_object *)object;
        result->threadCount = 1;
        goto done;
    }
}
// 新建 SyncData
posix_memalign((void **)&result, alignof(SyncData), sizeof(SyncData));
result->object = (objc_object *)object;
result->threadCount = 1;
new (&result->mutex) recursive_mutex_t(fork_unsafe_lock);
result->nextData = *listp;
*listp = result;
```

4. 第四步：Done 保存 SyncData 对象

```    
 done:
    // 释放前面一步添加的锁
    lockp->unlock();
    if (result) {
        // Only new ACQUIRE should get here.
        // All RELEASE and CHECK and recursive ACQUIRE are 
        // handled by the per-thread caches above.
        // 释放
        if (why == RELEASE) {
            // Probably some thread is incorrectly exiting 
            // while the object is held by another thread.
            return nil;
        }
        // 不是获取加锁，调用 `_objc_fatal`
        if (why != ACQUIRE) _objc_fatal("id2data is buggy");
        // 如果不是当前对象，同样调用 `_objc_fatal`
        if (result->object != object) _objc_fatal("id2data is buggy");

#if SUPPORT_DIRECT_THREAD_KEYS
        // 支持快速缓存，则加入到快速缓存中
        if (!fastCacheOccupied) {
            // Save in fast thread cache
            tls_set_direct(SYNC_DATA_DIRECT_KEY, result);
            tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)1);
        } else 
#endif
        {
            // 不支持快速缓存，则加入到线程缓存中
            // Save in thread cache
            if (!cache) cache = fetch_cache(YES);
            cache->list[cache->used].data = result;
            cache->list[cache->used].lockCount = 1;
            cache->used++;
        }
    }
    // 返回 SyncData
    return result;
```
这里解释一下 `_objc_fatal` ，如果是 `Debug` 环境下，App不会崩溃，但会调用 `_objc_syslog` 打印日志，之后调用 `_Exit(1)` 正常退出程序执行。

如果是 `release` 环境下，则会调用 `_objc_crashlog` 将信息添加到崩溃日志中，然后调用 `abort_with_reason` 函数来中断程序运行

**2. NSLock 互斥锁**



---
### 拓展知识

---
### 总结


---
**最后：？！**

### 参考资料：

* [Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html) ( https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html )

* [More than you want to know about @synchronized](https://www.rykap.com/objective-c/2015/05/09/synchronized/) （ https://www.rykap.com/objective-c/2015/05/09/synchronized/ ）
# iOS Teach Team iOS 常用锁的底层实现及使用详解

### 引言

> 在 iOS 开发过程中我们通过异步和多线程来提高程序的运行性能，与此同时多线程安全也就成为了一个我们必须要面对的问题，从安全上来说应该尽量避免资源在线程之间共享，以减少线程间的相互作用，因此线程锁就应运而生。在使用锁的过程中一定要小心，避免造成死锁而引起程序无法正常运行。本文就着重介绍一下 iOS 日常开发中常用的几种锁的底层实现原理及如何使用它们。

---
* ### 常用的几种锁
> 1. @synchronized        // 互斥锁（互斥递归锁）
> 2. pthread_mutex        // 互斥锁
> 3. NSLock               // 互斥锁
> 4. NSRecursiveLock      // 递归锁
> 5. NSCondition          // 条件锁
> 6. NSConditionLock      // 条件锁
> 7. dispatch_semaphore   // 信号量（严格的来说不能算锁）
> 8. OSSpinLock           // 自旋锁


**按照功能来说上面的锁可以划分为2两种：**

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

**递归锁**也称为可重入锁。**互斥锁**可以分为**非递归锁/递归锁**两种，主要区别在于： 同一个线程可以重复获取递归锁，不会死锁; 同一个线程重复获取非递归锁，则会产生死锁。

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

`@synchronized` 是一个互斥锁->递归锁，内部搭配 `nil` 防止死锁，通过表的结构存要锁的对象，表内部的对象又是通过哈希存储的。

`objc_sync_enter(id obj)` 源码：

```
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        // 执行 ACQUIRE 操作，并返回对应 SyncData
        SyncData* data = id2data(obj, ACQUIRE); 
        ASSERT(data);
        // 加锁操作
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

`objc_sync_exit(id obj)` 源码：

```
int objc_sync_exit(id obj)
{
    int result = OBJC_SYNC_SUCCESS;
    
    if (obj) {
        // 执行 RELEASE 操作，并返回对应 SyncData
        SyncData* data = id2data(obj, RELEASE); 
        if (!data) {
            result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
        } else {
            // 尝试解锁操作
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

1. 在多线程异步同时操作同一个对象时，因为递归锁会不停的 `objc_sync_enter(id obj)` ，有些特殊场景下 `obj` 对象可能会为 `nil` ，而此时 `objc_sync_enter(id obj)` 内部会进行判断，如果 `obj == nil` ，就不会再加锁，进而导致线程访问冲突。
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

快速缓存没找到，则从线程缓存中查找，如果匹配到对象，根据 `usage` 枚举值进行相应操作，然后返回 `result` 。

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

4. 第四步：`done` 保存 `SyncData` 对象

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

如果是 `release` 环境下，则会调用 `_objc_crashlog` 将信息添加到崩溃日志中，然后调用 `abort_with_reason` 函数来中断程序运行。

**2. pthread_mutex 互斥锁**

在 `POSIX`（Portable Operating System Interface：可移植操作系统）中，`pthread_mutex` 是一套用于多线程同步的 `mutex` 锁，如同名一样，使用起来非常简单，性能比较高。`pthread_mutex` 不是使用忙等，而是同信号量一样，会阻塞线程并进行等待，调用时进行线程上下文切换。

```
// 导入头文件
#import <pthread.h>

// 声明互斥锁
pthread_mutex_t _lock;

// 声明互斥锁属性对象
pthread_mutexattr_t attr;

// 初始化属性对象
pthread_mutexattr_init(&attr);

// 初始化互斥锁
pthread_mutex_init(&lock, &attr);
pthread_mutex_init(&lock, NULL);

// 加锁
pthread_mutex_lock(&_lock);

// 解锁 
pthread_mutex_unlock(&_lock);

// 释放锁
pthread_mutex_destroy(&_lock);

// 设置互斥锁的类型属性
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_NORMAL);

// 互斥锁支持的协议类型宏定义
/*
 * Mutex protocol attributes
 */
#define PTHREAD_PRIO_NONE            0
#define PTHREAD_PRIO_INHERIT         1
#define PTHREAD_PRIO_PROTECT         2

// 互斥锁支持的类型宏定义
/*
 * Mutex type attributes
 */
#define PTHREAD_MUTEX_NORMAL		0
#define PTHREAD_MUTEX_ERRORCHECK	1
#define PTHREAD_MUTEX_RECURSIVE		2
#define PTHREAD_MUTEX_DEFAULT       PTHREAD_MUTEX_NORMAL
```
如果对 `pthread_mutex` 详细使用有兴趣的，可以阅读[互斥锁属性](https://docs.oracle.com/cd/E19253-01/819-7051/6n919hpaf/index.html#sync-26886)深入了解一下。

**3. NSLock 互斥锁**

`NSLock` 底层是对 `pthread_mutex` 的一层封装，其性能比 `pthread_mutex` 略差。但由于缓存的存在，多次调用并不会对性能参数太大影响。


`NSLock` 源码在 `CoreFundation` 框架中，无法进行查看，但 `swift` 版本开源的 `CoreFoundation` 中我们可以看到关于 `NSLock` 的完整定义 [NSLock.swift](https://github.com/apple/swift-corelibs-foundation/blob/ce2827f06ca218fcb8756d9f0bc086b9746dffbe/Sources/Foundation/NSLock.swift)。

```
private typealias _MutexPointer = UnsafeMutablePointer<pthread_mutex_t>
private typealias _RecursiveMutexPointer = UnsafeMutablePointer<pthread_mutex_t>
private typealias _ConditionVariablePointer = UnsafeMutablePointer<pthread_cond_t>

open class NSLock: NSObject, NSLocking {
    internal var mutex = _MutexPointer.allocate(capacity: 1)

    private var timeoutCond = _ConditionVariablePointer.allocate(capacity: 1)
    private var timeoutMutex = _MutexPointer.allocate(capacity: 1)

    public override init() {
        // 初始化互斥锁，没有添加 pthread_mutexattr_t
        pthread_mutex_init(mutex, nil)
        pthread_cond_init(timeoutCond, nil)
        pthread_mutex_init(timeoutMutex, nil)
    }
    
    deinit {
        // 销毁互斥锁
        pthread_mutex_destroy(mutex)
        mutex.deinitialize(count: 1)
        mutex.deallocate()

        deallocateTimedLockData(cond: timeoutCond, mutex: timeoutMutex)
    }
    
    open func lock() {
        // pthread_mutex 加锁
        pthread_mutex_lock(mutex)
    }

    open func unlock() {
        // pthread_mutex 解锁
        pthread_mutex_unlock(mutex)

        // Wakeup any threads waiting in lock(before:)
        pthread_mutex_lock(timeoutMutex)
        // 广播当前锁的状态
        pthread_cond_broadcast(timeoutCond)
        pthread_mutex_unlock(timeoutMutex)
    }

    // 尝试图获取锁，但是如果锁不可用的时候，它不会阻塞线程，只会返回 false
    open func `try`() -> Bool {
        return pthread_mutex_trylock(mutex) == 0
    }
    
    // 在某一个时间点之前不断尝试加锁
    open func lock(before limit: Date) -> Bool {
        // pthread_mutex_trylock尝试加锁，如果加锁成功则直接返回true
        if pthread_mutex_trylock(mutex) == 0 {
            return true
        }

        // 内部通过 while 循环调用 pthread_mutex_trylock 不断尝试加锁。如果失败失败则返回false，如果成功则返回true
        return timedLock(mutex: mutex, endTime: limit, using: timeoutCond, with: timeoutMutex)
    }
    // 可以使用一个字符串名称作为锁的标识，Cocoa 会将此名称用作涉及接收方的错误的一部分。
    open var name: String?
}
```
从代码可见，`NSLock` 还有对 `timeout` 超时控制，需要注意的是 `NSLock` 并不支持递归调用，即同一个线程不支持锁两次，如果出现递归调用则会造成死锁。如果需要实现递归调用，可以使用 `NSRecursiveLock` 。

**4. NSRecursiveLock 递归锁**

同 `NSLock` 类似，我们也可以在 [NSLock.swift](https://github.com/apple/swift-corelibs-foundation/blob/ce2827f06ca218fcb8756d9f0bc086b9746dffbe/Sources/Foundation/NSLock.swift) 中查看 `NSRecursiveLock` 的完整定义：
```
private typealias _RecursiveMutexPointer = UnsafeMutablePointer<pthread_mutex_t>

open class NSRecursiveLock: NSObject, NSLocking {
    internal var mutex = _RecursiveMutexPointer.allocate(capacity: 1)

    private var timeoutCond = _ConditionVariablePointer.allocate(capacity: 1)
    private var timeoutMutex = _MutexPointer.allocate(capacity: 1)

    public override init() {
        super.init()
        var attrib = pthread_mutexattr_t()

        withUnsafeMutablePointer(to: &attrib) { attrs in
            pthread_mutexattr_init(attrs)
            // 初始化 pthread_mutexattr_t 互斥属性，设置递归类型为 PTHREAD_MUTEX_RECURSIVE
            let type = Int32(PTHREAD_MUTEX_RECURSIVE)
            pthread_mutexattr_settype(attrs, type)
            pthread_mutex_init(mutex, attrs)
        }
        pthread_cond_init(timeoutCond, nil)
        pthread_mutex_init(timeoutMutex, nil)
    }
    
    deinit {
        pthread_mutex_destroy(mutex)
        mutex.deinitialize(count: 1)
        mutex.deallocate()

        deallocateTimedLockData(cond: timeoutCond, mutex: timeoutMutex)
    }
    
    open func lock() {
        pthread_mutex_lock(mutex)
    }
    
    open func unlock() {
        pthread_mutex_unlock(mutex)

        // Wakeup any threads waiting in lock(before:)
        pthread_mutex_lock(timeoutMutex)
        pthread_cond_broadcast(timeoutCond)
        pthread_mutex_unlock(timeoutMutex)
    }
    
    open func `try`() -> Bool {
        return pthread_mutex_trylock(mutex) == 0
    }
    
    open func lock(before limit: Date) -> Bool {
        if pthread_mutex_trylock(mutex) == 0 {
            return true
        }

        return timedLock(mutex: mutex, endTime: limit, using: timeoutCond, with: timeoutMutex)
    }

    open var name: String?
}
```

可以看到 `NSRecursiveLock` 同 `NSLock` 类似，只是初始化时有些差别，正是这些差别，使得 `NSRecursiveLock` 支持递归调用。
```
// NSLock 初始化没有添加 pthread_mutexattr_t
pthread_mutex_init(mutex, nil)

// NSRecursiveLock 初始化添加 pthread_mutexattr_t ，设置类型为递归类型 PTHREAD_MUTEX_RECURSIVE
var attrib = pthread_mutexattr_t()
withUnsafeMutablePointer(to: &attrib) { attrs in
    pthread_mutexattr_init(attrs)
    // 初始化 pthread_mutexattr_t 互斥属性，设置递归类型为 PTHREAD_MUTEX_RECURSIVE
    let type = Int32(PTHREAD_MUTEX_RECURSIVE)
    pthread_mutexattr_settype(attrs, type)
    pthread_mutex_init(mutex, attrs)
}
```

**5. NSCondition 条件锁**

```
private typealias _MutexPointer = UnsafeMutablePointer<pthread_mutex_t>
private typealias _ConditionVariablePointer = UnsafeMutablePointer<pthread_cond_t>

open class NSCondition: NSObject, NSLocking {
    internal var mutex = _MutexPointer.allocate(capacity: 1)
    // 条件变量 pthread_cond_t
    internal var cond = _ConditionVariablePointer.allocate(capacity: 1)

    public override init() {
        pthread_mutex_init(mutex, nil)
        pthread_cond_init(cond, nil)
    }
    
    deinit {
        pthread_mutex_destroy(mutex)
        pthread_cond_destroy(cond)
        mutex.deinitialize(count: 1)
        cond.deinitialize(count: 1)
        mutex.deallocate()
        cond.deallocate()
    }
    
    open func lock() {
        pthread_mutex_lock(mutex)
    }
    
    open func unlock() {
        pthread_mutex_unlock(mutex)
    }
    // 挂起当前线程等待唤醒信号
    open func wait() {
        pthread_cond_wait(cond, mutex)
    }

    open func wait(until limit: Date) -> Bool {
        guard var timeout = timeSpecFrom(date: limit) else {
            return false
        }
        return pthread_cond_timedwait(cond, mutex, &timeout) == 0
    }
    // 发送条件信号，唤醒线程
    open func signal() {
        pthread_cond_signal(cond)
    }
    // 发生广播条件信号，唤醒所有在等待的线程
    open func broadcast() {
        pthread_cond_broadcast(cond)
    }
    
    open var name: String?
}
```
从代码来看，`NSCondition` 底层也是对 `pthread_mutex` 的封装，另外增加了条件变量 `cond`，其是一个 `pthread_cond_t` 类型的变量，用于访问和操作特定类型数据的指针。

其中 `wait` 操作会阻塞线程，使线程进入休眠状态，如果超时则返回 `false`。 `signal` 操作会唤醒一个正在休眠等待的线程。而 `broadcast` 会唤醒所有正在等待的线程。


**6. NSConditionLock 条件锁**

```
open class NSConditionLock : NSObject, NSLocking {
    internal var _cond = NSCondition()
    internal var _value: Int
    // 当前线程
    internal var _thread: _swift_CFThreadRef?
    
    public convenience override init() {
        self.init(condition: 0)
    }
    
    public init(condition: Int) {
        _value = condition
    }

    open func lock() {
        let _ = lock(before: Date.distantFuture)
    }

    open func unlock() {
        _cond.lock()
        _thread = nil
        _cond.broadcast()
        _cond.unlock()
    }
    
    open var condition: Int {
        return _value
    }
    // 只有满足条件变量等于指定值 condition，才能获取锁，一直等待
    open func lock(whenCondition condition: Int) {
        let _ = lock(whenCondition: condition, before: Date.distantFuture)
    }
    // 尝试加锁，返回是否加锁成功，不阻塞线程
    open func `try`() -> Bool {
        return lock(before: Date.distantPast)
    }
    // 满足条件变量等于指定值，尝试加锁，返回是否加锁成功
    open func tryLock(whenCondition condition: Int) -> Bool {
        return lock(whenCondition: condition, before: Date.distantPast)
    }

    // 解锁，并设置条件变量为指定值 condition
    open func unlock(withCondition condition: Int) {
        _cond.lock()
        _thread = nil
        _value = condition
        _cond.broadcast()
        _cond.unlock()
    }
    // 在指定时间之前一直尝试加锁，返回是否加锁成功
    open func lock(before limit: Date) -> Bool {
        _cond.lock()
        while _thread != nil {
            if !_cond.wait(until: limit) {
                _cond.unlock()
                return false
            }
        }
        _thread = pthread_self()
        _cond.unlock()
        return true
    }
    // 满足条件变量为指定值，且在指定时间之前一直尝试加锁，返回是否加锁成功
    open func lock(whenCondition condition: Int, before limit: Date) -> Bool {
        _cond.lock()
        while _thread != nil || _value != condition {
            if !_cond.wait(until: limit) {
                _cond.unlock()
                return false
            }
        }
        _thread = pthread_self()
        _cond.unlock()
        return true
    }
    
    open var name: String?
}
```
可以看到，`NSConditionLock` 是对 `NSCondition` 的进一步封装，增加了 `Int` 类型的 `_value` 条件变量，可以使用 `NSConditionLock` 来实现任务之间的依赖。

**7. dispatch_semaphore 信号量**

`dispatch_semaphore` 常用于保证资源的多线程安全。其本质是基于内核的信号量接口来实现的，是一种基于计数器的多线程同步机制。

信号量（ `semaphore` ）是 `PV` 操作的载体，基本操作有四种：初始化、等信号、给信号、清理（**备注：** ARC 下，`semaphore` 的清理工作由 `GCD` 在底层自动完成）。对应的 `dispatch_semaphore` 接口定义在 `libdispatch` 源码的 `semaphore.h` 头文件里面，我们可以看到有如下三个方法定义：

```
// 创建一个信号量，初始值为 value，如果 value 小于0，则返回NULL
dispatch_semaphore_t dispatch_semaphore_create(long value);

// 等待减少信号量，dsema 为目标信号量对象，即 dsema 的信号量值减一，如果 dsema 的信号量 value 为0，则线程会被阻塞，直到 value 大于0或者 timeout
long dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout);

// 传入的信号量对象 dsema 的 value 值加1
long dispatch_semaphore_signal(dispatch_semaphore_t dsema);

```

下面跟着源码详细介绍一下每个函数具体的实现：

1. `dispatch_semaphore_create` :
```
dispatch_semaphore_t
dispatch_semaphore_create(long value)
{
	dispatch_semaphore_t dsema;
    // 如果 value 值小于 0，则直接返回 DISPATCH_BAD_INPUT
	if (value < 0) {
        // #define DISPATCH_BAD_INPUT   ((void *_Nonnull)0)
		return DISPATCH_BAD_INPUT;
	}

	dsema = _dispatch_object_alloc(DISPATCH_VTABLE(semaphore),
			sizeof(struct dispatch_semaphore_s));
	dsema->do_next = DISPATCH_OBJECT_LISTLESS;
    // 目标队列
	dsema->do_targetq = _dispatch_get_default_queue(false);
    // 当前值
	dsema->dsema_value = value;
	_dispatch_sema4_init(&dsema->dsema_sema, _DSEMA4_POLICY_FIFO);
    // 初始值
	dsema->dsema_orig = value;
	return dsema;
}
```
可以看到如果 `value` 值小于 0 则会直接返回 `NULL`。传递一个大于零的值会返回一个 `dispatch_semaphore_t` 类型的信号量对象 `dsema` 。

2. `dispatch_semaphore_wait` :
```
long
dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout)
{
    // 原子性操作信号量 dsema_value 减 1 
	long value = os_atomic_dec2o(dsema, dsema_value, acquire);
    // 如果 value 大于等于0，则直接 return
	if (likely(value >= 0)) {
		return 0;
	}
    // 如果 value 小于 0，则调用 _dispatch_semaphore_wait_slow 函数进行阻塞，等待信号量唤醒或者超时
	return _dispatch_semaphore_wait_slow(dsema, timeout);
}

static long
_dispatch_semaphore_wait_slow(dispatch_semaphore_t dsema,
		dispatch_time_t timeout)
{
	long orig;

	_dispatch_sema4_create(&dsema->dsema_sema, _DSEMA4_POLICY_FIFO);
	switch (timeout) {
	default:
        // timeout 是一个特定时间会调用 _dispatch_sema4_timedwait 进行 timeout 时间的等待
		if (!_dispatch_sema4_timedwait(&dsema->dsema_sema, timeout)) {
			break;
		}
		// Fall through and try to undo what the fast path did to
		// dsema->dsema_value
	case DISPATCH_TIME_NOW:
		orig = dsema->dsema_value;
		while (orig < 0) {
            // dsema_value 加 1 抵消掉 dispatch_semaphore_wait 函数中的减 1 操作
			if (os_atomic_cmpxchgvw2o(dsema, dsema_value, orig, orig + 1,
					&orig, relaxed)) {
                // 返回超时
				return _DSEMA4_TIMEOUT();
			}
		}
		// Another thread called semaphore_signal().
		// Fall through and drain the wakeup.
	case DISPATCH_TIME_FOREVER:
        // _dispatch_sema4_wait 里面是一个 do-while 循环，当不满足条件时，会一直循环下去，从而导致流程的阻塞
		_dispatch_sema4_wait(&dsema->dsema_sema);
		break;
	}
	return 0;
}
```
等待减少信号量（ `dsema` 为目标信号量对象），即 `dsema` 的信号量值减一，如果信号量 `value` 为 0，则线程会被阻塞，直到 `value` 大于0或者超时。

`timeout` 有 `DISPATCH_TIME_NOW` 和 `DISPATCH_TIME_FOREVER` ，也可以通过 `dispatch_time(dispatch_time_t when, int64_t delta)` 自定义时长。如果返回 0，说明 `P` 操作成功，如果返回非 0，说明 `P` 操作失败，执行超时。


3. `dispatch_semaphore_signal` :
```
long dispatch_semaphore_signal(dispatch_semaphore_t dsema)
{
    // 原子性操作信号量 dsema_value 加1
	long value = os_atomic_inc2o(dsema, dsema_value, release);
    // value 大于 0，有资源可用，直接返回 0
	if (likely(value > 0)) {
		return 0;
	}
    // 过度操作导致signal信号不平衡，抛出 CRASH 异常
	if (unlikely(value == LONG_MIN)) {
		DISPATCH_CLIENT_CRASH(value,
				"Unbalanced call to dispatch_semaphore_signal()");
	}
    // value 小于等于 0 时，表示目前有线程需要唤醒
	return _dispatch_semaphore_signal_slow(dsema);
}

long
_dispatch_semaphore_signal_slow(dispatch_semaphore_t dsema)
{
    // 如果 &dsema->dsema_sema 为 NULL，则调用 _dispatch_sema4_create_slow 为 &dsema->dsema_sema 赋值
	_dispatch_sema4_create(&dsema->dsema_sema, _DSEMA4_POLICY_FIFO);

    // count 传 1，唤醒一条线程，如果有多个等待线程，则根据线程优先级来唤醒
	_dispatch_sema4_signal(&dsema->dsema_sema, 1);
	return 1;
}
```
对应信号量的 `V` 操作，会将 `dsema` 的 `value` 值 +1，如果 `value` 大于 0 直接返回 0，否则进入 `_dispatch_semaphore_signal_slow` 方法，该函数会调用内核的 `semaphore_signal` 函数唤醒等待中的线程。

`dispatch_semaphore` 常用于实现功能：

> * 用于保持线程同步，将异步执行任务转换为同步执行任务
> * 保证线程安全，为线程加锁，相当于自旋锁

值得注意的是，信号量运行效率比自旋锁略低，相对其他锁比较高。因此，在性能要求比较高的场景，信号量是一个优先级比较高的选择。


**8. OSSpinLock 自旋锁**

由于 `OSSpinLock` 不再安全，主要原因发生在低优先级线程拿到锁时，高优先级线程进入忙等 `busy-wait` 状态，占用大量 `CPU` 时间片。从而导致低优先级线程拿不到 `CPU` 时间片，无法完成任务并释放锁，这就造成了任务的优先级反转。

从 iOS 10/macOS 10.12 开始 `OSSpinLock` 被弃用，其替代方案是内部封装了 `os_unfair_lock`，而 `os_unfair_lock` 在加锁时会处于休眠状态，而不是自旋锁的忙等状态。

---

### **性能对比**

最后，借用一下YY大神的性能测试结果图，对比一下各种锁的性能：

![](https://github.com/minhechen/iOSTechTeam/blob/main/Blogs/resource/iOSTechTeam_04/lock_benchmark.png)

---
### **拓展知识**

1. 什么是死锁？

当两个以上的运算单元，双方都在等待对方停止执行，以获得系统资源，但是没有一方提前退出时，就称为死锁。

2. 什么是 `PV` 操作？

`PV` 操作是一种实现进程互斥与同步的有效方法。`PV` 操作与信号量的处理相关，具体定义如下： 
* P(S)： ①将信号量 S 的值减 1，即S = S - 1； ②如果 S > 0，则该进程继续执行；否则该进程置为等待状态，排入等待队列。

* V(S)： ①将信号量 S 的值加 1，即 S = S + 1；


---
### **总结**

同步工具是使代码线程安全的一种行之有效的方法，但它们不是万能的。与非线程安全性能相比，锁和其他类型的同步原语使用过多，实际上会降低应用程序的线程性能。在安全性和性能之间找到正确的平衡是一门需要经验的艺术。因此我们在实际开发过程中，需要根据具体需要选择合适的实现方案，已达到性能与安全兼顾的最优解。以上就是本文对 iOS 锁相关知识点的介绍，感谢阅读。

---
### **参考资料：**

* [Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html) ( https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html )

* [More than you want to know about @synchronized](https://www.rykap.com/objective-c/2015/05/09/synchronized/) ( https://www.rykap.com/objective-c/2015/05/09/synchronized/ )

* [互斥锁属性](https://docs.oracle.com/cd/E19253-01/819-7051/6n919hpaf/index.html#sync-26886) ( https://docs.oracle.com/cd/E19253-01/819-7051/6n919hpaf/index.html#sync-26886 )

* [不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/) (https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

---
### **关于技术组**
iOS 技术组主要用来学习、分享日常开发中使用到的技术，一起保持学习，保持进步。文章仓库在这里：https://github.com/minhechen/iOSTechTeam 微信公众号：iOS技术组，欢迎联系进群学习交流，感谢阅读。
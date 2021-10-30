##  4. Pooled 池化内存分配

### 1. PooledByteBufAllocator 简述

现在开始, 我们来分析Pooled 池化内存的分配原理, 我们首先还是先找到 AbstractByteBufAllocator 的子类 PooledByteBufAllocator  实现的分配内存的两个方法. newDirectBuffer()方法和 newHeapBuffer()方法：

```java
@Override
    protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();
        PoolArena<byte[]> heapArena = cache.heapArena;

        ByteBuf buf;
        if (heapArena != null) {
            buf = heapArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            buf = new UnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
        }

        return toLeakAwareBuffer(buf);
    }

    @Override
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();
        PoolArena<ByteBuffer> directArena = cache.directArena;

        ByteBuf buf;
        if (directArena != null) {
            buf = directArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            if (PlatformDependent.hasUnsafe()) {
                buf = UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
            } else {
                buf = new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
            }
        }

        return toLeakAwareBuffer(buf);
    }
```

我们发现这两个方法大体结构上是一样的. 我们以newDirectBuffer 为例简单的分析一下：

```java
    @Override
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();
        PoolArena<ByteBuffer> directArena = cache.directArena;

        ByteBuf buf;
        if (directArena != null) {
            buf = directArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            if (PlatformDependent.hasUnsafe()) {
                buf = UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
            } else {
                buf = new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
            }
        }

        return toLeakAwareBuffer(buf);
    }
```

首先,我们看到它是通过 threadCache.get() 拿到一个类型为PoolThreadCache 的cache 对象, 然后, 通过  cache 拿到 PoolArena 对象, 最终会调用 directArena.allocate()  方法分配 ByteBuf. 这个地方大家可能会看的比较懵, 我们来详细分析一下，我们跟进到  threadCache  对象其实是PoolThreadLocalCache 类型的变量, 跟进到 PoolThreadLocalCache 的源码:

```java
final class PoolThreadLocalCache extends FastThreadLocal<PoolThreadCache> {

        @Override
        protected synchronized PoolThreadCache initialValue() {
            final PoolArena<byte[]> heapArena = leastUsedArena(heapArenas);
            final PoolArena<ByteBuffer> directArena = leastUsedArena(directArenas);

            return new PoolThreadCache(
                    heapArena, directArena, tinyCacheSize, smallCacheSize, normalCacheSize,
                    DEFAULT_MAX_CACHED_BUFFER_CAPACITY, DEFAULT_CACHE_TRIM_INTERVAL);
        }

        @Override
        protected void onRemoval(PoolThreadCache threadCache) {
            threadCache.free();
        }

        private <T> PoolArena<T> leastUsedArena(PoolArena<T>[] arenas) {
            if (arenas == null || arenas.length == 0) {
                return null;
            }

            PoolArena<T> minArena = arenas[0];
            for (int i = 1; i < arenas.length; i++) {
                PoolArena<T> arena = arenas[i];
                if (arena.numThreadCaches.get() < minArena.numThreadCaches.get()) {
                    minArena = arena;
                }
            }

            return minArena;
        }
    }
```



从名字上来看, 我们发现PoolThreadLocalCache 的initialValue()  方法就是用来初始化PoolThreadLocalCache  的, 首先就调用了leastUsedArena() 方法分别获取类型为 PoolArena 的 heapArena 对象和directArena 对象. 然后把heapArena 对象和directArena 对象 当做参数出传递到了 PoolThreadLocalCache  的构造器中, 那么  heapArena 对象和directArena 对象 是在哪里被初始化的呢? 我们查找一下发现在PooledByteBufAllocator 的 构造器中调用 newArenaArray() 方法给heapArenas 和directArenas 赋值了。

```java
   public PooledByteBufAllocator(boolean preferDirect, int nHeapArena, int nDirectArena, int pageSize, int maxOrder,
                                  int tinyCacheSize, int smallCacheSize, int normalCacheSize) {
        super(preferDirect);
        threadCache = new PoolThreadLocalCache();
        this.tinyCacheSize = tinyCacheSize;
        this.smallCacheSize = smallCacheSize;
        this.normalCacheSize = normalCacheSize;
        final int chunkSize = validateAndCalculateChunkSize(pageSize, maxOrder);

        if (nHeapArena < 0) {
            throw new IllegalArgumentException("nHeapArena: " + nHeapArena + " (expected: >= 0)");
        }
        if (nDirectArena < 0) {
            throw new IllegalArgumentException("nDirectArea: " + nDirectArena + " (expected: >= 0)");
        }

        int pageShifts = validateAndCalculatePageShifts(pageSize);

        if (nHeapArena > 0) {
            heapArenas = newArenaArray(nHeapArena);
            List<PoolArenaMetric> metrics = new ArrayList<PoolArenaMetric>(heapArenas.length);
            for (int i = 0; i < heapArenas.length; i ++) {
                PoolArena.HeapArena arena = new PoolArena.HeapArena(this, pageSize, maxOrder, pageShifts, chunkSize);
                heapArenas[i] = arena;
                metrics.add(arena);
            }
            heapArenaMetrics = Collections.unmodifiableList(metrics);
        } else {
            heapArenas = null;
            heapArenaMetrics = Collections.emptyList();
        }

        if (nDirectArena > 0) {
            directArenas = newArenaArray(nDirectArena);
            List<PoolArenaMetric> metrics = new ArrayList<PoolArenaMetric>(directArenas.length);
            for (int i = 0; i < directArenas.length; i ++) {
                PoolArena.DirectArena arena = new PoolArena.DirectArena(
                        this, pageSize, maxOrder, pageShifts, chunkSize);
                directArenas[i] = arena;
                metrics.add(arena);
            }
            directArenaMetrics = Collections.unmodifiableList(metrics);
        } else {
            directArenas = null;
            directArenaMetrics = Collections.emptyList();
        }
    }
```



进去到newArenaArray() 方法:

```java
 private static <T> PoolArena<T>[] newArenaArray(int size) {
        return new PoolArena[size];
    }

```

其实就是创建一个固定大小的 PoolArena 数组, 数组大小由传入的参数 数 nHeapArena 和 nDirectArena 来决定, 我们再回到PooledByteBufAllocator 的构造器源码, 看 nHeapArena 和 nDirectArena 是怎么初始化的, 我们找到了PooledByteBufAllocator 的重载构造器:

```java
   public PooledByteBufAllocator(boolean preferDirect) {
        this(preferDirect, DEFAULT_NUM_HEAP_ARENA, DEFAULT_NUM_DIRECT_ARENA, DEFAULT_PAGE_SIZE, DEFAULT_MAX_ORDER);
    }
```

我们发现, nHeapArena 和 nDirectArena  是通过  DEFAULT_NUM_HEAP_ARENA, DEFAULT_NUM_DIRECT_ARENA 这两个常量默认赋值的, 再继续跟进常量的定义:

```java
  DEFAULT_NUM_HEAP_ARENA = Math.max(0,
                SystemPropertyUtil.getInt(
                        "io.netty.allocator.numHeapArenas",
                        (int) Math.min(
                                defaultMinNumArena,
                                runtime.maxMemory() / defaultChunkSize / 2 / 3)));
        DEFAULT_NUM_DIRECT_ARENA = Math.max(0,
                SystemPropertyUtil.getInt(
                        "io.netty.allocator.numDirectArenas",
                        (int) Math.min(
                                defaultMinNumArena,
                                PlatformDependent.maxDirectMemory() / defaultChunkSize / 2 / 3)));
```

到这里为止 , 我们才知道 nHeapArena 和 nDirectArena 的默认赋值, 默认是分配CPU 核数*2, 也就是把 defaultMinNumArena的值赋值给 nHeapArena 和 nDirectArena . 那么Netty 为什么这样设计呢? 其实主要目的就是为了保证Netty 中的每一个线程任务都可以有一个独享的 Arena, 保证在每个线程分配内存的是是不用加锁的.

基于上面的分析, 我们知道 Arena 有HeapArena 和 DirectArena , 这里我们统称为Arena . 假设我们有四个线程, 那么对应会分配四个 Arena , 在创建ByteBuf 的时候, 首先通过 PoolThreadCache 获取Arena  对象并 赋值给其成员变量, 然后, 每个线程通过PoolThreadCache  调用get 方法的时候会拿到它底层的Arena , 也就是说 EventLoop1 拿到 Arena1,  EventLoop2拿到 Arena2, 以此类推, 如下图所示：

 ![](http://files.luyanan.com//img/20191007173949.png)

那么PoolThreadCache 除了可以只在Arena  上进行内存分配，还可以在他底层维护ByteBuf 缓存列表进行分配.举个例子: 我们通过PooledByteBufAllocator 去创建了一个1024字节大小的ByteBuf, 当我们用完释放之后, 我们可能会在其他地方继续分配1024 字节大小的ByteBuf. 这个时候, 其实不需要要在 Arena  上进行内存分配, 而是直接通过 PoolThreadCache 中维护的ByteBuf 的缓存列表直接拿过来返回, 那么, 在  PooledByteBufAllocator 中维护三种规定大小的缓存列表, 分别是三个值 tinyCacheSize、smallCacheSize、normalCacheSize：

```java
public class PooledByteBufAllocator extends AbstractByteBufAllocator {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(PooledByteBufAllocator.class);
    private static final int DEFAULT_NUM_HEAP_ARENA;
    private static final int DEFAULT_NUM_DIRECT_ARENA;

    private static final int DEFAULT_PAGE_SIZE;
    private static final int DEFAULT_MAX_ORDER; // 8192 << 11 = 16 MiB per chunk
    private static final int DEFAULT_TINY_CACHE_SIZE;
    private static final int DEFAULT_SMALL_CACHE_SIZE;
    private static final int DEFAULT_NORMAL_CACHE_SIZE;
    private static final int DEFAULT_MAX_CACHED_BUFFER_CAPACITY;
    private static final int DEFAULT_CACHE_TRIM_INTERVAL;

    private static final int MIN_PAGE_SIZE = 4096;
    private static final int MAX_CHUNK_SIZE = (int) (((long) Integer.MAX_VALUE + 1) / 2);

    static {
        int defaultPageSize = SystemPropertyUtil.getInt("io.netty.allocator.pageSize", 8192);
        Throwable pageSizeFallbackCause = null;
        try {
            validateAndCalculatePageShifts(defaultPageSize);
        } catch (Throwable t) {
            pageSizeFallbackCause = t;
            defaultPageSize = 8192;
        }
        DEFAULT_PAGE_SIZE = defaultPageSize;

        int defaultMaxOrder = SystemPropertyUtil.getInt("io.netty.allocator.maxOrder", 11);
        Throwable maxOrderFallbackCause = null;
        try {
            validateAndCalculateChunkSize(DEFAULT_PAGE_SIZE, defaultMaxOrder);
        } catch (Throwable t) {
            maxOrderFallbackCause = t;
            defaultMaxOrder = 11;
        }
        DEFAULT_MAX_ORDER = defaultMaxOrder;

        // Determine reasonable default for nHeapArena and nDirectArena.
        // Assuming each arena has 3 chunks, the pool should not consume more than 50% of max memory.
        final Runtime runtime = Runtime.getRuntime();

        // Use 2 * cores by default to reduce condition as we use 2 * cores for the number of EventLoops
        // in NIO and EPOLL as well. If we choose a smaller number we will run into hotspots as allocation and
        // deallocation needs to be synchronized on the PoolArena.
        // See https://github.com/netty/netty/issues/3888
        final int defaultMinNumArena = runtime.availableProcessors() * 2;
        final int defaultChunkSize = DEFAULT_PAGE_SIZE << DEFAULT_MAX_ORDER;
        DEFAULT_NUM_HEAP_ARENA = Math.max(0,
                SystemPropertyUtil.getInt(
                        "io.netty.allocator.numHeapArenas",
                        (int) Math.min(
                                defaultMinNumArena,
                                runtime.maxMemory() / defaultChunkSize / 2 / 3)));
        DEFAULT_NUM_DIRECT_ARENA = Math.max(0,
                SystemPropertyUtil.getInt(
                        "io.netty.allocator.numDirectArenas",
                        (int) Math.min(
                                defaultMinNumArena,
                                PlatformDependent.maxDirectMemory() / defaultChunkSize / 2 / 3)));

        // cache sizes
        DEFAULT_TINY_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.tinyCacheSize", 512);
        DEFAULT_SMALL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.smallCacheSize", 256);
        DEFAULT_NORMAL_CACHE_SIZE = SystemPropertyUtil.getInt("io.netty.allocator.normalCacheSize", 64);

        // 32 kb is the default maximum capacity of the cached buffer. Similar to what is explained in
        // 'Scalable memory allocation using jemalloc'
        DEFAULT_MAX_CACHED_BUFFER_CAPACITY = SystemPropertyUtil.getInt(
                "io.netty.allocator.maxCachedBufferCapacity", 32 * 1024);

        // the number of threshold of allocations when cached entries will be freed up if not frequently used
        DEFAULT_CACHE_TRIM_INTERVAL = SystemPropertyUtil.getInt(
                "io.netty.allocator.cacheTrimInterval", 8192);

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.allocator.numHeapArenas: {}", DEFAULT_NUM_HEAP_ARENA);
            logger.debug("-Dio.netty.allocator.numDirectArenas: {}", DEFAULT_NUM_DIRECT_ARENA);
            if (pageSizeFallbackCause == null) {
                logger.debug("-Dio.netty.allocator.pageSize: {}", DEFAULT_PAGE_SIZE);
            } else {
                logger.debug("-Dio.netty.allocator.pageSize: {}", DEFAULT_PAGE_SIZE, pageSizeFallbackCause);
            }
            if (maxOrderFallbackCause == null) {
                logger.debug("-Dio.netty.allocator.maxOrder: {}", DEFAULT_MAX_ORDER);
            } else {
                logger.debug("-Dio.netty.allocator.maxOrder: {}", DEFAULT_MAX_ORDER, maxOrderFallbackCause);
            }
            logger.debug("-Dio.netty.allocator.chunkSize: {}", DEFAULT_PAGE_SIZE << DEFAULT_MAX_ORDER);
            logger.debug("-Dio.netty.allocator.tinyCacheSize: {}", DEFAULT_TINY_CACHE_SIZE);
            logger.debug("-Dio.netty.allocator.smallCacheSize: {}", DEFAULT_SMALL_CACHE_SIZE);
            logger.debug("-Dio.netty.allocator.normalCacheSize: {}", DEFAULT_NORMAL_CACHE_SIZE);
            logger.debug("-Dio.netty.allocator.maxCachedBufferCapacity: {}", DEFAULT_MAX_CACHED_BUFFER_CAPACITY);
            logger.debug("-Dio.netty.allocator.cacheTrimInterval: {}", DEFAULT_CACHE_TRIM_INTERVAL);
        }
    }

    public static final PooledByteBufAllocator DEFAULT =
            new PooledByteBufAllocator(PlatformDependent.directBufferPreferred());

    private final PoolArena<byte[]>[] heapArenas;
    private final PoolArena<ByteBuffer>[] directArenas;
    private final int tinyCacheSize;
    private final int smallCacheSize;
    private final int normalCacheSize;
    private final List<PoolArenaMetric> heapArenaMetrics;
    private final List<PoolArenaMetric> directArenaMetrics;
    private final PoolThreadLocalCache threadCache;

    public PooledByteBufAllocator() {
        this(false);
    }

    public PooledByteBufAllocator(boolean preferDirect) {
        this(preferDirect, DEFAULT_NUM_HEAP_ARENA, DEFAULT_NUM_DIRECT_ARENA, DEFAULT_PAGE_SIZE, DEFAULT_MAX_ORDER);
    }

    public PooledByteBufAllocator(int nHeapArena, int nDirectArena, int pageSize, int maxOrder) {
        this(false, nHeapArena, nDirectArena, pageSize, maxOrder);
    }

    public PooledByteBufAllocator(boolean preferDirect, int nHeapArena, int nDirectArena, int pageSize, int maxOrder) {
        this(preferDirect, nHeapArena, nDirectArena, pageSize, maxOrder,
                DEFAULT_TINY_CACHE_SIZE, DEFAULT_SMALL_CACHE_SIZE, DEFAULT_NORMAL_CACHE_SIZE);
    }
 ..............   
}
```



我们看到, 在 PooledByteBufAllocator 的构造器中, 分 别 赋 值 了 tinyCacheSize=512 ， smallCacheSize=256 ，
normalCacheSize=64。通过这样一种方式，Netty 中给我们预创建固定规格的内存池，大大提高了内存分配的性能。

### 2. DirectArena 内存分配流程

Arena 分配内存的基本流程有三个步骤:

1. 从对象池中拿到 PooledByteBuf 进行复用
2. 从缓存中进行内存分配
3. 从内存堆中进行内存分配

我们以directBuff 为例, 首先来看从对象池中拿到 PooledByteBuf  对象进行复用的情况, 我们依旧跟进到 PooledByteBufAllocator 的 newDirectBuffer()方法

```java
 @Override
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();
        PoolArena<ByteBuffer> directArena = cache.directArena;

        ByteBuf buf;
        if (directArena != null) {
            buf = directArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            if (PlatformDependent.hasUnsafe()) {
                buf = UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
            } else {
                buf = new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
            }
        }

        return toLeakAwareBuffer(buf);
    }
```



上面的PoolArena 我们已经清楚, 现在我们直接跟进到 PoolArena 的allocate()  方法:

```java
    PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
        PooledByteBuf<T> buf = newByteBuf(maxCapacity);
        allocate(cache, buf, reqCapacity);
        return buf;
    }

```



在这个地方其实就非常清晰了, 首先调用 newByteBuf()  方法拿到一个PooledByteBuf 对象, 接下来通过 allocate()  方法在线程私有的 PoolThreadCache 中分配一块内存, 然后buf 里面的内存地址之类的值进行初始化,我们跟进到  newByteBuf()  看一下, 选择 DirectArena 对象：

```java
     @Override
        protected PooledByteBuf<ByteBuffer> newByteBuf(int maxCapacity) {
            if (HAS_UNSAFE) {
                return PooledUnsafeDirectByteBuf.newInstance(maxCapacity);
            } else {
                return PooledDirectByteBuf.newInstance(maxCapacity);
            }
        }
```



我们发现首先判断是否支持 Unsafe, 默认情况下一般是支持 Unsafe 的, 虽然我们继续进入到 PooledUnsafeDirectByteBuf 的 newInstance()方法：

```java
final class PooledUnsafeDirectByteBuf extends PooledByteBuf<ByteBuffer> {
    private static final Recycler<PooledUnsafeDirectByteBuf> RECYCLER = new Recycler<PooledUnsafeDirectByteBuf>() {
        @Override
        protected PooledUnsafeDirectByteBuf newObject(Handle<PooledUnsafeDirectByteBuf> handle) {
            return new PooledUnsafeDirectByteBuf(handle, 0);
        }
    };

    static PooledUnsafeDirectByteBuf newInstance(int maxCapacity) {
        PooledUnsafeDirectByteBuf buf = RECYCLER.get();
        buf.reuse(maxCapacity);
        return buf;
    }
    
}
```

顾名思义, 我看到首先就是从 RECYCLER (也就是内存回收站) 对象的 get() 方法中拿到一个buf, 从上面的代码片段来看, RECYCLER 对象实现了一个newObject() 方法, 当回收站里面没有可用的buf时就会去创建一个新的buf. 因为创建的buf 可能是从回收站拿出来的 , 复用前需要重置,因此继续往下看就会调用buf 的reuse()  方法.

```java
   /**
     * Method must be called before reuse this {@link PooledByteBufAllocator}
     */
    final void reuse(int maxCapacity) {
        maxCapacity(maxCapacity);
        setRefCnt(1);
        setIndex0(0, 0);
        discardMarks();
    }
```

我们发现 reuse() 就是对所有的参数重新归为初始状态, 到这里我们已经清楚从内存池获取buf 对象的全过程. 那么接下来, 再回到PoolArena 的 allocate()方法. 看看真实的内存是如何分配出来的, buf 的内存分配主要有两种情况: 分别是:从缓存中进行内存分配和从内存堆中进行内存分配, 我们来看代码, 进入 allocate() 方法具体逻辑：

```java
 private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
        final int normCapacity = normalizeCapacity(reqCapacity);
        if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
            int tableIdx;
            PoolSubpage<T>[] table;
            boolean tiny = isTiny(normCapacity);
            if (tiny) { // < 512
                if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = tinyIdx(normCapacity);
                table = tinySubpagePools;
            } else {
                if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = smallIdx(normCapacity);
                table = smallSubpagePools;
            }

            final PoolSubpage<T> head = table[tableIdx];

            /**
             * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
             * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
             */
            synchronized (head) {
                final PoolSubpage<T> s = head.next;
                if (s != head) {
                    assert s.doNotDestroy && s.elemSize == normCapacity;
                    long handle = s.allocate();
                    assert handle >= 0;
                    s.chunk.initBufWithSubpage(buf, handle, reqCapacity);

                    if (tiny) {
                        allocationsTiny.increment();
                    } else {
                        allocationsSmall.increment();
                    }
                    return;
                }
            }
            allocateNormal(buf, reqCapacity, normCapacity);
            return;
        }
        if (normCapacity <= chunkSize) {
            if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            allocateNormal(buf, reqCapacity, normCapacity);
        } else {
            // Huge allocations are never served via the cache so just call allocateHuge
            allocateHuge(buf, reqCapacity);
        }
    }
```

这段代码逻辑看起来非常复杂, 其实大部分的逻辑基本上都是判断不同的规格大小, 从其对应的缓存中获取内存, 如果所有的规格都不满足, 那就直接调用allocateHuge()  方法进行真实的内存分配.

### 3. 内存池的内存规格

在前面的源码分析中, 关于内存规格大小我们应该还有些印象, 其实在Netty 内存池中主要设置了四种规格大小的内存: 

- tiny 是指0-512byte 之间的规格大小

- small 是指512byte - 8kb 之间的规格大小

- normal 是指8kb - 16MB 之间的规格大小

- buge 是指16MB 以上的

  为什么Netty 会选择这些值作为一个分界点呢? 其实在Netty 底层还有一个内存单位的封装. 为了更高效的管理内存, 避免内存浪费, 把每一个区间的内存规格都做了细分. 默认情况下, Netty 将内存规格分为四个部分. Netty 中所有的内存申请是以Chunk 为单位向内存申请的, 大小为16M， 后续的内存分配都是在这个Chunk里面的操作. 8KB 对应的是一个Page, 一个Chunk 会以Page 为单位进行切分. 8KB 对应的Chunk 被划分为2048个Page. 小于8K的对应的是SubPage. 例如: 我们申请的是一段内存空间只有1K , 却给我们分配了一个Page, 显然另外7K 就会被浪费, 所以继续把Page 进行划分, 节省空间 . 如下图所示:

  ![](http://files.luyanan.com//img/20191008220148.png)

至此, 小伙伴应该已经非常清楚Netty的内存池缓存管理机制了. 

### 4. 命中缓存的分配

前面我们简单分析了directArena 内容分配大概流程, 知道其先命中缓存, 如果命中不到, 则去分配一款连续内存. 现在带大家剖析命中缓存的相关逻辑. 前面我们也讲到 PoolThreadCache 中维护了三个缓存数组(实际上是六个, 这
里 仅 仅 以 Direct 为 例 , Heap 类 型 的 逻 辑 是 一 样 的 ): tinySubPageDirectCaches, smallSubPageDirectCaches, 和
normalDirectCaches 分别代表 tiny 类型, small 类型和 normal 类型的缓存数组）。这三个数组保存在 PoolThreadCache
的成员变量中:

```java
final class PoolThreadCache {

   ...
    private final MemoryRegionCache<ByteBuffer>[] tinySubPageDirectCaches;
    private final MemoryRegionCache<ByteBuffer>[] smallSubPageDirectCaches;
    private final MemoryRegionCache<ByteBuffer>[] normalDirectCaches;
    ...
}
```

其实是在构造方法中进行了初始化:

```java
PoolThreadCache(PoolArena<byte[]> heapArena, PoolArena<ByteBuffer> directArena,
                    int tinyCacheSize, int smallCacheSize, int normalCacheSize,
                    int maxCachedBufferCapacity, int freeSweepAllocationThreshold) {
      ...
        if (directArena != null) {
            tinySubPageDirectCaches = createSubPageCaches(
                    tinyCacheSize, PoolArena.numTinySubpagePools, SizeClass.Tiny);
            smallSubPageDirectCaches = createSubPageCaches(
                    smallCacheSize, directArena.numSmallSubpagePools, SizeClass.Small);

            numShiftsNormalDirect = log2(directArena.pageSize);
            normalDirectCaches = createNormalCaches(
                    normalCacheSize, maxCachedBufferCapacity, directArena);

            directArena.numThreadCaches.getAndIncrement();
        } 
    ...
    }
```

我们以tiny 类型为例 跟到createSubPageCaches 方法中:

```java
 private static <T> MemoryRegionCache<T>[] createSubPageCaches(
            int cacheSize, int numCaches, SizeClass sizeClass) {
        if (cacheSize > 0) {
            @SuppressWarnings("unchecked")
            MemoryRegionCache<T>[] cache = new MemoryRegionCache[numCaches];
            for (int i = 0; i < cache.length; i++) {
                // TODO: maybe use cacheSize / cache.length
                cache[i] = new SubPageMemoryRegionCache<T>(cacheSize, sizeClass);
            }
            return cache;
        } else {
            return null;
        }
    }
```

从代码来看, 其实就是创建了一个缓存数组, 这个缓存数组的长度, 也就是 numCaches, 在不同的类型, 这个长度不一样 . tiny类型长度是32,small 类型长度为4, normal 类型长度为3.我们知道， 缓存数组中每个节点代表一个缓存对象, 里面维护一个队列. 队列大小由PooledByteBufAllocator 类 中 的 tinyCacheSize, smallCacheSize, normalCacheSize 属性决定的。其中每个缓存对象, 队列中缓存的ByteBuf 大小的固定的. netty 将每种缓存区类型分为了不同的长度规格 ,而每个缓存中的队列缓存的Bytebuf的长度, 都是同一个规格的长度, 而缓冲区数组的长度, 就是规格的数量. 比如: 在 tiny 类型中, Netty将其长度为了32个规格, 每个规格都是16的整数倍, 也就是包含0Byte、16Byte、32Byte、48Byte、64Byte，96Byte...总共32种规格. 而在其缓存数组tinySubPageDirectCaches 中, 这每一种规格代表数组中的一个缓存对象缓存的ByteBuf的大小, 我们以tinySubPageDirectCaches[1]为例,(这里下标选择1是因为下标为0的规格是0Byte, 其实就代表一个空的缓存, 这里不进行举例). 在  `tinySubPageDirectCaches[1]`  的缓存对象中所缓存的ByteBuf的缓存长度是16Byte, tinySubPageDirectCaches[2] 中缓存的ByteBuf 长度为32Byte. 以此类推, tinySubPageDirectCaches[31]中缓存的 ByteBuf 长度为 496Byte。其具体类型规则的配置如下:
tiny:总共 32 个规格, 均是 16 的整数倍, 0Byte, 16Byte, 32Byte, 48Byte, 64Byte, 80Byte, 96Byte......496Byte；

small:4 种规格, 512Byte, 1KB, 2KB, 4KB；

normal: 3种规格, 8KB、16KB、32KB;

 如此我们得出结论PoolThreadCache 中缓存数组的数据结构如下图所示:

![](http://files.luyanan.com//img/20191010211930.png)

在基本了解缓存数组的数据结构后, 我们再继续剖析在缓存中分配内存的逻辑. 回到到 PoolArena 的 allocate() 方法中:

```java
    private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
        final int normCapacity = normalizeCapacity(reqCapacity);
        //  规格化
        if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
            int tableIdx;
            PoolSubpage<T>[] table;
            // 判断是不是tiny
            boolean tiny = isTiny(normCapacity);
            if (tiny) { // < 512
                // 缓存分配
                if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                // 通过tinyIdx 拿到tableIdx
                tableIdx = tinyIdx(normCapacity);
                // subpage 的数组
                table = tinySubpagePools;
            } else {
                if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = smallIdx(normCapacity);
                table = smallSubpagePools;
            }

            // 拿到对应的节点
            final PoolSubpage<T> head = table[tableIdx];

            /**
             * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
             * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
             */
            synchronized (head) {
                final PoolSubpage<T> s = head.next;
                // 默认情况下head的next 也是自身
                if (s != head) {
                    assert s.doNotDestroy && s.elemSize == normCapacity;
                    long handle = s.allocate();
                    assert handle >= 0;
                    s.chunk.initBufWithSubpage(buf, handle, reqCapacity);

                    if (tiny) {
                        allocationsTiny.increment();
                    } else {
                        allocationsSmall.increment();
                    }
                    return;
                }
            }
            allocateNormal(buf, reqCapacity, normCapacity);
            return;
        }
        if (normCapacity <= chunkSize) {
            // 首先在缓存上进行内存分配
            if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                // 分配成功. 返回
                return;
            }
            // 分配不成功, 做实际的内存分配
            allocateNormal(buf, reqCapacity, normCapacity);
        } else {
            // Huge allocations are never served via the cache so just call allocateHuge
            // 大于这个值, 就不在缓存上分配
            allocateHuge(buf, reqCapacity);
        }
    }
```

首先是通过normalizeCapacity()  方法进行内存规格化, 我们跟到normalizeCapacity 方法中:

```java
 int normalizeCapacity(int reqCapacity) {
        if (reqCapacity < 0) {
            throw new IllegalArgumentException("capacity: " + reqCapacity + " (expected: 0+)");
        }
        if (reqCapacity >= chunkSize) {
            return reqCapacity;
        }

     // 如果 > tiny
        if (!isTiny(reqCapacity)) { // >= 512
            // Doubled

            // 找一个2的幂次方数值, 确保数值大小等于reqCapacity
            int normalizedCapacity = reqCapacity;
            normalizedCapacity --;
            normalizedCapacity |= normalizedCapacity >>>  1;
            normalizedCapacity |= normalizedCapacity >>>  2;
            normalizedCapacity |= normalizedCapacity >>>  4;
            normalizedCapacity |= normalizedCapacity >>>  8;
            normalizedCapacity |= normalizedCapacity >>> 16;
            normalizedCapacity ++;

            if (normalizedCapacity < 0) {
                normalizedCapacity >>>= 1;
            }

            return normalizedCapacity;
        }

        // Quantum-spaced
     // 如果是16 的倍数 , 
        if ((reqCapacity & 15) == 0) {
            return reqCapacity;
        }

     // 如果不是16的倍数, 变成最大小于当前值的值+ 16
        return (reqCapacity & ~15) + 16;
    }
```

 上面代码中 `if (!isTiny(reqCapacity))` 代表如果大于tiny 类型的大小，也就是512, 则会找一个2的幂次方的数值, 确保这个数值大小等于reqCapacity。如果是tiny , 则继续往下 ` if ((reqCapacity & 15) == 0)`, 这里判断如果是16 的倍数, 则直接返回 . 如果不是16的倍数, 则返回(reqCapacity & ~15) + 16 , 也就是变成最小大于当前值的16 的倍数值。从上面规格化逻辑看出, 这里将缓存大小规格化成固定大小, 确保每个缓存对象缓存的ByteBuf 容量统一. 回到allocate() 方法, ： if(isTinyOrSmall(normCapacity)) 这里是根据规格化后的大小判断是否是tiny 或者small类型, 我们跟进去:

```java
   // capacity < pageSize
    boolean isTinyOrSmall(int normCapacity) {
        return (normCapacity & subpageOverflowMask) == 0;
    }
```

这个方法是判断如果normCapacity 小于一个page 的大小, 则就是8KB 代表其是tiny 或者small.

继续看allocate() 方法，如果当前大小是tiny 或者small, 则 isTiny(normCapacity)判断是否是 tiny 类型,跟进去:

```java
   // normCapacity < 512
    static boolean isTiny(int normCapacity) {
        return (normCapacity & 0xFFFFFE00) == 0;
    }
```

这个方法是判断如果小于512, 就认为是tiny.

再继续看 allocate() 方法, 如果是tiny, 则通过 cache.allocateTiny(this, buf, reqCapacity, normCapacity)在缓存上分配. 我们就以tiny 类型为例, 分析在缓存上分配ByteBuf 的流, allocateTiny 是缓存分配的入口. 我们跟进去: 进入到了PoolThreadCache 的 allocateTiny()方法中:

```java
    boolean allocateTiny(PoolArena<?> area, PooledByteBuf<?> buf, int reqCapacity, int normCapacity) {
        return allocate(cacheForTiny(area, normCapacity), buf, reqCapacity);
    }
```

这里有个方法cacheForTiny(area, normCapacity), 这个方法的作用是根据normCapacity 找到tiny类型缓存数组中的一个缓存对象, 我们跟进去cacheForTiny方法:

```java
  private MemoryRegionCache<?> cacheForTiny(PoolArena<?> area, int normCapacity) {
        int idx = PoolArena.tinyIdx(normCapacity);
        if (area.isDirect()) {
            return cache(tinySubPageDirectCaches, idx);
        }
        return cache(tinySubPageHeapCaches, idx);
    }
```

PoolArena.tinyIdx(normCapacity)是找到tiny类型缓存数组的下标, 继续跟进到 tinyIdx 方法:

```java
 static int tinyIdx(int normCapacity) {
        return normCapacity >>> 4;
    }
```

这里相当于直接将 normCapacity 除以16, 通过前面的内容我们知道，  tiny 类型缓存数组中的每个元素规格化的数据都是16的倍数, 所以通过这种方式可以找到其下标, 如果是16Byte 会拿到下标为1的元素， 如果是32Byte 会拿到下标为2的元素.

回到acheForTiny()方法中： if (area.isDirect()) 这里是判断是否分配堆外内存, 因为我们是按照堆外内存进行举例的, 所以这里是true, 再继续跟到到 cache(tinySubPageDirectCaches, idx)方法：

```java
  private static <T> MemoryRegionCache<T> cache(MemoryRegionCache<T>[] cache, int idx) {
        if (cache == null || idx > cache.length - 1) {
            return null;
        }
        return cache[idx];
    }
```

这里我们看到直接通过下标的方式拿到了缓存数组中的对象, 回到到 PoolThreadCache 的 allocateTiny()方法中：

```java
    boolean allocateTiny(PoolArena<?> area, PooledByteBuf<?> buf, int reqCapacity, int normCapacity) {
        return allocate(cacheForTiny(area, normCapacity), buf, reqCapacity);
    }
```

拿到了缓存对象后, 我们跟到 `allocate(cacheForTiny(area, normCapacity), buf, reqCapacity)`  方法中:

```java
  private boolean allocate(MemoryRegionCache<?> cache, PooledByteBuf buf, int reqCapacity) {
        if (cache == null) {
            // no cache found so just return false here
            return false;
        }
        boolean allocated = cache.allocate(buf, reqCapacity);
        if (++ allocations >= freeSweepAllocationThreshold) {
            allocations = 0;
            trim();
        }
        return allocated;
    }
```

分析上面的代码, 看到cache.allocate(buf, reqCapacity) 进行继续分配, 再继续往里跟, 来到内部类MemoryRegionCache 的 allocate(PooledByteBuf buf, int reqCapacity)方法：

```java
   /**
         * Allocate something out of the cache if possible and remove the entry from the cache.
         */
        public final boolean allocate(PooledByteBuf<T> buf, int reqCapacity) {
            Entry<T> entry = queue.poll();
            if (entry == null) {
                return false;
            }
            initBuf(entry.chunk, entry.handle, buf, reqCapacity);
            entry.recycle();

            // allocations is not thread-safe which is fine as this is only called from the same thread all time.
            ++ allocations;
            return true;
        }
```

这个方法, 首先通过 queue.poll()这种方式弹出一个Entry, 我们之前分析过MemoryRegionCache 维护这一个队列, 而队列中的每一个值是一个entry, 我们简单来看一下Entry 这个类

```java
  static final class Entry<T> {
            final Handle<Entry<?>> recyclerHandle;
            PoolChunk<T> chunk;
            long handle = -1;

            Entry(Handle<Entry<?>> recyclerHandle) {
                this.recyclerHandle = recyclerHandle;
            }

            void recycle() {
                chunk = null;
                handle = -1;
                recyclerHandle.recycle(this);
            }
        }
```

 我们重点关注 chunk 和handler 这两个属性, 	chunk 代表一块连续的内存, 我们之前简单介绍过, Netty 是通过chunk 为单位进行内存分配的. handler 相当于一个指针, 可以唯一定位到chunk 里面的一块连续的内存. 这样, 通过chunk和handler 就可以定位到ByteBuf 中的指定的一块连续的内存, 有关ByteBuf 相关的读写， 都会在这块内存中进行, 现在再回到MemoryRegionCache 的 allocate(PooledByteBuf buf,int reqCapacity)方法：

```java
       /**
         * Allocate something out of the cache if possible and remove the entry from the cache.
         */
        public final boolean allocate(PooledByteBuf<T> buf, int reqCapacity) {
            Entry<T> entry = queue.poll();
            if (entry == null) {
                return false;
            }
            initBuf(entry.chunk, entry.handle, buf, reqCapacity);
            entry.recycle();

            // allocations is not thread-safe which is fine as this is only called from the same thread all time.
            ++ allocations;
            return true;
        }
```



弹出entry后, 通过`initBuf(entry.chunk, entry.handle, buf, reqCapacity)`  这种方式给ByteBuf 初始化， 这里参数传入我们刚才分析过的 当前的entry的chunk 和handler, 因为我们分析的 tiny 类型的缓存对象是SubPageMemoryRegionCache 类型, 所以继续回到SubPageMemoryRegionCache 类的initBuf(entry.chunk, entry.handle, buf, reqCapacity)方法中：

```java
    @Override
        protected void initBuf(
                PoolChunk<T> chunk, long handle, PooledByteBuf<T> buf, int reqCapacity) {
            chunk.initBufWithSubpage(buf, handle, reqCapacity);
        }
```

这里的chunk 调用了了 initBufWithSubpage(buf, handle, reqCapacity)方法, 其实就是PoolChunk 类中的方法, 我们继续initBufWithSubpage() 方法:

```java
  void initBufWithSubpage(PooledByteBuf<T> buf, long handle, int reqCapacity) {
        initBufWithSubpage(buf, handle, bitmapIdx(handle), reqCapacity);
    }
```



上面代码中, 调用了 bitmapIdx()方法, 有关 bitmapIdx()方法相关的逻辑, 会再后面进行剖析,这里继续往里看, 看 initBufWithSubpage 的逻辑:

```java
  private void initBufWithSubpage(PooledByteBuf<T> buf, long handle, int bitmapIdx, int reqCapacity) {
        assert bitmapIdx != 0;

        int memoryMapIdx = memoryMapIdx(handle);

        PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
        assert subpage.doNotDestroy;
        assert reqCapacity <= subpage.elemSize;

        buf.init(
            this, handle,
            runOffset(memoryMapIdx) + (bitmapIdx & 0x3FFFFFFF) * subpage.elemSize, reqCapacity, subpage.elemSize,
            arena.parent.threadCache());
    }
```

我们先关注 init()  方法, 因为我们是以PooledUnsafeDirectByteBuf 为例,所以这里走的是PooledUnsafeDirectByteBuf 的  init() 方法, 进入init() 方法：

```java
  @Override
    void init(PoolChunk<ByteBuffer> chunk, long handle, int offset, int length, int maxLength,
              PoolThreadCache cache) {
        super.init(chunk, handle, offset, length, maxLength, cache);
        initMemoryAddress();
    }	
```

首先调用了父类的init()  方法，继续跟进去:

```java
  void init(PoolChunk<T> chunk, long handle, int offset, int length, int maxLength, PoolThreadCache cache) {
        assert handle >= 0;
        assert chunk != null;

        this.chunk = chunk;
        this.handle = handle;
        memory = chunk.memory;
        this.offset = offset;
        this.length = length;
        this.maxLength = maxLength;
        tmpNioBuf = null;
        this.cache = cache;
    }
```

上面的代码就是将PooledUnsafeDirectByteBuf 的各个属性进行了初始化:

this.chunk = chunk 这里初始化了 chunk, 代表当前的 ByteBuf 是在哪一块内存中分配的。

this.handle = handle 这里初始化了 handle, 代表当前的 ByteBuf 是这块内存的哪个连续内存。

有关 offset 和 length, 我们会在之后再分析, 在这里我们只需要知道, 通过缓存分配 ByteBuf, 我们只需要通过一个chunk 和 handle, 就可以确定一块内存，以上就是通过缓存分配 ByteBuf 对象的全过程。

现在，我们回到 MemoryRegionCache 的 allocate(PooledByteBuf buf, int reqCapacity)方法：

```java
 public final boolean allocate(PooledByteBuf<T> buf, int reqCapacity) {
            Entry<T> entry = queue.poll();
            if (entry == null) {
                return false;
            }
            initBuf(entry.chunk, entry.handle, buf, reqCapacity);
            entry.recycle();

            // allocations is not thread-safe which is fine as this is only called from the same thread all time.
            ++ allocations;
            return true;
        }
```

分析完了initBuf() 方法，再继续往下看, entry.recycle() 这步是将 entry 对象进行回收, 因为entry 对象弹出后没有在被引用，可能gc 会将entry 对象进行回收, netty为了将对象进行循环利用, 就将其对象放在回收站进行回收,我们跟进到recycle() 方法:

```java
  void recycle() {
                chunk = null;
                handle = -1;
                recyclerHandle.recycle(this);
            }
```

chunk = null  和handler = -1  表示当前entry 不指向任何一块内存,recyclerHandle.recycle(this) 将当前entry 回收, 以上就是命中缓存的流程, 因为这里我们是假设缓存中有值的情况下， 如果第一次分配, 缓存中是没有值的, 那么在缓存中没有值的i情况下, Netty 是如何进行分配的呢? 我们在后面再详细分析：

最后, 我们简单总结一下MemoryRegionCache 对象的基本结构, 如下图所示:

![](http://files.luyanan.com//img/20191010224444.png)

###  5. Page 级别的内存分配

之前我们介绍过,Netty 内存分配的单位是chunk, 一个chunk 的大小是16MB,实际上每个chunk都以双向链表的形式保存在一个chunkList中, 而多个chunkList 同样也是双向链表进行关联的, 大概结构如下:

![](http://files.luyanan.com//img/20191010225320.png)

在chunkList 中, 是根据chunk 的内存使用率归到一个chunkList中, 这样, 在内存分配时, 会根据百分比找到相应的chunkList, 在chunkList 中选择一个chunk 进行内存分配, 我们来看PoolArena 有关 chunkList 的成员变量:

```java
        q100 = new PoolChunkList<T>(null, 100, Integer.MAX_VALUE, chunkSize);
        q075 = new PoolChunkList<T>(q100, 75, 100, chunkSize);
        q050 = new PoolChunkList<T>(q075, 50, 100, chunkSize);
        q025 = new PoolChunkList<T>(q050, 25, 75, chunkSize);
        q000 = new PoolChunkList<T>(q025, 1, 50, chunkSize);
        qInit = new PoolChunkList<T>(q000, Integer.MIN_VALUE, 25, chunkSize);
```

这里总共定义了6个chunkList, 并在构造方法中将其进行初始化, 我们跟到其构造方法:

```java
 protected PoolArena(PooledByteBufAllocator parent, int pageSize, int maxOrder, int pageShifts, int chunkSize) {
        this.parent = parent;
        this.pageSize = pageSize;
        this.maxOrder = maxOrder;
        this.pageShifts = pageShifts;
        this.chunkSize = chunkSize;
        subpageOverflowMask = ~(pageSize - 1);
        tinySubpagePools = newSubpagePoolArray(numTinySubpagePools);
        for (int i = 0; i < tinySubpagePools.length; i ++) {
            tinySubpagePools[i] = newSubpagePoolHead(pageSize);
        }

        numSmallSubpagePools = pageShifts - 9;
        smallSubpagePools = newSubpagePoolArray(numSmallSubpagePools);
        for (int i = 0; i < smallSubpagePools.length; i ++) {
            smallSubpagePools[i] = newSubpagePoolHead(pageSize);
        }

        q100 = new PoolChunkList<T>(null, 100, Integer.MAX_VALUE, chunkSize);
        q075 = new PoolChunkList<T>(q100, 75, 100, chunkSize);
        q050 = new PoolChunkList<T>(q075, 50, 100, chunkSize);
        q025 = new PoolChunkList<T>(q050, 25, 75, chunkSize);
        q000 = new PoolChunkList<T>(q025, 1, 50, chunkSize);
        qInit = new PoolChunkList<T>(q000, Integer.MIN_VALUE, 25, chunkSize);

        q100.prevList(q075);
        q075.prevList(q050);
        q050.prevList(q025);
        q025.prevList(q000);
        q000.prevList(null);
        qInit.prevList(qInit);

        List<PoolChunkListMetric> metrics = new ArrayList<PoolChunkListMetric>(6);
        metrics.add(qInit);
        metrics.add(q000);
        metrics.add(q025);
        metrics.add(q050);
        metrics.add(q075);
        metrics.add(q100);
        chunkListMetrics = Collections.unmodifiableList(metrics);
    }
```

首先通过new PoolChunkList() 这种方式将每个chunkList 进行创建,我们以q050 = new PoolChunkList(q075, 50, 100, chunkSize)  为例进行简单介绍, q075 表示当前q50 的下一个节点是q075, 刚才我们讲过ChunkList 是通过双向链表进行关联的. 所以这里不难理解, 参数50 和100 表示当前chunkList 中存储的chunk 的内存使用率都在50% 到100% 之间. 最后chunkSize 为其设置大小, 创建完ChunkList 之后, 再设置其上一个节点,, q050.prevList(q025)  为例, 这里代表当前chunkList 的上一个节点是q025, 以这种方式创建完成之后, chunkList 的节点关系就变成了如下图所示:

![](http://files.luyanan.com//img/20191011103228.png)

Netty中,chunk 又包含了多个Page, 每个page 的大小为8KB, 如果要分配16KB的内存, 就要在chunk 汇总找到连续的两个page就可以分配, 对应关系如下:

![](http://files.luyanan.com//img/20191011103351.png)

很多场景下,为缓冲区分配8KB的内存也是一种浪费, 比如只需要分配2KB的缓存区, 如果使用8KB的就会造成6KB的浪费, 这种情况下，Netty 又会将page 切分成多个subpage, 每个subpage 大小要根据分配的缓冲区大小而指定, 比如要分配2KB 的内存, 就会将一个page切分成4个subpage, 每个subpage 的大小为2KB, 如下图:

![](http://files.luyanan.com//img/20191011104117.png)

来看看PoolSubpage 的基本结构:

```java
final class PoolSubpage<T> implements PoolSubpageMetric {

    final PoolChunk<T> chunk;
    private final int memoryMapIdx;
    private final int runOffset;
    private final int pageSize;
    private final long[] bitmap;

    PoolSubpage<T> prev;
    PoolSubpage<T> next;

    boolean doNotDestroy;
    int elemSize;
    private int maxNumElems;
    private int bitmapLength;
    private int nextAvail;
    private int numAvail;
    
}
```

chunk  代表其子页属于哪个chunk, bitmap 用于记录子页的内存分配情况, prev和 next, 代表子页是按照双向链表进行关联的, 这里分别指向上一个和下一个节点, elemSize 属性代表就是这个子页是按照多大内存进行划分的, 如果按照1KB 进行划分, 则可以划分出8个子页, 简单介绍了内存分配的数据结构.

我们开始剖析Netty 级别上分配内存的流程, 还是回到PoolArena 的 allocate 方法：

```java
 private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
        final int normCapacity = normalizeCapacity(reqCapacity);
        if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
            int tableIdx;
            PoolSubpage<T>[] table;
            boolean tiny = isTiny(normCapacity);
            if (tiny) { // < 512
                if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = tinyIdx(normCapacity);
                table = tinySubpagePools;
            } else {
                if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = smallIdx(normCapacity);
                table = smallSubpagePools;
            }

            final PoolSubpage<T> head = table[tableIdx];

            /**
             * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
             * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
             */
            synchronized (head) {
                final PoolSubpage<T> s = head.next;
                if (s != head) {
                    assert s.doNotDestroy && s.elemSize == normCapacity;
                    long handle = s.allocate();
                    assert handle >= 0;
                    s.chunk.initBufWithSubpage(buf, handle, reqCapacity);

                    if (tiny) {
                        allocationsTiny.increment();
                    } else {
                        allocationsSmall.increment();
                    }
                    return;
                }
            }
            allocateNormal(buf, reqCapacity, normCapacity);
            return;
        }
        if (normCapacity <= chunkSize) {
            if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            allocateNormal(buf, reqCapacity, normCapacity);
        } else {
            // Huge allocations are never served via the cache so just call allocateHuge
            allocateHuge(buf, reqCapacity);
        }
    }
```

我们之前讲过, 如果在缓存中分配不成功, 则会开辟一块连续的内存进行缓冲区分配, 这里我们先跳过isTinyOrSmall(normCapacity)往后的代码, 之后再来分析。

首先if (normCapacity <= chunkSize) 说明其小于16KB , 然后首先在缓存中分配, 以为最初缓存中没有值, 所以会走到 allocateNormal(buf, reqCapacity, normCapacity), 这里实际上就是在page级别上进行分配, 分配一个或者多个page 的空间, 我们跟进到 allocateNormal()  方法:

```java
private synchronized void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
     // 首先在原来的chunk 上进行内存分配(1)   
    if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
            q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
            q075.allocate(buf, reqCapacity, normCapacity)) {
            ++allocationsNormal;
            return;
        }

        // Add a new chunk.
    // 创建chunk 进行内存分配(2)
        PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
        long handle = c.allocate(normCapacity);
        ++allocationsNormal;
        assert handle > 0;
    // 初始化 byteBuf(3)
        c.initBuf(buf, handle, reqCapacity);
        qInit.add(c);
    }
```

这里主要拆解了如下步骤:

1. 在原有的chunk 中进行分配
2. 创建chunk 进行分配
3. 初始化byteBuf 

首先我们来看第一步, 在原有的chunk 中进行分配:

```java
   if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
            q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
            q075.allocate(buf, reqCapacity, normCapacity)) {
            ++allocationsNormal;
            return;
        }
```

我们之前讲过, chunkList 是存储不同内存使用量的chunk 集合,每个chunkList 通过双向链表的形式进行关联, 这里的 q050.allocate(buf, reqCapacity, normCapacity)  就代表首先在q050 这个chunkList 进行内存分配, 我们以q050 为例进行分析, 跟到q050.allocate(buf, reqCapacity, normCapacity)方法中：

```java
boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
        if (head == null || normCapacity > maxCapacity) {
            // Either this PoolChunkList is empty or the requested capacity is larger then the capacity which can
            // be handled by the PoolChunks that are contained in this PoolChunkList.
            return false;
        }

    //  从head 节点往下遍历
        for (PoolChunk<T> cur = head;;) {
            long handle = cur.allocate(normCapacity);
            if (handle < 0) {
                cur = cur.next;
                if (cur == null) {
                    return false;
                }
            } else {
                cur.initBuf(buf, handle, reqCapacity);
                if (cur.usage() >= maxUsage) {
                    remove(cur);
                    nextList.add(cur);
                }
                return true;
            }
        }
    }
```

首先会从head节点进行往下遍历, `ong handle = cur.allocate(normCapacity)`  表示对于每个chunk 都尝试去分配, if (handle < 0) 说明没有分配到, 则通过 cur = cur.next 找到下一个节点继续进行分配, 我们讲过chunk 也是通过双向链表进行关联的. 所以对这块逻辑应该不会陌生, 如果handler 大于0 说明已经分配到内存, 则通过cur.initBuf(buf, handle, reqCapacity)对 byteBuf 进行初始化；if (cur.usage() >= maxUsage) 代表当前 chunk 的内存使用率大于其最大
使用率, 则通过 remove(cur)从当前的 chunkList 中移除, 再通过 nextList.add(cur)添加到下一个 chunkList 中。
我们再回到 PoolArena 的 allocateNormal()方法中：

看第二步 PoolChunk c = newChunk(pageSize, maxOrder, pageShifts, chunkSize)，这里的参数 pageSize 是 8192, 也就是 8KB。

maxOrder 为11;

pageShifts 为 13, 2 的 13 次方正好是 8192, 也就是 8KB；
chunkSize 为 16777216, 也就是 16MB。

因为我们分析的是堆外内存, newChunk(pageSize, maxOrder, pageShifts, chunkSize)所以会走到 DirectArena 的
	()方法：

````java
     @Override
        protected PoolChunk<ByteBuffer> newChunk(int pageSize, int maxOrder, int pageShifts, int chunkSize) {
            return new PoolChunk<ByteBuffer>(
                    this, allocateDirect(chunkSize),
                    pageSize, maxOrder, pageShifts, chunkSize);
        }
````

这里直接通过构造函数创建了一个chunk , allocateDirect(chunkSize) 这里是通过JDK 的api 申请一块直接内存, 我们跟到 PoolChunk 的构造函数:

```java
 PoolChunk(PoolArena<T> arena, T memory, int pageSize, int maxOrder, int pageShifts, int chunkSize) {
        unpooled = false;
        this.arena = arena;
     // memory 为一个ByteBuf
        this.memory = memory;
     // 8KB
        this.pageSize = pageSize;
     // 13
        this.pageShifts = pageShifts;
     // 11   
     this.maxOrder = maxOrder;
        this.chunkSize = chunkSize;
        unusable = (byte) (maxOrder + 1);
        log2ChunkSize = log2(chunkSize);
        subpageOverflowMask = ~(pageSize - 1);
        freeBytes = chunkSize;

        assert maxOrder < 30 : "maxOrder should be < 30, but is: " + maxOrder;
        maxSubpageAllocs = 1 << maxOrder;

        // Generate the memory map.
      //  节点数量为4096   
     memoryMap = new byte[maxSubpageAllocs << 1];
     // 也就是4096个节点
        depthMap = new byte[memoryMap.length];
        int memoryMapIndex = 1;
     // d 相当于一个深度, 赋值的内容代表当前节点的深度
        for (int d = 0; d <= maxOrder; ++ d) { // move down the tree one level at a time
            int depth = 1 << d;
            for (int p = 0; p < depth; ++ p) {
                // in each level traverse left to right and set value to the depth of subtree
                memoryMap[memoryMapIndex] = (byte) d;
                depthMap[memoryMapIndex] = (byte) d;
                memoryMapIndex ++;
            }
        }

        subpages = newSubpageArray(maxSubpageAllocs);
    }
```

首先将传入的参数的值进行赋值 `this.memory = memory`  就是将参数中创建的堆外内存进行保存, 就是chunk 所指向的那块连续的内存, 在这个chunk 中所分配的BteBuf, 都会在这块内存中进行读写.

我们重点关注 memoryMap = new byte[maxSubpageAllocs << 1] 和 depthMap = new byte[memoryMap.length]
这两步：首先看 memoryMap = new byte[maxSubpageAllocs << 1]；这里初始化了一个字节数组 memoryMap, 大
小为 maxSubpageAllocs << 1, 也就是 4096；depthMap = new byte[memoryMap.length] 同样也是初始化了一个字
节数组, 大小为 memoryMap 的大小, 也就是 4096。继续往下分析之前, 我们看 chunk 的一个层级关系。

![](http://files.luyanan.com//img/20191011112610.png)

这是一个二叉树的结构，左侧的数字代表层级, 右侧代表一块连续的内存, 每个父节下又拆分成多个子节点, 最顶层表示的内存范围为0-16KB， 其下又分为两层, 范围为0-8MB，8-16MB, 以此类推, 最后到11层, 以8KB 的大小划分, 也就是一个page 的大小.

如果我们分配一个8MB 的缓冲区, 则会将第二层的第一个节点, 也就是0-8 这样连续的内存进行分配, 分配完成后会将这个节点设置为不可用, 结合上面的图, 我们再看构造函数中的for 循环：

```java
   for (int d = 0; d <= maxOrder; ++ d) { // move down the tree one level at a time
            int depth = 1 << d;
            for (int p = 0; p < depth; ++ p) {
                // in each level traverse left to right and set value to the depth of subtree
                memoryMap[memoryMapIndex] = (byte) d;
                depthMap[memoryMapIndex] = (byte) d;
                memoryMapIndex ++;
            }
        }
```

实际上这个for循环就是将上面的结构包装成一个字节数组memoryMap, 外层循环用于控制层数, 内层循环用于控制里面每层的节点, 这里经过循环后， memoryMap 和depthMap 内容为以下表现形式：

> [0, 0, 1, 1, 2, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3, 3, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4...........]

这里注意一下, 因为程序汇总数组的下标是从1开始设置的, 所以第零个节点元素为默认值0.

这里数字代表层级, 同时也代表了当前层级的节点, 相同的数字个数就是这一层级的节点数.

其中0 为2(因为这里分配时下标是从1开始的, 所以第0个位置的默认是0, 实际上第0层元素只有一个, 就是头节点), 1为2, 2为4, 3为8, 4为16, n 为2的n次方个, 也及时11有2 的11次方个.

我们再回到PoolArena 的 allocateNormal()方法：

```java
   private synchronized void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
        if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
            q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
            q075.allocate(buf, reqCapacity, normCapacity)) {
            ++allocationsNormal;
            return;
        }

        // Add a new chunk.
        PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
        long handle = c.allocate(normCapacity);
        ++allocationsNormal;
        assert handle > 0;
        c.initBuf(buf, handle, reqCapacity);
        qInit.add(c);
    }
```

我们继续剖析 long handle = c.allocate(normCapacity) 这步，跟到 allocate(normCapacity)中：

```jade
   long allocate(int normCapacity) {
        if ((normCapacity & subpageOverflowMask) != 0) { // >= pageSize
            return allocateRun(normCapacity);
        } else {
            return allocateSubpage(normCapacity);
        }
    }
```



如果分配的以page 为单位, 则走到allocateRun(normCapacity) 方法中, 跟进去

```java
    private long allocateRun(int normCapacity) {
        int d = maxOrder - (log2(normCapacity) - pageShifts);
        int id = allocateNode(d);
        if (id < 0) {
            return id;
        }
        freeBytes -= runLength(id);
        return id;
    }
```

int d = maxOrder - (log2(normCapacity) - pageShifts) 表示根据 normCapacity 计算出第几层；

int id = allocateNode(d) 表示根据层级关系, 去分配一个节点, 其中 id 代表 memoryMap 中的下标。

我们跟到 allocateNode()方法中：

```java
private int allocateNode(int d) {
    // 下标初始值为1
        int id = 1;
    // 代表 当前层级第一个节点的初始下标
        int initial = - (1 << d); // has last d bits = 0 and rest all = 1
    // 获取第一个节点的值
        byte val = value(id);
    // 如果值大于层级, 说明chunk 不可用
        if (val > d) { // unusable
            return -1;
        }
    // 当前下标对应的节点值如果小于层级,或者当前下标小于层级的初始下标
        while (val < d || (id & initial) == 0) { // id & initial == 1 << d for all ids at depth d, for < d it is 0
            // 当前下标乘以2, 代表当前节点的子节点的初始位置
            id <<= 1;
            // 获取id 位置的值
            val = value(id);
            // 如果当前节点值大于层数(节点不可用)
            if (val > d) {
                // id 为偶数则+1, id 为奇数则-1(拿的是其兄弟节点)
                id ^= 1;
                // 获取id 的值
                val = value(id);
            }
        }
        byte value = value(id);
        assert value == d && (id & initial) == 1 << d : String.format("val = %d, id & initial = %d, d = %d",
                value, id & initial, d);
    // 将找到的节点设置为不可用
        setValue(id, unusable); // mark as unusable
    // 逐层往上标记被使用    
    updateParentsAlloc(id);
        return id;
    }
```

这里是实际上从一个节点往下找, 找到层级为d 未被使用的节点，我们可以通过注释体会其逻辑, 找到相关节点后通过setValue 将当前节点设置为不可用, 其中id 是当前节点的下标, unusable 代表一个不可用的值, 这里是12, 以为我们的层级只有12层, 所以设置为12之后就相当于标记不可用,设置成不可用之后, 通过updateParentsAlloc(id) 逐层设置为被使用, 我们跟进 updateParentsAlloc() 方法:

```java
    private void updateParentsAlloc(int id) {
        while (id > 1) {
            // 取到当前节点的父节点id
            int parentId = id >>> 1;
            // 获取当前节点的值
            byte val1 = value(id);
            // 找到当前节点的兄弟节点
            byte val2 = value(id ^ 1);
            // 如果当前节点值大小兄弟节点,则保存当前节点值到val,否则, 保存兄弟节点值到val.
            // 如果当前节点是不可用的, 则当前节点值是12, 大于兄弟节点的值,所以这里将兄弟节点的值进行保存
            byte val = val1 < val2 ? val1 : val2;
            // 将val 的值设置为父节点下标所对应的值
            setValue(parentId, val);
            // id 设置为父节点id, 继续循环
            id = parentId;
        }
    }
```

这里其实是循环将兄弟节点的值替换成父节点的值, 我们可以通过注释仔细的进行逻辑分析, 如果实在理解有困难, 我通过画图帮助大家理解, 简单起见, 我们这里只设置为三层：

![](http://files.luyanan.com//img/20191011120814.png)

我们模拟其分配场景, 假设只有三层, 其中 index 代表数组 memoryMap 的下标, value 代表其值, memoryMap 中的值
就为[0, 0, 1, 1, 2, 2, 2, 2]。我们要分配一个 4MB 的 byteBuf, 在我们调用 allocateNode(int d)中传入的 d 是 2, 也就是
第二层。根据我们上面分分析的逻辑这里会找到第二层的第一个节点, 也就是 0-4mb 这个节点, 找到之后将其设置为 不可用, 这样 memoryMap 中的值就为[0, 0, 1, 1, 12, 2, 2, 2]，二叉树的结构就会变为：

![](http://files.luyanan.com//img/20191011120905.png)

注意标红部分, 将 index 为 4 的节点设置为了不可用。将这个节点设置为不可用之后, 则会将进行向上设置不可用, 循
环将兄弟节点数值较小的节点替换到父节点, 也就是将 index 为 2 的节点的值替换成了 index 的为 5 的节点的值, 这样
数组的值就会变为[0, 1, 2, 1, 12, 2, 2, 2]，二叉树的结构变为：

![](http://files.luyanan.com//img/20191011120943.png)

> 注意: 这里节点标红仅仅代表节点变化,并不是当前节点为不可用状态, 真正不可用状态的判断依据是value 的值为12

这样, 如果再次分配一个 4MB 内存的 ByteBuf, 根据其逻辑, 则会找到第二层的第二个节点, 也就是 4-8MB。再根据我
们的逻辑, 通过向上设置不可用, index为2就会设置成不可用状态, 将value的值设置为12, 数组数值变为[0, 1, 12, 1, 12, 12, 2, 2]二叉树如下图所示：

![](http://files.luyanan.com//img/20191011121137.png)

这样我们看到, 通过分配两个4mb 的ByteBuf 之后, 当前节点和其父节点都会设置成不可用状态, 当index =2 的节点设置为不可用之后, 将不会再找到这个节点下的子节点. 以此类推, 直到所有的内存分配完毕的时候, index 为1 的节点也会变成不可用状态, 这样所有的page 就分配完毕, chunk 中再无可用的节点.  现在再回到 PoolArena的allocateNormal()
方法：

 ```java
 private synchronized void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
        if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
            q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
            q075.allocate(buf, reqCapacity, normCapacity)) {
            ++allocationsNormal;
            return;
        }

        // Add a new chunk.
        PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
        long handle = c.allocate(normCapacity);
        ++allocationsNormal;
        assert handle > 0;
        c.initBuf(buf, handle, reqCapacity);
        qInit.add(c);
    }
 ```

通过以上的逻辑我们知道`ong handle = c.allocate(normCapacity)`  这一步其实返回的就是 memoryMap 的一个下标, 通过这个下标我们能唯一的定义一块内存, 继续往下跟, 通过过 c.initBuf(buf, handle, reqCapacity)初始化 ByteBuf 之后, 通过 qInit.add(c)将新创建的 chunk 添加到 chunkList 中，我们跟到 initBuf 方法中去：

```java
  void initBuf(PooledByteBuf<T> buf, long handle, int reqCapacity) {
        int memoryMapIdx = memoryMapIdx(handle);
        int bitmapIdx = bitmapIdx(handle);
        if (bitmapIdx == 0) {
            byte val = value(memoryMapIdx);
            assert val == unusable : String.valueOf(val);
            buf.init(this, handle, runOffset(memoryMapIdx), reqCapacity, runLength(memoryMapIdx),
                     arena.parent.threadCache());
        } else {
            initBufWithSubpage(buf, handle, bitmapIdx, reqCapacity);
        }
    }
```

从上面的代码中, 看出通过memoryMapIdx(handle)找到 memoryMap 的下标, 其实就是 handle 的值。bitmapIdx(handle)
是有关 subPage 中使用到的逻辑, 如果是 page 级别的分配, 这里只返回 0, 所以进入到 if 块中。if 中首先断言当前节
点是不是不可用状态, 然后通过 init 方法进行初始化。其中 runOffset(memoryMapIdx)表示偏移量, 偏移量相当于分配
给缓冲区的这块内存相对于 chunk 中申请的内存的首地址偏移了多少。参数 runLength(memoryMapIdx), 表示根据下
标获取可分配的最大长度。我们跟到 init()方法中, 这里会走到 PooledByteBuf 的 init()方法：

```java
void init(PoolChunk<T> chunk, long handle, int offset, int length, int maxLength, PoolThreadCache cache) {
        assert handle >= 0;
        assert chunk != null;

        this.chunk = chunk;
        this.handle = handle;
        memory = chunk.memory;
        this.offset = offset;
        this.length = length;
        this.maxLength = maxLength;
        tmpNioBuf = null;
        this.cache = cache;
    }
```

这段代码又是我们熟悉的, 将属性进行了初始化, 以上就是完整的DirectUnsafePooledByteBuf 在Page 级别的完整分配的流程, 逻辑也是复杂的.

### 6. SubPage 级别的内存分配.

通过之前的学习我们知道,如果我们分配一个缓冲区大小远小于page, 则直接在一个page 上进行分配则会造成内存浪费, 所以需要将page 继续进行切分成多个子块进行分配, 子块分配的个数根据你要分配的缓冲区大小而定, 比如只需要分配1KB的内存, 就会将一个page 分为8等分. 简单起见, 我们这里仅仅以16字节为例, 讲解其分配逻辑, 在分析其逻辑前, 首先看 PoolArean 的一个属性:

> ```
> private final PoolSubpage<T>[] tinySubpagePools;
> ```

这个属性是一个PoolSubpage 的数组, 有点类似于一个 subpage 的缓存, 我们创建一个subpage 之后, 会将创建的subpage 与该属性其中的每个关联, 下次在分配的的时候可以直接通过该属性的元素去找关联的subpage, 我们其中是在构造方法中初始化的, 看构造方法中其初始化的代码:

> ```
> tinySubpagePools = newSubpagePoolArray(numTinySubpagePools);
> ```

这里的 numTinySubpagePools 为32, 跟到 `newSubpagePoolArray(numTinySubpagePools)`  方法中:

```java
    @SuppressWarnings("unchecked")
    private PoolSubpage<T>[] newSubpagePoolArray(int size) {
        return new PoolSubpage[size];
    }
```

这里直接创建了一个 PoolSubpage 数组, 长度为32, 在构造方法中创建完毕后, 会通过循环为其赋值:

```java
  for (int i = 0; i < tinySubpagePools.length; i ++) {
            tinySubpagePools[i] = newSubpagePoolHead(pageSize);
        }
```



继续跟进到 `newSubpagePoolHead(pageSize);` 方法中:

```java
    private PoolSubpage<T> newSubpagePoolHead(int pageSize) {
        PoolSubpage<T> head = new PoolSubpage<T>(pageSize);
        head.prev = head;
        head.next = head;
        return head;
    }
```

在 `newSubpagePoolHead` 方法中创建了一个 PoolSubpage 对象head.

```java
    head.prev = head;
    head.next = head;
```

这种写法我们知道SubPage 其实也是个双向链表,这里的将head的上一个节点和下一个节点设置为自身,有关PoolSubpage 的关联关系, 我们稍后分析, 这样通过循环创建 PoolSubpage, 总共会创建出来32个subpage, 其中每个subpage 实际代表一块内存大小。

![](http://files.luyanan.com//img/20191011140957.png)

tinySubpagePools 的结构就有点类似之前的缓存数据 tinySubPageDirectCaches 的结构, 了解了tinySubpagePools  属性, 我们看 看 PoolArean 的 allocate 方法, 也就是缓冲区的入口方法:

```java
private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
        final int normCapacity = normalizeCapacity(reqCapacity);
        if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
            int tableIdx;
            PoolSubpage<T>[] table;
            boolean tiny = isTiny(normCapacity);
            if (tiny) { // < 512
                if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = tinyIdx(normCapacity);
                table = tinySubpagePools;
            } else {
                if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = smallIdx(normCapacity);
                table = smallSubpagePools;
            }

            final PoolSubpage<T> head = table[tableIdx];

            /**
             * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
             * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
             */
            synchronized (head) {
                final PoolSubpage<T> s = head.next;
                if (s != head) {
                    assert s.doNotDestroy && s.elemSize == normCapacity;
                    long handle = s.allocate();
                    assert handle >= 0;
                    s.chunk.initBufWithSubpage(buf, handle, reqCapacity);

                    if (tiny) {
                        allocationsTiny.increment();
                    } else {
                        allocationsSmall.increment();
                    }
                    return;
                }
            }
            allocateNormal(buf, reqCapacity, normCapacity);
            return;
        }
        if (normCapacity <= chunkSize) {
            if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            allocateNormal(buf, reqCapacity, normCapacity);
        } else {
            // Huge allocations are never served via the cache so just call allocateHuge
            allocateHuge(buf, reqCapacity);
        }
    }
```

 之前我们在这个方法剖析过page级别相关内存分配逻辑, 现在我们来看subpage 级别分配的相关逻辑, 假设我们分配16字节的缓存区, `isTinyOrSmall(normCapacity)`  就会返回true, 进入if块, 同样 `if (tiny)`  这里会返回true. 继续跟进到 `if (tiny)` 的逻辑, 首先会在缓存中分配缓冲区, 如果分配不到, 就开辟一块内存进行内存分配, 先看着一步:

> ```
> tableIdx = tinyIdx(normCapacity);
> ```

这是通过 normCapacity 拿到tableIdx, 我们跟进去:

```java
   static int tinyIdx(int normCapacity) {
        return normCapacity >>> 4;
    }
```

这里将 normCapacity  除以16, 其实也就是1, 我们回到 到 PoolArena 的 allocate()方法继续看：

> ```
> table = tinySubpagePools;
> ```

这里将tinySubpagePools 赋值到局部变量table, 继续往下看:

`final PoolSubpage head = table[tableIdx]` 这步时通过下标拿到一个 PoolSubpage, 因为我们以16字节为例, 所以我们拿到下标为1的PoolSubpage, 对应的内存大小也就是16Byte, 再看 `final PoolSubpage head = table[tableIdx]` 这一步, 跟我们刚才了解到了tinySubpagePools 属性, 默认情况下 head.next 也是自身, 所以`if (s != head)` 会返回false, 我们继续往下看, 会走到 `allocateNormal(buf, reqCapacity, normCapacity)`   这个方法:

```java
private synchronized void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
        if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
            q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
            q075.allocate(buf, reqCapacity, normCapacity)) {
            ++allocationsNormal;
            return;
        }

        // Add a new chunk.
        PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
        long handle = c.allocate(normCapacity);
        ++allocationsNormal;
        assert handle > 0;
        c.initBuf(buf, handle, reqCapacity);
        qInit.add(c);
    }
```

这里的逻辑我们之前已经剖析过了, 首先在原来的chunk 中分配, 如果分配不成功, 则会创建chunk 进行分配, 我们看这一步`long handle = c.allocate(normCapacity);`  跟到  `allocate(normCapacity);` 这个方法:

```java
  long allocate(int normCapacity) {
        if ((normCapacity & subpageOverflowMask) != 0) { // >= pageSize
            return allocateRun(normCapacity);
        } else {
            return allocateSubpage(normCapacity);
        }
    }
```

前面我们分析page级别的时候, 剖析的是 `allocateRun(normCapacity)` 方法, 因为这里我们是以16字节举例, 所以这次我们剖析 `allocateSubpage(normCapacity)` 方法, 也就是在 subpage级别进行内存分配的.

```java
private long allocateSubpage(int normCapacity) {
        // Obtain the head of the PoolSubPage pool that is owned by the PoolArena and synchronize on it.
        // This is need as we may add it back and so alter the linked-list structure.
        PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity);
        synchronized (head) {
            int d = maxOrder; // subpages are only be allocated from pages i.e., leaves
            int id = allocateNode(d);
            if (id < 0) {
                return id;
            }

            final PoolSubpage<T>[] subpages = this.subpages;
            final int pageSize = this.pageSize;

            freeBytes -= pageSize;

            int subpageIdx = subpageIdx(id);
            PoolSubpage<T> subpage = subpages[subpageIdx];
            if (subpage == null) {
                subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
                subpages[subpageIdx] = subpage;
            } else {
                subpage.init(head, normCapacity);
            }
            return subpage.allocate();
        }
    }
```

首先, 通过`PoolSubpage head = arena.findSubpagePoolHead(normCapacity)` 这种方式找到head节点, 实际上这里head 就是我们分析的tinySubpagePools 属性的第一个节点, 也就是对应16KB的那个节点, int d = maxOrder 是将11赋值给d, 也就是在内存树的第11层取值, 这部分前面剖析过了,。int id = allocateNode(d)  这个也是前面分析过的, 字节数组 memoryMap 的下标, 这里指向一个page, 如果第一次分配, 指向的是0-8KB 的那个page, 前面已经对此进行详细的剖析了, `final PoolSubpage[] subpages = this.subpages`  这一步, 是拿到PoolChunk 中成员变量subpage的值, 也就是 PoolSubpage 数组, 在PoolChunk 进行初始化的时候， 也会初始化该数组, 长度为2048, 也就是每个chunk 都维护着一个subpage 的列表, 如果每一个page 级别的内存都需要被切分成子page, 则会将这个page 放入该列表中, 专门用于分配子page, 所以这个列表中的subpage 其实就是一个用于切分的page.

![](http://files.luyanan.com//img/20191011144649.png)

`int subpageIdx = subpageIdx(id)`  这一步是通过id 拿到这个 PoolSubpage 数组的下标, 如果id 对应的page 是0-8KB 的节点, 这里拿到的下标就是0, 在 if (subpage == null) 中, 因为默认 subpages 只是创建一个数组, 并没有数组中赋值, 所以第一次走到这里会返回true, 跟到if 块中:

> ```
> subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
> ```

这里通过 `new PoolSubpage` 创建一个新的subpage 之后, 通过`subpages[subpageIdx] = subpage`  这种方式将新创建的subpage 根据下表赋值到subpages 中的元素中, 在new PoolSubpage 的构造方法中, 传入head, 就是我们刚才提到过的tinySubpagePools 属性中的节点, 如果我们分配的16字节的缓冲区, 则这里对应的就是第一个节点, 我们跟到PoolSubpage  的构造方法中

```java
 PoolSubpage(PoolSubpage<T> head, PoolChunk<T> chunk, int memoryMapIdx, int runOffset, int pageSize, int elemSize) {
        this.chunk = chunk;
        this.memoryMapIdx = memoryMapIdx;
        this.runOffset = runOffset;
        this.pageSize = pageSize;
        bitmap = new long[pageSize >>> 10]; // pageSize / 16 / 64
        init(head, elemSize);
    }
```

这里重点关注属性 bitmap, 这是一个long类型的数组, 初始大小为8, 这里只是初始化的大小, 真正的大小要根据将page 切分多少块而确定的, 这里将属性进行了赋值, 我们跟到init()  方法:

```java
 void init(PoolSubpage<T> head, int elemSize) {
        doNotDestroy = true;
        this.elemSize = elemSize;
        if (elemSize != 0) {
            maxNumElems = numAvail = pageSize / elemSize;
            nextAvail = 0;
            bitmapLength = maxNumElems >>> 6;
            if ((maxNumElems & 63) != 0) {
                bitmapLength ++;
            }

            for (int i = 0; i < bitmapLength; i ++) {
                bitmap[i] = 0;
            }
        }
        addToPool(head);
    }
```

`this.elemSize = elemSize`  表示保存当前分配的缓存区大小, 这里我们以16字节举例, `maxNumElems = numAvail = pageSize / elemSize`   这里初始化了两个属性 性 maxNumElems, numAvail, 值都为 pageSize / elemSize, 表示一个 page 大小除以分配的缓冲区大小, 也就是表示当前 page 被划分了多少分。

nextAvail 则表示剩余可用的块数, 由于第一次分配都是可用的, 所以 numAvail=maxNumElems；

bitmapLength 表示bitmap 的实际大小, 刚才我们分析过, bitmap 初始化的大小为8, 但实际上并不一定需要8个元素, 元素个数要根据page切分的子块而定, 这里的大小是所切分的子块数除以64.

再往下看, `if ((maxNumElems & 63) != 0)`   判断maxNumElems 也就是当前配置所切分的子块是不是64的倍数, 如果不是, 则bigmapLength 加1 , 最后通过循环, 将其分配的大小中的元素赋值为0.

这里详细分析一下bitmap, 这里是个long 类型的数组, long 数组中的每一个值, 也就是long 类型的数字, 其中的每一个比特位, 都标记着page 中每一个子块的内存是否已分配, 如果比特位是1, 表示该快已经分配, 如果比特位是0, 表示该子块未分配. 标记顺序是其二进制数从低位到高位进行排列的, 我们应该知道为什么bitmap 大小要设置为子块数量乘以64, 因为long类型的数字是64位, 每一个元素能记录64个子块的数量, 这样就可以通过 子page 个数除以64 的方式决定bitmap中元素的数量, 如果子块不能整除64, 则通过元素数量+1 方式, 除以64之后剩余的子块通过long中比特位由低到高进行排列记录, 其逻辑结构如下图所示：

![](http://files.luyanan.com//img/20191011153603.png)

进入到 PoolSubpage 的addToPool(head) 方法:

```java
    private void addToPool(PoolSubpage<T> head) {
        assert prev == null && next == null;
        prev = head;
        next = head.next;
        next.prev = this;
        head.next = this;
    }
```

这里的head我们刚才讲过, 是Arena 中数组 tinySubpagePools 中的元素, 通过以上逻辑, 就会将新创建的Subpage 通过双向链表的方式关联到 tinySubpagePools 中的元素, 我们以16字节为例, 关联关系如下图所示：

![](http://files.luyanan.com//img/20191011154018.png)

这样, 下次如果还需要分配16字节的内存, 就可以通过tinySubpagePools 找到其元素关联的subpage 进行分配了, 我们再回到PoolChunk 的 allocateSubpage()方法：

```java
    private long allocateSubpage(int normCapacity) {
        // Obtain the head of the PoolSubPage pool that is owned by the PoolArena and synchronize on it.
        // This is need as we may add it back and so alter the linked-list structure.
        PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity);
        synchronized (head) {
            int d = maxOrder; // subpages are only be allocated from pages i.e., leaves
            int id = allocateNode(d);
            if (id < 0) {
                return id;
            }

            final PoolSubpage<T>[] subpages = this.subpages;
            final int pageSize = this.pageSize;

            freeBytes -= pageSize;

            int subpageIdx = subpageIdx(id);
            PoolSubpage<T> subpage = subpages[subpageIdx];
            if (subpage == null) {
                subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
                subpages[subpageIdx] = subpage;
            } else {
                subpage.init(head, normCapacity);
            }
            return subpage.allocate();
        }
    }
```

创建完一个subpage， 我们就可以通过subpage.allocate() 方法进行内存分配, 我们跟到allocate()  方法：

```java
long allocate() {
        if (elemSize == 0) {
            return toHandle(0);
        }

        if (numAvail == 0 || !doNotDestroy) {
            return -1;
        }

        final int bitmapIdx = getNextAvail();
        int q = bitmapIdx >>> 6;
        int r = bitmapIdx & 63;
        assert (bitmap[q] >>> r & 1) == 0;
        bitmap[q] |= 1L << r;

        if (-- numAvail == 0) {
            removeFromPool();
        }

        return toHandle(bitmapIdx);
    }
```

其中 bitmapIdx 表示从 bitmap 中找到一个可用的bit 位的下标, 注意, 这里是bit的下标, 并不是数组的下标, 我们之前分析过，因为每一个比特位代表一个子块的内存分配情况, 通过这个下标就可以知道哪个比特位是未分配状态, 我们跟进去：

```java
 private int getNextAvail() {
        int nextAvail = this.nextAvail;
        if (nextAvail >= 0) {
            this.nextAvail = -1;
            return nextAvail;
        }
        return findNextAvail();
    }
```

上述代表片段中的nextAvail 表示下一个可用的bitmapIdx, 在释放的时候就会被标记, 标记被释放的子块对应  bitmapIdx的下标, 如果< 0 则代表没有被释放的子块, 则通过findNextAvail() 方法进行查找, 继续跟进 findNextAvail() 方法:

```java
 private int findNextAvail() {
        final long[] bitmap = this.bitmap;
        final int bitmapLength = this.bitmapLength;
        for (int i = 0; i < bitmapLength; i ++) {
            long bits = bitmap[i];
            if (~bits != 0) {
                return findNextAvail0(i, bits);
            }
        }
        return -1;
    }
```

这里会遍历bitmap 中的每一个元素, 如果当前元素中所有的比特位并没有全部标记被使用, 则通过findNextAvail() 方法一个一个往后找标记未使用的比特位, 再继续跟findNextAvail0

```java
private int findNextAvail0(int i, long bits) {
        final int maxNumElems = this.maxNumElems;
        final int baseVal = i << 6;

        for (int j = 0; j < 64; j ++) {
            if ((bits & 1) == 0) {
                int val = baseVal | j;
                if (val < maxNumElems) {
                    return val;
                } else {
                    break;
                }
            }
            bits >>>= 1;
        }
        return -1;
    }
```

这里从当前元素的第一个比特位开始查找, 知道找到一个标记位0的比特位, 并返回当前比特位的下标,大致流程如下图所示:

![](http://files.luyanan.com//img/20191011155250.png)

我们回到allocate() 方法中:

```java
 long allocate() {
        if (elemSize == 0) {
            return toHandle(0);
        }

        if (numAvail == 0 || !doNotDestroy) {
            return -1;
        }

     // 取一个bitmap 中可用的id(绝对id)
        final int bitmapIdx = getNextAvail();
     // 除以64 (bitmap 的相对下标)
        int q = bitmapIdx >>> 6;
     // 除以64 取余, 其实就是当前绝对id 的偏移量
        int r = bitmapIdx & 63;
        assert (bitmap[q] >>> r & 1) == 0;
     // 当前位标记位1
        bitmap[q] |= 1L << r;

     // 如果可用的page 为0
     // 可用的子page -1
        if (-- numAvail == 0) {
            removeFromPool();
        }

     // 将bitmapIdx 转换为handler
        return toHandle(bitmapIdx);
    }
```



找到可用的bitmapIdx , 通过 `int q = bitmapIdx >>>6` 获取bitmap 中 bitmapIdx 所属元素的数组下标, `int r =
bitmapIdx & 63` 表示获取bitmapIdx 的位置是从当前元素最低开始的第几个比特位, `bitmap[q] |= 1L << r` 是将bitmap 的位置设置为不可用, 特就是比特位设置为1, 表示已占用, 然后将可用子配置的数量numAvail 减1.  如果没有可用子page 的数量, 则会将 PoolArena 中的数组 tinySubpagePools 所关联的 subpage 进行移除。最后通过toHandle(bitmapIdx)  获取当前子块的handle, 上一小节我们知道handler 指向的是当前chunk 中的唯一的一块内存, 我们跟进toHandle(bitmapIdx) 中：

```java
 private long toHandle(int bitmapIdx) {
        return 0x4000000000000000L | (long) bitmapIdx << 32 | memoryMapIdx;
    }
```

(long) bitmapIdx << 32 是将 bitmapIdx 右移 32 位, 而 32 位正好是一个 int 的长度, 这样, 通过 (long) bitmapIdx <<
32 | memoryMapIdx 计算, 就可以将 memoryMapIdx, 也就是 page 所属的下标的二进制数保存在 (long) bitmapIdx
<< 32 的低 32 位中。0x4000000000000000L 是一个最高位是 1 并且所有低位都是 0 的二进制数, 这样通过按位或的
方式可以将 (long) bitmapIdx << 32 | memoryMapIdx 计算出来的结果保存在 0x4000000000000000L 的所有低位中, 这样, 返回对的数字就可以指向 chunk 中唯一的一块内存，我们回到 PoolArena 的 allocateNormal 方法中：

```java
private synchronized void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
        if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
            q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
            q075.allocate(buf, reqCapacity, normCapacity)) {
            ++allocationsNormal;
            return;
        }

        // Add a new chunk.
        PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
        long handle = c.allocate(normCapacity);
        ++allocationsNormal;
        assert handle > 0;
        c.initBuf(buf, handle, reqCapacity);
        qInit.add(c);
    }
```

分析完了 `long handle = c.allocate(normCapacity)` 这步, 这里返回的handle 就指向 chunk 中的某个page 中的某个子块所对应的连续内存, 最后, 通过 initBuf() 初始化后， 将创建的chunk 加到chunkList 里面, 我们跟进到initBuf 方法里面

```java
 void initBuf(PooledByteBuf<T> buf, long handle, int reqCapacity) {
        int memoryMapIdx = memoryMapIdx(handle);
        int bitmapIdx = bitmapIdx(handle);
        if (bitmapIdx == 0) {
            byte val = value(memoryMapIdx);
            assert val == unusable : String.valueOf(val);
            buf.init(this, handle, runOffset(memoryMapIdx), reqCapacity, runLength(memoryMapIdx),
                     arena.parent.threadCache());
        } else {
            initBufWithSubpage(buf, handle, bitmapIdx, reqCapacity);
        }
    }
```

这部分在前面已经剖析过, 相信大家不会陌生, 这里有区别是的`if (bitmapIdx == 0)` 的判断, 这里的bitmapIdx 不会是0, 这样, 就会走到 initBufWithSubpage(buf, handle, bitmapIdx, reqCapacity)方法中，跟到 initBufWithSubpage()方法：

```java
 private void initBufWithSubpage(PooledByteBuf<T> buf, long handle, int bitmapIdx, int reqCapacity) {
        assert bitmapIdx != 0;

        int memoryMapIdx = memoryMapIdx(handle);

        PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
        assert subpage.doNotDestroy;
        assert reqCapacity <= subpage.elemSize;

        buf.init(
            this, handle,
            runOffset(memoryMapIdx) + (bitmapIdx & 0x3FFFFFFF) * subpage.elemSize, reqCapacity, subpage.elemSize,
            arena.parent.threadCache());
    }
```

首先拿到memoryMapIdx , 这里会将我们之前计算的 handle 传入, 跟进去:

```java
  private static int memoryMapIdx(long handle) {
        return (int) handle;
    }

```

这里将其强制转换为一个int 类型, 也就是去掉高32位, 这样就能得到memoryMapIdx, 回到initBufWithSubpage()  方法中:我们注意在buf 调用init()  方法中的一个参数 `runOffset(memoryMapIdx) + (bitmapIdx & 0x3FFFFFFF) *
subpage.elemSize`  这里的偏移量就是原来page 的偏移量 + 子块的偏移量: bitmapIdx & 0x3FFFFFFF 代表当前分配
的子 page 是属于第几个子 page。(bitmapIdx & 0x3FFFFFFF) * subpage.elemSize 表示在当前 page 的偏移量。这样, 分
配的 ByteBuf 在内存读写的时候, 就会根据偏移量进行读写。最后，我们跟到 init()方法中：

```java

    void init(PoolChunk<T> chunk, long handle, int offset, int length, int maxLength, PoolThreadCache cache) {
        assert handle >= 0;
        assert chunk != null;

        this.chunk = chunk;
        this.handle = handle;
        memory = chunk.memory;
        this.offset = offset;
        this.length = length;
        this.maxLength = maxLength;
        tmpNioBuf = null;
        this.cache = cache;
    }
```

这里又是我们熟悉的逻辑, 初始化属性之后, 一个缓冲区分配完成, 以上就是SubPage 级别的缓冲区分配逻辑。

### 7. 内存池ByteBuf 内存回收

我们知道,堆外内存是不受JVM 垃圾回收机制控制的, 所以我们分配一块堆外内存进行ByteBuf 操作时, 使用完毕要对对象进行回收, 本节就以 PooledUnsafeDirectByteBuf 为例讲解有关内存分配的相关逻辑, PooledUnsafeDirectByteBuf 中内存释放的入口方法是其父类 AbstractReferenceCountedByteBuf 中的release() 方法:

```java
 @Override
    public boolean release(int decrement) {
        return release0(checkPositive(decrement, "decrement"));
    }

    private boolean release0(int decrement) {
        for (;;) {
            int refCnt = this.refCnt;
            if (refCnt < decrement) {
                throw new IllegalReferenceCountException(refCnt, -decrement);
            }

            if (refCntUpdater.compareAndSet(this, refCnt, refCnt - decrement)) {
                if (refCnt == decrement) {
                    deallocate();
                    return true;
                }
                return false;
            }
        }
    }
```

`if (refCnt == decrement)`  中判断当前ByteBuf 是否没有被引用了, 如果没有被引用, 则通过deallocate() 方法进行释放 ,因为我们是以因为我们是以 PooledUnsafeDirectByteBuf 为例, 所以这里会调用其父类 PooledByteBuf 的 deallocate()方法：

```java
    @Override
    protected final void deallocate() {
        if (handle >= 0) {
            final long handle = this.handle;
            this.handle = -1;
            memory = null;
            chunk.arena.free(chunk, handle, maxLength, cache);
            recycle();
        }
    }
```

我们首先来分析  free()  方法:

```java
void free(PoolChunk<T> chunk, long handle, int normCapacity, PoolThreadCache cache) {
        if (chunk.unpooled) {
            int size = chunk.chunkSize();
            destroyChunk(chunk);
            activeBytesHuge.add(-size);
            deallocationsHuge.increment();
        } else {
            SizeClass sizeClass = sizeClass(normCapacity);
            if (cache != null && cache.add(this, chunk, handle, normCapacity, sizeClass)) {
                // cached so not free it.
                return;
            }

            freeChunk(chunk, handle, sizeClass);
        }
    }
```

首先判断是不是unpooled, 我们这里是 Pooled, 所以会走到else 块中：

sizeClass(normCapacity) 计算是哪种级别的size, 我们按照tiny 级别进行分析

cache.add(this, chunk, handle, normCapacity, sizeClass) 是将当前ByteBuf  进行缓存.

我们之前讲过, 在分配ByteBuf 时首先在缓存上分配, 而这步, 就是将其缓存的过程, 继续跟进去:

```java
 boolean add(PoolArena<?> area, PoolChunk chunk, long handle, int normCapacity, SizeClass sizeClass) {
        MemoryRegionCache<?> cache = cache(area, normCapacity, sizeClass);
        if (cache == null) {
            return false;
        }
        return cache.add(chunk, handle);
    }
```

首先根据类型拿到相关类型的缓存节点, 这里会根据不同的内存规格去找不同的对象, 我们简单回顾一下, 每个缓存对象都包含一个queue, queue 中每个节点是entry, 每一个entry 中包含一个chunk 和handle, 可以指向唯一的连续的内存, 我们跟到cache中:

```java
  private MemoryRegionCache<?> cache(PoolArena<?> area, int normCapacity, SizeClass sizeClass) {
        switch (sizeClass) {
        case Normal:
            return cacheForNormal(area, normCapacity);
        case Small:
            return cacheForSmall(area, normCapacity);
        case Tiny:
            return cacheForTiny(area, normCapacity);
        default:
            throw new Error();
        }
    }
```

假设我们是tiny类型, 这里就会走到   `cacheForTiny(area, normCapacity)` 方法中, 跟进去:

```java
 private MemoryRegionCache<?> cacheForTiny(PoolArena<?> area, int normCapacity) {
        int idx = PoolArena.tinyIdx(normCapacity);
        if (area.isDirect()) {
            return cache(tinySubPageDirectCaches, idx);
        }
        return cache(tinySubPageHeapCaches, idx);
    }
```

这个方法我们之前剖析过, 就是根据大小找到第几个缓存中的第几个缓存, 拿到下标后, 通过cache 去找相对应的缓存对象.

```java
  private static <T> MemoryRegionCache<T> cache(MemoryRegionCache<T>[] cache, int idx) {
        if (cache == null || idx > cache.length - 1) {
            return null;
        }
        return cache[idx];
    }
```

我们这里看到, 是直接通过下标拿到的缓存对象, 回到 add()  方法:

```java
  @SuppressWarnings({ "unchecked", "rawtypes" })
    boolean add(PoolArena<?> area, PoolChunk chunk, long handle, int normCapacity, SizeClass sizeClass) {
        MemoryRegionCache<?> cache = cache(area, normCapacity, sizeClass);
        if (cache == null) {
            return false;
        }
        return cache.add(chunk, handle);
    }
```

这里的cache 对象调用了一个add方法, 这个方法就是将chunk 和handle 封装成一个 entry 加到queue, 我们跟到add() 方法中:

```java
  public final boolean add(PoolChunk<T> chunk, long handle) {
            Entry<T> entry = newEntry(chunk, handle);
            boolean queued = queue.offer(entry);
            if (!queued) {
                // If it was not possible to cache the chunk, immediately recycle the entry
                entry.recycle();
            }

            return queued;
        }
```

我们之前介绍过, 从在缓存对象中分配的时候从queue 弹出一个entry, 会放到一个对象池里面, 而这里的` Entry<T> entry = newEntry(chunk, handle);`  就是从对象池中去取一个entry 对象, 然后将chunk 和handle 进行赋值, 然后通过`queue.offer(entry)`  加到queue, 我们回到free() 方法:

```java
   void free(PoolChunk<T> chunk, long handle, int normCapacity, PoolThreadCache cache) {
        if (chunk.unpooled) {
            int size = chunk.chunkSize();
            destroyChunk(chunk);
            activeBytesHuge.add(-size);
            deallocationsHuge.increment();
        } else {
            SizeClass sizeClass = sizeClass(normCapacity);
            if (cache != null && cache.add(this, chunk, handle, normCapacity, sizeClass)) {
                // cached so not free it.
                return;
            }

            freeChunk(chunk, handle, sizeClass);
        }
    }
```

这里加到缓存之后, 如果成功, 就会return, 如果不成功, 就会调用 ` freeChunk(chunk, handle, sizeClass);`  方法, 这个方法的意思是： 将原来给ByteBuf 分配的内存区段标记位未使用,跟进到 ` freeChunk()  方法：

```java
void freeChunk(PoolChunk<T> chunk, long handle, SizeClass sizeClass) {
        final boolean destroyChunk;
        synchronized (this) {
            switch (sizeClass) {
            case Normal:
                ++deallocationsNormal;
                break;
            case Small:
                ++deallocationsSmall;
                break;
            case Tiny:
                ++deallocationsTiny;
                break;
            default:
                throw new Error();
            }
            destroyChunk = !chunk.parent.free(chunk, handle);
        }
        if (destroyChunk) {
            // destroyChunk not need to be called while holding the synchronized lock.
            destroyChunk(chunk);
        }
    }
```

我们在跟到free() 方法中：

```java
 boolean free(PoolChunk<T> chunk, long handle) {
        chunk.free(handle);
        if (chunk.usage() < minUsage) {
            remove(chunk);
            // Move the PoolChunk down the PoolChunkList linked-list.
            return move0(chunk);
        }
        return true;
    }
```

`chunk.free(handle);` 的意思是通过chunk 释放一段连续的内存, 再跟回到free() 方法中：

```java
void free(long handle) {
        int memoryMapIdx = memoryMapIdx(handle);
        int bitmapIdx = bitmapIdx(handle);

        if (bitmapIdx != 0) { // free a subpage
            PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
            assert subpage != null && subpage.doNotDestroy;

            // Obtain the head of the PoolSubPage pool that is owned by the PoolArena and synchronize on it.
            // This is need as we may add it back and so alter the linked-list structure.
            PoolSubpage<T> head = arena.findSubpagePoolHead(subpage.elemSize);
            synchronized (head) {
                if (subpage.free(head, bitmapIdx & 0x3FFFFFFF)) {
                    return;
                }
            }
        }
        freeBytes += runLength(memoryMapIdx);
        setValue(memoryMapIdx, depth(memoryMapIdx));
        updateParentsFree(memoryMapIdx);
    }
```

if (bitmapIdx != 0)  这里判断是当前缓存区分配的级别是page 还是subpage, 如果是subpage , 则会找到相关的subpage 将其位标记为0， 如果不是subpage, 这里通过分配内存的反向标记, 将该内存标记为未使用. 回 到 PooledByteBuf 的 deallocate方法中：

```java
  @Override
    protected final void deallocate() {
        if (handle >= 0) {
            final long handle = this.handle;
            this.handle = -1;
            memory = null;
            chunk.arena.free(chunk, handle, maxLength, cache);
            recycle();
        }
    }
```

最后, 通过recycle()  将释放的ByteBuf 放入对象回收站

### 8. SocketChannel 读取ByteBuf 的过程

我们首先看NioEventLoop 的processSelectedKey 方法:

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        if (!k.isValid()) {
            final EventLoop eventLoop;
            try {
                eventLoop = ch.eventLoop();
            } catch (Throwable ignored) {
                // If the channel implementation throws an exception because there is no event loop, we ignore this
                // because we are only trying to determine if ch is registered to this event loop and thus has authority
                // to close ch.
                return;
            }
            // Only close ch if ch is still registerd to this EventLoop. ch could have deregistered from the event loop
            // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
            // still healthy and should not be closed.
            // See https://github.com/netty/netty/issues/5125
            if (eventLoop != this || eventLoop == null) {
                return;
            }
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
            return;
        }

        try {
            int readyOps = k.readyOps();
            // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
            // the NIO JDK channel implementation may throw a NotYetConnectedException.
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
                // See https://github.com/netty/netty/issues/924
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);

                unsafe.finishConnect();
            }

            // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
                ch.unsafe().forceFlush();
            }

            // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
            // to a spin loop
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
                if (!ch.isOpen()) {
                    // Connection already closed - no need to handle write.
                    return;
                }
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }

```

`if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0)` 这里的判断表示轮询到大事件是OP_READ 或者 OP_ACCEPT 事件.  如果当前 **NioEventLoop** 是work 线程的话, 那么这里就是OP_READ  事件, 也就是读事件, 表示客户端发来了数据流, 这里会调用 unsafe 的redis()  方法进行读取， 如果是work 线程, 那么这里的channel  是NioServerSocketChannel, 其绑定的 unsafe 是NioByteUnsafe, 这里会走到 NioByteUnsafe 的 read()方法中：

```java
   @Override
        public final void read() {
            final ChannelConfig config = config();
            final ChannelPipeline pipeline = pipeline();
            final ByteBufAllocator allocator = config.getAllocator();
            final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
            allocHandle.reset(config);

            ByteBuf byteBuf = null;
            boolean close = false;
            try {
                do {
                    byteBuf = allocHandle.allocate(allocator);
                    allocHandle.lastBytesRead(doReadBytes(byteBuf));
                    if (allocHandle.lastBytesRead() <= 0) {
                        // nothing was read. release the buffer.
                        byteBuf.release();
                        byteBuf = null;
                        close = allocHandle.lastBytesRead() < 0;
                        break;
                    }

                    allocHandle.incMessagesRead(1);
                    readPending = false;
                    pipeline.fireChannelRead(byteBuf);
                    byteBuf = null;
                } while (allocHandle.continueReading());

                allocHandle.readComplete();
                pipeline.fireChannelReadComplete();

                if (close) {
                    closeOnRead(pipeline);
                }
            } catch (Throwable t) {
                handleReadException(pipeline, byteBuf, t, close, allocHandle);
            } finally {
                // Check if there is a readPending which was not processed yet.
                // This could be for two reasons:
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
                //
                // See https://github.com/netty/netty/issues/2254
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
        }
```

首先获取SocketChannel 的 config, pipeline 等相关属性，final ByteBufAllocator allocator = config.getAllocator(); 这
一步是获取一个 ByteBuf 的内存分配器, 用于分配 ByteBuf。这里会走到 DefaultChannelConfig 的 getAllocator 方法中:

```java
  @Override
    public ByteBufAllocator getAllocator() {
        return allocator;
    }
```

这里返回的 DefaultChannelConfig  的成员变量, 我们看这个成员变量：

> ```
> private volatile ByteBufAllocator allocator = ByteBufAllocator.DEFAULT;
> ```

这里调用的 ByteBufAllocator 的DEFAULT属性, 跟进去:

> ```
> ByteBufAllocator DEFAULT = ByteBufUtil.DEFAULT_ALLOCATOR;
> ```

看到这里又调用了ByteBufUtil 的静态属性DEFAULT_ALLOCATOR, 再跟进去

> ```
> 
> static final ByteBufAllocator DEFAULT_ALLOCATOR;
> ```

DEFAULT_ALLOCATOR  这个属性是在static 块中初始化的, 我们跟到static 块中：

```java
 static {
        String allocType = SystemPropertyUtil.get(
                "io.netty.allocator.type", PlatformDependent.isAndroid() ? "unpooled" : "pooled");
        allocType = allocType.toLowerCase(Locale.US).trim();

        ByteBufAllocator alloc;
        if ("unpooled".equals(allocType)) {
            alloc = UnpooledByteBufAllocator.DEFAULT;
            logger.debug("-Dio.netty.allocator.type: {}", allocType);
        } else if ("pooled".equals(allocType)) {
            alloc = PooledByteBufAllocator.DEFAULT;
            logger.debug("-Dio.netty.allocator.type: {}", allocType);
        } else {
            alloc = PooledByteBufAllocator.DEFAULT;
            logger.debug("-Dio.netty.allocator.type: pooled (unknown: {})", allocType);
        }

        DEFAULT_ALLOCATOR = alloc;

        THREAD_LOCAL_BUFFER_SIZE = SystemPropertyUtil.getInt("io.netty.threadLocalDirectBufferSize", 64 * 1024);
        logger.debug("-Dio.netty.threadLocalDirectBufferSize: {}", THREAD_LOCAL_BUFFER_SIZE);

        MAX_CHAR_BUFFER_SIZE = SystemPropertyUtil.getInt("io.netty.maxThreadLocalCharBufferSize", 16 * 1024);
        logger.debug("-Dio.netty.maxThreadLocalCharBufferSize: {}", MAX_CHAR_BUFFER_SIZE);
    }
```

首先判断运行环境是不是安卓, 如果不是安卓, 在返回"pooled"  字符串保存到  allocType 中, 然后通过if 判断, 最后局部变量alloc = PooledByteBufAllocator.DEFAULT;  最后将alloc  赋值到成员变量 DEFAULT_ALLOCATOR, 我们跟到PooledByteBufAllocator 的 DEFAULT 属性中：

> ```
> public static final PooledByteBufAllocator DEFAULT =        new PooledByteBufAllocator(PlatformDependent.directBufferPreferred());
> ```

我们看到这里直接通过new 的方式, 创建了一个PooledByteBufAllocator 对象， 也就是基于申请一块连续内存进行缓冲区分配的缓存区分配器, 回到NioByteUnsafe 的 read()  方法中：

```java
   @Override
        public final void read() {
            final ChannelConfig config = config();
            final ChannelPipeline pipeline = pipeline();
            final ByteBufAllocator allocator = config.getAllocator();
            final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
            allocHandle.reset(config);

            ByteBuf byteBuf = null;
            boolean close = false;
            try {
                do {
                    byteBuf = allocHandle.allocate(allocator);
                    allocHandle.lastBytesRead(doReadBytes(byteBuf));
                    if (allocHandle.lastBytesRead() <= 0) {
                        // nothing was read. release the buffer.
                        byteBuf.release();
                        byteBuf = null;
                        close = allocHandle.lastBytesRead() < 0;
                        break;
                    }

                    allocHandle.incMessagesRead(1);
                    readPending = false;
                    pipeline.fireChannelRead(byteBuf);
                    byteBuf = null;
                } while (allocHandle.continueReading());

                allocHandle.readComplete();
                pipeline.fireChannelReadComplete();

                if (close) {
                    closeOnRead(pipeline);
                }
            } catch (Throwable t) {
                handleReadException(pipeline, byteBuf, t, close, allocHandle);
            } finally {
                // Check if there is a readPending which was not processed yet.
                // This could be for two reasons:
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
                //
                // See https://github.com/netty/netty/issues/2254
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
        }
```



这里ByteBufAllocator allocator = config.getAllocator()中的 allocator , 就是 PooledByteBufAllocator。
final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle() 是创建一个 handle,handle是对RecvByteBufAllocator 进行实际操作的对象, 我们跟进recvBufAllocHandle：

```java
 @Override
        public RecvByteBufAllocator.Handle recvBufAllocHandle() {
            if (recvHandle == null) {
                recvHandle = config().getRecvByteBufAllocator().newHandle();
            }
            return recvHandle;
        
```

这里是我们之前剖析过的逻辑, 如果不存在, 则创建handle 的实例. 同样, `allocHandle.reset(config)` 是将配置重置, 重置完配置后, 进行 `do-while` 	循环, 有关循环终止条件 `allocHandle.continueReading()`. 在  `do-while ` 循环中, 首先看 `byteBuf = allocHandle.allocate(allocator)` 这一步, 这里传入了刚才创建的allocate 对象 , 也就是PooledByteBufAllocator，这里会进入 DefaultMaxMessagesRecvByteBufAllocator 类的 allocate()方法中：

```java
   @Override
        public ByteBuf allocate(ByteBufAllocator alloc) {
            return alloc.ioBuffer(guess());
        }
```

这里的guess() 方法, 会调用AdaptiveRecvByteBufAllocator 的 guess()  方法：

```java
  @Override
        public int guess() {
            return nextReceiveBufferSize;
        }
```

这里会返回AdaptiveRecvByteBufAllocator 的成员变量nextReceiveBufferSize, 也就是下次所分配的缓冲区的大小,第一个分配的时候u也会分配初始大小, 也就是1024字节, 这样, alloc.ioBuffer(guess())  就会分配一个PooledByteBuf，我们跟到AbstractByteBufAllocator 的 ioBuffer 方法中：

```java
  public ByteBuf ioBuffer() {
        if (PlatformDependent.hasUnsafe()) {
            return directBuffer(DEFAULT_INITIAL_CAPACITY);
        }
        return heapBuffer(DEFAULT_INITIAL_CAPACITY);
    }
```

这里首先判断是否能够获取jdk 的unsafe 对象, 默认为true, 所以会走到directBuffer(initialCapacity) 中, 这里最终会分配一个 PooledUnsafeDirectByteBuf 对象 , 回到NioByteUnsafe 的read(） 方法中, 分配完了ByteBuf 之后, 再看这一步 allocHandle.lastBytesRead(doReadBytes(byteBuf))。

首先看参数doReadBytes(byteBuf)方法, 这步是将channel 中的数据读取到我们刚分配到了ByteBuf 中, 并返回读取到的字节数, 这里会调用NioSocketChannel 的 doReadBytes()方法：

```java
  @Override
    protected int doReadBytes(ByteBuf byteBuf) throws Exception {
        final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
        allocHandle.attemptedBytesRead(byteBuf.writableBytes());
        return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());
    }
```

首先拿到绑定在channel 中的handle, 因为我们已经创建了handle,所以这里会直接拿到, 再看allocHandle.attemptedBytesRead(byteBuf.writableBytes()) 这步, , byteBuf.writableBytes()  返回的是ByteBuf 可写的字节数, 也就是最多能从channel 中读取多少字节写到ByteBuf,  allocate 的 attemptedBytesRead 会把可写字节数设置到
DefaultMaxMessagesRecvByteBufAllocator 类 的 attemptedBytesRead 属 性 中 ， 跟 到
DefaultMaxMessagesRecvByteBufAllocator 中的 attemptedBytesRead 我们会看到：

```java
  @Override
        public void attemptedBytesRead(int bytes) {
            attemptedBytesRead = bytes;
        }
```



继续看doReadBytes()  方法, 往下看到最后, 通过`byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead())`  将JDK 底层的channel 数据写入到我们创建的ByteBuf 中, 并返回实际写入的字节数.  回到 NioByteUnsafe 的 read() 方法中继续看`allocHandle.lastBytesRead(doReadBytes(byteBuf))`  这一步, 刚才我们剖析过 `doReadBytes(byteBuf)`  返回的是刚才写入的ByteBuf 的字节数, 再看 lastBytesRead()   方法, 跟到DefaultMaxMessagesRecvByteBufAllocator 的lastBytesRead()   方法中:

```java
  public final void lastBytesRead(int bytes) {
            lastBytesRead = bytes;
            // Ignore if bytes is negative, the interface contract states it will be detected externally after call.
            // The value may be "invalid" after this point, but it doesn't matter because reading will be stopped.
            totalBytesRead += bytes;
            if (totalBytesRead < 0) {
                totalBytesRead = Integer.MAX_VALUE;
            }
        }
```

这里会赋值两个属性, lastBytesRead 代表最后读取的字节数, 这里赋值为我们刚才写入ByteBuf 的字节数, totalBytesRead 代表总共读取的字节数, 这里将写入的字节数相加。继续来到 NioByteUnsafe 的read(） 方法, 如果最后一次性读取数据为0, 说明已经将channel 中的数据全部读取完毕, 将新创建的ByteBuf 释放循环使用, 并跳出循环, `allocHandle.incMessagesRead(1);` 这步是增加消息的读取次数, 因为我们这里循环最多16次, 所以当消息增加次数增加到16 会结束循环。 读取完毕后, 会通过`pipeline.fireChannelReadComplete()`   将传入 channelRead 事件.

至此, 小伙伴应该有个疑问, 如果一次读取不完, 就传递channelRead 事件. 那么server 接受到的数据就有可能不是完整的, 其实关于这点, Netty 也做了相应的处理, 循环结束后, 会执行到`allocHandle.readComplete();`   这一步.

其实我们知道第一次分配ByteBuf 的初始容量是1024, 但是初始容量不一定一定满足所有的业务场景, netty 中, 将每次读取数据的字节数进行记录, 然后之后分配ByteBuf 的时候, 容量会尽可能的符合业务场景所需要的大小, 具体实现方式在 allocHandle.readComplete(）这一步体现的,  我们跟到 AdaptiveRecvByteBufAllocator 的 readComplete()  方法中:

```java
  @Override
        public void readComplete() {
            record(totalBytesRead());
        }

```

这里调用了record()  方法, 并传入了这一次所读取的字节总数, 跟到record() 方法:

```java
private void record(int actualReadBytes) {
            if (actualReadBytes <= SIZE_TABLE[Math.max(0, index - INDEX_DECREMENT - 1)]) {
                if (decreaseNow) {
                    index = Math.max(index - INDEX_DECREMENT, minIndex);
                    nextReceiveBufferSize = SIZE_TABLE[index];
                    decreaseNow = false;
                } else {
                    decreaseNow = true;
                }
            } else if (actualReadBytes >= nextReceiveBufferSize) {
                index = Math.min(index + INDEX_INCREMENT, maxIndex);
                nextReceiveBufferSize = SIZE_TABLE[index];
                decreaseNow = false;
            }
        }
```

首先看判断条件 `if (actualReadBytes <= SIZE_TABLE[Math.max(0, index - INDEX_DECREMENT - 1)]) ` . 这里index 是当前分配的缓存区大小所在的SIZE_TABLE 的索引, 将这个索引进行缩进, 然后根据缩进后的索引找出 SIZE_TABLE 所存储的内存值, 再判断是否大于等于这次读取的最大字节数， 如果条件成立, 说明分配的内存过大, 需要缩容操作, 我们看if 块中缩容相关的逻辑, 首先 `if (decreaseNow)`   会判断是否立即进行收缩操作, 通常第一次不会进行收缩操作, 然后会将 decreaseNow 设置为true, 代表下一次直接进行收缩操作, 假设需要立即进行收缩操作,我们看收缩操作的相关逻辑。

` index = Math.max(index - INDEX_DECREMENT, minIndex);` 这一步将索引缩进一步, 但不能小于最小索引值, 然后通过`nextReceiveBufferSize = SIZE_TABLE[index]` 获取设置索引之后的内存, 赋值在nextReceiveBufferSize , 也就是下次需要分配的大小, 下次就会根据这个大小分配ByteBuf了, 这样就实现了缩容操作.

再看 `else if (actualReadBytes >= nextReceiveBufferSize)` 这里判断这次读取字节的总量比上次分配的大小还要大, 则进行扩容操作. 扩容操作也非常简单, 索引步进 , 然后拿到步进后的索引锁对应的内存值, 作为下次所需要的分配的大小在NioByteUnsafe 的 read()  方法， 经过了缩容或者扩容操作后， 通过pipeline.fireChannelReadComplete()传播
ChannelReadComplete()事件 , 以上就是读取客户端消息的相关流程.






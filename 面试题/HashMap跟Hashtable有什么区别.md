#  HashMap跟Hashtable有什么区别?

HashMap和HashTable都是基于哈希表来实现键值映射的工具类,从公开的方法上来看，这两个类提供的，是一样的功能。都提供键值映射的服务，可以增、删、查、改键值对，可以对键、值、键值对提供遍历视图。支持浅拷贝，支持序列化。

- 从数据结构上看 :
   HashMap和HashTable都使用哈希表来存储键值对。在数据结构上是基本相同的，都创建了一个继承自Map.Entry的私有的内部类Entry，每一个Entry对象表示存储在哈希表中的一个键值对。可以说，有多少个键值对，就有多少个Entry对象.
   得出结论，HashMap/HashTable内部用Entry数组实现哈希表，而对于映射到同一个哈希桶（数组的同一个位置）的键值对，使用Entry链表来存储(解决hash冲突)。



```csharp
Entry对象唯一表示一个键值对，有四个属性：
-K key 键对象
-V value 值对象
-int hash 键对象的hash值
-Entry entry 指向链表中下一个Entry对象，可为null，表示当前Entry对象在链表尾部
```

- 从继承体系上看 :
   虽然都实现了Map、Cloneable、Serializable三个接口。但是HashMap继承自抽象类AbstractMap，而HashTable继承自抽象类Dictionary。其中Dictionary类是一个已经被废弃的类.

- 从公共方法上看 :
   HashMap是支持null键和null值的，而HashTable在遇到null时，会抛出NullPointerException异常。这并不是因为HashTable有什么特殊的实现层面的原因导致不能支持null键和null值，这仅仅是因为HashMap在实现时对null做了特殊处理，将null的hashCode值定为了0，从而将其存放在哈希表的第0个bucket中。在HashMap中不能由get()方法来判断HashMap中是否存在某个键， 而应该用containsKey()方法来判断。

- 从算法上看 :
   HashTable默认的初始大小为11，之后每次扩充为原来的2n+1。HashMap默认的初始化大小为16，之后每次扩充为原来的2倍。如果在创建时给定了初始化大小，那么HashTable会直接使用你给定的大小，而HashMap会将其扩充为2的幂次方大小。
   因为哈希表的大小为素数时，简单的取模哈希的结果会更加均匀.这样设计可以减少哈希值冲突.
   但是在取模计算时，如果模数是2的幂，那么我们可以直接使用位运算来得到结果，效率要大大高于做除法。hash计算的效率HashMap更胜一筹.HashMap由于使用了2的幂次方，所以在取模运算时不需要做除法，只需要位的与运算就可以了。但是由于引入的hash冲突加剧问题，HashMap在调用了对象的hashCode方法之后，又做了一些位运算在打散数据。

- 从线程安全上看 :
   Hashtable是线程安全的，它的每个方法中都加入了Synchronize方法。在多线程并发的环境下，可以直接使用Hashtable，不需要自己为它的方法实现同步.
   HashMap不是线程安全的，在多线程并发的环境下，可能会产生死锁等问题。使用HashMap时就必须要自己增加同步处理.HashMap进行同步： `Map m = Collections.synchronizeMap(hashMap);`
   虽然HashMap不是线程安全的，但是它的效率会比Hashtable要好很多。这样设计是合理的。在我们的日常使用当中，大部分时间是单线程操作的。HashMap把这部分操作解放出来了。当需要多线程操作的时候可以使用线程安全的ConcurrentHashMap。ConcurrentHashMap虽然也是线程安全的，但是它的效率比Hashtable要高好多倍。因为ConcurrentHashMap使用了分段锁，并不对整个数据进行锁定。

- 从遍历方式上看 :
   Hashtable、HashMap都使用了 Iterator。而由于历史原因，Hashtable还使用了Enumeration的方式 .

  

  HashMap和jdk1.8以后的HashTable迭代器使用fast-fail机制 , 当有其它线程改变了HashMap的结构（增加，删除，修改元素），将会抛出ConcurrentModificationException。不过，通过Iterator的remove()方法移除元素则不会抛出ConcurrentModificationException异常。

  ![img](https:////upload-images.jianshu.io/upload_images/8884551-9c116a6102a410b4.png?imageMogr2/auto-orient/strip|imageView2/2/w/688/format/webp)

  每次在发生增删改的时候都会出现modCount++的动作。而modcount可以理解为是当前hashtable的状态。每发生一次操作，状态就向前走一步。设置这个状态，主要是由于hashtable等容器类在迭代时，判断数据是否过时时使用的。尽管hashtable采用了原生的同步锁来保护数据安全。但是在出现迭代数据的时候，则无法保证边迭代，边正确操作。于是使用这个值来标记状态。一旦在迭代的过程中状态发生了改变，则会快速抛出一个异常，终止迭代行为。

- 从使用角度上看 :
   如果不需要线程安全，那么使用HashMap，如果需要线程安全，那么使用ConcurrentHashMap。HashTable已经被淘汰了，不要在新的代码中再使用它。
   每一版本的JDK，都会对HashMap和HashTable的内部实现做优化，比如JDK 1.8的红黑树优化。所以尽可能的使用新版本的JDK，会有性能上有提升。
   为什么HashTable已经淘汰了，还要优化它？因为有老的代码还在使用它，所以优化了它之后，这些老的代码也能获得性能提升。


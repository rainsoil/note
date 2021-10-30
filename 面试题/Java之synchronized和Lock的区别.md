

# Java之synchronized和Lock的区别

1. Lock是java的一个interface接口，而synchronized是Java中的关键字，synchronized是由JDK实现的，不需要程序员编写代码去控制加锁和释放；Lock的接口如下：

```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```

2. synchronized修饰的代码在执行异常时，jdk会自动释放线程占有的锁，不需要程序员去控制释放锁，因此不会导致死锁现象发生；但是，当Lock发生异常时，如果程序没有通过unLock()去释放锁，则很可能造成死锁现象，因此Lock一般都是在finally块中释放锁；格式如下：



```csharp
Lock lock = new LockImpl; // new 一个Lock的实现类
lock.lock();  // 加锁
try{
    //todo
}catch(Exception ex){
     // todo
}finally{
    lock.unlock();   //释放锁
}
```

3. Lock可以让等待锁的线程响应中断处理，如tryLock(long time, TimeUnit unit)，而synchronized却不行，使用synchronized时，等待的线程会一直等待下去，不能够中断，程序员无法控制；
4.  synchronized是非公平锁，Lock可以设置是否公平锁，默认是非公平锁；
5. Lock的实现类ReentrantReadWriteLock提供了readLock()和writeLock()用来获取读锁和写锁的两个方法，这样多个线程可以进行同时读操作；
6. Lock锁的范围有局限性，仅适用于代码块范围，而synchronized可以锁住代码块、对象实例、类；
7. Lock可以绑定条件，实现分组唤醒需要的线程；synchronized要么随机唤醒一个，要么唤醒全部线程。




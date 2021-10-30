# redis之事务

##   1. 官网介绍

https://redis.io/topics/transactions/
http://redisdoc.com/topic/transaction.html

## 2. 为什么要使用事务

我们知道Redis的单个命令是原子性的(比如`get`,`set`,`mget`,`mset`),如果涉及到多个命令的时候, 需要把多个命令作为一个不可分割的处理序列, 就需要用到事务. 

例如当使用`sentx` 实现分布式锁的时候，我们先`set`, 然后对`key` 设置`expire`, 防止`del` 发生异常的时候锁不会被释放, 业务处理完了之后再`del`, 这三个动作我们希望他们作为一组命令执行. 

  redis的事务涉及到四个命令:

- `multi`: 开启事务
- `exec`: 执行事务
- `discard`: 取消事务
- `watch`:监视事务



##  3. 事务的用法

案例场景: 

> tom 和 mic 各有 1000 元，tom 需要向 mic 转账 100 元。tom 的账户余额减少 100 元，mic 的账户余额增加 100 元。

```bash
127.0.0.1:6379> set tom 1000
OK
127.0.0.1:6379> set mic 1000
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> decrby tom 100
QUEUED
127.0.0.1:6379> incrby mic 100
QUEUED
127.0.0.1:6379> exec
1) (integer) 900
2) (integer) 1100
127.0.0.1:6379> get tom
"900" 
127.0.0.1:6379> get mic
"1100"
```



通过`multi` 的命令开启事务. 事务不能嵌套, 多个`multi` 命令的效果一样. 

`multi` 执行后, 客户端可以继续向服务器发送任意多条命令,这些命令不会立即被执行, 而是被放到一个队列中, 当`exec` 命名被调用时, 所有队列中的这些命令才会被执行. 

通过`exec` 命令执行事务, 如果没有执行`exec`, 所有的命令都不会被执行. 

如果中途不想执行事务了, 怎么办?

可以调用`discard` 清空事务队列, 放弃执行. 

```bash
multi
set k1 1
set k2 2
set k3 3
discard
```

##   4 `watch` 命令

在`redis` 中还提供了一个`watch` 命令.

他可以为`redis` 事务提供`CAS` 乐观锁行为(Check and Set / Compare and Swap) , 也就是多个线程更新变量的时候, 会跟原值做比较， 只有他没有被其他线程做修改的情况下, 才更新成新的值. 

我们可以用`watch`监视一个或者多个key, 如果开启事务后, 至少有一个被监视 key 键在`exec` 执行之前被修改了, 那么整个事务都会被取消(key提前过期除外). 可以使用`unwatch` 取消. 



## 5. 事务中可能遇到的问题?

我们把事务中可能遇到的问题分成两种:

1. 在执行`exec` 之前发生的错误
2. 在执行`exec` 之后发生的错误. 



### 5.1 在执行`exec` 之前发生的错误

比如, 入队的命令存在语法错误, 包括参数变量, 参数名等等(编译器错误)

```bash
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set test 666
QUEUED
127.0.0.1:6379> hset test1 2673
(error) ERR wrong number of arguments for 'hset' command
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
```



在这种情况下, 事务会被拒绝执行, 也就是队列中的所有的命令都不会被执行 

###  5.2 在执行`exec` 之后发生错误

比如: 类型错误, 比如对String执行了`Hset` 的命令, 这是一种运行时错误 

```hash
127.0.0.1:6379> flushall
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set k1 1
QUEUED
127.0.0.1:6379> hset k1 a b
QUEUED
127.0.0.1:6379> exec
1) OK
2) (error) WRONGTYPE Operation against a key holding the wrong kind of value
127.0.0.1:6379> get k1
"1"
```

 最后我们发现`set k1 1`的命令是成功的, 也就是在这种发生了运行时异常的情况下, 只有错误的命令不会被执行, 但是其他命令没有收到影响. 

这个显然不符合我们对原子性的定义, 也就是我们没办法用redis的这种事务机制来实现原子性, 保证事务的一致. 


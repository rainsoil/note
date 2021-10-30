# redis使用lua脚本

`lua`(`ˈluə`) 是一种轻量级脚本语言, 他是用C语言编写的, 跟数据库的存储过程有点类型, 使用`lua` 脚本来执行redis命令的好处: 

1. 一次发送多条命令, 较少网络开销
2.  redis会将整个脚本来作为一个整体执行, 不会被其他请求打断, 保持原子性. 
3. 对于复杂的组合命令, 我们可以放到文件中, 可以实现程序之间的命令复用. 

##  1 在Redis 中调用`lua` 脚本

使用`eval`(`ɪ'væl`) 方法, 语法格式为: 

```bash
redis> eval lua-script key-num [key1 key2 key3 ....] [value1 value2 value3 ....]
```

- eval 代表执行 Lua 语言的命令。
- lua-script 代表 Lua 语言脚本内容。
- key-num 表示参数中有多少个 key，需要注意的是 Redis 中 key 是从 1 开始的，如果没有 key 的参数，那么写 0。
- [key1 key2 key3…]是 key 作为参数传递给 Lua 语言，也可以不填，但是需要和 key-num 的个数对应起来。
- [value1 value2 value3 ….]这些参数传递给 Lua 语言，它们是可填可不填的

**示例**: 返回一个字符串, 0个参数

```bash
redis> eval "return 'Hello World'" 0

```

##  2. 在`lua` 脚本中调用`redis` 的命令

使用`redis.call(command, key [param1, param2…])` 进行操作。语法格式: 

```bash
redis> eval "redis.call('set',KEYS[1],ARGV[1])" 1 lua-key lua-value

```

- `command` 是命令，包括 set、get、del 等。
- `key` 是被操作的键
- `param1,param2…` 代表给`key`的参数

注意跟`java` 不一样, 定义只有形参, 调用只有实参

`lua` 是在调用时用`key`表示形参, `argv` 表示参数值(实参)



###  2.1 设置键值对

在redis中调用`lua` 脚本执行redis命令

```bash
redis> eval "return redis.call('set',KEYS[1],ARGV[1])" 1 test 2673
redis> get test	
```

以上命令相当于`set test 2673`

在`redis-cli` 中直接写`lua` 脚本不够方便, 也不能实现编辑和复用, 通常我们会把脚本放在文件里, 然后执行这个文件. 



### 2.2 在redis中调用`lua` 脚本中的命令, 操作redis

创建`lua` 脚本文件: 

```bash
cd /usr/local/soft/redis5.0.5/src
vim test.lua
```

`lua` 脚本内容： 先设置, 再取值

```bash
redis.call('set','test','lua666')
return redis.call('get','test')
```

在redis 客户端中调用`lua` 脚本

```bash
cd /usr/local/soft/redis5.0.5/src
redis-cli --eval test.lua 0
```

得到返回值: 

```bash
[root@localhost src]# redis-cli --eval test.lua 0
"lua666"
```

##  3 案例： 对IP 进行限流

需求: 在X 秒中只能访问Y次. 

设计思路: 用key 记录ip, 用value记录访问数

拿到`ip`后，对`ip`+1. 如果是第一次访问, 对key设置过期时间(参数1). 否则判断次数, 超过限定的次数(参数2), 返回0. 如果没有超过次数则返回1. 超过时间, key过期, 可以再次访问. 

`KEY[1]`是ip,`ARGV[1]`是过期时间X, `ARGV[2]` 是限制访问次数Y

```lua
-- ip_limit.lua
-- IP 限流，对某个 IP 频率进行限制 ，6 秒钟访问 10 次
local num=redis.call('incr',KEYS[1])
if tonumber(num)==1 then
    redis.call('expire',KEYS[1],ARGV[1])
    return 1
 elseif tonumber(num)>tonumber(ARGV[2]) then
    return 0
 else
    return 1
end
```



6 秒钟内限制访问10次, 调用次数(连续调用10次)

```bash
./redis-cli --eval "ip_limit.lua" app:ip:limit:192.168.8.111 , 6 10 
```

- `app:ip:limit:192.168.8.111` 是key值, 后面是参数值, 中间要加上一个空格和一个逗号, 再加上一个空格

   即: `./redis-cli –eval [lua 脚本] [key…]空格,空格[args…]`

- 多个参数之间用一个空格分割

```java
public class LuaLimitTest {
    public static void main(String[] args) {
        Jedis jedis = getJedisUtil();
        jedis.eval("return redis.call('set',KEYS[1],ARGV[1])", 1, "test:lua:key", "2673lua");
        System.out.println(jedis.get("test:lua:key"));
        for (int i = 0; i < 10; i++) {
            limit();
        }
    }


    /**
     * 10秒内限制访问5次
     */
    public static void limit() {
        Jedis jedis = getJedisUtil();
        // 只在第一次对key设置过期时间
        String lua = "local num = redis.call('incr', KEYS[1])\n" +
                "if tonumber(num) == 1 then\n" +
                "\tredis.call('expire', KEYS[1], ARGV[1])\n" +
                "\treturn 1\n" +
                "elseif tonumber(num) > tonumber(ARGV[2]) then\n" +
                "\treturn 0\n" +
                "else \n" +
                "\treturn 1\n" +
                "end\n";
        Object result = jedis.evalsha(jedis.scriptLoad(lua), Arrays.asList("localhost"), Arrays.asList("10", "5"));
        System.out.println(result);
    }

    private static Jedis getJedisUtil() {
        String ip = ResourceUtil.getKey("redis.host");
        int port = Integer.valueOf(ResourceUtil.getKey("redis.port"));
        String password = ResourceUtil.getKey("redis.password");
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        JedisPool pool = new JedisPool(jedisPoolConfig, ip, port, 10000, password);
        return pool.getResource();
    }


}

```



## 4. 缓存`lua` 脚本

### 4.1 为什么要缓存

在脚本比较长的情况下, 如果每次调用脚本都需要将整个脚本传给redis服务端, 会产生比较大的网络开销. 为了解决这个问题, redis 提供了`EVALSHA`命令, 允许开发者通过脚本内容的SHA1 摘要来执行脚本. 



### 4.2 如何缓存

redis在执行`script load` 命令时会计算脚本的SHA1 摘要并记录在脚本缓存中, 执行`EVALSHA`命令时redis 会根据提供的摘要从脚本中查找对应的脚本内容, 如果找到了则执行脚本, 否则会返回错误: `"NOSCRIPT No matching script. Please use EVAL."`. 

```bash
127.0.0.1:6379> script load "return 'Hello World'"
"470877a599ac74fbfda41caa908de682c5fc7d4b" 127.0.0.1:6379> evalsha "470877a599ac74fbfda41caa908de682c5fc7d4b" 0
"Hello World"		
```

自乘案例: 

redis有`incrby` 这样的自增命令, 但是没有自乘, 比如乘以3, 乘以5

我们可以写一个自乘的运算,让他乘以后面的参数

```lua
local curVal = redis.call("get", KEYS[1])
if curVal == false then
 curVal = 0
else
 curVal = tonumber(curVal)
end
curVal = curVal * tonumber(ARGV[1])
redis.call("set", KEYS[1], curVal)
return curVal
```

把这个脚本换成单行, 语句之间使用分号隔开

```lua
local curVal = redis.call("get", KEYS[1]); if curVal == false then curVal = 0 else curVal = tonumber(curVal) end; curVal = curVal * tonumber(ARGV[1]); redis.call("set", KEYS[1], curVal); return curVal	
```



使用`script load` 命令

```bash
127.0.0.1:6379> script load 'local curVal = redis.call("get", KEYS[1]); if curVal == false then curVal = 0 else curVal =
tonumber(curVal) end; curVal = curVal * tonumber(ARGV[1]); redis.call("set", KEYS[1], curVal); return curVal'
"be4f93d8a5379e5e5b768a74e77c8a4eb0434441"
```

调用

```bash
127.0.0.1:6379> set num 2
OK
127.0.0.1:6379> evalsha be4f93d8a5379e5e5b768a74e77c8a4eb0434441 1 num 6
(integer) 12
```



### 5. 脚本超时

redis的指令执行本身是单线程的, 这个线程还要执行客户端的`lua` 脚本, 如果`lua` 脚本执行超时或者陷入了死循环, 是不是就没办法为客户端提供服务了呢? 

```bash
eval 'while(true) do end' 0
```

为了防止某个脚本执行时间过长导致redis无法提供服务,redis提供了`lua-time-limit` 参数限制脚本的最长运行时间,默认为5秒钟

```properties
lua-time-limit 5000（redis.conf 配置文件中）

```

当脚本运行时间超时这一限制后, redis将开始接收命名但不会执行(以确保脚本的原子性, 因为此时脚本并没有被终止), 而是会返回`BUSY` 错误. 

redis 提供了一个`script kill` 命令来终止脚本的运行。 新开一个客户端: 

```bash 
script kill

```



如果当前执行的`lua`脚本对redis的数据进行了修改(`set`,`del`等), 那么通过`script kill`  命令是不能终止脚本运行的. 

```bash
127.0.0.1:6379> eval "redis.call('set','gupao','666') while true do end" 0

```

 因为要保证脚本运行的原子性, 如果脚本执行了一部分终止, 那就违背了脚本原子性的要求, 最终要保证脚本要么全部执行, 要么都不执行. 

```bash
127.0.0.1:6379> script kill
(error) UNKILLABLE Sorry the script already executed write commands against the dataset. You can either wait the script termination or kill the server in a hard way using the SHUTDOWN NOSAVE command.
```



遇到这种情况, 只能通过`shutdown nosave` 命名来强制终止redis

`shutdown nosave` 和`shutdown `的区别在于`shutdown nosave` 不会进行持久化操作,意味着发生在上一次快照后的数据库修改都会丢失. 

总结: 如果我们有一些特殊的需求, 可以用`lua` 来实现, 但是要注意那些耗时的操作. 


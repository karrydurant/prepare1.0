# Redis

## 基于 Redis 实现分布式锁

### ✔️基于 Redis 实现一个最简易的分布式锁

不论是本地锁还是分布式锁，核心都在于"互斥"。在 Redis 中，SETNX 命令可以帮助实现互斥。SETNX 即 SET if Not eXists，如果 key 不存在的话才会设置 key 的值。如果 key 已经存在，SETNX 什么也不做。

```bash
> SETNX lockKey uniqueValue
(integer) 1
> SETNX lockKey uniqueValue
(Integer) 0
```

释放锁的话，直接通过 DEL 命令删除对应的 key 即可

```bash
>DEL lockKey
(integer) 1
```

为防止误删到其他锁，这里建议使用 Lua 脚本先比对 key 对应的 value 是否为加锁时写入的唯一值，校验通过后再删除。

选用 Lua 脚本是为了保证解锁操作的原子性。因为 Redis 在执行 Lua 脚本时，可以以原子性的方式执行，从而保证了锁释放操作的原子性。

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

这是一种最简易的 Redis 分布式锁实现，实现方式比较简单，性能也很高效。不过，这种方式实现分布式锁存在一些问题，比如释放锁的逻辑突然挂掉，可能会导致锁无法被释放，进而造成共享资源无法再被其他线程/进程访问。

### ✔️给锁设置一个过期时间

为了避免锁无法被释放，我们可以想到的一个解决办法就是：给这个 key（也就是锁）设置一个过期时间。

```bash
127.0.0.1:6379> SET lockKey uniqueValue EX 3 NX
OK
```

lockKey：加锁的锁名

uniqueValue：能够唯一标识锁的随机字符串

NX：只有当 lockKey 对应的 key 值不存在的时候才能 SET 成功

EX：过期时间设置（秒为单位）。EX 3 表示这个锁有一个 3 秒的自动过期时间。与 EX 对应的是 PX（毫秒为单位），这两个都是过期时间设置。

一定要保证设置指定 key 的值和过期时间是一个原子操作。不然的话，依然可能会出现锁无法被释放的问题。

为什么要用 SET NX EX，而不是先 SETNX 再 EXPIRE 呢？早期常见的 SETNX 后再 EXPIRE 是两步操作，如果客户端在 SETNX 成功后、EXPIRE 前崩溃，会留下永久锁。

这种方式同样存在漏洞：如果操作共享资源的时间大于过期时间，就会出现锁提前过期，进而导致分布式锁直接失效。如果锁的超时时间设置过长，又会影响到性能。

### 如何实现锁的优雅续期

对于 Java ，已经有了现成的解决方案：Redisson

Redisson 是一个开源的 Java 语言 Redis 客户端，提供了很多开箱即用的功能，不仅仅包括多种分布式锁的实现。并且，Redisson 还支持 Redis 单机、Redis Sentinel、Redis Cluster 等多种部署架构。

Redisson 中的分布式锁自带自动续期机制，使用起来非常简单，原理也比较简单，其提供了一个专门用来监控和续期锁的 Watch Dog，如果操作共享资源的线程还未执行完的话，Watch Dog 会不断地延长锁的过期时间，进而保证锁不会因为超时而被释放。

默认情况下，每过 10 秒，看门狗就会执行续期操作，将锁的超时时间设置为 30 秒。看门狗续期前也会先判断是否需要执行续期操作，需要才会执行续期，否则取消续期操作。

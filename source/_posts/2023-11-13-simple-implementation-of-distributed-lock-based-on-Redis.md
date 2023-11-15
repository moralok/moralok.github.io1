---
title: 基于 Redis 的分布式锁的简单实现
date: 2023-11-13 14:02:49
tags:
    - distributed lock
    - redis
---

在分布式应用中，并发访问资源需要谨慎考虑。比如读取和修改保存并不是一个原子操作，在并发时，就可能发生修改的结果被覆盖的问题。

{% asset_img "Pasted image 20231110180458.png" 分布式应用中的并发问题 %}

很多人都了解在必要的时候需要使用分布式锁来限制程序的并发执行，但是在具体的细节上，往往并不正确。

## 基于 Redis 的分布式锁简单实现

本质上要实现的目标就是在 Redis 中占坑，告诉后来者资源已经被锁定，放弃或者稍后重试。Redis 原生支持 set if not exists 的语义。

```console
> setnx lock:user1 true
OK

... do something

> del lock:user1
(integer) 1
```

### 死锁问题

#### 问题一：异常引发死锁 1

如果在处理过程中，程序出现异常，将导致 del 指令没有执行成功。锁无法释放，其他线程将无法再获取锁。

{% asset_img "Pasted image 20231110182254.png" 异常引发死锁 1 %}

#### 改进一：设置超时时间

对 key 设置过期时间，如果在处理过程中，程序出现异常，导致 del 指令没有执行成功，设置的过期时间一到，key 将自动被删除，锁也就等于被释放了。

```console
> setnx lock:user1 true
OK
> expire lock:user1 5

... do something

> del lock:user1
(integer) 1
```

#### 问题二：异常引发死锁 2

事实上，上述措施并没有彻底解决问题。如果在设置 key 的超时时间之前，程序出现异常，一切仍旧会发生。

{% asset_img "Pasted image 20231110190407.png" 异常引发死锁 2 %}

本质原因是 setnx 和 expire 两个指令不是一个原子操作。那么是否可以使用 Redis 的事务解决呢？不行。因为 expire 依赖于 setnx 的执行结果，如果 setnx 没有成功，expire 就不应该执行。

#### 改进二：setnx + expire 的原子指令

如果 setnx 和 expire 可以用一个原子指令实现就好了。

{% asset_img "Pasted image 20231110190442.png" setnx + expire 原子指令 %}

##### 基于原生指令的实现

在 Redis 2.8 版本中，Redis 的作者加入 set 指令扩展参数，允许 setnx 和 expire 组合成一个原子指令。

```console
> set lock:user1 true ex 5 nx
OK

... do something

> del lock:user1
(integer) 1
```

##### 基于 Lua 脚本的实现

除了使用原生的指令外，还可以使用 Lua 脚本，将多个 Redis 指令组合成一个原子指令。

```lua
if redis.call('setnx', KEYS[1], ARGV[1]) == 1 then
  redis.call('expire', KEYS[1], ARGV[2])
  return true
else
  return false
end
```

### 超时问题

基于 Redis 的分布式锁还会面临超时问题。如果在加锁和释放之间的处理逻辑过于耗时，以至于超出了 key 的过期时间，锁将在处理结束前被释放，就可能发生问题。

{% asset_img "Pasted image 20231110191738.png" 其他线程提前进入临界区 %}

#### 问题一：其他线程提前进入临界区

如果第一个线程因为处理逻辑过于耗时导致在处理结束前锁已经被释放，其他线程将可以提前获得锁，临界区的代码将不能保证严格串行执行。

#### 问题二：错误释放其他线程的锁

如果在第二个线程获得锁后，第一个线程刚好处理逻辑结束去释放锁，将导致第二个线程的锁提前被释放，引发连锁问题。

#### 改进一：不要用于较长时间的任务

与其说是改进，不如说是注意事项。如果真的出现问题，造成的数据错误可能需要人工介入解决。

如果真的存在这样的业务场景，应考虑使用其他解决方案加以优化。

#### 改进二：使用 watchdog 实现锁续期

为 Redis 的 key 设置过期时间，其实是为了解决死锁问题而做出的兜底措施。可以为获得的锁设置定时任务定期地为锁续期，以避免锁被提前释放。

```java
private void scheduleRenewal() {
    String value = lockValue.get();
    ScheduledFuture<?> scheduledFuture = sScheduler.scheduleAtFixedRate(
        () -> this.renewal(value), RENEWAL_INTERVAL, RENEWAL_INTERVAL, TimeUnit.MILLISECONDS
    );
    renewalTask.set(scheduledFuture);
}
```

但是这个方式仍然不能避免解锁失败时的其他线程的等待时间。

#### 改进三：加锁时指定 tag

可以将 set 指令的 value 参数设置为一个随机数，释放锁时先匹配持有的 tag 是否和 value 一致，如果一致再删除 key，以此避免锁被其他线程错误释放。

##### 基于原生指令的实现

```console
tag = random.nextint()
if redis.set(key, tag, nx= True, ex=5):
	do_something()
	redis.delifequals(key, tag)
```

但是注意，Redis 并没有提供语义为 delete if equals 的原子指令，这样的话问题并不能被彻底解决。如果在第一个线程判断 tag 是否和 value 相等之后，第二个线程刚好获得了锁，然后第一个线程因为匹配成功执行删除 key 操作，仍然将导致第二个线程获得的锁被第一个线程错误释放。

{% asset_img "Pasted image 20231110193919.png" 错误释放其他线程的锁 %}

##### 基于 Lua 脚本的实现

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("del", KEYS[1])
else
  return 0
end
```

### 可重入性

可重入性是指线程在已经持有锁的情况下再次请求加锁，如果一个锁支持同一个线程多次加锁，那么就称这个锁是可重入的，类似 Java 的 ReentrantLock。

#### 使用 ThreadLocal 实现锁计数

Redis 分布式锁如果要支持可重入，可以使用线程的 ThreadLocal 变量存储当前持有的锁计数。但是在多次获得锁后，过期时间并没有得到延长，后续获得锁后持有锁的时间其实比设置的时间更短。

```java
private ThreadLocal<Integer> lockCount = ThreadLocal.withInitial(() -> 0);

public boolean tryLock() {  
    Integer count = lockCount.get();  
    if (count != null && count > 0) {  
        lockCount.set(count + 1);   
        return true;    
    }  
    String result = commands.set(lockKey, lockValue.get(), SetArgs.Builder.nx().px(RedisLockManager.LOCK_EXPIRE));  
    if ("OK".equals(result)) {  
        lockCount.set(1);  
        scheduleRenewal();  
        return true;    
    }  
    return false;  
}
```

#### 使用 Redis hash 实现锁计数

还可以使用 Redis 的 hash 数据结构实现锁计数，支持重新获取锁后重置过期时间。

```lua
if (redis.call('exists', KEYS[1]) == 0) then 
	redis.call('hset', KEYS[1], ARGV[2], 1);
	redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
    end;
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
	redis.call('hincrby', KEYS[1], ARGV[2], 1);
	redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
return redis.call('pttl', KEYS[1]);
```

书的作者**不推荐使用可重入锁**，他提出可重入锁会加重客户端的复杂度，如果在编写代码时注意在逻辑结构上进行调整，完全可以避免使用可重入锁。

## 代码实现

[redis-lock](https://github.com/moralok/redis-lock)

## 参考文章

- 《Redis 深度历险，核心原理与应用实践》
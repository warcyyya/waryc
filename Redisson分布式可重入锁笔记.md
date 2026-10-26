# Redisson 分布式可重入锁笔记

## 一、Redis 中如何保存锁

Redisson 通常使用 Redis Hash 保存锁：

```text
Key:   lock:order
Field: 客户端ID:线程ID
Value: 重入次数
```

例如 `thread1: 2` 表示 thread1 持有锁，并且已经重入 2 次。Hash 的 Key 还会设置过期时间，避免程序崩溃后永久占锁。

## 二、获取锁

`redissonClient.getLock("lock:order")` 只是得到锁对象；真正访问 Redis 发生在 `tryLock()` 或 `lock()` 时。

加锁 Lua 脚本的核心逻辑：

```lua
if 锁不存在 then
    创建 Hash
    重入次数设为 1
    设置过期时间
    return nil
end

if 锁属于当前线程 then
    重入次数加 1
    刷新过期时间
    return nil
end

return 当前锁的剩余有效时间
```

Redisson 对返回值的约定是：

```text
null      加锁成功
数字      锁被其他线程占用，数字是剩余有效时间
```

这里的 `null` 是 Lua 脚本的返回约定，不等同于直接执行 Redis `TTL` 命令的结果。

## 三、可重入锁

同一个线程可以重复获取同一把锁：

```java
method1() {
    lock.tryLock();   // 次数 1
    method2();
    lock.unlock();    // 次数减为 0
}

method2() {
    lock.tryLock();   // 次数 2
    lock.unlock();    // 次数减为 1
}
```

只有重入次数减到 `0` 时，锁才会真正从 Redis 中删除。

## 四、`HEXISTS` 的作用

```redis
HEXISTS key field
```

用于判断 Hash 中某个 field 是否存在：

```text
1 = 存在
0 = 不存在
```

释放锁时，Redisson 用它检查当前线程是不是持有者：

```lua
if HEXISTS(key, threadId) == 0 then
    return
end
```

区别：

```text
EXISTS   判断整个 Redis Key 是否存在
HEXISTS  判断 Hash 内某个 Field 是否存在
```

## 五、释放锁

`unlock()` 的核心逻辑：

```lua
if 当前线程不是持有者 then
    释放失败
end

重入次数减 1

if 重入次数 > 0 then
    刷新过期时间
else
    删除锁
    发布释放消息
end
```

释放锁必须放在 `finally` 中：

```java
try {
    // 业务代码
} finally {
    lock.unlock();
}
```

## 六、看门狗（WatchDog）自动续期

没有指定固定租期时，Redisson 会启动看门狗：

```java
lock.lock();
lock.tryLock();
```

默认锁有效期通常为 30 秒，看门狗大约每 10 秒续期一次：

```text
t=0 秒：有效期到 30 秒
t=10 秒：重新设置为 30 秒
t=20 秒：重新设置为 30 秒
```

每次是重新设置完整的 30 秒，不是只增加 10 秒。只有当前线程仍持有锁时才会续期。

重入次数降为 `0` 后：

```text
删除锁
停止看门狗
通知等待线程
```

程序宕机后看门狗停止，锁最终自动过期，从而避免永久死锁。

指定固定 `leaseTime` 时不会启动看门狗：

```java
lock.tryLock(5, 60, TimeUnit.SECONDS);
```

这表示锁最多持有 60 秒，超过后自动过期。

## 七、等待获取锁

如果锁被其他线程占用：

```text
尝试加锁
  -> 锁被占用
  -> 订阅释放消息
  -> 收到消息
  -> 再次尝试加锁
```

等待时间用完仍未成功时，`tryLock` 返回 `false`。无参 `tryLock()` 通常立即尝试，失败就返回。

## 八、完整执行过程

```text
method1 加锁，次数 1
  -> method2 加锁，次数 2
  -> method2 解锁，次数 1
  -> method1 解锁，次数 0
  -> 删除 Redis 锁并通知等待者
```

## 记忆口诀

```text
Hash 保存锁
Field 标识线程
Value 保存重入次数
HEXISTS 判断是不是自己
null 表示加锁成功
数字表示锁被占用
次数归零才真正释放
WatchDog 负责自动续期
finally 中必须 unlock
```

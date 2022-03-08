### 前言

在项目中某些场景需要先读取值，然后修改再保存，在这个过程中如果有并发请求，可能会突破程序设置的边界：在库存为 1 的情况下，同时有两个请求下单购买，如果不加处理，库存和订单总要异常一个。在单体服务中使用语言提供的锁就可以解决这一问题，但是在集群环境下，多节点的无状态服务，并发请求可能会通过负载均衡分发到不同的节点上，这时候就需要一个能在这种环境下也能生效的分布式锁了。

### 实现

利用 Redis 实现分布式锁，实际上就是在 Redis 里占个坑，当别的请求也要占坑时，发现已经没坑了，只能放弃这次请求。

占坑正确的操作是使用官方提供的 setex 指令，但 setex 指令是在 Redis 2.8 版本中加入的，那么在此之前，是如何实现的呢？

占坑使用 setnx(set if not exists) 指令，只有一个客户端可以设置值成功，调用 del 指令删除 key 释放锁：

```shell
> setnx key 1
(integer) 1
> setnx key 1
(integer) 0
...

> del key
(integer) 1
```

看上去没什么问题，但是如果在程序执行到释放锁这一步之前就挂掉了，那么这个锁将永远无法释放。

所以在成功设置锁之后，可以给锁设置一个过期时间，比如 5s，这样即使程序异常退出，锁依旧会在 5s 后自动释放

```shell
> setnx key 1
(integer) 1
> expire key 5
(integer) 1
...

> del key
(integer) 1
```

这样解决了异常情况下锁永远无法释放的情况，但是因为 setnx 和 expire 指令不是原子操作，很可能 setnx 指令执行成功了，但是 expire 指令却没有执行成功。虽然这种情况发生的可能很小，但是凡是可能会发生的，最终一定会发生，如果没有更方便、安全的替代方案，这种实现方案也能勉强接受。

好在有更好的实现方案：Redis 在 2.6 版本加入了 eval 指令，使得客户端可以执行 lua 脚本，同时当 lua 脚本在执行的时候，不会有其他脚本和命令同时执行。从别的客户端的视角来看，一个 lua 脚本要么不可见，要么已经执行完。

所以这里可以使用 lua 脚本保证 setnx 和 expire 指令一起执行：

```lua
local value = redis.call('setnx', KEYS[1], 1)
if value == 1 then
  redis.call('expire', KEYS[1], ARGV[1])
end
return value
```

在 redis-cli 中使用 eval 执行这条 lua 脚本是这样的：

```shell
eval "local value = redis.call('setnx', KEYS[1], 1) if value == 1 then redis.call('expire', KEYS[1], ARGV[1]) end return value" 1 test 5
```

其中 eval 后跟的是 lua 脚本，1 是 key 的数量，test 是 key 的值（lua 中通过 KEYS 取），5 是参数（lua 中通过 ARGV 取），在 golang 中则需要使用 `github.com/go-redis/redis/v7` ：

```go
client.Eval(luaScript, []string{test}, 5)
```

在 Redis 2.8 中加入了 setex 指令，使得这一操作更加方便：

```shell
setex key seconds value
```

直接将 setnx 和 expire 两条指令合二为一，官方逼死同人这是，所以占坑直接使用 setex 指令即可：

```go
client.SetNX(key, value, expire)
```

嗯，go 的这个 client 包中依旧是叫 setNX ，不过加了个持续时间参数。

 对于释放锁，一般情况下使用 del 指令将其删除即可，但是如果集群中某个节点故障，导致本来 50 ms 就可以处理完并响应客户端的请求耗时超过了锁设置的超时时间，那么很有可能导致超时的 A 请求错误的释放了正在使用的 B 请求的锁。避免这种情况，就需要在 setex 时将能够标记请求的值设置到 key 中，在 del 时传入标记，利用 lua 脚本判断同时执行 get 和 del 指令：

```lua
local old = redis.call('get', KEYS[1])
if old == ARGV[1] then
  return redis.call('del', KEYS[1])
end
return 0
```

```go
client.Eval(unlockLua, []string{key}, val).Int64()
```

完整版 go 实现：

````go
import "time"

type (
	LockRedis struct{}
)

unlockLua := ```
local old = redis.call('get', KEYS[1])
if old == ARGV[1] then
	return redis.call('del', KEYS[1])
end
return 0
```
)

func (l *LockRedis) Lock(key, val string, expire time.Duration) (bool, error) {
	ok, err := RedisConn.SetNX(key, val, expire).Result()
	return ok, err
}

func (l *LockRedis) UnLock(key, val string) (bool, error) {
	value, err := RedisConn.Eval(unlockLua, []string{key}, val).Int64()
	if err != nil {
		return false, err
	}

	return value == 1, err
}
````

到这，基本可以应对绝大部分情况了，但是仍然无法解决 Redis 发生故障，主从切换时同时有两个请求获取到锁的问题……

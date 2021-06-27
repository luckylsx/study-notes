## redis锁使用的正确姿势

可靠的分布式锁，要具备以下几个特性

1. 互斥性。（在任意时刻，只有一个客户端能持有锁）
2. 不会发生死锁。(即使有一个客户端在持有琐的期间崩溃而没有主动释放锁，也能保证后续其他客户端能加锁)
3. 具有容错性。（只要大部分的Redis正常运行，客户端就可以加锁和解锁）
4. 解铃还需系铃人。（加锁和解锁必须是同一个客户端，客户端自己不能把别人加的锁给解了）
5. 锁不能自己失效。（正常执行过程中锁不能因为某些原因自己失效，会造成多个程序运行同一个任务）

### 错误案例

#### setNx

SETNX key value

redis> SETNX job "programmer"    # job 设置成功

(integer) 1

**条件分析**

保证了redis里只有唯一的key存在

有效时间避免了死锁的发生

但不满足3，4，5特点。

#### 采用Lua脚本实现

golang实现

```golang
package redis

import (
	"context"
	"fmt"
	"math/rand"
	"sync"
	"time"

	"github.com/go-kratos/kratos/v2/log"
	"github.com/go-redis/redis/v8"
)

const (
	_LockDistributedLua = "local v;" +
		"v = redis.call('setnx',KEYS[1],ARGV[1]);" +
		"if tonumber(v) == 1 then\n" +
		"    redis.call('expire',KEYS[1],ARGV[2])\n" +
		"end\n" +
		"return v"

	_UnLockDistributedLua = "if redis.call('get',KEYS[1]) == ARGV[1]\n" +
		"then\n" +
		"    return redis.call('del',KEYS[1])\n" +
		"else\n" +
		"    return 0\n" +
		"end"

	_DistributedTimeOut = 4
	_DistributedSuccess = 1
)

var (
	_LockDistributedLuaScript   = redis.NewScript(_LockDistributedLua)
	_UnLockDistributedLuaScript = redis.NewScript(_UnLockDistributedLua)
)

type Option struct {
	Addrs        []string
	Pwd          string
	DB           int
	DialTimeout  time.Duration
	WriteTimeout time.Duration
	ReadTimeout  time.Duration
}

type Redis struct {
	cluster     *redis.ClusterClient
	single      *redis.Client
	clusterMode bool
	mutex       *sync.Mutex
}
func NewRedis(c *Option) *Redis {
	log := log.NewHelper(log.DefaultLogger)
	if len(c.Addrs) == 0 {
		return nil
	}

	r := &Redis{}
	r.log = log
	if len(c.Addrs) == 1 {
		r.single = redis.NewClient(
			&redis.Options{
				Addr:         c.Addrs[0], // use default Addr
				Password:     c.Pwd,      // no password set
				DB:           c.DB,       // use default DB
				DialTimeout:  c.DialTimeout,
				ReadTimeout:  c.ReadTimeout,
				WriteTimeout: c.WriteTimeout,
			})
		if err := r.single.Ping(context.Background()).Err(); err != nil {
			r.log.Errorf(err.Error())
			return nil
		}
		r.single.Do(context.Background(), "CONFIG", "SET", "notify-keyspace-events", "AKE")
		r.clusterMode = false
		r.mutex = new(sync.Mutex)
		return r
	}

	r.cluster = redis.NewClusterClient(
		&redis.ClusterOptions{
			Addrs:        c.Addrs,
			Password:     c.Pwd,
			DialTimeout:  c.DialTimeout,
			ReadTimeout:  c.ReadTimeout,
			WriteTimeout: c.WriteTimeout,
		})
	if err := r.cluster.Ping(context.Background()).Err(); err != nil {
		r.log.Errorf("cluster init failed, error : ", err.Error())
	}
	r.cluster.Do(context.Background(), "CONFIG", "SET", "notify-keyspace-events", "AKE")
	r.clusterMode = true
	return r
}
func (r *Redis) Run(ctx context.Context, script *redis.Script, keys []string, argv ...interface{}) interface{} {
	r.mutex.Lock()
	defer r.mutex.Unlock()
	if r.clusterMode {
		return script.Run(ctx, r.cluster, keys, argv...).Val()
	}
	return script.Run(ctx, r.single, keys, argv...).Val()
}

func (r *Redis) TryGetDistributedLock(ctx context.Context, key string) bool {
	var res interface{}
	if r.clusterMode {
		res = _LockDistributedLuaScript.Run(ctx, r.cluster, []string{key}, "_", _DistributedTimeOut).Val()
	} else {
		res = _LockDistributedLuaScript.Run(ctx, r.single, []string{key}, "_", _DistributedTimeOut).Val()
	}
	return res.(int64) == _DistributedSuccess
}

func (r *Redis) TryGetDistributedLockWithTimeOut(ctx context.Context, key string, duration time.Duration) bool {
	end := duration.Milliseconds()
	for getNowMillisecond() <= end {
		suc := r.TryGetDistributedLock(ctx, key)
		if suc {
			return true
		}
		sleepMillisecond(80 + int64(rand.Int31n(30)))
	}
	return false
}

func (r *Redis) ReleaseDistributedLock(ctx context.Context, key string) bool {
	r.mutex.Lock()
	defer r.mutex.Unlock()
	var res interface{}
	if r.clusterMode {
		res = _UnLockDistributedLuaScript.Run(ctx, r.cluster, []string{key}, "_").Val()
	} else {
		res = _UnLockDistributedLuaScript.Run(ctx, r.single, []string{key}, "_").Val()
	}
	if res.(int64) == _DistributedSuccess {
		return true
	}
	return false
}
```

### 参考

* https://www.bilibili.com/video/BV1r5411V7Sb
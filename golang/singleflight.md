## Golang之singleflight并发处理浅谈
我们在开发golang时候，一般都使用过sync包下的同步原语；例如互斥锁Mutex、读写锁RWMutex、Once、WaitGroup等。现在介绍sync扩展包的`SingleFlight`，它对于瞬间请求量大的场景有很好的并发处理效果。

## 应用场景
在开发中，我们可能需要缓存数据到redis并设置超时时间。在超时失效的时刻，可能有许多请求在redis中找不到记录进而到下游的数据库或者访问第三方接口获取。这些请求可能都是返回相同的结果，这样无疑消耗了许多资源。实际上我们只需请求一次，再把数据同步给其他请求即可。

使用SingleFlight可以对同一个key只进行一次函数调用，并把结果放回到缓存；其余相同key的请求只需获取新的缓存中的结果就好，这样可以减少多次数据库/第三方请求的操作。

## 具体测试
直接上代码看一下`SingleFlight`包的效果

```go
func main() {
    sf := singleflight.Group{}
    wg := sync.WaitGroup{}
    //count := 1000 * runtime.GOMAXPROCS(0)
    count := 100
    for i := 0; i < count; i++ {
        wg.Add(1)
        go func(i int) {
            ret, err, _ := sf.Do("test", DoReq)
            if err != nil {
                fmt.Println(err)
            }
            fmt.Printf("%s \n", ret)
            wg.Done()
        }(i)
    }
	
    wg.Wait()
}
```
![](/Users/lhh/Desktop/singleflight_1.jpg)
demo中可以看到，我们并发了100个协程同时去访问第三方服务，其中DoReq是具体访问接口服务的方法；我们可以看到100个协程中，只有1个协程真正去请求了服务，并把数据同步到其余的99个协程，大大减少了网络的请求。

## 源码分析
我们从源码的角度来看看，SingleFlight是如何实现的。我们先看看两个结构体：

```
type call struct {
    wg sync.WaitGroup // 内部用于同步

    // These fields are written once before the WaitGroup is done
    // and are only read after the WaitGroup is done.
    val interface{} // 缓存数据
    err error       // 缓存error

    // forgotten indicates whether Forget was called with this call's key
    // while the call was still in flight.
    forgotten bool //是否删除key的标志位

    // These fields are read and written with the singleflight
    // mutex held before the WaitGroup is done, and are read but
    // not written after the WaitGroup is done.
    dups  int             //是否有多个请求
    chans []chan<- Result // 异步用的channel，用于保存数据
}

// Group represents a class of work and forms a namespace in
// which units of work can be executed with duplicate suppression.
type Group struct {
    mu sync.Mutex // protects m
    // 缓存同一个key的具体数据
    m map[string]*call // lazily initialized
}
```
源码和具体解释标识在上面，`SingleFlight`主要通过`Group`, `call`结构体实现同步。`Group`中有一个互斥锁以及`key`对应`call`结构体的映射。其中`call`保存了**当次调用**对应的信息。这里的当次调用后面会解释到，具体使用的时候一定要注意，防止踩坑。
再看看具体的`Do()`方法：

```
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
    g.mu.Lock()
    if g.m == nil {
        g.m = make(map[string]*call)
    }
    if c, ok := g.m[key]; ok {
        c.dups++
        g.mu.Unlock()
        c.wg.Wait()
        return c.val, c.err, true
    }
    c := new(call)
    c.wg.Add(1)
    g.m[key] = c
    g.mu.Unlock()

    g.doCall(c, key, fn)
    return c.val, c.err, c.dups > 0
}
```
其实`Do()`方法比较简单。在瞬间大量请求时，请求A发现不存在`key`时，初始化一个新的`call`结构体指针，同时增加`WaitGroup`计数器，并释放锁。当其他请求（请求B）拿到锁，并发现存在key，dups增加，表示当前属于重复调用，并等待请求A`WaitGroup`释放计数器。

请求A进行`doCall()`,具体处理`fn`函数：

```
func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
    c.val, c.err = fn()
    c.wg.Done()

    g.mu.Lock()
    if !c.forgotten {
        delete(g.m, key)
    }
    for _, ch := range c.chans {
        ch <- Result{c.val, c.err, c.dups > 0}
    }
    g.mu.Unlock()
}
```
当前调用的数据保存到`val`,`err`中，并释放`WaitGroup`计数，此时请求B就可以同步请求A的数据`c.val，c.err`，并返回了。最后请求A再做一些收尾的工作，其中chans是异步`DoChan()`时候用到，和`Do()`的区别不大，这里不展开说。

注意到`delete(g.m, key)`方法，会删除`key`，其实这里表示只有当次调用有效。如果我们串行调用`Do()`方法，实际上会多次调用`fn()`的。或者并发的数量十分大的时候，并不能保证`SingleFlight`只调用一次`fn()`函数。例如在我自己的笔记本上调到10000的并发，有时会调用3次请求，不过也比请求10000次要好很多：
![](/Users/lhh/Desktop/singleflight_2)

##基准测试
最后并发测试一下使用`SingleFlight`的效果
![](/Users/lhh/Desktop/singleflight_3.jpg)
其实肯定是使用SingleFlight的性能会好。可以看到不使用SF的情况下，每次都要去请求第三方，性能肯定是不好的。其次并发越高的情况，命中缓存的请求就会越多，性能就会越高。总体来说，如果是那种瞬间爆发的并发重复请求，使用SingleFlight可以很好的提高性能。具体能不能使用该包，需要根据具体的业务场景来衡量。

对上述有什么疑问或者不足，欢迎一起探讨。

## 参考
https://github.com/golang/groupcache/blob/master/singleflight/singleflight.go

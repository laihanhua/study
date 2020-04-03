### Context
context一般为“上下文”，用来追踪goruntine,达到控制它们的目的。

例如：在Go服务器程序中，每个请求都会有一个goroutine去处理。然而，处理程序往往还需要创建额外的goroutine去访问后端资源，比如数据库、RPC服务等。由于这些goroutine都是在处理同一个请求，所以它们往往需要访问一些共享的资源，比如用户身份信息、认证token、请求截止时间等。而且如果请求超时或者被取消后，所有的goroutine都应该马上退出并且释放相关的资源。这种情况也需要用Context来为我们取消掉所有goroutine

### context接口
	```
	type Context interface {

	Deadline() (deadline time.Time, ok bool)
	
	//WithCancel arranges for Done to be closed when cancel is called;
	//WithDeadline arranges for Done to be closed when the deadline expired;
	//WithTimeout arranges for Done to be closed when the timeout elapses.
	Done() <-chan struct{}
	
	// If Done is closed, Err returns a non-nil error explaining why
	Err() error
	
	Value(key interface{}) interface{}
	
	}
	```
1. Deadline方法是获取设置的截止时间的意思，第一个返回式是截止时间，到了这个时间点，Context会**自动发起取消请求**；第二个返回值ok==false时表示没有设置截止时间，如果需要取消的话，需要手动**调用取消函数**进行取消。
2. Done()是一个**只读**的channel,当该channel可读时，说明parent context已经发起了context取消请求。当Context被**canceled**或是**timeout**，Done返回一个被closed 的channel
3. Err():当Done()关闭时，Err()返回关闭的原因
4. Value()：获取context绑定的值，线程安全的。

### context衍生关系
1. context有两个最顶层的`backgroud, todo`两个context，它们什么都没有，是一个不可取消，没有设置截止时间，没有携带任何值的Context。
2. 生成子context,通过如下4个方法，可以衍生出context树。

	```
	func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
	
	func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
	
	func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
	
	func WithValue(parent Context, key, val interface{}) Context

	```
		
##### WithCancel()
传递一个父Context作为参数，返回子Context，以及一个取消函数`cancel()`用来取消Context,这个函数就是用来关闭ctxWithCancel中的 Done channel 函数。

1. 调用cancel()函数时，会close(c.done),这个done是双向channel，可以被关闭。关闭后，<-ctx.Done()就不会再阻塞，可以往下执行。且所有的子context也都会被取消。

```
var ctx1, cancle1 = context.WithCancel(context.Background())
var ctx2, cancle2 = context.WithCancel(ctx1)
func main() {
    fmt.Println("begin")
    go func() {
        time.Sleep(time.Second * 4)
        fmt.Println("sleep 4s")
        cancle1()
    }()
    select {
    case <-ctx1.Done():
        fmt.Println("ctx1.Done()")
   case <-ctx2.Done():
        fmt.Println("ctx2.Done()")
    }

}

begin
sleep 4s
ctx1.Done() // 随机出现ctx2.Done()其中一个，说明两个都被关闭了
```

##### WithValue()
通过WithValue()可以设置Key-value值，提供上下文给后面通过context.Value()获取到。
```
ctxReq := context.WithValue(r.Context(), "ToBeDel_IP", true)
next.ServeHTTP(w, r.WithContext(ctxReq))


del, _ := r.Context().Value("ToBeDel_IP").(bool)
```
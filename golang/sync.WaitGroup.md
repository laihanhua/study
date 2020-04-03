#### sync.WaitGroup
1. 它能够一直等到所有的goroutine执行完成，并且阻塞主线程的执行，直到所有的goroutine执行完成。（当wg.Add()后，所有调用了wg.Wait()的协程都会阻塞，直到wg.Done()后，所有协程都会得到执行。）

2. sync.WaitGroup只有3个方法，Add()，Done()，Wait()。

3. Done()是Add(-1)的别名。简单的来说，使用Add()添加计数，Done()减掉一个计数，计数不为0, 阻塞Wait()的运行。

	```
	func main() {
	
	    var wg sync.WaitGroup
	    fmt.Println("begin")
	    wg.Add(3)
	    for i := 0; i < 3; i++ {
	        go func(a int) {
	            time.Sleep(1 * time.Second)
	            defer wg.Done()
	            fmt.Println(a)
	        }(i)
	    }
	    wg.Wait()
	    fmt.Println("end")
	
	}
	begin
	1
	0
	2
	end
	```

4. 如果wg.Done()的时候，计数已经为0了，此时会panic`sync: negative WaitGroup counter`

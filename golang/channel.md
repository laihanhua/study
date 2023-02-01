## channel关闭问题

#### 造成panic情况
1. 未初始化关闭
2. 重复关闭
3. 关闭后发送

```golang
// 1.未初始化时关闭
func TestCloseNilChan(t *testing.T) {
   var errCh chan error
   close(errCh)
   
   // Output:
   // panic: close of nil channel
}

// 2.重复关闭
func TestRepeatClosingChan(t *testing.T) {
   errCh := make(chan error)
   var wg sync.WaitGroup
   wg.Add(1)

   go func() {
      defer wg.Done()
      close(errCh)
      close(errCh)
   }()

   wg.Wait()
   
   // Output:
   // panic: close of closed channel
}

// 3.关闭后发送
func TestSendOnClosingChan(t *testing.T) {
   errCh := make(chan error)
   var wg sync.WaitGroup
   wg.Add(1)

   go func() {
      defer wg.Done()
      close(errCh)
      errCh <- errors.New("chan error")
   }()

   wg.Wait()
   
   // Output:
   // panic: send on closed channel
}

// 4.发送时关闭
func TestCloseOnSendingToChan(t *testing.T) {
   errCh := make(chan error)
   var wg sync.WaitGroup
   wg.Add(1)

   go func() {
      defer wg.Done()
      defer close(errCh)

      go func() {
         errCh <- errors.New("chan error") // 由于 chan 没有缓冲队列，代码会一直在此处阻塞
      }()

      time.Sleep(time.Second) // 等待向 errCh 发送数据
   }()

   wg.Wait()

   // Output:
   // panic: send on closed channel
}
```

关闭原则：
1. 发送端关闭
2. 多个发送者时，使用专门的协程关闭

#### 不关闭channel情况
1. channel发送次数等于接受次数，channel会被垃圾回收，此时不手动关闭无任何影响
2. channel发送次数小于接受次数，接受者会等待发送从而阻塞，导致接受者协程和channel的内存泄漏
3. 主要channel不阻塞，可以不手动关闭channel
4. channel关闭后，接受者会输出对应类型的零值，不会阻塞

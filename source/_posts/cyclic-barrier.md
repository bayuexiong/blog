---
title: "Go 的 CyclicBarrier 模式"
date: "2022-09-23"
---

最近在工作中遇到了需要控制多个Goroutine并发的问题，大致可以描述为，有一组需要计算的 Work，每个work都会得出一个结果，最后汇总每个work的结果，再进行下一轮计算。

这里结算结果是严格要求一组一组的计算，这里很像是使用WaitGroup 的场景，但是因为Add和Wait只能在一个Goroutine里面，所以最后实现的逻辑效率并不高。

大致的实现如下：

```go
func WaitGroupLoop() {
	wg := sync.WaitGroup{}
	cnt := 0
	for cnt >= 5  {
		wg.Add(10)
		for i := 0; i < 10; i++ {
			go func() {
				// do some work
				time.Sleep(time.Second)
				wg.Done()
			}()
		}
		wg.Wait()
		cnt++
	}
	fmt.Println(cnt) // 5
}

```
可以看到每次执行一次计算，都创建了很多Goroutine，后面从鸟窝的[啊哈，一道有趣的Go并发题](https://colobu.com/2022/09/01/fizzbuzz-in-go/)了解到 `CyclicBarrier` 。
看上去可以简化我这里的代码，就稍微研究了一下。

从搜索引擎中查询 `CyclicBarrier` 大部分结果是 java.util.concurrent 中的 [CyclicBarrier](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/package-summary.html)，官方文档写得很清晰，大致翻译一下就是。

> 它允许一组线程互相等待，直到到达某个公共屏障点 (common barrier point)，才得以继续执行。阻塞子线程，当阻塞数量到达定义的参与线程数后，才可继续向下执行。

这看上去很像WaitGroup是不是，不过重点是文档后面出现的 `re-used`, WaitGroup 重复使用很容易出现panic，这里写个例子

```go

func TestWaitGroupPanic(t *testing.T) {
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		wg.Done()
		wg.Add(1)
	}()
	wg.Wait() // 无法保证 Add 在 Wait之后
}
```

Go 也有人实现了 [marusama/cyclicbarrier](https://github.com/marusama/cyclicbarrier)，用这个库来改写上面的逻辑

```go

func WaitCyclicBarrier() {
	cnt := 0
	cb := cyclicbarrier.NewWithAction(10, func() error {
		cnt++
		return nil
	})
	wg := sync.WaitGroup{}
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			for j := 0; j < 5; j++ {
				time.Sleep(100 * time.Millisecond)
				cb.Await(nil)
			}
			wg.Done()
		}()
	}
	wg.Wait()
}
```

这里明显减少了Goroutine的生成，每次只需要在一开始生成Goroutine就可以了

----

参考资料：

[啊哈，一道有趣的Go并发题](https://colobu.com/2022/09/01/fizzbuzz-in-go/)

[CyclicBarrier的用法](https://www.cnblogs.com/liuling/p/2013-8-21-01.html)
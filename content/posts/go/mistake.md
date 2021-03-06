---
title: "在Go中你犯过的错"
date: 2021-04-21T23:54:52+08:00
toc: true
tags :

- go
- mistake

---

### 在迭代器变量上使用 goroutine

这算高频吧。

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	items := []int{1, 2, 3, 4, 5}
	for index, _ := range items {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Printf("item:%v\n", items[index])
		}()
	}
	wg.Wait()
}

```

一个很简单的利用 `sync.waitGroup` 做任务编排的场景，看一下好像没啥问题，运行看看结果。

![image](https://image.syst.top/image/mistake/mistake-1.png)

为啥不是1-5(当然不是顺序的)。

原因很简单，循环器中的 i 实际上是一个单变量，`go func`  里的闭包只绑定在一个变量上，
每个 `goroutine` 可能要等到循环结束才真正的运行，这时候运行的 i 值大概率就是5了,
没人能保证这个过程，有的只是手段。

正确的做法

```go
func main() {
	var wg sync.WaitGroup

	items := []int{1, 2, 3, 4, 5}
	for index, _ := range items {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			fmt.Printf("item:%v\n", items[i])
		}(index)
	}
	wg.Wait()
}
```

通过将 `i`  作为一个参数传入闭包中，i 每次迭代都会被求值，
并放置在 `goroutine` 的堆栈中，因此每个切片元素最终都会被执行打印。

或者这样。
```go
for index, _ := range items {
		wg.Add(1)
		i:=index
		go func() {
			defer wg.Done()
			fmt.Printf("item:%v\n", items[i])
		}()
	}
```

### WaitGroup


上面的例子有用到 `sync.waitGroup`，使用不当，也会犯错。

我把上面的例子稍微改动复杂一点点。
```go
package main

import (
	"errors"
	"github.com/prometheus/common/log"
	"sync"
)

type User struct {
	userId int
}

func main() {
	var userList []User
	for i := 0; i < 10; i++ {
		userList = append(userList, User{userId: i})
	}

	var wg sync.WaitGroup
	for i, _ := range userList {
		wg.Add(1)
		go func(item int) {
			_, err := Do(userList[item])
			if err != nil {
				log.Infof("err message:%v\n", err)
				return
			}
			wg.Done()
		}(i)
	}
	wg.Wait()
	
	// 处理其他事务
}

func Do(user User) (string, error) {
	// 处理杂七杂八的业务....
	if user.userId == 9 {
		// 此人是非法用户
		return "失败", errors.New("非法用户")
	}
	return "成功", nil
}
```
发现问题严重性了吗？

当用户`id`等于9的时候，`err !=nil` 直接 `return` 了，导致 `waitGroup` 计数器根本没机会减1，
最终 `wait` 会阻塞，多么可怕的 `bug`。

在绝大多数的场景下，我们都必须这样:
```go
func main() {
	var userList []User
	for i := 0; i < 10; i++ {
		userList = append(userList, User{userId: i})
	}
	var wg sync.WaitGroup
	for i, _ := range userList {
		wg.Add(1)
		go func(item int) {
			defer wg.Done()

			//....业务代码
			//....业务代码
			_, err := Do(userList[item])
			if err != nil {
				log.Infof("err message:%v\n", err)
				return
			}
		}(i)
	}
	wg.Wait()
}
```

### 野生 goroutine

我不知道你们公司是咋么处理异步操作的，是下面这样吗
```go
func main() {
	// doSomething
	go func() {
		// doSomething
	}()
}
```
我们为了防止程序中出现不可预知的 `panic`，导致程序直接挂掉，都会加入 `recover`，
```go
func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("%v\n", err)
		}
	}()
	panic("处理失败")
}
```

但是如果这时候我们直接开启一个 `goroutine`，在这个 `goroutine` 里面发生了  `panic`，

```go
func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("%v\n", err)
		}
	}()
	go func() {
		panic("处理失败")
	}()

	time.Sleep(2 * time.Second)
}
```
此时最外层的 `recover` 并不能捕获，程序会直接挂掉。
![image](https://image.syst.top/image/mistake/mistake-2.png)

但是你总不能每次开启一个新的 `goroutine` 就在里面 `recover`,
```go
func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("%v\n", err)
		}
	}()

	// func1
	go func() {
		defer func() {
			if err := recover(); err != nil {
				fmt.Printf("%v\n", err)
			}
		}()
		panic("错误失败")
	}()

	// func2
	go func() {
		defer func() {
			if err := recover(); err != nil {
				fmt.Printf("%v\n", err)
			}
		}()
		panic("请求错误")
	}()

	time.Sleep(2 * time.Second)
}
```
多蠢啊。所以基本上大家都会包一层。
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("%v\n", err)
		}
	}()

	// func1
	Go(func() {
		panic("错误失败")
	})

	// func2
	Go(func() {
		panic("请求错误")
	})

	time.Sleep(2 * time.Second)
}

func Go(fn func()) {
	go RunSafe(fn)
}

func RunSafe(fn func()) {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("错误:%v\n", err)
		}
	}()
	fn()
}
```

当然我这里只是简单都打印一些日志信息，一般还会带上堆栈都信息。


### channel 

`channel` 在 `go` 中的地位实在太高了，各大开源项目到处都是 `channel` 的影子，
以至于你在工业级的项目 issues 中搜索 `channel` ，能看到很多的 `bug`，
比如 etcd 这个 `issue`,
![image](https://image.syst.top/image/mistake/mistake-3.png)

一个往已关闭的 `channel` 中发送数据引发的 `panic`,等等类似场景很多。

这个故事告诉我们，否管大不大佬，改写的 `bug` 还是会写，手动狗头。


`channel` 除了上述高频出现的错误，还有以下几点:

#### 直接关闭一个 nil 值 channel 会引发 panic
```go
package main

func main() {
  var ch chan struct{}
  close(ch)
}

```

#### 关闭一个已关闭的 channel 会引发 panic。
```go
package main

func main() {
  ch := make(chan struct{})
  close(ch)
  close(ch)
}
```

另外，有时候使用 `channel` 不小心会导致 `goroutine` 泄露，比如下面这种情况,

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ch := make(chan struct{})
	cx, _ := context.WithTimeout(context.Background(), time.Second)
	go func() {
		time.Sleep(2 * time.Second)
		ch <- struct{}{}
		fmt.Println("goroutine 结束")
	}()

	select {
	case <-ch:
		fmt.Println("res")
	case <-cx.Done():
		fmt.Println("timeout")
	}
	time.Sleep(5 * time.Second)
}

```

启动一个 `goroutine` 去处理业务，业务需要执行2秒，而我们设置的超时时间是1秒。
这就会导致 `channel` 从未被读取，
我们知道没有缓冲的 `channel` 必须等发送方和接收方都准备好才能操作。
此时 `goroutine` 会被永久阻塞在  `ch <- struct{}{}` 这行代码，除非程序结束。
而这就是 `goroutine` 泄露。

解决这个也很简单，把无缓冲的 `channel` 改成缓冲为1。 

### 总结
这篇文章主要介绍了使用 `Go` 在日常开发中容易犯下的错。
当然还远远不止这些，你可以在下方留言中补充你犯过的错。
 























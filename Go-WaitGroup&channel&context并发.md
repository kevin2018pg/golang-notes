## WaitGroup + channel + context 使用

### sync.WaitGroup

等待组是用于线程（协程）同步的方式之一，它实现了一个类似队列的结构，WaitGroup等待一组线程执行完成，才会继续向下执行。 

主线程(goroutine)调用 Add 来设置等待的线程(goroutine)数量（初始默认为0），然后每个线程(goroutine)运行完成后调用 Done。 同时， Wait 方法用来阻塞而不需要用传统 sleep 一个固定时间的方式（如果此时等待组维护计数为零，则 Wait 操作为一个空操作），直到所有线程(goroutine)完成才会继续执行。

值得一提的地方 -_- ：

- Add:添加等待goroutine的数量，Done:相当于Add(-1),减掉一个goroutine计数，Wait:执行阻塞，直到所有的WaitGroup数量变成0

* Add(-1) 和 Done() 效果一致，如果计数器变为负数，则panic
* WaitGroup 缺点是无法指定固定的 Goroutine 数目,可以通过使用 channel 解决这个问题
* 在运行 main 函数的 goroutine 里运行 Add() 方法，在其他的 goroutine 里面运行 Done() 方法

**基本用法**

```go
package main
import (
    "fmt"
    "sync"
)
func main() {
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(p int) {
            defer wg.Done()
            fmt.Println(p)
        }(i)
    }
    wg.Wait()
}
```

**使用场景**

将每次循环的数 sleep 一段时间输出，这个程序如果不用 WaitGroup，那么将无法输出结果。因为 Goroutine 还没执行完，主线程已经执行完毕。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i = i + 1 {
		wg.Add(1)
		go func(n int) {
			//defer wg.Done(),注意这个Done的位置，是另一个函数
			defer wg.Add(-1)
			EchoNumber(n)
		}(i)
	}
	wg.Wait()
}
func EchoNumber(i int) {
	time.Sleep(time.Millisecond * 2000)
	fmt.Println(i)
}
```





### 融合使用

**三者结合的使用场景**

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func main() {
	docList := []int64{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20}
	done := make(chan int)
	ctx, cancel := context.WithTimeout(context.Background(), time.Duration(1)*time.Millisecond)
	defer cancel()
	var wg sync.WaitGroup
	for startPos := 0; startPos < len(docList); {
		endPos := startPos + 3
		if endPos > len(docList) {
			endPos = len(docList)
		}

		wg.Add(1)
		go func(docList []int64) {
			defer wg.Done()
			processLoop(docList)
			select {
			case <-ctx.Done():
				fmt.Println("time out 1")
				break
			default:
				fmt.Println("default branch")
				break
			}
		}(docList[startPos:endPos])

		startPos = endPos
	}
	go func() {
		wg.Wait()
		fmt.Println("wg.Wait")
		done <- 1
	}()
	select {
	case <-ctx.Done():
		fmt.Println("time out 2")
		break
	case <-done:
		fmt.Println("channel done")
		break
	}
}
func processLoop(docList []int64) {
	for _, v := range docList {
		fmt.Println(v)
	}
}

```

**返回结果**

```bash
1
2
3
default branch
7
8
9
default branch
10
11
12
default branch
13
14
15
default branch
4
5
6
default branch
16
17
18
default branch
19
20
default branch
wg.Wait
channel done
```


## slice和map并发安全

并发安全也叫线程安全，在实际操作中发现map和slice都是并发不安全的，出现了数据的丢失。

## 场景和解决办法

**场景1:** 10000个协程并发添加元素至切片

```go
package main

import (
	"fmt"
)

var array []int

func appendStack(i int) {
	array = append(array, i)
}

func main() {
	for i := 0; i < 10000; i++ { // 10000个协程同时入栈
		go appendStack(i)
	}

	for i, v := range array { // 获取索引和值
		fmt.Println(i, ":", v)
	}
}
```

结果没有到最大值9999，证实有数据丢失。

<p align="center"><img width="30%" src="image/slice append并发.png" /></p>



**解决方法:** 加锁

```go
package main

import (
	"fmt"
	"sort"
	"sync"
	"time"
)

var array []int
var lock sync.Mutex //互斥锁
func appendStack(i int) {
	lock.Lock() //加锁
	array = append(array, i)
	lock.Unlock() //解锁
}

func main() {
	for i := 0; i < 10000; i++ {
		go appendStack(i)
	}
	sort.Ints(array)        //给切片排序
	time.Sleep(time.Second) //间隔1s再打印,防止一边插入数据一边打印时数据乱序
	for i, v := range array {
		fmt.Println(i, ":", v)
	}
}
```

**总结:** slice在并发执行中不会报错，但是数据会丢失



**场景2:** 2个协程同时读和写map

```go
package main

import "fmt"

func main() {
	m := make(map[int]int)
	go func() { //开一个协程写map
		for i := 0; i < 10000; i++ {
			m[i] = i
		}
	}()

	go func() { //开一个协程读map
		for i := 0; i < 10000; i++ {
			fmt.Println(m[i])
		}
	}()

	//time.Sleep(time.Second * 20)
}
```

**解决方法：**尽量不要做map的并发，如果用并发要加锁，保证map的操作只读或只写。

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var lock sync.Mutex

func main() {
	m := make(map[int]int)
	go func() { //开一个协程写map
		for i := 0; i < 10000; i++ {
			lock.Lock() //加锁
			m[i] = i
			lock.Unlock() //解锁
		}
	}()
	go func() { //开一个协程读map
		for i := 0; i < 10000; i++ {
			lock.Lock() //加锁
			fmt.Println(m[i])
			lock.Unlock() //解锁
		}
	}()
	time.Sleep(time.Second * 20)
}
```

**总结:** map在并发执行中会直接报错


## TODO sync.Map

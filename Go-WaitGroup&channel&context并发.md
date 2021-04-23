## WaitGroup + channel + context 使用

**使用场景**

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


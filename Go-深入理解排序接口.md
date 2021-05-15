## 1 排序接口

golang的排序和 c 有些差别，可以对任何对象进行排序， 常用场景就是对 slice ，或是包含 slice 的一个对象进行排序，另外就是对结构体进行排序。

**实现排序(接口)的三要素：**

1. 待排序元素个数 `len` ；
2. 第 i 和第 j 个元素的比较函数  `less`；
3. 第 i 和 第 j 个元素的交换 `swap` 。 



## 2 基本类型排序

**升序排序**

对于 `int` 、 `float64` 和 `string` 基本类型数组或是切片的排序， go 分别提供了 `sort.Ints()` 、 `sort.Float64s()` 和 `sort.Strings()` 函数， 默认都是升序排序。(*注：没有`sort.Float32s()` 函数)

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	intArray := []int{2, 14, 3, 51, 7, 26, 9, 8, 11, 10}
	float64Array := []float64{4.2, 5.9, 12.3, 10.0, 50.4, 99.9, 31.4, 27.81828, 3.14}
	// float32Array := [] float32 {4.2, 5.9, 12.3, 10.0, 50.4, 99.9, 31.4, 27.81828, 3.14}    // no function : sort.Float32s
	stringArray := []string{"ac", "cd", "bh", "da", "fg", "ig", "za", "xas", "ww", "yw"}

	sort.Ints(intArray)
	sort.Float64s(float64Array)
	sort.Strings(stringArray)

	fmt.Printf("%v\n%v\n%v\n", intArray, float64Array, stringArray)

}
```

**降序排序**

`int` 、 `float64` 和 `string` 默认是升序排序， 那如果是降序呢 ？ go 中对某个 Type 的对象 obj 排序， 可以使用 `sort.Sort(obj)` 即可，但是需要对 Type 类型绑定三个方法 ： `Len()` 求长度、 `Less(i,j)` 比较第 i 和 第 j 个元素大小的函数、 `Swap(i,j)` 交换第 i 和第 j 个元素的函数。sort 包下的三个类型 IntSlice 、 Float64Slice 、 StringSlice 分别实现了这三个方法， 对应排序的对象是 [] int 、 [] float64 和 [] string 。如果需要降序排序， 只需要将对应的 Less 函数修改一下即可。

go 的 sort 包可以使用 sort.Reverse(slice) 来调换 slice.Interface.Less ，也就是比较函数，所以， int 、 float64 和 string 的降序排序也很简便。

```go
package main

import (
	"fmt"
	"sort"
)

func main() {
	intArray := []int{2, 14, 3, 51, 7, 26, 9, 8, 11, 10}
	float64Array := []float64{4.2, 5.9, 12.3, 10.0, 50.4, 99.9, 31.4, 27.81828, 3.14}
	// float32Array := [] float32 {4.2, 5.9, 12.3, 10.0, 50.4, 99.9, 31.4, 27.81828, 3.14}    // no function : sort.Float32s
	stringArray := []string{"ac", "cd", "bh", "da", "fg", "ig", "za", "xas", "ww", "yw"}

	sort.Sort(sort.Reverse(sort.IntSlice(intArray)))
	sort.Sort(sort.Reverse(sort.Float64Slice(float64Array)))
	sort.Sort(sort.Reverse(sort.StringSlice(stringArray)))
	fmt.Printf("%v\n%v\n%v\n", intArray, float64Array, stringArray)

}
```



## 3 深入理解排序接口

sort 包中含有 sort.Interface 接口，该接口有三个方法 Len() 、 Less(i,j) 和 Swap(i,j) 。 排序函数 sort.Sort 可以排序任何实现了 sort.Inferface 接口的对象(变量)。

对于 [] int 、[] float64 和 [] string 除了使用指定的函数外，还可以使用改装过的类型 IntSclice 、 Float64Slice 和 StringSlice ， 然后直接调用 Sort() 方法；由于这三种类型也实现了 sort.Interface 接口， 所以可以通过 sort.Reverse 来转换这三种类型的 Interface.Less 方法来实现降序排序。

下面使用了一个自定义(用户定义)的 Reverse 结构体， 而不是 sort.Reverse 函数， 来实现逆向排序。

```go
package main

import (
	"fmt"
	"sort"
)

// 自定义的 Reverse 类型
type Reverse struct {
	sort.Interface // 这样， Reverse 可以接纳任何实现了 sort.Interface (包括 Len, Less, Swap 三个方法) 的对象
}

// Reverse 只是将其中的 Inferface.Less 的顺序对调了一下
func (r Reverse) Less(i, j int) bool {
	return r.Interface.Less(j, i)
}

func main() {
	intArray := []int{5, 2, 6, 3, 1, 4} // 未排序
	sort.Ints(intArray)
	floatArray := []float64{2.3, 3.2, 6.7, 10.9, 5.4, 1.8}
	sort.Float64s(floatArray)
	stringArray := []string{"hello", "world", "da", "ji", "da", "li"}
	sort.Strings(stringArray)

	ipos := sort.SearchInts(intArray, -1) // int 搜索
	fmt.Printf("pos of 5 is %d th\n", ipos)

	dpos := sort.SearchFloat64s(floatArray, 20.1) // float64 搜索
	fmt.Printf("pos of 5.0 is %d th\n", dpos)

	fmt.Printf("floatArray is asc ? %v\n", sort.Float64sAreSorted(floatArray))

	floatArray = []float64{3.5, 4.2, 8.9, 100.98, 20.14, 79.32}
	(sort.Float64Slice(floatArray)).Sort()           // float64 排序方法 3
	fmt.Println("after sort by Sort:\t", floatArray) // [3.5 4.2 8.9 20.14 79.32 100.98]

	sort.Sort(Reverse{sort.Float64Slice(floatArray)})         // float64 逆序排序
	fmt.Println("after sort by Reversed Sort:\t", floatArray) // [100.98 79.32 20.14 8.9 4.2 3.5]
}

```

search 函数是测试是否有序的函数，发现搜索函数只能定位到如果存在的话的位置，不存在的话，位置就是不对的。



## 4 结构体类型排序

结构体排序在工作使用中会用到很多，结构体类型的排序是通过使用 `sort.Sort(slice)` 实现的， 只要 slice 实现了 `sort.Interface` 的三个方法就可以。 虽然这么说，但是排序的方法却有那么好几种。首先就是模拟排序 `[] int` 构造对应的 `IntSlice` 类型，对 `IntSlice` 类型实现 Interface 的三个方法。

**方法1：**

```go
package main

import (
	"fmt"
	"sort"
)

type Person struct {
	Name string // 姓名
	Age  int    // 年纪
}

// 按照 Person.Age 从大到小排序，相等就按照姓名从小大排序
type PersonSlice []Person

func (a PersonSlice) Len() int { // 重写 Len() 方法
	return len(a)
}
func (a PersonSlice) Swap(i, j int) { // 重写 Swap() 方法
	a[i], a[j] = a[j], a[i]
}
func (a PersonSlice) Less(i, j int) bool { // 重写 Less() 方法， 从大到小排序
    if a[j].Age == a[i].Age{
        return a[j].Name < a[i].Name
    }
	return a[j].Age < a[i].Age
}

func main() {
	people := []Person{
		{"zhang san", 12},
		{"li si", 30},
		{"wang wu", 52},
		{"zhao liu", 26},
	}

	fmt.Println(people)

	sort.Sort(PersonSlice(people)) // 按照 Age 的逆序排序
	fmt.Println(people)

	sort.Sort(sort.Reverse(PersonSlice(people))) // 按照 Age 的升序排序
	fmt.Println(people)

}
```

**方法 2：** 

方法 1 有个缺点 ： 根据 Age 排序需要重新定义 PersonSlice 方法，绑定 Len 、 Less 和 Swap 方法， 如果需要根据 Name 排序， 又需要重新写三个函数； 那么根据不同的标准 Age 或是 Name， 真正不同的体现在 Less 方法上，所以， 可以将 Less 抽象出来， 每种排序的 Less 让其变成动态的。

```go
package main
 
import (
    "fmt"
    "sort"
)
 
type Person struct {
    Name string    // 姓名
    Age  int    // 年纪
}
 
type PersonWrapper struct {
    people [] Person
    by func(p, q * Person) bool
}
 
func (pw PersonWrapper) Len() int {    // 重写 Len() 方法
    return len(pw.people)
}
func (pw PersonWrapper) Swap(i, j int){     // 重写 Swap() 方法
    pw.people[i], pw.people[j] = pw.people[j], pw.people[i]
}
func (pw PersonWrapper) Less(i, j int) bool {    // 重写 Less() 方法
    return pw.by(&pw.people[i], &pw.people[j])
}
 
func main() {
    people := [] Person{
        {"zhang san", 12},
        {"li si", 30},
        {"wang wu", 52},
        {"zhao liu", 26},
    }
 
    fmt.Println(people)
 
    sort.Sort(PersonWrapper{people, func (p, q *Person) bool {
        return q.Age < p.Age    // Age 递减排序
    }})
 
    fmt.Println(people)
    sort.Sort(PersonWrapper{people, func (p, q *Person) bool {
        return p.Name < q.Name    // Name 递增排序
    }})
 
    fmt.Println(people)
 
}
```

方法 2 将 [] Person 和比较的准则 封装在了一起，形成了 PersonWrapper 函数，然后在其上绑定 Len 、 Less 和 Swap 方法。 实际上 sort.Sort(pw) 排序的是 pw 中的 people， 这就是前面说的， go 的排序未必就是针对的一个数组或是 slice， 而可以是一个对象中的数组或是 slice 。



**方法3：**

方法 2 有一个缺点是，在 main 中使用的时候暴露了 sort.Sort 的使用，还有就是 PersonWrapper 的构造。 为了让 main 中使用起来更为方便，可以再简单的封装一下， 构造一个 SortPerson 方法。

```go
package main
 
import (
    "fmt"
    "sort"
)
 
type Person struct {
    Name string    // 姓名
    Age  int    // 年纪
}
 
type PersonWrapper struct {
    people [] Person
    by func(p, q * Person) bool
}
 
type SortBy func(p, q *Person) bool
 
func (pw PersonWrapper) Len() int {    // 重写 Len() 方法
    return len(pw.people)
}
func (pw PersonWrapper) Swap(i, j int){     // 重写 Swap() 方法
    pw.people[i], pw.people[j] = pw.people[j], pw.people[i]
}
func (pw PersonWrapper) Less(i, j int) bool {    // 重写 Less() 方法
    return pw.by(&pw.people[i], &pw.people[j])
}
 
 
func SortPerson(people [] Person, by SortBy){    // SortPerson 方法
    sort.Sort(PersonWrapper{people, by})
}
 
func main() {
    people := [] Person{
        {"zhang san", 12},
        {"li si", 30},
        {"wang wu", 52},
        {"zhao liu", 26},
    }
 
    fmt.Println(people)
 
    sort.Sort(PersonWrapper{people, func (p, q *Person) bool {
        return q.Age < p.Age    // Age 递减排序
    }})
 
    fmt.Println(people)
 
    SortPerson(people, func (p, q *Person) bool {
        return p.Name < q.Name    // Name 递增排序
    })
 
    fmt.Println(people)
 
}
```

在方法 2 基础上构造了 SortPerson 函数，使用的时候传一个 [] Person 和一个 比较 函数。

这么多方法，个人觉得动态排序更好用，并且待排序属性都可以存储起来，方便比较，学以致用！

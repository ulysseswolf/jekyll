---
layout: post
title: "Golang Escape Analysis"
description: ""
categories: 
- Algorithm
tags: [Golang]
---
{{ page.title }}
================

> 注意我们此处谈到的堆和栈是对操作系统中的，这个和数据结构中的堆和栈还是又一定区别的。

1. 关于 堆和栈
---------

栈 可以简单得理解成一次函数调用内部申请到的内存，它们会随着函数的返回把内存还给系统。

```
func F() {
	temp := make([]int, 0, 20)
	...
}

```

类似于上面代码里面的 temp 变量，只是内函数内部申请的临时变量，并不会作为返回值返回，它就是被编译器申请到栈里面。

### 申请到 栈内存 好处：函数返回直接释放，不会引起垃圾回收，对性能没有影响。

再来看看堆得情况之一如下代码：

```
func F() []int{
	a := make([]int, 0, 20)
	return a
}

```

而上面这段代码，申请的代码一模一样，但是申请后作为返回值返回了，编译器会认为变量之后还会被使用，当函数返回之后并不会将其内存归还，那么它就会被申请到 堆 上面了。

### 申请到堆上面的内存才会引起垃圾回收，如果这个过程（特指垃圾回收不断被触发）过于高频就会导致 gc 压力过大，程序性能出问题。

我们再看看如下几个例子：

```
func F() {
	a := make([]int, 0, 20)     
	b := make([]int, 0, 20000) 
 
	l := 20
	c := make([]int, 0, l) 
}

```

### 像是 b 这种 即使是临时变量，申请过大也会在堆上面申请。

### 对于 c 编译器对于这种不定长度的申请方式，也会在堆上面申请，即使申请的长度很短。

2. 逃逸分析（Escape analysis）
------------------------

> 所谓逃逸分析（Escape analysis）是指由编译器决定内存分配的位置，不需要程序员指定。

在函数中申请一个新的对象：

*   如果分配 在栈中，则函数执行结束可自动将内存回收；
    
*   如果分配在堆中，则函数执行结束可交给 GC（垃圾回收）处理;
    

> 注意，对于函数外部没有引用的对象，也有可能放到堆中，比如内存过大超过栈的存储能力。

3. 逃逸场景（什么情况才分配到堆中）
-------------------

### 3.1 指针逃逸

Go 可以返回局部变量指针，这其实是一个典型的变量逃逸案例，示例代码如下：

```
package main

type Student struct {
    Name string
    Age  int
}

func StudentRegister(name string, age int) *Student {
    s := new(Student) 

    s.Name = name
    s.Age = age

    return s
}

func main() {
    StudentRegister("Jim", 18)
}

```

虽然 在函数 StudentRegister() 内部 s 为局部变量，其值通过函数返回值返回，s 本身为一指针，其指向的内存地址不会是栈而是堆，这就是典型的逃逸案例。

终端运行命令查看逃逸分析日志：

![](https://ws3.sinaimg.cn/large/006tNc79gy1g04xkehoe8j30l80aa7ab.jpg)

可见在 StudentRegister() 函数中，也即代码第 9 行显示”escapes to heap”，代表该行内存分配发生了逃逸现象。

### 3.2 栈空间不足逃逸（空间开辟过大）

```
package main

func Slice() {
    s := make([]int, 1000, 1000)

    for index, _ := range s {
        s[index] = index
    }
}

func main() {
    Slice()
}

```

上面代码 Slice() 函数中分配了一个 1000 个长度的切片，是否逃逸取决于栈空间是否足够大。 直接查看编译提示，如下：

![](https://ws2.sinaimg.cn/large/006tNc79gy1g04xn62r3gj30pd050whm.jpg)

所以只是 1000 的长度还不足以发生逃逸现象。然后就 x10 倍吧

```
package main

func Slice() {
    s := make([]int, 10000, 10000)

    for index, _ := range s {
        s[index] = index
    }
}

func main() {
    Slice()
}

```

分析如下：

![](https://ws2.sinaimg.cn/large/006tNc79gy1g04xpbqx37j30nu04uju8.jpg)

当切片长度扩大到 10000 时就会逃逸。

实际上当栈空间不足以存放当前对象时或无法判断当前切片长度时会将对象分配到堆中。

### 3.3 动态类型逃逸（不确定长度大小）

很多函数参数为 interface 类型，比如 fmt.Println(a …interface{})，编译期间很难确定其参数的具体类型，也能产生逃逸。

如下代码所示：

```
package main

import "fmt"

func main() {
    s := "Escape"
    fmt.Println(s)
}

```

逃逸分下如下：

```
D:\SourceCode\GoExpert\src>go build -gcflags=-m
# _/D_/SourceCode/GoExpert/src
.\main.go:7: s escapes to heap
.\main.go:7: main ... argument does not escape

```

又或者像前面提到的例子：

```
func F() {
	a := make([]int, 0, 20)     
	b := make([]int, 0, 20000) 
 
	l := 20
	c := make([]int, 0, l) 
}

```

### 3.4 闭包引用对象逃逸

Fibonacci 数列的函数：

```
package main

import "fmt"

func Fibonacci() func() int {
    a, b := 0, 1
    return func() int {
        a, b = b, a+b
        return a
    }
}

func main() {
    f := Fibonacci()

    for i := 0; i < 10; i++ {
        fmt.Printf("Fibonacci: %d\n", f())
    }
}

```

输出如下：

```
 ~/go/src/gitHub/test/pool  go run main.go
Fibonacci: 1
Fibonacci: 1
Fibonacci: 2
Fibonacci: 3
Fibonacci: 5
Fibonacci: 8
Fibonacci: 13
Fibonacci: 21
Fibonacci: 34
Fibonacci: 55

```

逃逸如下：

```
 ~/go/src/gitHub/test/pool  go build -gcflags=-m
# gitHub/test/pool
./main.go:7:9: can inline Fibonacci.func1
./main.go:7:9: func literal escapes to heap
./main.go:7:9: func literal escapes to heap
./main.go:8:10: &b escapes to heap
./main.go:6:5: moved to heap: b
./main.go:8:13: &a escapes to heap
./main.go:6:2: moved to heap: a
./main.go:17:34: f() escapes to heap
./main.go:17:13: main ... argument does not escape

```

Fibonacci() 函数中原本属于局部变量的 a 和 b 由于闭包的引用，不得不将二者放到堆上，以致产生逃逸。

### 逃逸分析的作用是什么呢？

1.  逃逸分析的好处是为了减少 gc 的压力，不逃逸的对象分配在栈上，当函数返回时就回收了资源，不需要 gc 标记清除。
    
2.  逃逸分析完后可以确定哪些变量可以分配在栈上，栈的分配比堆快，性能好 (逃逸的局部变量会在堆上分配 , 而没有发生逃逸的则有编译器在栈上分配)。
    
3.  同步消除，如果你定义的对象的方法上有同步锁，但在运行时，却只有一个线程在访问，此时逃逸分析后的机器码，会去掉同步锁运行。
    

### 逃逸总结：

*   栈上分配内存比在堆中分配内存有更高的效率
    
*   栈上分配的内存不需要 GC 处理
    
*   堆上分配的内存使用完毕会交给 GC 处理
    
*   逃逸分析目的是决定内分配地址是栈还是堆
    
*   逃逸分析在编译阶段完成
    

提问：函数传递指针真的比传值效率高吗？

> 我们知道传递指针可以减少底层值的拷贝，可以提高效率，但是如果拷贝的数据量小，由于指针传递会产生逃逸，可能会使用堆，也可能会增加 GC 的负担，所以传递指针不一定是高效的。

在官网 (golang.org) FAQ 上有一个关于变量分配的问题如下：

> From a correctness standpoint, you don’t need to know. Each variable in Go exists as long as there are references to it. The storage location chosen by the implementation is irrelevant to the semantics of the language.
> 
> The storage location does have an effect on writing efficient programs. When possible, the Go compilers will allocate variables that are local to a function in that function’s stack frame.
> 
> However, if the compiler cannot prove that the variable is not referenced after the function returns, then the compiler must allocate the variable on the garbage-collected heap to avoid dangling pointer errors. Also, if a local variable is very large, it might make more sense to store it on the heap rather than the stack.
> 
> In the current compilers, if a variable has its address taken, that variable is a candidate for allocation on the heap. However, a basic escape analysis recognizes some cases when such variables will not live past the return from the function and can reside on the stack.

翻译如下：

如何得知变量是分配在栈（stack）上还是堆（heap）上？

准确地说，你并不需要知道。Golang 中的变量只要被引用就一直会存活，存储在堆上还是栈上由内部实现决定而和具体的语法没有关系。

知道变量的存储位置确实和效率编程有关系。如果可能，Golang 编译器会将函数的局部变量分配到函数栈帧（stack frame）上。 然而，如果编译器不能确保变量在函数 return 之后不再被引用，编译器就会将变量分配到堆上。而且，如果一个局部变量非常大，那么它也应该被分配到堆上而不是栈上。

当前情况下，如果一个变量被取地址，那么它就有可能被分配到堆上。然而，还要对这些变量做逃逸分析，如果函数 return 之后，变量不再被引用，则将其分配到栈上。

* * *

1. 内存碎片化问题：
-----------

实际项目基本都是通过

来申请内存，长度都是不确定的，自然而然这些变量都会申请到堆上面了。

Golang 使用的垃圾回收算法是『标记——清除』。

简单得说，就是程序要从操作系统申请一块比较大的内存，内存分成小块，通过链表链接。

每次程序申请内存，就从链表上面遍历每一小块，找到符合的就返回其地址，没有合适的就从操作系统再申请。如果申请内存次数较多，而且申请的大小不固定，就会引起内存碎片化的问题。

申请的堆内存并没有用完，但是用户申请的内存的时候却没有合适的空间提供。这样会遍历整个链表，还会继续向操作系统申请内存。这就能解释我一开始描述的问题，申请一块内存变成了慢语句。

* * *

2. 为了解决以上问题引入 临时对象池 sync.Pool
-----------------------------

* * *

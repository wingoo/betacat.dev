+++
title = "循环中的闭包陷阱"
tags = [
    "golang",
    "closure",
    "goroutine",
    "dev",
]
date = "2019-03-22"
toc = true
+++

在 go 中进行循环操作时, 对于批量任务, 大多都会用 goroutine 来并发处理. 下面代码使用了 goroutine + closure

{{< highlight go "linenos=table,linenostart=1" >}}
for _, val := range values {
	go func() {
		fmt.Println(val)
	}()
}
{{< / highlight >}}

 相信熟悉的同学们已经看出来了, 这段代码是有问题的. 循环中`变动`的变量, 如果直接在闭包使用, 会导致 goroutine 处理时拿到的值可能是重复的值. 因为循环迭代使用了同一个变量, 所以每个闭包也都共享了这一个变量. 当闭包开始运行, 程序在 println 执行时打印了`val`的值, 但这个值在 goroutine 启动时可能已经被修改了, 或者说可能被修改了很多次. 可以用[go vet]来判断是否有此类似的问题. 

为了绑定当前的`val`到每个启动后的闭包中, 需要在循环中新增变量. 一个方法是 将变量对闭包进行参数传递, 如下:

{{< highlight go "linenos=table,linenostart=1" >}}
for _, val := range values {
	go func(val interface{}) {
		fmt.Println(val)
	}(val)
}
{{< / highlight >}}

通过在闭包中传递参数, `val`变量在循环中被放在栈上,所以每一个循环中的变量在 goroutine 中都是有效的.

另外也可以在循环中定义局部变量来解决, 循环的每一次, 变量都会重新定义, 所以也不会在循环中被共享. 

{{< highlight go "linenos=table,linenostart=1" >}}
for i := range valslice {
	val := valslice[i]
	go func() {
		fmt.Println(val)
	}()
}
{{< / highlight >}}

**如果不在循环中定义新的变量, 可能会造成预想之外的行为. 这个行为在未来的 go 版本中可能会修改, 但在 version 1的大版本中不会做改动, 详见[golang FAQ]**

如果闭包不通过 goroutine 来执行, 则不会有问题, 下面的代码会打印1 -> 10

{{< highlight go "linenos=table,linenostart=1" >}}
for i := 1; i <= 10; i++ {
	func() {
		fmt.Println(i)
	}()
}
{{< / highlight >}}

正确执行的原因是, 在每次循环迭代时, 匿名函数都会在拿到当前变量后同步执行

下面是另外一种类似的情况

{{< highlight go "linenos=table,linenostart=1" >}}
for _, val := range values {
	go val.MyMethod()
}

func (v *val) MyMethod() {
        fmt.Println(v)
}
{{< / highlight >}}

上面的代码依旧会只打印最后一个`val`的值, 原因同上, 要解决的话, 只需要在循环中新增一个变量即可

{{< highlight go "linenos=table,linenostart=1" >}}
for _, val := range values {
        newVal := val
	go newVal.MyMethod()
}

func (v *val) MyMethod() {
        fmt.Println(v)
}
{{< / highlight >}}

---

现在, 我们可以整理总结下, 循环中变量的使用规则

1. 循环中如果不存在延迟执行, 则没有问题. 
2. 循环之外定义的变量, 如果只是读取也没有问题
3. 可以通过临时变量保存循环的变量值, 以保证后续延迟执行的调用可靠

--- 

好吧, 之所以翻译这篇文章, 是因为遇到了异常, 直接看代码吧 [go paly]

{{< highlight go "linenos=table,linenostart=1" >}}
bb := []ccc{{aa: 11, bb: 22}, {aa: 111, bb: 222},}
for _, i := range bb {
    go func(i *ccc) {
        fmt.Println(i)
    }(&i)
}
{{< / highlight >}}

第五行, 传递的 i 的引用, 尽管遵从了闭包参数传递的规则, 但因为传送的是变量地址, 所以仍然导致问题. 如果去除引用传递, 则没有问题, 不管`ccc`的值是 struct 还是指针.

Reference:

- [CommonMistakes]
- [golang FAQ]

[CommonMistakes]: https://github.com/golang/go/wiki/CommonMistakes
[go vet]: https://golang.org/cmd/go/#hdr-Run_go_tool_vet_on_packages
[golang FAQ]: https://golang.org/doc/faq#closures_and_goroutines
[go paly]: https://play.golang.org/p/CMJWCOM4OCl

# 5.1 for 和 range [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#51-for-和-range)

循环是所有编程语言都有的控制结构，除了使用经典的三段式循环之外，Go 语言还引入了另一个关键字 `range` 帮助我们快速遍历数组、切片、哈希表以及 Channel 等集合类型。本节将深入分析 Go 语言的两种循环，也就是 for 循环和 for-range 循环，我们会分析这两种循环的运行时结构以及它们的实现原理，

for 循环能够将代码中的数据和逻辑分离，让同一份代码能够多次复用相同的处理逻辑。我们先来看一下 Go 语言 for 循环对应的汇编代码，下面是一段经典的三段式循环的代码，我们将它编译成汇编指令：

```go
package main

func main() {
	for i := 0; i < 10; i++ {
		println(i)
	}
}

"".main STEXT size=98 args=0x0 locals=0x18
	00000 (main.go:3)	TEXT	"".main(SB), $24-0
	...
	00029 (main.go:3)	XORL	AX, AX                   ;; i := 0
	00031 (main.go:4)	JMP	75
	00033 (main.go:4)	MOVQ	AX, "".i+8(SP)
	00038 (main.go:5)	CALL	runtime.printlock(SB)
	00043 (main.go:5)	MOVQ	"".i+8(SP), AX
	00048 (main.go:5)	MOVQ	AX, (SP)
	00052 (main.go:5)	CALL	runtime.printint(SB)
	00057 (main.go:5)	CALL	runtime.printnl(SB)
	00062 (main.go:5)	CALL	runtime.printunlock(SB)
	00067 (main.go:4)	MOVQ	"".i+8(SP), AX
	00072 (main.go:4)	INCQ	AX                       ;; i++
	00075 (main.go:4)	CMPQ	AX, $10                  ;; 比较变量 i 和 10
	00079 (main.go:4)	JLT	33                           ;; 跳转到 33 行如果 i < 10
	...
```

Go

这里将上述汇编指令的执行过程分成三个部分进行分析：

1. 0029 ~ 0031 行负责循环的初始化；

   1. 对寄存器 `AX` 中的变量 `i` 进行初始化并执行 `JMP 75` 指令跳转到 0075 行；

2. 0075 ~ 0079 行负责检查循环的终止条件，将寄存器中存储的数据

    

   ```
   i
   ```

    

   与 10 比较；

   1. `JLT 33` 命令会在变量的值小于 10 时跳转到 0033 行执行循环主体；
   2. `JLT 33` 命令会在变量的值大于 10 时跳出循环体执行下面的代码；

3. 0033 ~ 0072 行是循环内部的语句；

   1. 通过多个汇编指令打印变量中的内容；
   2. `INCQ AX` 指令会将变量加一，然后再与 10 进行比较，回到第二步；

经过优化的 for-range 循环的汇编代码有着相同的结构。无论是变量的初始化、循环体的执行还是最后的条件判断都是完全一样的，所以这里也就不展开分析对应的汇编指令了。

```go
package main

func main() {
	arr := []int{1, 2, 3}
	for i, _ := range arr {
		println(i)
	}
}
```

Go

在汇编语言中，无论是经典的 for 循环还是 for-range 循环都会使用 `JMP` 等命令跳回循环体的开始位置复用代码。从不同循环具有相同的汇编代码可以猜到，使用 for-range 的控制结构最终也会被 Go 语言编译器转换成普通的 for 循环，后面的分析也会印证这一点。

## 5.1.1 现象 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#511-现象)

在深入语言源代码了解两种不同循环的实现之前，我们可以先来看一下使用 `for` 和 `range` 会遇到的一些问题，我们可以带着问题去源代码中寻找答案，这样能更高效地理解它们的实现原理。

### 循环永动机 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#循环永动机)

如果我们在遍历数组的同时修改数组的元素，能否得到一个永远都不会停止的循环呢？你可以尝试运行下面的代码：

```go
func main() {
	arr := []int{1, 2, 3}
	for _, v := range arr {
		arr = append(arr, v)
	}
	fmt.Println(arr)
}

$ go run main.go
1 2 3 1 2 3
```

Go

上述代码的输出意味着循环只遍历了原始切片中的三个元素，我们在遍历切片时追加的元素不会增加循环的执行次数，所以循环最终还是停了下来。

### 神奇的指针 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#神奇的指针)

第二个例子是使用 Go 语言经常会犯的错误[1](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#fn:1)。当我们在遍历一个数组时，如果获取 `range` 返回变量的地址并保存到另一个数组或者哈希时，会遇到令人困惑的现象，下面的代码会输出 “3 3 3”：

```go
func main() {
	arr := []int{1, 2, 3}
	newArr := []*int{}
	for _, v := range arr {
		newArr = append(newArr, &v)
	}
	for _, v := range newArr {
		fmt.Println(*v)
	}
}

$ go run main.go
3 3 3
```

Go

一些有经验的开发者不经意也会犯这种错误，正确的做法应该是使用 `&arr[i]` 替代 `&v`，我们会在下面分析这一现象背后的原因。

### 遍历清空数组 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#遍历清空数组)

当我们想要在 Go 语言中清空一个切片或者哈希时，一般都会使用以下的方法将切片中的元素置零：

```go
func main() {
	arr := []int{1, 2, 3}
	for i, _ := range arr {
		arr[i] = 0
	}
}
```

Go

依次遍历切片和哈希看起来是非常耗费性能的，因为数组、切片和哈希占用的内存空间都是连续的，所以最快的方法是直接清空这片内存中的内容，当我们编译上述代码时会得到以下的汇编指令：

```go
"".main STEXT size=93 args=0x0 locals=0x30
	0x0000 00000 (main.go:3)	TEXT	"".main(SB), $48-0
	...
	0x001d 00029 (main.go:4)	MOVQ	"".statictmp_0(SB), AX
	0x0024 00036 (main.go:4)	MOVQ	AX, ""..autotmp_3+16(SP)
	0x0029 00041 (main.go:4)	MOVUPS	"".statictmp_0+8(SB), X0
	0x0030 00048 (main.go:4)	MOVUPS	X0, ""..autotmp_3+24(SP)
	0x0035 00053 (main.go:5)	PCDATA	$2, $1
	0x0035 00053 (main.go:5)	LEAQ	""..autotmp_3+16(SP), AX
	0x003a 00058 (main.go:5)	PCDATA	$2, $0
	0x003a 00058 (main.go:5)	MOVQ	AX, (SP)
	0x003e 00062 (main.go:5)	MOVQ	$24, 8(SP)
	0x0047 00071 (main.go:5)	CALL	runtime.memclrNoHeapPointers(SB)
	...
```

Go

从生成的汇编代码我们可以看出，编译器会直接使用 [`runtime.memclrNoHeapPointers`](https://draveness.me/golang/tree/runtime.memclrNoHeapPointers) 清空切片中的数据，这也是我们在下面的小节会介绍的内容。

### 随机遍历 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#随机遍历)

当我们在 Go 语言中使用 `range` 遍历哈希表时，往往都会使用如下的代码结构，但是这段代码在每次运行时都会打印出不同的结果：

```go
func main() {
	hash := map[string]int{
		"1": 1,
		"2": 2,
		"3": 3,
	}
	for k, v := range hash {
		println(k, v)
	}
}
```

Go

两次运行上述代码可能会得到不同的结果，第一次会打印 `2 3 1`，第二次会打印 `1 2 3`，如果我们运行的次数足够多，最后会得到几种不同的遍历顺序。

```bash
$ go run main.go
2 2
3 3
1 1

$ go run main.go
1 1
2 2
3 3
```

Bash

Go 语言在运行时为哈希表的遍历引入了不确定性，也是告诉所有 Go 语言的使用者，程序不要依赖于哈希表的稳定遍历，我们在下面的小节会介绍在遍历的过程是如何引入不确定性的。

## 5.1.2 经典循环 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#512-经典循环)

Go 语言中的经典循环在编译器看来是一个 `OFOR` 类型的节点，这个节点由以下四个部分组成：

1. 初始化循环的 `Ninit`；
2. 循环的继续条件 `Left`；
3. 循环体结束时执行的 `Right`；
4. 循环体 `NBody`：

```go
for Ninit; Left; Right {
    NBody
}
```

Go

在生成 SSA 中间代码的阶段，[`cmd/compile/internal/gc.state.stmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.stmt) 方法在发现传入的节点类型是 `OFOR` 时会执行以下的代码块，这段代码会将循环中的代码分成不同的块：

```go
func (s *state) stmt(n *Node) {
	switch n.Op {
	case OFOR, OFORUNTIL:
		bCond, bBody, bIncr, bEnd := ...

		b := s.endBlock()
		b.AddEdgeTo(bCond)
		s.startBlock(bCond)
		s.condBranch(n.Left, bBody, bEnd, 1)

		s.startBlock(bBody)
		s.stmtList(n.Nbody)

		b.AddEdgeTo(bIncr)
		s.startBlock(bIncr)
		s.stmt(n.Right)
		b.AddEdgeTo(bCond)
		s.startBlock(bEnd)
	}
}
```

Go

一个常见的 for 循环代码会被 [`cmd/compile/internal/gc.state.stmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.stmt) 转换成下面的控制结构，该结构中包含了 4 个不同的块，这些代码块之间的连接表示汇编语言中的跳转关系，与我们理解的 for 循环控制结构没有太多的差别。

![golang-for-loop-ssa](https://img.draveness.me/2020-01-17-15792766877627-golang-for-loop-ssa.png)

**图 5-1 Go 语言循环生成的 SSA 代码**

[机器码生成](https://draveness.me/golang/docs/part1-prerequisite/ch02-compile/golang-machinecode/)阶段会将这些代码块转换成机器码，以及指定 CPU 架构上运行的机器语言，就是我们在前面编译得到的汇编指令。

## 5.1.3 范围循环 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#513-范围循环)

与简单的经典循环相比，范围循环在 Go 语言中更常见、实现也更复杂。这种循环同时使用 `for` 和 `range` 两个关键字，编译器会在编译期间将所有 for-range 循环变成经典循环。从编译器的视角来看，就是将 `ORANGE` 类型的节点转换成 `OFOR` 节点:

![Golang-For-Range-Loop](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409192236210.png)

**图 5-2 范围循环、普通循环和 SSA**

节点类型的转换过程都发生在中间代码生成阶段，所有的 for-range 循环都会被 [`cmd/compile/internal/gc.walkrange`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkrange) 转换成不包含复杂结构、只包含基本表达式的语句。接下来，我们按照循环遍历的元素类型依次介绍遍历数组和切片、哈希表、字符串以及管道时的过程。

### 数组和切片 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#数组和切片)

对于数组和切片来说，Go 语言有三种不同的遍历方式，这三种不同的遍历方式分别对应着代码中的不同条件，它们会在 [`cmd/compile/internal/gc.walkrange`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkrange) 函数中转换成不同的控制逻辑，我们会分成几种情况分析该函数的逻辑：

1. 分析遍历数组和切片清空元素的情况；
2. 分析使用 `for range a {}` 遍历数组和切片，不关心索引和数据的情况；
3. 分析使用 `for i := range a {}` 遍历数组和切片，只关心索引的情况；
4. 分析使用 `for i, elem := range a {}` 遍历数组和切片，关心索引和数据的情况；

```go
func walkrange(n *Node) *Node {
	switch t.Etype {
	case TARRAY, TSLICE:
		if arrayClear(n, v1, v2, a) {
			return n
		}
```

Go

[`cmd/compile/internal/gc.arrayClear`](https://draveness.me/golang/tree/cmd/compile/internal/gc.arrayClear) 是一个非常有趣的优化，它会优化 Go 语言遍历数组或者切片并删除全部元素的逻辑：

```go
// 原代码
for i := range a {
	a[i] = zero
}

// 优化后
if len(a) != 0 {
	hp = &a[0]
	hn = len(a)*sizeof(elem(a))
	memclrNoHeapPointers(hp, hn)
	i = len(a) - 1
}
```

Go

相比于依次清除数组或者切片中的数据，Go 语言会直接使用 [`runtime.memclrNoHeapPointers`](https://draveness.me/golang/tree/runtime.memclrNoHeapPointers) 或者 [`runtime.memclrHasPointers`](https://draveness.me/golang/tree/runtime.memclrHasPointers) 清除目标数组内存空间中的全部数据，并在执行完成后更新遍历数组的索引，这也印证了我们在遍历清空数组一节中观察到的现象。

处理了这种特殊的情况之后，我们可以回到 `ORANGE` 节点的处理过程了。这里会设置 for 循环的 `Left` 和 `Right` 字段，也就是终止条件和循环体每次执行结束后运行的代码：

```go
		ha := a

		hv1 := temp(types.Types[TINT])
		hn := temp(types.Types[TINT])

		init = append(init, nod(OAS, hv1, nil))
		init = append(init, nod(OAS, hn, nod(OLEN, ha, nil)))

		n.Left = nod(OLT, hv1, hn)
		n.Right = nod(OAS, hv1, nod(OADD, hv1, nodintconst(1)))

		if v1 == nil {
			break
		}
```

Go

如果循环是 `for range a {}`，那么就满足了上述代码中的条件 `v1 == nil`，即循环不关心数组的索引和数据，这种循环会被编译器转换成如下形式：

```go
ha := a
hv1 := 0
hn := len(ha)
v1 := hv1
for ; hv1 < hn; hv1++ {
    ...
}
```

Go

这是 `ORANGE` 结构在编译期间被转换的最简单形式，由于原代码不需要获取数组的索引和元素，只需要使用数组或者切片的数量执行对应次数的循环，所以会生成一个最简单的 for 循环。

如果我们在遍历数组时需要使用索引 `for i := range a {}`，那么编译器会继续会执行下面的代码：

```go
		if v2 == nil {
			body = []*Node{nod(OAS, v1, hv1)}
			break
		}
```

Go

`v2 == nil` 意味着调用方不关心数组的元素，只关心遍历数组使用的索引。它会将 `for i := range a {}` 转换成下面的逻辑，与第一种循环相比，这种循环在循环体中添加了 `v1 := hv1` 语句，传递遍历数组时的索引：

```go
ha := a
hv1 := 0
hn := len(ha)
v1 := hv1
for ; hv1 < hn; hv1++ {
    v1 = hv1
    ...
}
```

Go

上面两种情况虽然也是使用 range 会经常遇到的情况，但是同时去遍历索引和元素也很常见。处理这种情况会使用下面这段的代码：

```go
		tmp := nod(OINDEX, ha, hv1)
		tmp.SetBounded(true)
		a := nod(OAS2, nil, nil)
		a.List.Set2(v1, v2)
		a.Rlist.Set2(hv1, tmp)
		body = []*Node{a}
	}
	n.Ninit.Append(init...)
	n.Nbody.Prepend(body...)

	return n
}
```

Go

这段代码处理的使用者同时关心索引和切片的情况。它不仅会在循环体中插入更新索引的语句，还会插入赋值操作让循环体内部的代码能够访问数组中的元素：

```go
ha := a
hv1 := 0
hn := len(ha)
v1 := hv1
v2 := nil
for ; hv1 < hn; hv1++ {
    tmp := ha[hv1]
    v1, v2 = hv1, tmp
    ...
}
```

Go

对于所有的 range 循环，Go 语言都会在编译期将原切片或者数组赋值给一个新变量 `ha`，在赋值的过程中就发生了拷贝，而我们又通过 `len` 关键字预先获取了切片的长度，所以在循环中追加新的元素也不会改变循环执行的次数，这也就解释了循环永动机一节提到的现象。

而遇到这种同时遍历索引和元素的 range 循环时，Go 语言会额外创建一个新的 `v2` 变量存储切片中的元素，**循环中使用的这个变量 v2 会在每一次迭代被重新赋值而覆盖，赋值时也会触发拷贝**。

```go
func main() {
	arr := []int{1, 2, 3}
	newArr := []*int{}
	for i, _ := range arr {
		newArr = append(newArr, &arr[i])
	}
	for _, v := range newArr {
		fmt.Println(*v)
	}
}
```

Go

因为在循环中获取返回变量的地址都完全相同，所以会发生神奇的指针一节中的现象。因此当我们想要访问数组中元素所在的地址时，不应该直接获取 range 返回的变量地址 `&v2`，而应该使用 `&a[index]` 这种形式。

### 哈希表 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#哈希表)

在遍历哈希表时，编译器会使用 [`runtime.mapiterinit`](https://draveness.me/golang/tree/runtime.mapiterinit) 和 [`runtime.mapiternext`](https://draveness.me/golang/tree/runtime.mapiternext) 两个运行时函数重写原始的 for-range 循环：

```go
ha := a
hit := hiter(n.Type)
th := hit.Type
mapiterinit(typename(t), ha, &hit)
for ; hit.key != nil; mapiternext(&hit) {
    key := *hit.key
    val := *hit.val
}
```

Go

上述代码是展开 `for key, val := range hash {}` 后的结果，在 [`cmd/compile/internal/gc.walkrange`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkrange) 处理 `TMAP` 节点时，编译器会根据 range 返回值的数量在循环体中插入需要的赋值语句：

![golang-range-map](https://img.draveness.me/2020-01-17-15792766877639-golang-range-map.png)

**图 5-3 不同方式遍历哈希插入的语句**

这三种不同的情况分别向循环体插入了不同的赋值语句。遍历哈希表时会使用 [`runtime.mapiterinit`](https://draveness.me/golang/tree/runtime.mapiterinit) 函数初始化遍历开始的元素：

```go
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	it.t = t
	it.h = h
	it.B = h.B
	it.buckets = h.buckets

	r := uintptr(fastrand())
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))
	it.bucket = it.startBucket
	mapiternext(it)
}
```

Go

该函数会初始化 [`runtime.hiter`](https://draveness.me/golang/tree/runtime.hiter) 结构体中的字段，并通过 [`runtime.fastrand`](https://draveness.me/golang/tree/runtime.fastrand) 生成一个随机数帮助我们随机选择一个遍历桶的起始位置。Go 团队在设计哈希表的遍历时就不想让使用者依赖固定的遍历顺序，所以引入了随机数保证遍历的随机性。

遍历哈希会使用 [`runtime.mapiternext`](https://draveness.me/golang/tree/runtime.mapiternext)，我们在这里简化了很多逻辑，省去了一些边界条件以及哈希表扩容时的兼容操作，这里只需要关注处理遍历逻辑的核心代码，我们会将该函数分成桶的选择和桶内元素的遍历两部分，首先是桶的选择过程：

```go
func mapiternext(it *hiter) {
	h := it.h
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	alg := t.key.alg

next:
	if b == nil {
		if bucket == it.startBucket && it.wrapped {
			it.key = nil
			it.value = nil
			return
		}
		b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
```

Go

这段代码主要有两个作用：

1. 在待遍历的桶为空时，选择需要遍历的新桶；
2. 在不存在待遍历的桶时。返回 `(nil, nil)` 键值对并中止遍历；

[`runtime.mapiternext`](https://draveness.me/golang/tree/runtime.mapiternext) 剩余代码的作用是从桶中找到下一个遍历的元素，在大多数情况下都会直接操作内存获取目标键值的内存地址，不过如果哈希表处于扩容期间就会调用 [`runtime.mapaccessK`](https://draveness.me/golang/tree/runtime.mapaccessK) 获取键值对：

```go
	for ; i < bucketCnt; i++ {
		offi := (i + it.offset) & (bucketCnt - 1)
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.valuesize))
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey() || alg.equal(k, k)) {
			it.key = k
			it.value = v
		} else {
			rk, rv := mapaccessK(t, h, k)
			it.key = rk
			it.value = rv
		}
		it.bucket = bucket
		it.i = i + 1
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}
```

Go

当上述函数已经遍历了正常桶后，会通过 [`runtime.bmap.overflow`](https://draveness.me/golang/tree/runtime.bmap.overflow) 遍历哈希中的溢出桶。

![golang-range-map-and-buckets](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409192236506.png)

**图 5-4 哈希表的遍历过程**

简单总结一下哈希表遍历的顺序，首先会选出一个绿色的正常桶开始遍历，随后遍历所有黄色的溢出桶，最后依次按照索引顺序遍历哈希表中其他的桶，直到所有的桶都被遍历完成。

### 字符串 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#字符串)

遍历字符串的过程与数组、切片和哈希表非常相似，只是在遍历时会获取字符串中索引对应的字节并将字节转换成 `rune`。我们在遍历字符串时拿到的值都是 `rune` 类型的变量，`for i, r := range s {}` 的结构都会被转换成如下所示的形式：

```go
ha := s
for hv1 := 0; hv1 < len(ha); {
    hv1t := hv1
    hv2 := rune(ha[hv1])
    if hv2 < utf8.RuneSelf {
        hv1++
    } else {
        hv2, hv1 = decoderune(ha, hv1)
    }
    v1, v2 = hv1t, hv2
}
```

Go

在前面的字符串一节中我们曾经介绍过字符串是一个只读的字节数组切片，所以范围循环在编译期间生成的框架与切片非常类似，只是细节有一些不同。

使用下标访问字符串中的元素时得到的就是字节，但是这段代码会将当前的字节转换成 `rune` 类型。如果当前的 `rune` 是 ASCII 的，那么只会占用一个字节长度，每次循环体运行之后只需要将索引加一，但是如果当前 `rune` 占用了多个字节就会使用 [`runtime.decoderune`](https://draveness.me/golang/tree/runtime.decoderune) 函数解码，具体的过程就不在这里详细介绍了。

### 通道 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-for-range/#通道)

使用 range 遍历 Channel 也是比较常见的做法，一个形如 `for v := range ch {}` 的语句最终会被转换成如下的格式：

```go
ha := a
hv1, hb := <-ha
for ; hb != false; hv1, hb = <-ha {
    v1 := hv1
    hv1 = nil
    ...
}
```

Go

这里的代码可能与编译器生成的稍微有一些出入，但是结构和效果是完全相同的。该循环会使用 `<-ch` 从管道中取出等待处理的值，这个操作会调用 [`runtime.chanrecv2`](https://draveness.me/golang/tree/runtime.chanrecv2) 并阻塞当前的协程，当 [`runtime.chanrecv2`](https://draveness.me/golang/tree/runtime.chanrecv2) 返回时会根据布尔值 `hb` 判断当前的值是否存在：

- 如果不存在当前值，意味着当前的管道已经被关闭；
- 如果存在当前值，会为 `v1` 赋值并清除 `hv1` 变量中的数据，然后重新陷入阻塞等待新数据；

# 5.2 select [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#52-select)

`select` 是操作系统中的系统调用，我们经常会使用 `select`、`poll` 和 `epoll` 等函数构建 I/O 多路复用模型提升程序的性能。Go 语言的 `select` 与操作系统中的 `select` 比较相似，本节会介绍 Go 语言 `select` 关键字常见的现象、数据结构以及实现原理。

C 语言的 `select` 系统调用可以同时监听多个文件描述符的可读或者可写的状态，Go 语言中的 `select` 也能够让 Goroutine 同时等待多个 Channel 可读或者可写，在多个文件或者 Channel状态改变之前，`select` 会一直阻塞当前线程或者 Goroutine。

![Golang-Select-Channels](https://img.draveness.me/2020-01-19-15794018429532-Golang-Select-Channels.png)

**图 5-5 Select 和 Channel**

`select` 是与 `switch` 相似的控制结构，与 `switch` 不同的是，`select` 中虽然也有多个 `case`，但是这些 `case` 中的表达式必须都是 Channel 的收发操作。下面的代码就展示了一个包含 Channel 收发操作的 `select` 结构：

```go
func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}
```

Go

上述控制结构会等待 `c <- x` 或者 `<-quit` 两个表达式中任意一个返回。无论哪一个表达式返回都会立刻执行 `case` 中的代码，当 `select` 中的两个 `case` 同时被触发时，会随机执行其中的一个。

## 5.2.1 现象 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#521-现象)

当我们在 Go 语言中使用 `select` 控制结构时，会遇到两个有趣的现象：

1. `select` 能在 Channel 上进行非阻塞的收发操作；
2. `select` 在遇到多个 Channel 同时响应时，会随机执行一种情况；

这两个现象是学习 `select` 时经常会遇到的，我们来深入了解具体场景并分析这两个现象背后的设计原理。

### 非阻塞的收发 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#非阻塞的收发)

在通常情况下，`select` 语句会阻塞当前 Goroutine 并等待多个 Channel 中的一个达到可以收发的状态。但是如果 `select` 控制结构中包含 `default` 语句，那么这个 `select` 语句在执行时会遇到以下两种情况：

1. 当存在可以收发的 Channel 时，直接处理该 Channel 对应的 `case`；
2. 当不存在可以收发的 Channel 时，执行 `default` 中的语句；

当我们运行下面的代码时就不会阻塞当前的 Goroutine，它会直接执行 `default` 中的代码。

```go
func main() {
	ch := make(chan int)
	select {
	case i := <-ch:
		println(i)

	default:
		println("default")
	}
}

$ go run main.go
default
```

Go

只要我们稍微想一下，就会发现 Go 语言设计的这个现象很合理。`select` 的作用是同时监听多个 `case` 是否可以执行，如果多个 Channel 都不能执行，那么运行 `default` 也是理所当然的。

非阻塞的 Channel 发送和接收操作还是很有必要的，在很多场景下我们不希望 Channel 操作阻塞当前 Goroutine，只是想看看 Channel 的可读或者可写状态，如下所示：

```go
errCh := make(chan error, len(tasks))
wg := sync.WaitGroup{}
wg.Add(len(tasks))
for i := range tasks {
    go func() {
        defer wg.Done()
        if err := tasks[i].Run(); err != nil {
            errCh <- err
        }
    }()
}
wg.Wait()

select {
case err := <-errCh:
    return err
default:
    return nil
}
```

Go

在上面这段代码中，我们不关心到底多少个任务执行失败了，只关心是否存在返回错误的任务，最后的 `select` 语句能很好地完成这个任务。然而使用 `select` 实现非阻塞收发不是最初的设计，Go 语言在最初版本使用 `x, ok := <-c` 实现非阻塞的收发，以下是与非阻塞收发相关的提交：

1. [select default](https://github.com/golang/go/commit/79fbbe37a76502e6f5f9647d2d82bab953ab1546) 提交支持了 `select` 语句中的 `default`[1](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#fn:1)；
2. [gc: special case code for single-op blocking and non-blocking selects](https://github.com/golang/go/commit/5038792837355abde32f2e9549ef132fc5ffbd16) 提交引入了基于 `select` 的非阻塞收发[2](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#fn:2)。
3. [gc: remove non-blocking send, receive syntax](https://github.com/golang/go/commit/cb584707af2d8803adba88fd9692e665ecd2f059) 提交将 `x, ok := <-c` 语法删除[3](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#fn:3)；
4. [gc, runtime: replace closed(c) with x, ok := <-c](https://github.com/golang/go/commit/8bf34e335686816f7fe7e28614b2c7a3e04e9e7c) 提交使用 `x, ok := <-c` 语法替代 `closed(c)` 语法判断 Channel 的关闭状态[4](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#fn:4)；

我们可以从上面的几个提交中看到非阻塞收发从最初版本到现在的演变。

### 随机执行 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#随机执行)

另一个使用 `select` 遇到的情况是同时有多个 `case` 就绪时，`select` 会选择哪个 `case` 执行的问题，我们通过下面的代码可以简单了解一下：

```go
func main() {
	ch := make(chan int)
	go func() {
		for range time.Tick(1 * time.Second) {
			ch <- 0
		}
	}()

	for {
		select {
		case <-ch:
			println("case1")
		case <-ch:
			println("case2")
		}
	}
}

$ go run main.go
case1
case2
case1
...
```

Go

从上述代码输出的结果中我们可以看到，`select` 在遇到多个 `<-ch` 同时满足可读或者可写条件时会随机选择一个 `case` 执行其中的代码。

这个设计是在十多年前被 [select](https://github.com/golang/go/commit/cb9b1038db77198c2b0961634cf161258af2374d) 提交[5](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#fn:5)引入并一直保留到现在的，虽然中间经历过一些修改[6](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#fn:6)，但是语义一直都没有改变。在上面的代码中，两个 `case` 都是同时满足执行条件的，如果我们按照顺序依次判断，那么后面的条件永远都会得不到执行，而随机的引入就是为了避免饥饿问题的发生。

## 5.2.2 数据结构 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#522-数据结构)

`select` 在 Go 语言的源代码中不存在对应的结构体，但是我们使用 [`runtime.scase`](https://draveness.me/golang/tree/runtime.scase) 结构体表示 `select` 控制结构中的 `case`：

```go
type scase struct {
	c    *hchan         // chan
	elem unsafe.Pointer // data element
}
```

Go

因为非默认的 `case` 中都与 Channel 的发送和接收有关，所以 [`runtime.scase`](https://draveness.me/golang/tree/runtime.scase) 结构体中也包含一个 [`runtime.hchan`](https://draveness.me/golang/tree/runtime.hchan) 类型的字段存储 `case` 中使用的 Channel。

## 5.2.3 实现原理 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#523-实现原理)

`select` 语句在编译期间会被转换成 `OSELECT` 节点。每个 `OSELECT` 节点都会持有一组 `OCASE` 节点，如果 `OCASE` 的执行条件是空，那就意味着这是一个 `default` 节点。

![golang-oselect-and-ocases](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409192245320.png)

**图 5-7 OSELECT 和多个 OCASE**

上图展示的就是 `select` 语句在编译期间的结构，每一个 `OCASE` 既包含执行条件也包含满足条件后执行的代码。

编译器在中间代码生成期间会根据 `select` 中 `case` 的不同对控制语句进行优化，这一过程都发生在 [`cmd/compile/internal/gc.walkselectcases`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkselectcases) 函数中，我们在这里会分四种情况介绍处理的过程和结果：

1. `select` 不存在任何的 `case`；
2. `select` 只存在一个 `case`；
3. `select` 存在两个 `case`，其中一个 `case` 是 `default`；
4. `select` 存在多个 `case`；

上述四种情况不仅会涉及编译器的重写和优化，还会涉及 Go 语言的运行时机制，我们会从编译期间和运行时两个角度分析上述情况。

### 直接阻塞 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#直接阻塞)

首先介绍的是最简单的情况，也就是当 `select` 结构中不包含任何 `case`。我们截取 [`cmd/compile/internal/gc.walkselectcases`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkselectcases) 函数的前几行代码：

```go
func walkselectcases(cases *Nodes) []*Node {
	n := cases.Len()

	if n == 0 {
		return []*Node{mkcall("block", nil, nil)}
	}
	...
}
```

Go

这段代码很简单并且容易理解，它直接将类似 `select {}` 的语句转换成调用 [`runtime.block`](https://draveness.me/golang/tree/runtime.block) 函数：

```go
func block() {
	gopark(nil, nil, waitReasonSelectNoCases, traceEvGoStop, 1)
}
```

Go

[`runtime.block`](https://draveness.me/golang/tree/runtime.block) 的实现非常简单，它会调用 [`runtime.gopark`](https://draveness.me/golang/tree/runtime.gopark) 让出当前 Goroutine 对处理器的使用权并传入等待原因 `waitReasonSelectNoCases`。

简单总结一下，空的 `select` 语句会直接阻塞当前 Goroutine，导致 Goroutine 进入无法被唤醒的永久休眠状态。

### 单一管道 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#单一管道)

如果当前的 `select` 条件只包含一个 `case`，那么编译器会将 `select` 改写成 `if` 条件语句。下面对比了改写前后的代码：

```go
// 改写前
select {
case v, ok <-ch: // case ch <- v
    ...    
}

// 改写后
if ch == nil {
    block()
}
v, ok := <-ch // case ch <- v
...
```

Go

[`cmd/compile/internal/gc.walkselectcases`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkselectcases) 在处理单操作 `select` 语句时，会根据 Channel 的收发情况生成不同的语句。当 `case` 中的 Channel 是空指针时，会直接挂起当前 Goroutine 并陷入永久休眠。

### 非阻塞操作 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#非阻塞操作)

当 `select` 中仅包含两个 `case`，并且其中一个是 `default` 时，Go 语言的编译器就会认为这是一次非阻塞的收发操作。[`cmd/compile/internal/gc.walkselectcases`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkselectcases) 会对这种情况单独处理。不过在正式优化之前，该函数会将 `case` 中的所有 Channel 都转换成指向 Channel 的地址，我们会分别介绍非阻塞发送和非阻塞接收时，编译器进行的不同优化。

#### 发送 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#发送)

首先是 Channel 的发送过程，当 `case` 中表达式的类型是 `OSEND` 时，编译器会使用条件语句和 [`runtime.selectnbsend`](https://draveness.me/golang/tree/runtime.selectnbsend) 函数改写代码：

```go
select {
case ch <- i:
    ...
default:
    ...
}

if selectnbsend(ch, i) {
    ...
} else {
    ...
}
```

Go

这段代码中最重要的就是 [`runtime.selectnbsend`](https://draveness.me/golang/tree/runtime.selectnbsend)，它为我们提供了向 Channel 非阻塞地发送数据的能力。我们在 Channel 一节介绍了向 Channel 发送数据的 [`runtime.chansend`](https://draveness.me/golang/tree/runtime.chansend) 函数包含一个 `block` 参数，该参数会决定这一次的发送是不是阻塞的：

```go
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}
```

Go

由于我们向 [`runtime.chansend`](https://draveness.me/golang/tree/runtime.chansend) 函数传入了非阻塞，所以在不存在接收方或者缓冲区空间不足时，当前 Goroutine 都不会阻塞而是会直接返回。

#### 接收 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#接收)

由于从 Channel 中接收数据可能会返回一个或者两个值，所以接收数据的情况会比发送稍显复杂，不过改写的套路是差不多的：

```go
// 改写前
select {
case v <- ch: // case v, ok <- ch:
    ......
default:
    ......
}

// 改写后
if selectnbrecv(&v, ch) { // if selectnbrecv2(&v, &ok, ch) {
    ...
} else {
    ...
}
```

Go

返回值数量不同会导致使用函数的不同，两个用于非阻塞接收消息的函数 [`runtime.selectnbrecv`](https://draveness.me/golang/tree/runtime.selectnbrecv) 和 [`runtime.selectnbrecv2`](https://draveness.me/golang/tree/runtime.selectnbrecv2) 只是对 [`runtime.chanrecv`](https://draveness.me/golang/tree/runtime.chanrecv) 返回值的处理稍有不同：

```go
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
	selected, _ = chanrecv(c, elem, false)
	return
}

func selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool) {
	selected, *received = chanrecv(c, elem, false)
	return
}
```

Go

因为接收方不需要，所以 [`runtime.selectnbrecv`](https://draveness.me/golang/tree/runtime.selectnbrecv) 会直接忽略返回的布尔值，而 [`runtime.selectnbrecv2`](https://draveness.me/golang/tree/runtime.selectnbrecv2) 会将布尔值回传给调用方。与 [`runtime.chansend`](https://draveness.me/golang/tree/runtime.chansend) 一样，[`runtime.chanrecv`](https://draveness.me/golang/tree/runtime.chanrecv) 也提供了一个 `block` 参数用于控制这次接收是否阻塞。

### 常见流程 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#常见流程)

在默认的情况下，编译器会使用如下的流程处理 `select` 语句：

1. 将所有的 `case` 转换成包含 Channel 以及类型等信息的 [`runtime.scase`](https://draveness.me/golang/tree/runtime.scase) 结构体；
2. 调用运行时函数 [`runtime.selectgo`](https://draveness.me/golang/tree/runtime.selectgo) 从多个准备就绪的 Channel 中选择一个可执行的 [`runtime.scase`](https://draveness.me/golang/tree/runtime.scase) 结构体；
3. 通过 `for` 循环生成一组 `if` 语句，在语句中判断自己是不是被选中的 `case`；

一个包含三个 `case` 的正常 `select` 语句其实会被展开成如下所示的逻辑，我们可以看到其中处理的三个部分：

```go
selv := [3]scase{}
order := [6]uint16
for i, cas := range cases {
    c := scase{}
    c.kind = ...
    c.elem = ...
    c.c = ...
}
chosen, revcOK := selectgo(selv, order, 3)
if chosen == 0 {
    ...
    break
}
if chosen == 1 {
    ...
    break
}
if chosen == 2 {
    ...
    break
}
```

Go

展开后的代码片段中最重要的就是用于选择待执行 `case` 的运行时函数 [`runtime.selectgo`](https://draveness.me/golang/tree/runtime.selectgo)，这也是我们要关注的重点。因为这个函数的实现比较复杂， 所以这里分两部分分析它的执行过程：

1. 执行一些必要的初始化操作并确定 `case` 的处理顺序；
2. 在循环中根据 `case` 的类型做出不同的处理；

#### 初始化 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#初始化)

[`runtime.selectgo`](https://draveness.me/golang/tree/runtime.selectgo) 函数首先会进行执行必要的初始化操作并决定处理 `case` 的两个顺序 — 轮询顺序 `pollOrder` 和加锁顺序 `lockOrder`：

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))
	
	ncases := nsends + nrecvs
	scases := cas1[:ncases:ncases]
	pollorder := order1[:ncases:ncases]
	lockorder := order1[ncases:][:ncases:ncases]

	norder := 0
	for i := range scases {
		cas := &scases[i]
	}

	for i := 1; i < ncases; i++ {
		j := fastrandn(uint32(i + 1))
		pollorder[norder] = pollorder[j]
		pollorder[j] = uint16(i)
		norder++
	}
	pollorder = pollorder[:norder]
	lockorder = lockorder[:norder]

	// 根据 Channel 的地址排序确定加锁顺序
	...
	sellock(scases, lockorder)
	...
}
```

Go

轮询顺序 `pollOrder` 和加锁顺序 `lockOrder` 分别是通过以下的方式确认的：

- 轮询顺序：通过 [`runtime.fastrandn`](https://draveness.me/golang/tree/runtime.fastrandn) 函数引入随机性；
- 加锁顺序：按照 Channel 的地址排序后确定加锁顺序；

随机的轮询顺序可以避免 Channel 的饥饿问题，保证公平性；而根据 Channel 的地址顺序确定加锁顺序能够避免死锁的发生。这段代码最后调用的 [`runtime.sellock`](https://draveness.me/golang/tree/runtime.sellock) 会按照之前生成的加锁顺序锁定 `select` 语句中包含所有的 Channel。

#### 循环 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#循环)

当我们为 `select` 语句锁定了所有 Channel 之后就会进入 [`runtime.selectgo`](https://draveness.me/golang/tree/runtime.selectgo) 函数的主循环，它会分三个阶段查找或者等待某个 Channel 准备就绪：

1. 查找是否已经存在准备就绪的 Channel，即可以执行收发操作；
2. 将当前 Goroutine 加入 Channel 对应的收发队列上并等待其他 Goroutine 的唤醒；
3. 当前 Goroutine 被唤醒之后找到满足条件的 Channel 并进行处理；

[`runtime.selectgo`](https://draveness.me/golang/tree/runtime.selectgo) 函数会根据不同情况通过 `goto` 语句跳转到函数内部的不同标签执行相应的逻辑，其中包括：

- `bufrecv`：可以从缓冲区读取数据；
- `bufsend`：可以向缓冲区写入数据；
- `recv`：可以从休眠的发送方获取数据；
- `send`：可以向休眠的接收方发送数据；
- `rclose`：可以从关闭的 Channel 读取 EOF；
- `sclose`：向关闭的 Channel 发送数据；
- `retc`：结束调用并返回；

我们先来分析循环执行的第一个阶段，查找已经准备就绪的 Channel。循环会遍历所有的 `case` 并找到需要被唤起的 [`runtime.sudog`](https://draveness.me/golang/tree/runtime.sudog) 结构，在这个阶段，我们会根据 `case` 的四种类型分别处理：

1. 当

    

   ```
   case
   ```

    

   不包含 Channel 时；

   - 这种 `case` 会被跳过；

2. 当

    

   ```
   case
   ```

    

   会从 Channel 中接收数据时；

   - 如果当前 Channel 的 `sendq` 上有等待的 Goroutine，就会跳到 `recv` 标签并从缓冲区读取数据后将等待 Goroutine 中的数据放入到缓冲区中相同的位置；
   - 如果当前 Channel 的缓冲区不为空，就会跳到 `bufrecv` 标签处从缓冲区获取数据；
   - 如果当前 Channel 已经被关闭，就会跳到 `rclose` 做一些清除的收尾工作；

3. 当

    

   ```
   case
   ```

    

   会向 Channel 发送数据时；

   - 如果当前 Channel 已经被关，闭就会直接跳到 `sclose` 标签，触发 `panic` 尝试中止程序；
   - 如果当前 Channel 的 `recvq` 上有等待的 Goroutine，就会跳到 `send` 标签向 Channel 发送数据；
   - 如果当前 Channel 的缓冲区存在空闲位置，就会将待发送的数据存入缓冲区；

4. 当

    

   ```
   select
   ```

    

   语句中包含

    

   ```
   default
   ```

    

   时；

   - 表示前面的所有 `case` 都没有被执行，这里会解锁所有 Channel 并返回，意味着当前 `select` 结构中的收发都是非阻塞的；

![golang-runtime-selectgo](https://img.draveness.me/2020-01-18-15793463657488-golang-runtime-selectgo.png)

**图 5-8 运行时 selectgo 函数**

第一阶段的主要职责是查找所有 `case` 中是否有可以立刻被处理的 Channel。无论是在等待的 Goroutine 上还是缓冲区中，只要存在数据满足条件就会立刻处理，如果不能立刻找到活跃的 Channel 就会进入循环的下一阶段，按照需要将当前 Goroutine 加入到 Channel 的 `sendq` 或者 `recvq` 队列中：

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	...
	gp = getg()
	nextp = &gp.waiting
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c
		sg := acquireSudog()
		sg.g = gp
		sg.c = c

		if casi < nsends {
			c.sendq.enqueue(sg)
		} else {
			c.recvq.enqueue(sg)
		}
	}

	gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)
	...
}
```

Go

除了将当前 Goroutine 对应的 [`runtime.sudog`](https://draveness.me/golang/tree/runtime.sudog) 结构体加入队列之外，这些结构体都会被串成链表附着在 Goroutine 上。在入队之后会调用 [`runtime.gopark`](https://draveness.me/golang/tree/runtime.gopark) 挂起当前 Goroutine 等待调度器的唤醒。

![Golang-Select-Waiting](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409192245579.png)

**图 5-9 Goroutine 上等待收发的 sudog 链表**

等到 `select` 中的一些 Channel 准备就绪之后，当前 Goroutine 就会被调度器唤醒。这时会继续执行 [`runtime.selectgo`](https://draveness.me/golang/tree/runtime.selectgo) 函数的第三部分，从 [`runtime.sudog`](https://draveness.me/golang/tree/runtime.sudog) 中读取数据：

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	...
	sg = (*sudog)(gp.param)
	gp.param = nil

	casi = -1
	cas = nil
	sglist = gp.waiting
	for _, casei := range lockorder {
		k = &scases[casei]
		if sg == sglist {
			casi = int(casei)
			cas = k
		} else {
			c = k.c
			if int(casei) < nsends {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}

	c = cas.c
	goto retc
	...
}
```

Go

第三次遍历全部 `case` 时，我们会先获取当前 Goroutine 接收到的参数 `sudog` 结构，我们会依次对比所有 `case` 对应的 `sudog` 结构找到被唤醒的 `case`，获取该 `case` 对应的索引并返回。

由于当前的 `select` 结构找到了一个 `case` 执行，那么剩下 `case` 中没有被用到的 `sudog` 就会被忽略并且释放掉。为了不影响 Channel 的正常使用，我们还是需要将这些废弃的 `sudog` 从 Channel 中出队。

当我们在循环中发现缓冲区中有元素或者缓冲区未满时就会通过 `goto` 关键字跳转到 `bufrecv` 和 `bufsend` 两个代码段，这两段代码的执行过程都很简单，它们只是向 Channel 中发送数据或者从缓冲区中获取新数据：

```go
bufrecv:
	recvOK = true
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	typedmemclr(c.elemtype, qp)
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount--
	selunlock(scases, lockorder)
	goto retc

bufsend:
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++
	selunlock(scases, lockorder)
	goto retc
```

Go

这里在缓冲区进行的操作和直接调用 [`runtime.chansend`](https://draveness.me/golang/tree/runtime.chansend) 和 [`runtime.chanrecv`](https://draveness.me/golang/tree/runtime.chanrecv) 差不多，上述两个过程在执行结束之后都会直接跳到 `retc` 字段。

两个直接收发 Channel 的情况会调用运行时函数 [`runtime.send`](https://draveness.me/golang/tree/runtime.send) 和 [`runtime.recv`](https://draveness.me/golang/tree/runtime.recv)，这两个函数会与处于休眠状态的 Goroutine 打交道：

```go
recv:
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	recvOK = true
	goto retc

send:
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	goto retc
```

Go

不过如果向关闭的 Channel 发送数据或者从关闭的 Channel 中接收数据，情况就稍微有一点复杂了：

- 从一个关闭 Channel 中接收数据会直接清除 Channel 中的相关内容；
- 向一个关闭的 Channel 发送数据就会直接 `panic` 造成程序崩溃：

```go
rclose:
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	goto retc

sclose:
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
```

Go

总体来看，`select` 语句中的 Channel 收发操作和直接操作 Channel 没有太多出入，只是由于 `select` 多出了 `default` 关键字所以会支持非阻塞的收发。

# 5.3 defer [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#53-defer)

很多现代的编程语言中都有 `defer` 关键字，Go 语言的 `defer` 会在当前函数返回前执行传入的函数，它会经常被用于关闭文件描述符、关闭数据库连接以及解锁资源。这一节会深入 Go 语言的源代码介绍 `defer` 关键字的实现原理，相信读者读完这一节会对 `defer` 的数据结构、实现以及调用过程有着更清晰的理解。

作为一个编程语言中的关键字，`defer` 的实现一定是由编译器和运行时共同完成的，不过在深入源码分析它的实现之前我们还是需要了解 `defer` 关键字的常见使用场景以及使用时的注意事项。

使用 `defer` 的最常见场景是在函数调用结束后完成一些收尾工作，例如在 `defer` 中回滚数据库的事务：

```go
func createPost(db *gorm.DB) error {
    tx := db.Begin()
    defer tx.Rollback()
    
    if err := tx.Create(&Post{Author: "Draveness"}).Error; err != nil {
        return err
    }
    
    return tx.Commit().Error
}
```

Go

在使用数据库事务时，我们可以使用上面的代码在创建事务后就立刻调用 `Rollback` 保证事务一定会回滚。哪怕事务真的执行成功了，那么调用 `tx.Commit()` 之后再执行 `tx.Rollback()` 也不会影响已经提交的事务。

## 5.3.1 现象 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#531-现象)

我们在 Go 语言中使用 `defer` 时会遇到两个常见问题，这里会介绍具体的场景并分析这两个现象背后的设计原理：

- `defer` 关键字的调用时机以及多次调用 `defer` 时执行顺序是如何确定的；
- `defer` 关键字使用传值的方式传递参数时会进行预计算，导致不符合预期的结果；

### 作用域 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#作用域)

向 `defer` 关键字传入的函数会在函数返回之前运行。假设我们在 `for` 循环中多次调用 `defer` 关键字：

```go
func main() {
	for i := 0; i < 5; i++ {
		defer fmt.Println(i)
	}
}

$ go run main.go
4
3
2
1
0
```

Go

运行上述代码会倒序执行传入 `defer` 关键字的所有表达式，因为最后一次调用 `defer` 时传入了 `fmt.Println(4)`，所以这段代码会优先打印 4。我们可以通过下面这个简单例子强化对 `defer` 执行时机的理解：

```go
func main() {
    {
        defer fmt.Println("defer runs")
        fmt.Println("block ends")
    }
    
    fmt.Println("main ends")
}

$ go run main.go
block ends
main ends
defer runs
```

Go

从上述代码的输出我们会发现，`defer` 传入的函数不是在退出代码块的作用域时执行的，它只会在当前函数和方法返回之前被调用。

### 预计算参数 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#预计算参数)

Go 语言中所有的函数调用都是传值的，虽然 `defer` 是关键字，但是也继承了这个特性。假设我们想要计算 `main` 函数运行的时间，可能会写出以下的代码：

```go
func main() {
	startedAt := time.Now()
	defer fmt.Println(time.Since(startedAt))
	
	time.Sleep(time.Second)
}

$ go run main.go
0s
```

Go

然而上述代码的运行结果并不符合我们的预期，这个现象背后的原因是什么呢？经过分析，我们会发现调用 `defer` 关键字会立刻拷贝函数中引用的外部参数，所以 `time.Since(startedAt)` 的结果不是在 `main` 函数退出之前计算的，而是在 `defer` 关键字调用时计算的，最终导致上述代码输出 0s。

想要解决这个问题的方法非常简单，我们只需要向 `defer` 关键字传入匿名函数：

```go
func main() {
	startedAt := time.Now()
	defer func() { fmt.Println(time.Since(startedAt)) }()
	
	time.Sleep(time.Second)
}

$ go run main.go
1s
```

Go

虽然调用 `defer` 关键字时也使用值传递，但是因为拷贝的是函数指针，所以 `time.Since(startedAt)` 会在 `main` 函数返回前调用并打印出符合预期的结果。

## 5.3.2 数据结构 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#532-数据结构)

在介绍 `defer` 函数的执行过程与实现原理之前，我们首先来了解一下 `defer` 关键字在 Go 语言源代码中对应的数据结构：

```go
type _defer struct {
	siz       int32
	started   bool
	openDefer bool
	sp        uintptr
	pc        uintptr
	fn        *funcval
	_panic    *_panic
	link      *_defer
}
```

Go

[`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体是延迟调用链表上的一个元素，所有的结构体都会通过 `link` 字段串联成链表。

![golang-defer-link](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409192300392.png)

**图 5-10 延迟调用链表**

我们简单介绍一下 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体中的几个字段：

- `siz` 是参数和结果的内存大小；
- `sp` 和 `pc` 分别代表栈指针和调用方的程序计数器；
- `fn` 是 `defer` 关键字中传入的函数；
- `_panic` 是触发延迟调用的结构体，可能为空；
- `openDefer` 表示当前 `defer` 是否经过开放编码的优化；

除了上述的这些字段之外，[`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 中还包含一些垃圾回收机制使用的字段，这里为了减少理解的成本就都省去了。

## 5.3.3 执行机制 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#533-执行机制)

中间代码生成阶段的 [`cmd/compile/internal/gc.state.stmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.stmt) 会负责处理程序中的 `defer`，该函数会根据条件的不同，使用三种不同的机制处理该关键字：

```go
func (s *state) stmt(n *Node) {
	...
	switch n.Op {
	case ODEFER:
		if s.hasOpenDefers {
			s.openDeferRecord(n.Left) // 开放编码
		} else {
			d := callDefer // 堆分配
			if n.Esc == EscNever {
				d = callDeferStack // 栈分配
			}
			s.callResult(n.Left, d)
		}
	}
}
```

Go

堆分配、栈分配和开放编码是处理 `defer` 关键字的三种方法，早期的 Go 语言会在堆上分配 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体，不过该实现的性能较差，Go 语言在 1.13 中引入栈上分配的结构体，减少了 30% 的额外开销[1](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#fn:1)，并在 1.14 中引入了基于开放编码的 `defer`，使得该关键字的额外开销可以忽略不计[2](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#fn:2)，我们在一节中会分别介绍三种不同类型 `defer` 的设计与实现原理。

## 5.3.4 堆上分配 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#534-堆上分配)

根据 [`cmd/compile/internal/gc.state.stmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.stmt) 方法对 `defer` 的处理我们可以看出，堆上分配的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体是默认的兜底方案，当该方案被启用时，编译器会调用 [`cmd/compile/internal/gc.state.callResult`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.callResult) 和 [`cmd/compile/internal/gc.state.call`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call)，这表示 `defer` 在编译器看来也是函数调用。

[`cmd/compile/internal/gc.state.call`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call) 会负责为所有函数和方法调用生成中间代码，它的工作包括以下内容：

1. 获取需要执行的函数名、闭包指针、代码指针和函数调用的接收方；
2. 获取栈地址并将函数或者方法的参数写入栈中；
3. 使用 [`cmd/compile/internal/gc.state.newValue1A`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.newValue1A) 以及相关函数生成函数调用的中间代码；
4. 如果当前调用的函数是 `defer`，那么会单独生成相关的结束代码块；
5. 获取函数的返回值地址并结束当前调用；

```go
func (s *state) call(n *Node, k callKind, returnResultAddr bool) *ssa.Value {
	...
	var call *ssa.Value
	if k == callDeferStack {
		// 在栈上初始化 defer 结构体
		...
	} else {
		...
		switch {
		case k == callDefer:
			aux := ssa.StaticAuxCall(deferproc, ACArgs, ACResults)
			call = s.newValue1A(ssa.OpStaticCall, types.TypeMem, aux, s.mem())
		...
		}
		call.AuxInt = stksize
	}
	s.vars[&memVar] = call
	...
}
```

Go

从上述代码中我们能看到，`defer` 关键字在运行期间会调用 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc)，这个函数接收了参数的大小和闭包所在的地址两个参数。

编译器不仅将 `defer` 关键字都转换成 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 函数，它还会通过以下三个步骤为所有调用 `defer` 的函数末尾插入 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 的函数调用：

1. [`cmd/compile/internal/gc.walkstmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkstmt) 在遇到 `ODEFER` 节点时会执行 `Curfn.Func.SetHasDefer(true)` 设置当前函数的 `hasdefer` 属性；
2. [`cmd/compile/internal/gc.buildssa`](https://draveness.me/golang/tree/cmd/compile/internal/gc.buildssa) 会执行 `s.hasdefer = fn.Func.HasDefer()` 更新 `state` 的 `hasdefer`；
3. [`cmd/compile/internal/gc.state.exit`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.exit) 会根据 `state` 的 `hasdefer` 在函数返回之前插入 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 的函数调用；

```go
func (s *state) exit() *ssa.Block {
	if s.hasdefer {
		...
		s.rtcall(Deferreturn, true, nil)
	}
	...
}
```

Go

当运行时将 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 分配到堆上时，Go 语言的编译器不仅将 `defer` 转换成了 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc)，还在所有调用 `defer` 的函数结尾插入了 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn)。上述两个运行时函数是 `defer` 关键字运行时机制的入口，它们分别承担了不同的工作：

- [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 负责创建新的延迟调用；
- [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 负责在函数调用结束时执行所有的延迟调用；

我们以上述两个函数为入口介绍 `defer` 关键字在运行时的执行过程与工作原理。

### 创建延迟调用 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#创建延迟调用)

[`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 会为 `defer` 创建一个新的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体、设置它的函数指针 `fn`、程序计数器 `pc` 和栈指针 `sp` 并将相关的参数拷贝到相邻的内存空间中：

```go
func deferproc(siz int32, fn *funcval) {
	sp := getcallersp()
	argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
	callerpc := getcallerpc()

	d := newdefer(siz)
	if d._panic != nil {
		throw("deferproc: d.panic != nil after newdefer")
	}
	d.fn = fn
	d.pc = callerpc
	d.sp = sp
	switch siz {
	case 0:
	case sys.PtrSize:
		*(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
	default:
		memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
	}

	return0()
}
```

Go

最后调用的 [`runtime.return0`](https://draveness.me/golang/tree/runtime.return0) 是唯一一个不会触发延迟调用的函数，它可以避免递归 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 的递归调用。

[`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 中 [`runtime.newdefer`](https://draveness.me/golang/tree/runtime.newdefer) 的作用是想尽办法获得 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体，这里包含三种路径：

1. 从调度器的延迟调用缓存池 `sched.deferpool` 中取出结构体并将该结构体追加到当前 Goroutine 的缓存池中；
2. 从 Goroutine 的延迟调用缓存池 `pp.deferpool` 中取出结构体；
3. 通过 [`runtime.mallocgc`](https://draveness.me/golang/tree/runtime.mallocgc) 在堆上创建一个新的结构体；

```go
func newdefer(siz int32) *_defer {
	var d *_defer
	sc := deferclass(uintptr(siz))
	gp := getg()
	if sc < uintptr(len(p{}.deferpool)) {
		pp := gp.m.p.ptr()
		if len(pp.deferpool[sc]) == 0 && sched.deferpool[sc] != nil {
			for len(pp.deferpool[sc]) < cap(pp.deferpool[sc])/2 && sched.deferpool[sc] != nil {
				d := sched.deferpool[sc]
				sched.deferpool[sc] = d.link
				pp.deferpool[sc] = append(pp.deferpool[sc], d)
			}
		}
		if n := len(pp.deferpool[sc]); n > 0 {
			d = pp.deferpool[sc][n-1]
			pp.deferpool[sc][n-1] = nil
			pp.deferpool[sc] = pp.deferpool[sc][:n-1]
		}
	}
	if d == nil {
		total := roundupsize(totaldefersize(uintptr(siz)))
		d = (*_defer)(mallocgc(total, deferType, true))
	}
	d.siz = siz
	d.link = gp._defer
	gp._defer = d
	return d
}
```

Go

无论使用哪种方式，只要获取到 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体，它都会被追加到所在 Goroutine `_defer` 链表的最前面。

![golang-new-defer](https://img.draveness.me/2020-01-19-15794017184614-golang-new-defer.png)

**图 5-11 追加新的延迟调用**

`defer` 关键字的插入顺序是从后向前的，而 `defer` 关键字执行是从前向后的，这也是为什么后调用的 `defer` 会优先执行。

### 执行延迟调用 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#执行延迟调用)

[`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 会从 Goroutine 的 `_defer` 链表中取出最前面的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 并调用 [`runtime.jmpdefer`](https://draveness.me/golang/tree/runtime.jmpdefer) 传入需要执行的函数和参数：

```go
func deferreturn(arg0 uintptr) {
	gp := getg()
	d := gp._defer
	if d == nil {
		return
	}
	sp := getcallersp()
	...

	switch d.siz {
	case 0:
	case sys.PtrSize:
		*(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
	default:
		memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
	}
	fn := d.fn
	gp._defer = d.link
	freedefer(d)
	jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}
```

Go

[`runtime.jmpdefer`](https://draveness.me/golang/tree/runtime.jmpdefer) 是一个用汇编语言实现的运行时函数，它的主要工作是跳转到 `defer` 所在的代码段并在执行结束之后跳转回 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn)。

```go
TEXT runtime·jmpdefer(SB), NOSPLIT, $0-8
	MOVL	fv+0(FP), DX	// fn
	MOVL	argp+4(FP), BX	// caller sp
	LEAL	-4(BX), SP	// caller sp after CALL
#ifdef GOBUILDMODE_shared
	SUBL	$16, (SP)	// return to CALL again
#else
	SUBL	$5, (SP)	// return to CALL again
#endif
	MOVL	0(DX), BX
	JMP	BX	// but first run the deferred function
```

Go

[`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 会多次判断当前 Goroutine 的 `_defer` 链表中是否有未执行的结构体，该函数只有在所有延迟函数都执行后才会返回。

## 5.3.5 栈上分配 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#535-栈上分配)

在默认情况下，我们可以看到 Go 语言中 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体都会在堆上分配，如果我们能够将部分结构体分配到栈上就可以节约内存分配带来的额外开销。

Go 语言团队在 1.13 中对 `defer` 关键字进行了优化，当该关键字在函数体中最多执行一次时，编译期间的 [`cmd/compile/internal/gc.state.call`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call) 会将结构体分配到栈上并调用 [`runtime.deferprocStack`](https://draveness.me/golang/tree/runtime.deferprocStack)：

```go
func (s *state) call(n *Node, k callKind) *ssa.Value {
	...
	var call *ssa.Value
	if k == callDeferStack {
		// 在栈上创建 _defer 结构体
		t := deferstruct(stksize)
		...

		ACArgs = append(ACArgs, ssa.Param{Type: types.Types[TUINTPTR], Offset: int32(Ctxt.FixedFrameSize())})
		aux := ssa.StaticAuxCall(deferprocStack, ACArgs, ACResults) // 调用 deferprocStack
		arg0 := s.constOffPtrSP(types.Types[TUINTPTR], Ctxt.FixedFrameSize())
		s.store(types.Types[TUINTPTR], arg0, addr)
		call = s.newValue1A(ssa.OpStaticCall, types.TypeMem, aux, s.mem())
		call.AuxInt = stksize
	} else {
		...
	}
	s.vars[&memVar] = call
	...
}
```

Go

因为在编译期间我们已经创建了 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体，所以在运行期间 [`runtime.deferprocStack`](https://draveness.me/golang/tree/runtime.deferprocStack) 只需要设置一些未在编译期间初始化的字段，就可以将栈上的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 追加到函数的链表上：

```go
func deferprocStack(d *_defer) {
	gp := getg()
	d.started = false
	d.heap = false // 栈上分配的 _defer
	d.openDefer = false
	d.sp = getcallersp()
	d.pc = getcallerpc()
	d.framepc = 0
	d.varp = 0
	*(*uintptr)(unsafe.Pointer(&d._panic)) = 0
	*(*uintptr)(unsafe.Pointer(&d.fd)) = 0
	*(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))
	*(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

	return0()
}
```

Go

除了分配位置的不同，栈上分配和堆上分配的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 并没有本质的不同，而该方法可以适用于绝大多数的场景，与堆上分配的 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 相比，该方法可以将 `defer` 关键字的额外开销降低 ~30%。

## 5.3.5 开放编码 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#535-开放编码)

Go 语言在 1.14 中通过开放编码（Open Coded）实现 `defer` 关键字，该设计使用代码内联优化 `defer` 关键的额外开销并引入函数数据 `funcdata` 管理 `panic` 的调用[3](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#fn:3)，该优化可以将 `defer` 的调用开销从 1.13 版本的 ~35ns 降低至 ~6ns 左右：

```go
With normal (stack-allocated) defers only:         35.4  ns/op
With open-coded defers:                             5.6  ns/op
Cost of function call alone (remove defer keyword): 4.4  ns/op
```

Go

然而开放编码作为一种优化 `defer` 关键字的方法，它不是在所有的场景下都会开启的，开放编码只会在满足以下的条件时启用：

1. 函数的 `defer` 数量少于或者等于 8 个；
2. 函数的 `defer` 关键字不能在循环中执行；
3. 函数的 `return` 语句与 `defer` 语句的乘积小于或者等于 15 个；

初看上述几个条件可能会觉得不明所以，但是当我们深入理解基于开放编码的优化就可以明白上述限制背后的原因，除了上述几个条件之外，也有其他的条件会限制开放编码的使用，不过这些都是不太重要的细节，我们在这里也不会深究。

### 启用优化 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#启用优化)

Go 语言会在编译期间就确定是否启用开放编码，在编译器生成中间代码之前，我们会使用 [`cmd/compile/internal/gc.walkstmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkstmt) 修改已经生成的抽象语法树，设置函数体上的 `OpenCodedDeferDisallowed` 属性：

```go
const maxOpenDefers = 8

func walkstmt(n *Node) *Node {
	switch n.Op {
	case ODEFER:
		Curfn.Func.SetHasDefer(true)
		Curfn.Func.numDefers++
		if Curfn.Func.numDefers > maxOpenDefers {
			Curfn.Func.SetOpenCodedDeferDisallowed(true)
		}
		if n.Esc != EscNever {
			Curfn.Func.SetOpenCodedDeferDisallowed(true)
		}
		fallthrough
	...
	}
}
```

Go

就像我们上面提到的，如果函数中 `defer` 关键字的数量多于 8 个或者 `defer` 关键字处于 `for` 循环中，那么我们在这里都会禁用开放编码优化，使用上两节提到的方法处理 `defer`。

在 SSA 中间代码生成阶段的 [`cmd/compile/internal/gc.buildssa`](https://draveness.me/golang/tree/cmd/compile/internal/gc.buildssa) 中，我们也能够看到启用开放编码优化的其他条件，也就是返回语句的数量与 `defer` 数量的乘积需要小于 15：

```go
func buildssa(fn *Node, worker int) *ssa.Func {
	...
	s.hasOpenDefers = s.hasdefer && !s.curfn.Func.OpenCodedDeferDisallowed()
	...
	if s.hasOpenDefers &&
		s.curfn.Func.numReturns*s.curfn.Func.numDefers > 15 {
		s.hasOpenDefers = false
	}
	...
}
```

Go

中间代码生成的这两个步骤会决定当前函数是否应该使用开放编码优化 `defer` 关键字，一旦确定使用开放编码，就会在编译期间初始化延迟比特和延迟记录。

### 延迟记录 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#延迟记录)

延迟比特和延迟记录是使用开放编码实现 `defer` 的两个最重要结构，一旦决定使用开放编码，[`cmd/compile/internal/gc.buildssa`](https://draveness.me/golang/tree/cmd/compile/internal/gc.buildssa) 会在编译期间在栈上初始化大小为 8 个比特的 `deferBits` 变量：

```go
func buildssa(fn *Node, worker int) *ssa.Func {
	...
	if s.hasOpenDefers {
		deferBitsTemp := tempAt(src.NoXPos, s.curfn, types.Types[TUINT8]) // 初始化延迟比特
		s.deferBitsTemp = deferBitsTemp
		startDeferBits := s.entryNewValue0(ssa.OpConst8, types.Types[TUINT8])
		s.vars[&deferBitsVar] = startDeferBits
		s.deferBitsAddr = s.addr(deferBitsTemp)
		s.store(types.Types[TUINT8], s.deferBitsAddr, startDeferBits)
		s.vars[&memVar] = s.newValue1Apos(ssa.OpVarLive, types.TypeMem, deferBitsTemp, s.mem(), false)
	}
}
```

Go

延迟比特中的每一个比特位都表示该位对应的 `defer` 关键字是否需要被执行，如下图所示，其中 8 个比特的倒数第二个比特在函数返回前被设置成了 1，那么该比特位对应的函数会在函数返回前执行：

![golang-defer-bits](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409192300252.png)

**图 5-12 延迟比特**

因为不是函数中所有的 `defer` 语句都会在函数返回前执行，如下所示的代码只会在 `if` 语句的条件为真时，其中的 `defer` 语句才会在结尾被执行[4](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#fn:4)：

```go
deferBits := 0 // 初始化 deferBits

_f1, _a1 := f1, a1  // 保存函数以及参数
deferBits |= 1 << 0 // 将 deferBits 最后一位置位 1

if condition {
    _f2, _a2 := f2, a2  // 保存函数以及参数
    deferBits |= 1 << 1 // 将 deferBits 倒数第二位置位 1
}
exit:

if deferBits & 1 << 1 != 0 {
    deferBits &^= 1 << 1
    _f2(a2)
}

if deferBits & 1 << 0 != 0 {
    deferBits &^= 1 << 0
    _f1(a1)
}
```

Go

延迟比特的作用就是标记哪些 `defer` 关键字在函数中被执行，这样在函数返回时可以根据对应 `deferBits` 的内容确定执行的函数，而正是因为 `deferBits` 的大小仅为 8 比特，所以该优化的启用条件为函数中的 `defer` 关键字少于 8 个。

上述伪代码展示了开放编码的实现原理，但是仍然缺少了一些细节，例如：传入 `defer` 关键字的函数和参数都会存储在如下所示的 [`cmd/compile/internal/gc.openDeferInfo`](https://draveness.me/golang/tree/cmd/compile/internal/gc.openDeferInfo) 结构体中：

```go
type openDeferInfo struct {
	n           *Node
	closure     *ssa.Value
	closureNode *Node
	rcvr        *ssa.Value
	rcvrNode    *Node
	argVals     []*ssa.Value
	argNodes    []*Node
}
```

Go

当编译器在调用 [`cmd/compile/internal/gc.buildssa`](https://draveness.me/golang/tree/cmd/compile/internal/gc.buildssa) 构建中间代码时会通过 [`cmd/compile/internal/gc.state.openDeferRecord`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.openDeferRecord) 方法在栈上构建结构体，该结构体的 `closure` 中存储着调用的函数，`rcvr` 中存储着方法的接收者，而最后的 `argVals` 中存储了函数的参数。

很多 `defer` 语句都可以在编译期间判断是否被执行，如果函数中的 `defer` 语句都会在编译期间确定，中间代码生成阶段就会直接调用 [`cmd/compile/internal/gc.state.openDeferExit`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.openDeferExit) 在函数返回前生成判断 `deferBits` 的代码，也就是上述伪代码中的后半部分。

不过当程序遇到运行时才能判断的条件语句时，我们仍然需要由运行时的 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 决定是否执行 `defer` 关键字：

```go
func deferreturn(arg0 uintptr) {
	gp := getg()
	d := gp._defer
	sp := getcallersp()
	if d.openDefer {
		runOpenDeferFrame(gp, d)
		gp._defer = d.link
		freedefer(d)
		return
	}
	...
}
```

Go

该函数为开放编码做了特殊的优化，运行时会调用 [`runtime.runOpenDeferFrame`](https://draveness.me/golang/tree/runtime.runOpenDeferFrame) 执行活跃的开放编码延迟函数，该函数会执行以下的工作：

1. 从 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体中读取 `deferBits`、函数 `defer` 数量等信息；
2. 在循环中依次读取函数的地址和参数信息并通过 `deferBits` 判断该函数是否需要被执行；
3. 调用 [`runtime.reflectcallSave`](https://draveness.me/golang/tree/runtime.reflectcallSave) 调用需要执行的 `defer` 函数；

```go
func runOpenDeferFrame(gp *g, d *_defer) bool {
	fd := d.fd

	...
	deferBitsOffset, fd := readvarintUnsafe(fd)
	nDefers, fd := readvarintUnsafe(fd)
	deferBits := *(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset)))

	for i := int(nDefers) - 1; i >= 0; i-- {
		var argWidth, closureOffset, nArgs uint32 // 读取函数的地址和参数信息
		argWidth, fd = readvarintUnsafe(fd)
		closureOffset, fd = readvarintUnsafe(fd)
		nArgs, fd = readvarintUnsafe(fd)
		if deferBits&(1<<i) == 0 {
			...
			continue
		}
		closure := *(**funcval)(unsafe.Pointer(d.varp - uintptr(closureOffset)))
		d.fn = closure

		...

		deferBits = deferBits &^ (1 << i)
		*(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset))) = deferBits
		p := d._panic
		reflectcallSave(p, unsafe.Pointer(closure), deferArgs, argWidth)
		if p != nil && p.aborted {
			break
		}
		d.fn = nil
		memclrNoHeapPointers(deferArgs, uintptr(argWidth))
		...
	}
	return done
}
```

Go

Go 语言的编译器为了支持开放编码在中间代码生成阶段做出了很多修改，我们在这里虽然省略了很多细节，但是也可以很好地展示 `defer` 关键字的实现原理。

# 5.4 panic 和 recover [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#54-panic-和-recover)

本节将分析 Go 语言中两个经常成对出现的两个关键字 — `panic` 和 `recover`。这两个关键字与上一节提到的 `defer` 有紧密的联系，它们都是 Go 语言中的内置函数，也提供了互补的功能。

![golang-panic](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409192310678.png)

**图 5-12 panic 触发的递归延迟调用**

- `panic` 能够改变程序的控制流，调用 `panic` 后会立刻停止执行当前函数的剩余代码，并在当前 Goroutine 中递归执行调用方的 `defer`；
- `recover` 可以中止 `panic` 造成的程序崩溃。它是一个只能在 `defer` 中发挥作用的函数，在其他作用域中调用不会发挥作用；

## 5.4.1 现象 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#541-现象)

我们先通过几个例子了解一下使用 `panic` 和 `recover` 关键字时遇到的现象，部分现象也与上一节分析的 `defer` 关键字有关：

- `panic` 只会触发当前 Goroutine 的 `defer`；
- `recover` 只有在 `defer` 中调用才会生效；
- `panic` 允许在 `defer` 中嵌套多次调用；

### 跨协程失效 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#跨协程失效)

首先要介绍的现象是 `panic` 只会触发当前 Goroutine 的延迟函数调用，我们可以通过如下所示的代码了解该现象：

```go
func main() {
	defer println("in main")
	go func() {
		defer println("in goroutine")
		panic("")
	}()

	time.Sleep(1 * time.Second)
}

$ go run main.go
in goroutine
panic:
...
```

Go

当我们运行这段代码时会发现 `main` 函数中的 `defer` 语句并没有执行，执行的只有当前 Goroutine 中的 `defer`。

前面我们曾经介绍过 `defer` 关键字对应的 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 会将延迟调用函数与调用方所在 Goroutine 进行关联。所以当程序发生崩溃时只会调用当前 Goroutine 的延迟调用函数也是非常合理的。

![golang-panic-and-defers](https://img.draveness.me/2020-01-19-15794253176199-golang-panic-and-defers.png)

**图 5-13 panic 触发当前 Goroutine 的延迟调用**

如上图所示，多个 Goroutine 之间没有太多的关联，一个 Goroutine 在 `panic` 时也不应该执行其他 Goroutine 的延迟函数。

### 失效的崩溃恢复 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#失效的崩溃恢复)

初学 Go 语言的读者可能会写出下面的代码，在主程序中调用 `recover` 试图中止程序的崩溃，但是从运行的结果中我们也能看出，下面的程序没有正常退出。

```go
func main() {
	defer fmt.Println("in main")
	if err := recover(); err != nil {
		fmt.Println(err)
	}

	panic("unknown err")
}

$ go run main.go
in main
panic: unknown err

goroutine 1 [running]:
main.main()
	...
exit status 2
```

Go

仔细分析一下这个过程就能理解这种现象背后的原因，`recover` 只有在发生 `panic` 之后调用才会生效。然而在上面的控制流中，`recover` 是在 `panic` 之前调用的，并不满足生效的条件，所以我们需要在 `defer` 中使用 `recover` 关键字。

### 嵌套崩溃 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#嵌套崩溃)

Go 语言中的 `panic` 是可以多次嵌套调用的。一些熟悉 Go 语言的读者很可能也不知道这个知识点，如下所示的代码就展示了如何在 `defer` 函数中多次调用 `panic`：

```go
func main() {
	defer fmt.Println("in main")
	defer func() {
		defer func() {
			panic("panic again and again")
		}()
		panic("panic again")
	}()

	panic("panic once")
}

$ go run main.go
in main
panic: panic once
	panic: panic again
	panic: panic again and again

goroutine 1 [running]:
...
exit status 2
```

Go

从上述程序输出的结果，我们可以确定程序多次调用 `panic` 也不会影响 `defer` 函数的正常执行，所以使用 `defer` 进行收尾工作一般来说都是安全的。

## 5.4.2 数据结构 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#542-数据结构)

`panic` 关键字在 Go 语言的源代码是由数据结构 [`runtime._panic`](https://draveness.me/golang/tree/runtime._panic) 表示的。每当我们调用 `panic` 都会创建一个如下所示的数据结构存储相关信息：

```go
type _panic struct {
	argp      unsafe.Pointer
	arg       interface{}
	link      *_panic
	recovered bool
	aborted   bool
	pc        uintptr
	sp        unsafe.Pointer
	goexit    bool
}
```

Go

1. `argp` 是指向 `defer` 调用时参数的指针；
2. `arg` 是调用 `panic` 时传入的参数；
3. `link` 指向了更早调用的 [`runtime._panic`](https://draveness.me/golang/tree/runtime._panic) 结构；
4. `recovered` 表示当前 [`runtime._panic`](https://draveness.me/golang/tree/runtime._panic) 是否被 `recover` 恢复；
5. `aborted` 表示当前的 `panic` 是否被强行终止；

从数据结构中的 `link` 字段我们就可以推测出以下的结论：`panic` 函数可以被连续多次调用，它们之间通过 `link` 可以组成链表。

结构体中的 `pc`、`sp` 和 `goexit` 三个字段都是为了修复 [`runtime.Goexit`](https://draveness.me/golang/tree/runtime.Goexit) 带来的问题引入的[1](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#fn:1)。[`runtime.Goexit`](https://draveness.me/golang/tree/runtime.Goexit) 能够只结束调用该函数的 Goroutine 而不影响其他的 Goroutine，但是该函数会被 `defer` 中的 `panic` 和 `recover` 取消[2](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#fn:2)，引入这三个字段就是为了保证该函数的一定会生效。

## 5.4.3 程序崩溃 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#543-程序崩溃)

这里先介绍分析 `panic` 函数是终止程序的实现原理。编译器会将关键字 `panic` 转换成 [`runtime.gopanic`](https://draveness.me/golang/tree/runtime.gopanic)，该函数的执行过程包含以下几个步骤：

1. 创建新的 [`runtime._panic`](https://draveness.me/golang/tree/runtime._panic) 并添加到所在 Goroutine 的 `_panic` 链表的最前面；
2. 在循环中不断从当前 Goroutine 的 `_defer` 中链表获取 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 并调用 [`runtime.reflectcall`](https://draveness.me/golang/tree/runtime.reflectcall) 运行延迟调用函数；
3. 调用 [`runtime.fatalpanic`](https://draveness.me/golang/tree/runtime.fatalpanic) 中止整个程序；

```go
func gopanic(e interface{}) {
	gp := getg()
	...
	var p _panic
	p.arg = e
	p.link = gp._panic
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

	for {
		d := gp._defer
		if d == nil {
			break
		}

		d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

		reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))

		d._panic = nil
		d.fn = nil
		gp._defer = d.link

		freedefer(d)
		if p.recovered {
			...
		}
	}

	fatalpanic(gp._panic)
	*(*int)(nil) = 0
}
```

Go

需要注意的是，我们在上述函数中省略了三部分比较重要的代码：

1. 恢复程序的 `recover` 分支中的代码；

2. 通过内联优化

    

   ```
   defer
   ```

    

   调用性能的代码

   [3](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#fn:3)

   ；

   - [runtime: make defers low-cost through inline code and extra funcdata](https://github.com/golang/go/commit/be64a19d99918c843f8555aad580221207ea35bc)

3. 修复 [`runtime.Goexit`](https://draveness.me/golang/tree/runtime.Goexit) 异常情况的代码；

> Go 语言在 1.14 通过 [runtime: ensure that Goexit cannot be aborted by a recursive panic/recover](https://github.com/golang/go/commit/7dcd343ed641d3b70c09153d3b041ca3fe83b25e) 提交解决了递归 `panic` 和 `recover` 与 [`runtime.Goexit`](https://draveness.me/golang/tree/runtime.Goexit) 的冲突。

[`runtime.fatalpanic`](https://draveness.me/golang/tree/runtime.fatalpanic) 实现了无法被恢复的程序崩溃，它在中止程序之前会通过 [`runtime.printpanics`](https://draveness.me/golang/tree/runtime.printpanics) 打印出全部的 `panic` 消息以及调用时传入的参数：

```go
func fatalpanic(msgs *_panic) {
	pc := getcallerpc()
	sp := getcallersp()
	gp := getg()

	if startpanic_m() && msgs != nil {
		atomic.Xadd(&runningPanicDefers, -1)
		printpanics(msgs)
	}
	if dopanic_m(gp, pc, sp) {
		crash()
	}

	exit(2)
}
```

Go

打印崩溃消息后会调用 [`runtime.exit`](https://draveness.me/golang/tree/runtime.exit) 退出当前程序并返回错误码 2，程序的正常退出也是通过 [`runtime.exit`](https://draveness.me/golang/tree/runtime.exit) 实现的。

## 5.4.4 崩溃恢复 [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/#544-崩溃恢复)

到这里我们已经掌握了 `panic` 退出程序的过程，接下来将分析 `defer` 中的 `recover` 是如何中止程序崩溃的。编译器会将关键字 `recover` 转换成 [`runtime.gorecover`](https://draveness.me/golang/tree/runtime.gorecover)：

```go
func gorecover(argp uintptr) interface{} {
	gp := getg()
	p := gp._panic
	if p != nil && !p.recovered && argp == uintptr(p.argp) {
		p.recovered = true
		return p.arg
	}
	return nil
}
```

Go

该函数的实现很简单，如果当前 Goroutine 没有调用 `panic`，那么该函数会直接返回 `nil`，这也是崩溃恢复在非 `defer` 中调用会失效的原因。在正常情况下，它会修改 [`runtime._panic`](https://draveness.me/golang/tree/runtime._panic) 的 `recovered` 字段，[`runtime.gorecover`](https://draveness.me/golang/tree/runtime.gorecover) 函数中并不包含恢复程序的逻辑，程序的恢复是由 [`runtime.gopanic`](https://draveness.me/golang/tree/runtime.gopanic) 函数负责的：

```go
func gopanic(e interface{}) {
	...

	for {
		// 执行延迟调用函数，可能会设置 p.recovered = true
		...

		pc := d.pc
		sp := unsafe.Pointer(d.sp)

		...
		if p.recovered {
			gp._panic = p.link
			for gp._panic != nil && gp._panic.aborted {
				gp._panic = gp._panic.link
			}
			if gp._panic == nil {
				gp.sig = 0
			}
			gp.sigcode0 = uintptr(sp)
			gp.sigcode1 = pc
			mcall(recovery)
			throw("recovery failed")
		}
	}
	...
}
```

Go

上述这段代码也省略了 `defer` 的内联优化，它从 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 中取出了程序计数器 `pc` 和栈指针 `sp` 并调用 [`runtime.recovery`](https://draveness.me/golang/tree/runtime.recovery) 函数触发 Goroutine 的调度，调度之前会准备好 `sp`、`pc` 以及函数的返回值：

```go
func recovery(gp *g) {
	sp := gp.sigcode0
	pc := gp.sigcode1

	gp.sched.sp = sp
	gp.sched.pc = pc
	gp.sched.lr = 0
	gp.sched.ret = 1
	gogo(&gp.sched)
}
```

Go

当我们在调用 `defer` 关键字时，调用时的栈指针 `sp` 和程序计数器 `pc` 就已经存储到了 [`runtime._defer`](https://draveness.me/golang/tree/runtime._defer) 结构体中，这里的 [`runtime.gogo`](https://draveness.me/golang/tree/runtime.gogo) 函数会跳回 `defer` 关键字调用的位置。

[`runtime.recovery`](https://draveness.me/golang/tree/runtime.recovery) 在调度过程中会将函数的返回值设置成 1。从 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 的注释中我们会发现，当 [`runtime.deferproc`](https://draveness.me/golang/tree/runtime.deferproc) 函数的返回值是 1 时，编译器生成的代码会直接跳转到调用方函数返回之前并执行 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn)：

```go
func deferproc(siz int32, fn *funcval) {
	...
	return0()
}
```

Go

跳转到 [`runtime.deferreturn`](https://draveness.me/golang/tree/runtime.deferreturn) 函数之后，程序就已经从 `panic` 中恢复了并执行正常的逻辑，而 [`runtime.gorecover`](https://draveness.me/golang/tree/runtime.gorecover) 函数也能从 [`runtime._panic`](https://draveness.me/golang/tree/runtime._panic) 结构中取出了调用 `panic` 时传入的 `arg` 参数并返回给调用方。

# 5.5 make 和 new [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/#55-make-和-new)

当我们想要在 Go 语言中初始化一个结构时，可能会用到两个不同的关键字 — `make` 和 `new`。因为它们的功能相似，所以初学者可能会对这两个关键字的作用感到困惑[1](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/#fn:1)，但是它们两者能够初始化的变量却有较大的不同。

- `make` 的作用是初始化内置的数据结构，也就是我们在前面提到的切片、哈希表和 Channel[2](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/#fn:2)；
- `new` 的作用是根据传入的类型分配一片内存空间并返回指向这片内存空间的指针[3](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/#fn:3)；

我们在代码中往往都会使用如下所示的语句初始化这三类基本类型，这三个语句分别返回了不同类型的数据结构：

```go
slice := make([]int, 0, 100)
hash := make(map[int]bool, 10)
ch := make(chan int, 5)
```

Go

1. `slice` 是一个包含 `data`、`cap` 和 `len` 的结构体 [`reflect.SliceHeader`](https://draveness.me/golang/tree/reflect.SliceHeader)；
2. `hash` 是一个指向 [`runtime.hmap`](https://draveness.me/golang/tree/runtime.hmap) 结构体的指针；
3. `ch` 是一个指向 [`runtime.hchan`](https://draveness.me/golang/tree/runtime.hchan) 结构体的指针；

相比与复杂的 `make` 关键字，`new` 的功能就简单多了，它只能接收类型作为参数然后返回一个指向该类型的指针：

```go
i := new(int)

var v int
i := &v
```

Go

上述代码片段中的两种不同初始化方法是等价的，它们都会创建一个指向 `int` 零值的指针。

![golang-make-and-new](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409192311654.png)

**图 5-14 make 和 new 初始化的类型**

接下来我们将分别介绍 `make` 和 `new` 初始化不同数据结构的过程，我们会从编译期间和运行时两个不同阶段理解这两个关键字的原理，不过由于前面的章节已经详细地分析过 `make` 的原理，所以这里会将重点放在另一个关键字 `new` 上。

## 5.5.1 make [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/#551-make)

在前面的章节中我们已经谈到过 `make` 在创建切片、哈希表和 Channel 的具体过程，所以在这一小节，我们只是会简单提及 `make` 相关的数据结构的初始化原理。

![golang-make-typecheck](https://img.draveness.me/golang-make-typecheck.png)

**图 5-15 make 关键字的类型检查**

在编译期间的类型检查阶段，Go 语言会将代表 `make` 关键字的 `OMAKE` 节点根据参数类型的不同转换成了 `OMAKESLICE`、`OMAKEMAP` 和 `OMAKECHAN` 三种不同类型的节点，这些节点会调用不同的运行时函数来初始化相应的数据结构。

## 5.5.2 new [#](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/#552-new)

编译器会在中间代码生成阶段通过以下两个函数处理该关键字：

1. [`cmd/compile/internal/gc.callnew`](https://draveness.me/golang/tree/cmd/compile/internal/gc.callnew) 会将关键字转换成 `ONEWOBJ` 类型的节点[2](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/#fn:2)；

2. `cmd/compile/internal/gc.state.expr`

    

   会根据申请空间的大小分两种情况处理：

   1. 如果申请的空间为 0，就会返回一个表示空指针的 `zerobase` 变量；
   2. 在遇到其他情况时会将关键字转换成 [`runtime.newobject`](https://draveness.me/golang/tree/runtime.newobject) 函数：

```go
func callnew(t *types.Type) *Node {
	...
	n := nod(ONEWOBJ, typename(t), nil)
	...
	return n
}

func (s *state) expr(n *Node) *ssa.Value {
	switch n.Op {
	case ONEWOBJ:
		if n.Type.Elem().Size() == 0 {
			return s.newValue1A(ssa.OpAddr, n.Type, zerobaseSym, s.sb)
		}
		typ := s.expr(n.Left)
		vv := s.rtcall(newobject, true, []*types.Type{n.Type}, typ)
		return vv[0]
	}
}
```

Go

需要注意的是，无论是直接使用 `new`，还是使用 `var` 初始化变量，它们在编译器看来都是 `ONEW` 和 `ODCL` 节点。如果变量会逃逸到堆上，这些节点在这一阶段都会被 [`cmd/compile/internal/gc.walkstmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.walkstmt) 转换成通过 [`runtime.newobject`](https://draveness.me/golang/tree/runtime.newobject) 函数并在堆上申请内存：

```go
func walkstmt(n *Node) *Node {
	switch n.Op {
	case ODCL:
		v := n.Left
		if v.Class() == PAUTOHEAP {
			if prealloc[v] == nil {
				prealloc[v] = callnew(v.Type)
			}
			nn := nod(OAS, v.Name.Param.Heapaddr, prealloc[v])
			nn.SetColas(true)
			nn = typecheck(nn, ctxStmt)
			return walkstmt(nn)
		}
	case ONEW:
		if n.Esc == EscNone {
			r := temp(n.Type.Elem())
			r = nod(OAS, r, nil)
			r = typecheck(r, ctxStmt)
			init.Append(r)
			r = nod(OADDR, r.Left, nil)
			r = typecheck(r, ctxExpr)
			n = r
		} else {
			n = callnew(n.Type.Elem())
		}
	}
}
```

Go

不过这也不是绝对的，如果通过 `var` 或者 `new` 创建的变量不需要在当前作用域外生存，例如不用作为返回值返回给调用方，那么就不需要初始化在堆上。

[`runtime.newobject`](https://draveness.me/golang/tree/runtime.newobject) 函数会获取传入类型占用空间的大小，调用 [`runtime.mallocgc`](https://draveness.me/golang/tree/runtime.mallocgc) 在堆上申请一片内存空间并返回指向这片内存空间的指针：

```go
func newobject(typ *_type) unsafe.Pointer {
	return mallocgc(typ.size, typ, true)
}
```

Go

[`runtime.mallocgc`](https://draveness.me/golang/tree/runtime.mallocgc) 函数的实现大概有 200 多行代码，我们会在后面的章节中详细分析 Go 语言的内存管理机制。
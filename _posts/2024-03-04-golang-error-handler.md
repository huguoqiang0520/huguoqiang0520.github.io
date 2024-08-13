---
layout: post
title: Go｜error处理与传播
categories: [golang]
mermaid: true
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# 背景

## 错误（广泛）处理流派

目前主流语言对于错误（广泛）处理主要有两大流派（这里不口水战究竟哪派好）：

- 结构化异常处理
    - 语言层提供类Exception机制以及try、throw、catch、finally等语法
    - 例如Java、PHP、JS等
- 返回值判断：
    - 其实只针对从错误（广泛）中特定的错误（可处理/该处理的），作为返回值返回
    - 例如C、Golang、Rust

## Go的error

Go将错误（广泛）区分成错误（error，本文后续错误的定义均为go error）和紊乱（panic），并倡导开发者：

- 不要随意使用panic（关于panic的特点和场景后续再谈）
- 面对错误，显式处理错误

## 思维惯性的改变

我们需要顺着Go对错误的设计理念和使用原则，而不是带着结构化异常处理语言的思维习惯来看待和使用Go，否则很可能：

- 对go的错误处理骂骂咧咧（虽然确实也有不太优雅的地方）
- 肆意panic，毕竟看起来很像异常抛出和捕获机制

## 实际遇到的问题

当我们顺着Go对错误的设计理念和使用原则来使用Go时，势必会在实际的开发过程中遇到问题：

- 排查定位：结构化异常处理帮我们实现了完整的调用堆栈维护，但Go的error显然只是一个interface，那么针对错误的排查定位就成问题了；
- 错误区分：同样会遇到一些场景我们需要识别出不同的错误（可能是类型、值等）进行不同的业务处理。

## 相关方案演进历史

- Go 1.13之前，[github.com/pkg/errors](github.com/pkg/errors)这个三方扩展库能解决上述问题，而且非常受欢迎，但再2021年12月它归档了；
- Go 1.13开始，标准库针对error进行了扩充（其实是Go2的草案被不断融入实现），基本可以解决上述问题。

# 场景实战

本文基于Go 1.13之后error标准库针对各类场景提供较优处理方案和DEMO

## 基础场景

- error信息

```go
package main

import (
	"errors"
	"fmt"
)

// BAD CASE: 错误信息模糊 
func demo1(x int) error {
	// 处理x

	// 判断x
	if x%2 == 0 {
		return errors.New("错了")
	}
	return nil
}

// GOOD CASE: 错误信息清晰
func demo2(x int) error {
	// 处理x

	// 判断x
	if x%2 == 0 {
		return errors.New(fmt.Sprintf("x[%d]经过计算后不符合要求", x))
	}
	return nil
}
```

```go
package main

import "errors"

// BAD CASE: 多原因错误信息重复
func demo1(x, y int) error {
	// 处理x，y

	// 判断x
	if x%2 == 0 {
		return errors.New("错了")
	}

	// 判断y
	if y%2 == 1 {
		return errors.New("错了")
	}
	return nil
}

// GOOD CASE: 多原因错误信息唯一
func demo2(x, y int) error {
	// 处理x，y

	// 判断x
	if x%2 == 0 {
		return errors.New("错了1")
	}

	// 判断y
	if y%2 == 1 {
		return errors.New("错了2")
	}
	return nil
}
```

- error处理

```go
package main

import (
	"errors"
	"fmt"
)

func err1() error {
	return errors.New("一个错误")
}
func err2() (int, error) {
	return 0, errors.New("又是一个错误")
}

// BAD CASE: 忽略错误
func demo1() {
	err1()
	y, _ := err2()
}

// BAD CASE: 不完整处理错误
func demo2() {
	if err := err1(); nil != err {
		fmt.Print("发生了错误")
	}
}

// BAD CASE: 多次处理错误
func demo3() error {
	if err := err1(); nil != err {
		fmt.Print("发生了错误：" + err.Error())
		return err
	}
	return nil
}

// GOOD CASE: 完整记录信息
func demo4() {
	if err := err1(); nil != err {
		fmt.Print("发生了错误：" + err.Error())
	}
}

// GOOD CASE: 完整传播信息
func demo5() error {
	if err := err1(); nil != err {
		return fmt.Errorf("demo5：%v", err)
	}
	return nil
}
```

- error区分

```go
package main

import (
	"errors"
	"strings"
)

type me struct {
	s string
}

func New(s string) *me {
	return &me{s: s}
}

func (m *me) Error() string {
	return m.s
}

var e = errors.New("一个哨兵错误")
var ex = New("又一个哨兵错误")

func err1() error {
	return errors.New("一个错误")
}

func err2() error {
	return e
}

func err3() error {
	return ex
}

// BAD CASE: 对Error()进行匹配
func demo1() {
	err := err1()
	if err.Error() == "另一个错误" {
		// do some thing
	} else if strings.Contains(err.Error(), "一个") {
		// do other thing
	}
}

// BAD CASE: 直接对错误进行等值判断
func demo2() {
	err := err2()
	if err == e {
		// do some thing
	}
}

// GOOD CASE: 对错误进行Is判断
func demo3() {
	err := err2()
	if errors.Is(err, e) {
		// do some thing
	}
}

// BAD CASE: 直接对错误类型进行断言
func demo4() {
	err := err3()
	if err2, ok := err.(*me); ok {
		// do some thing
	}
}

// GOOD CASE: 对错误进行As判断，成功时抽取对应类型错误
func demo5() {
	err := err3()
	var err2 *me
	if errors.As(err, &err2) {
		// do some thing
	}
}
```

- error包装

```go
package main

import (
	"errors"
	"fmt"
)

func err1() error {
	return errors.New("一个错误")
}

func demo1() error {
	err := err1()
	if nil != err {
		return fmt.Errorf("demo1: %w", err)
	}
}
```

# 产生层场景

| 错误匹配      | 错误信息  | 适用方案                                      | DEMO  |
|-----------|-------|-------------------------------------------|-------|
| 调用层无需匹配识别 | /     | 直接产生错误并返回                                 | demo1 |
| 调用层需要匹配识别 | 无动态信息 | 定义顶层哨兵错误，对需要匹配的错误点返回哨兵错误                  | demo2 |
| 调用层需要匹配识别 | 有动态信息 | 定义顶层哨兵错误，对需要匹配的错误点包装哨兵错误和动态信息后返回/或者用自定义错误 | demo3 |

- demo1

```go
package main

import (
	"errors"
	"fmt"
)

func demo1() error {
	if xxx {
		return errors.New("错误")
	}
	return nil
}

func main() {
	if err := demo1(); nil != err {
		fmt.Print(err)
	}
}
```

- demo2

```go
package main

import (
	"errors"
)

var SomeErr = errors.New("某种哨兵错误")

func demo2() error {
	if xxx {
		return SomeErr
	}
	return nil
}

func main() {
	err := demo2()
	if errors.Is(err, SomeErr) {

	}
}
```

- demo3

```go
package main

import (
	"errors"
	"fmt"
)

var SomeErr = errors.New("某种哨兵错误")

func demo3(s string) error {
	if xxx {
		return fmt.Errorf("%w：错误数据[%s]", SomeErr, s)
	}
	return nil
}

func main() {
	err := demo3()
	if errors.Is(err, SomeErr) {

	}
}
```

# 中间层场景

| 错误匹配      | 错误定位      | 错误信息  | 适用方案                               | DEMO  |
|-----------|-----------|-------|------------------------------------|-------|
| 调用层无需匹配识别 | 原始错误足够定位  | 无动态信息 | 原样传递返回原始错误                         | demo1 |
| 调用层无需匹配识别 | 原始错误足够定位  | 有动态信息 | 包装动态信息/或者用自定义错误                    | demo2 |
| 调用层无需匹配识别 | 原始错误不足够定位 | 无动态信息 | /                                  | demo2 |
| 调用层无需匹配识别 | 原始错误不足够定位 | 有动态信息 | /                                  | demo2 |
| 调用层需要匹配识别 | 原始错误足够定位  | /     | 定义顶层哨兵错误，对需要匹配的错误点包装哨兵错误以及本层相关信息返回 | demo3 |

- demo1

```go
package main

import "errors"

func f1() error {
	return errors.New("f1错误")
}

func f2() error {
	return f1()
}

func f3() {
	err := f2()
	if nil != err {
		//
	}
}
```

- demo2

```go
package main

import (
	"errors"
	"fmt"
)

func f1() error {
	return errors.New("f1错误")
}

func f2(s string) error {
	err := f1()
	if nil != err {
		return fmt.Errorf("f2(%s): %w", s, err)
	}
	return nil
}

func f3() {
	err := f2("hello")
	if nil != err {
		//
	}
}

```

- demo3

```go
package main

import (
	"errors"
	"fmt"
)

var SomeErr = errors.New("某种哨兵错误")

func f1() error {
	return errors.New("f1错误")
}

func f2(s string) error {
	err := f1()
	if nil != err {
		return fmt.Errorf("f2(%s):%w:%w", s, SomeErr, err)
	}
	return nil
}

func f3() {
	err := f2("hello")
	if errors.Is(err, SomeErr) {
		//
	}
}
```

# 顶层场景

```go
package main

import (
	"errors"
	"fmt"
)

var (
	ex1 = errors.New("错误1")
	ex2 = errors.New("错误2")
)

func err1() error {
	return errors.New("错误")
}

// 匹配分发
func match() {
	err := err1()
	if nil == err {

	} else if errors.Is(err, ex1) {

	} else if errors.Is(err, ex2) {

	} else {

	}
}

// 记录降级
func log() error {
	err := err1()
	if nil != err {
		fmt.Print(err)
	}
	return nil
}

// 记录转换（一般不推荐）
func convert() error {
	err := err1()
	if nil != err {
		fmt.Print(err)
		return errors.New("本层错误")
	}
	return nil
}
```

# 跨层场景

比如A层->B层->C层的跨层场景：

- 允许跨层依赖：当A层明确依赖C层且B层对C层的依赖不变，则允许A层中出现C层开放的哨兵错误；
- 不允许跨层依赖：大部分情况下不应该让A层知道C层的存在并对其产生依赖，应当在B层中进行包装；
- 跨层依赖转换：如果项目层级划分较多，可抽离独立错误层共享错误，减少部分逐层包装的代码。

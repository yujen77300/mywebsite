+++
title = 'Bitwise operation, 以 Golang 為例'
date = 2025-05-13T01:14:00+08:00
draft = false
categories = ["Tech"]
tags = ["Go"]
[cover]
image = 'img/godeeper/exploring-bitwise-operations-using-golang.png'
alt = 'Exploring Bitwise Operations Using Golang'
+++

## Introduction
最近在寫[leetcode的第190題](https://leetcode.com/problems/reverse-bits/)，發現自己對Bitwise運算不太了解，所以想寫一篇文章當做學習的紀錄。Bitwise從名字來看就是要對bit做運算，對二進位的數字進行操作。這類運算常被用於嵌入式裝置這種記憶體受限的環境、權限管理的flag檢查，以及Unix檔案系統的檔案權限設置等實際應用場景。雖然工作上好像很少實際用到Bitwise 運算，但理解它不僅有助於解題，我想也會對系統底層的實現原理更有概念。

Bitwise所包含的operator如下:
* 右移 `>>`
* 左移 `<<`
* AND `&`
* OR `|`
* XOR `^`
* AND NOT `&^` (Go 特有)

---

## Right Shift `>>`
將數字向右移, 左邊根據最高位置決定補上 0（正數）或 1（負數），等同於數字除以 2^n。

```go
x := 20 // 00010100
x = x >> 2 // 00000101 = 5
```

---

## Left Shift `<<`

將數字向左移動，等於數字乘以 2^n 。

```go
x := 20 // 00010100
x = x << 2 // 01010000 = 80
```

---

## AND `&`

對每個位元進行 AND 運算。只有當兩個位元都為 1 時，結果才為 1，否則為 0。

```
0 0 0 1 0 1 0 0 = 20
0 0 0 0 0 1 0 0 = 4
After & operations
0 0 0 0 0 1 0 0 = 4
```

```go
x := 20 // 00010100
x := x & 4 //00010100 = 4
```

---

## OR `|`

對每個位元進行 OR 運算。只要其中一個位元為 1，結果就為 1。

```
0 0 0 1 0 1 0 0 = 20
0 0 0 0 1 0 0 0 = 8
After | operations
0 0 0 1 1 1 0 0 = 28
```

```go
x := 20 // 00010100
x = x | 8 // 00011100 = 28
```


---

## XOR `^`


對每個位元進行 XOR 運算。當兩個位元不同時，結果為 1；當兩個位元相同時，結果為 0。

```
0 0 0 1 0 1 0 0 = 20
0 0 0 0 1 0 1 0 = 10
After ^ operations
0 0 0 1 1 1 1 0 = 30
```

```go
x := 20 // 00010100
y := 10 // 00001010
x = x ^ y // 00011110 = 30
```



---

## AND NOT `&^` (Go 特有)

先對右側運算元進行 NOT 運算 (將 0 變成 1，將 1 變成 0)，然後再與左側運算元進行 AND 運算。

```
0 0 0 1 0 1 0 0 = 20
0 0 0 1 0 0 0 0 = 16
After &^ operations
0 0 0 0 0 1 0 0 = 4
```

```go
x := 20 // 00010100
x = x &^ 16 // 00000100 = 4
```

---

## Code Snippet

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    var a uint = 20
    fmt.Println("原始數值：", a)
    fmt.Println("二進位表示：", strconv.FormatUint(uint64(a), 2)) // 10100
    fmt.Println("========================= ")

    // Right Shift 右移
    fmt.Println("右移 (Right Shift) >>")
    b := a >> 2
    fmt.Println(a, ">> 2 =", b)
    fmt.Println("二進位結果：", strconv.FormatUint(uint64(b), 2)) // 101
    fmt.Println("========================= ")

    // Left Shift 左移
    fmt.Println("左移 (Left Shift) <<")
    c := a << 2
    fmt.Println(a, "<< 2 =", c)
    fmt.Println("二進位結果：", strconv.FormatUint(uint64(c), 2)) // 1010000
    fmt.Println("========================= ")

    // AND 運算
    fmt.Println("AND 運算 &")
    d := a & 4
    fmt.Println(a, "& 4 =", d)
    fmt.Println("二進位：", strconv.FormatUint(uint64(a), 2), "&", strconv.FormatUint(uint64(4), 2))
    fmt.Println("二進位結果：", strconv.FormatUint(uint64(d), 2)) // 100
    fmt.Println("========================= ")

    // OR 運算
    fmt.Println("OR 運算 |")
    e := a | 8
    fmt.Println(a, "| 8 =", e)
    fmt.Println("二進位：", strconv.FormatUint(uint64(a), 2), "|", strconv.FormatUint(uint64(8), 2))
    fmt.Println("二進位結果：", strconv.FormatUint(uint64(e), 2)) // 11100
    fmt.Println("========================= ")

    // XOR 運算
    fmt.Println("XOR 運算 ^")
    f := a ^ 10
    fmt.Println(a, "^ 10 =", f)
    fmt.Println("二進位：", strconv.FormatUint(uint64(a), 2), "^", strconv.FormatUint(uint64(10), 2))
    fmt.Println("二進位結果：", strconv.FormatUint(uint64(f), 2)) // 11110
    fmt.Println("========================= ")

    // AND NOT 運算
    fmt.Println("AND NOT 運算 &^")
    g := a &^ 16
    fmt.Println(a, "&^ 16 =", g)
    fmt.Println("二進位：", strconv.FormatUint(uint64(a), 2), "&^", strconv.FormatUint(uint64(16), 2))
    fmt.Println("二進位結果：", strconv.FormatUint(uint64(g), 2)) // 100
    fmt.Println("========================= ")
}
```

---

## References

* [良葛格Go語言運算子](https://openhome.cc/Gossip/Go/Operator.html)
* [BITWISE OPERATORS IN GOLANG](https://medium.com/@murat.bilal/bitwise-operators-in-golang-db4b524065f3)
* [Bitwise Operators and WHY we use them](https://www.youtube.com/watch?v=igIjGxF2J-w)
+++
title = 'Golang Pointer'
date = 2023-10-22T22:56:03+08:00
draft = false
tags = ["Go"]
categories = ["Tech"]
+++

<style>
    .important{
        color:red
    }
</style>

## 前言
指標大概是我一開始學習golang最不適應的概念，最一開始學程式是由JavaScript入門，到wehelp後再學習用python 的flask框架寫網頁，這兩種語言都沒有指標的概念。而Go為了實現更高效的演算法及展現其高併發的特性，所以存有指標的方法來操作內存，當然相信自己學習時間不算久，也非EE 或CS背景，對指標瞭解可能還不夠全面，只能盡力就我了解的進行說明，如果有錯還請各方大神指正。


## 值 & 指標
當我們將int、string、struct這類的值傳給函式的時候(圖一)，Go語言會在函式中複製這些值，建立新的變數，也就代表如果是在函式對參數做修改時候，原始的值是不會有影響的，而如果要修改原本的值，就要透過所謂的 <span class="important">**指標變數**</span>。

![](https://i.imgur.com/KSDiIh5.png)(圖一，圖片來源 : [Udemy Go: The Complete Developer's Guide (Golang)](https://www.udemy.com/course/go-the-complete-developers-guide/))

我們可以在記憶體儲存資料非常簡單看成是一個抽屜，每個變數到會找到一個抽屜來儲存資料，而每個抽屜都有一個地址對應一筆資料(圖二)，所謂的**值**與**指標變數**的關係可以參考圖三，也可以幫助理解。
* 宣告一個int型別的變數x，其**值**初始化為3，這個3在記憶體裡面會有個地址0x88FFF。並可以透過在變數前面加上ampersand符號，取得該變數在記憶體的地址。
* 宣告一個int型別的指標變數xPtr，*int代表一個指向int的指針，為一種type description。並透過將xPtr初始化為&x，取得該變數在記憶體的地址。
* 最後如果有需要**存取原本的值**，或**修改原本的值**，可以透過在指標變數前面加上`*`符號進行反解(dereferencing)。如果(ex : `fmt.Println(*xPtr)`可以得到3 )。

![](https://i.imgur.com/W0r56sN.png)(圖二，自行繪製)
![](https://i.imgur.com/ScCEoOb.png)(圖三，圖片來源 : [Golang 指標基礎 Pointer - 記憶體位址、指標變數與資料型態、反解指標 By 彭彭](https://www.youtube.com/watch?v=SC8MPfhh9_8&list=PL-g0fdC5RMbo9bdRzbKaCWYC2mXg2eEZE&index=10&t=571s))

介紹完值和指標宣告與反解的流程，會發現用到兩次`*`符號，但這兩次其實是代表不同個功用，參考下圖四

![](https://i.imgur.com/7EC8pUh.png)(圖四，自行繪製)
<br>

同時繪製圖五再次說明`*`符號與`&`符號的當作運算符號時候的功能
![](https://i.imgur.com/IY5sH3C.png)(圖五，自行繪製)




## 實作程式碼
以下面程式碼為例，一開始會先印出x等於3，但當我們將int傳給函式的時候，會在函式複製值，建立新的變數，所以如果要修改原本的值，傳入的變數必須為<span class="important">**指標變數**</span>，故modifyInt函式的參數為`(x *int)`。
而當我們定義型別為`*int`，傳入的參數就必須是記憶體位置，故使用`&x`作為參數帶入modifyInt()函式，最後透過在modifyInt()使用`*x`進行反解，取得原本記憶體位置的值並重新賦值為50，可以發現不管是一開始的x變數，或是執行完modifyInt()函式的x變數，其值都變成50了。
```go!
package main

import "fmt"

var x int=3

func modifyInt(x *int) {
  *x = 50
  fmt.Println("Inside function :", *x)
}

func main() {
  fmt.Println("At first, x eqaul to :", x)
  modifyInt(&x)
  fmt.Println("Outside function :", x)
}

// At first, x eqaul to : 3
// Inside function : 50
// Outside function : 50

```


## References
1. [Udemy Go: The Complete Developer's Guide (Golang)](https://www.udemy.com/course/go-the-complete-developers-guide/)
2. [Golang 指標基礎 Pointer - 記憶體位址、指標變數與資料型態、反解指標 By 彭彭](https://www.youtube.com/watch?v=SC8MPfhh9_8&list=PL-g0fdC5RMbo9bdRzbKaCWYC2mXg2eEZE&index=10&t=571s)
3. https://ithelp.ithome.com.tw/articles/10235479

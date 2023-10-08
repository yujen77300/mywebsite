+++
title = 'Golang String'
date = 2023-10-01T12:01:56+08:00
draft = false
categories = ["Tech"]
tags = ["Go"]
[cover]
image = 'img/gopher.png'
alt = 'This is a post image'
caption = 'This is the caption'
+++

# Golang 字串

## String in Go

在Go語言中透過雙引號 "" 傳達的文字訊息，其預設型態就是string。有以下特色
* 字串是唯讀的一但建立就無法改變，但可以使用 + 進行串接
* 為位元組並透過UTF-8進行編碼
* 使用雙引號 "" 不可換行，如要換行可以使用反引號 ` ，反引號會保留換行和空白字元

因為之前主要是學習JS和Python居多，關於第二點一開始在go做字串擷取時較為不習慣，下面先舉三個例子來說明在go裡面的字串其實就是位元組 :


一  : 透過%x進行16進位的輸出
```go!
fmt.Printf("%x","Golang 您好")
// 476f6c616e6720e682a8e5a5bd
```
我們可以發現`47`是「G」這個位元組以16進位表示，6f則是小寫的o，20為空白。
然後在UTF-8裡面一個中文字為3 bytes，所以`e682a8`代表「您」這個字以16進位輸出之結果

二 : 利用[]取得字串的**位元組**資料
```go!
str := "Golang 您好"
fmt.Println(str[0])
// 71
fmt.Println(str[0:3])
// Gol
```
因為string是以byte型式儲存，所以`[0]`會得到G的位元組資料，相較於第一個範例因為透過%x進行格式化輸出，也就是沒有轉為16進位，以10進位結果表示為71
有個蠻有趣的是，如果我透過索引只取一個數字，會得到該位置的byte並以Unicode表示，但如果索引有如上寫成[0,3]，就會印出位置為0 1 2的字串

三 : 使用len方法
```go!
str := "Golang 您好"
fmt.Println(len(str))
// 13
```
因為string是透過UTF-8進行編碼，而一個中文字代表 3bytes，所以str字串長度為13 bytes.

## 取得字串裡的文字
從上面的第二個例子，如果我們透過fmt.Println(str[0])只是取得，該字串的Unicode值，那我們可以如何取得?

第一個方法可以透過string方法
```go!
str := "Golang 您好"
fmt.Println(string(str[0]))
// G
fmt.Println(string(str[8]))
// ½
```
這方法如果是抓取英文字母或是數字沒有問題，因為每個字元後面就是一個byte。但因為中文字對應UTF-8的編碼有三個bytes，會造成不能訪問這個中文字**完整**的字元，而會出現錯誤。
為了解決這個問題，建議如果要讀取字串中的字元，可以將該字串先轉成rune類型的slice，`[]rune`，這樣會將UTF-8編碼的位元組，轉換為Unicode的碼點，可以在特定索引取得特定字元，範例程式碼如下
```go!
str := "Golang 您好"
strRune := []rune(str)
for _, char := range strRune {
	fmt.Print(string(char))
}
// Golang 您好
```

## References
1. https://openhome.cc/Gossip/Go/String.html
2. https://www.asciitable.com/
3. https://www.youtube.com/watch?v=ut74oHojxqo

+++
title = 'Golang Function'
date = 2023-10-07T12:06:31+08:00
draft = false
tags = ["Go"]
categories = ["Tech"]
+++


## 宣告函式

函數就像個果汁機，使用者丟什麼食材進去，果汁機可以根據不同的設定(運算邏輯)，產出不同的結果。在Go裡面使用函數的方式其實和JavaScript、Python很像，不管是傳入的參數值或是返回值得類型都是可選擇的，最大的不同則是是如果有return的話一定要定義輸出的類型，其宣告格式如下

```go!
func functionName([parameter list])[return type]{
    //運算邏輯
}
```
最簡單呼叫函式的方式就是沒有任何傳遞任何參數，與要求回傳的類型
```go!

func main() {
	newCard()
}
func newCard() {
	fmt.Println("Ace of Spades")
}
//  Ace of Spades
```
如在main函式裡面，呼叫add(5,3)，add函式無論如何都會回傳int類型的結果。
值得注意的是，因為Go是一個強型別(strongly typed)語言，在執行階段會檢查型別，因此如果我們函式定義返回int，在第三行如果是return字串就會出現錯誤。
```go=
func add(x int, y int) int {
	return x + y
    //return "8" 
}

func main() {
	fmt.Println(add(5, 3))
}
// 8
```
此外在函式裡面如果也可以返回多個值，如下面compute function 定義回傳兩個int類型的結果
```go!
func main() {
	fmt.Println(compute(5, 3))
}
func compute(x int, y int) (int, int) {
	return x * y, x + y
}
// 15, 8
```

## 參數傳遞
在Go語言裡面，參數的傳遞都是都透過<span style="color:red;">Pass by value</span>，實際傳遞時會將參數複製一份到函式中，
因此在函式中對參數進行修改並不會影響原本的值。
如下為[官網原文](https://go.dev/doc/faq#pass_by_value)
> As in all languages in the C family, everything in Go is passed by value. That is, a function always gets a copy of the thing being passed, as if there were an assignment statement assigning the value to the parameter. For instance, passing an int value to a function makes a copy of the int, and passing a pointer value makes a copy of the pointer, but not the data it points to

以下面例子為例，雖然將num1和num2傳入compute函式中，但在函式中的修改不會影響原本的值。

```go=
func main() {
	num1 := 6
	num2 := 8
	fmt.Printf("compute函式前的num1值: %d\n", num1)
	fmt.Printf("compute函式前的num2值: %d\n", num2)
	fmt.Println(compute(num1, num2))
	fmt.Printf("compute函式後的num1值: %d\n", num1)
	fmt.Printf("compute函式後的num2值: %d\n", num2)

}
func compute(x int, y int) int {
	x = 10
	y = 12
	return x + y
}
// compute函式前的num1值: 6
// compute函式前的num2值: 8
// 22
// compute函式後的num1值: 6
// compute函式後的num2值: 8
```
那如果我們想透過函式修改影響原本的變數要怎麼辦呢，就要傳遞**指針**。
**傳遞指針也是以值傳遞**，但複製的是指針本身，而指針所指向的地址是一樣的，所以在函式內部的修改會影響到函式外原本變數的值。<br>
如下面例子，如果將變數前面加上ampersand&，可以指向該變數的內存地址，並在傳入函式後，透過*將原本的變數進行反解構修改變數的值(L13、L14)，就會到函式外原本變數的值
```go=

func main() {
	num1 := 6
	num2 := 8
	fmt.Printf("compute函式前的num1值: %d\n", num1)
	fmt.Printf("compute函式前的num2值: %d\n", num2)
	fmt.Println(compute(&num1, &num2))
	fmt.Printf("compute函式後的num1值: %d\n", num1)
	fmt.Printf("compute函式後的num2值: %d\n", num2)

}
func compute(x *int, y *int) int {
	*x = 10
	*y = 12
	return *x+*y
}
// compute函式前的num1值: 6
// compute函式前的num2值: 8
// 22
// compute函式後的num1值: 10
// compute函式後的num2值: 12

```
## defer的使用
在函式的使用中，經常需要創建資源，例如使用資料庫連接取得資料，為了讓該函式執行完畢後可以及時釋放資源，Go提供defer這個方法來達成，這也蠻像JavaScript裡面的try..finally。<br>
其有下列特性 :

1. 當執行到一個defer的時候，不會立即執行defer所定義的函式，而是將其放入一個專門儲存defer的stack裡面，然後繼續執行函式
2. 當函式執行完畢後，會從該stack最上面取出開始執行 (**First In, Last Out**)，如下圖
![](https://i.imgur.com/TLfoKI1.png)

接下舉例兩種最常使用的例子，第一個為當我建立資料庫連線的時候，我要更新參加者的資訊，當執行完畢後，需要釋放資源，避免佔用記憶體資源
```go!
func UpdateParticipantInfo(participantInfo []byte, roomId string) {
	var p pcpInRoom
	err := json.Unmarshal(participantInfo, &p)
	if err != nil {
		fmt.Println("Error during json to struct")
		return
	}

	db, _ := ConnectToMYSQL()
	_, err = db.Exec("INSERT INTO participant(room_id,member_id,pcp_stream_id) values(?,?,?);", roomId, p.ParticipantId, p.ParticipantStreamId)
	if err != nil {
		fmt.Println("Insert participant failed")
		return
	}
	defer db.Close()
}
```
第二個則是和recover函式一起使用，在Go裡面當函式遇到panic錯誤時候，可以透過recover()恢復執行。我們可以在可能產生panic的函式裡面放入defer recover()的方法，讓內層函式即使遇到panic錯誤，外層函式還是可以繼續執行
```go!
func f() {
    defer func() {
        r := recover()
        if r != nil {
            fmt.Println("Recovered in f", r) 
        }
    }()

    fmt.Println("Calling g.")
    g()
    fmt.Println("Returned normally from g.")
}

```



## References
1. https://zhuanlan.zhihu.com/p/542218435
2. https://go.dev/doc/faq#pass_by_value
3. https://www.songx.fun/blog/%E5%BF%85%E4%BF%AE/7golang%E4%B8%AD%E7%9A%84defer%E5%BF%85%E6%8E%8C%E6%8F%A1%E7%9A%847%E7%9F%A5%E8%AF%86%E7%82%B9/
4. https://pjchender.dev/golang/defer-panic-recovery/


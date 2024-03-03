+++
title = 'Dependency Injection Mocking with Go'
date = 2024-03-03T12:01:56+08:00
draft = false
categories = ["Tech"]
tags = ["Go"]
[cover]
image = 'img/DI-Mocking/DILOGO.png'
alt = 'Dependency Injection Mocking with Go'
+++ 

在軟體開發有個有個很重要的概念叫做SOLID，這五個單字其實各自說明不同的設計方法，目的是為了讓軟體開發及維護更加容易。這邊特別要提到的是D代表的依賴反轉原則 (Dependency Inversion)，主要是高層次的模組不應該依賴低層次的模組，應該提供介面讓較低層次的模組實現，降低不同模組之間的耦合程度，提高程式碼的可測試性。此篇文章會先介紹何謂依賴注入，再分別透過[Learn Go with Tests](https://quii.gitbook.io/learn-go-with-tests/)書中的例子實作Dependency Injection和mocking。

## 依賴注入

“依賴”如果以蓋大樓來比喻，如果要蓋一樓需要依賴地基，蓋二樓則需要依賴一樓和地基已經蓋好。”注入”則是將我們蓋房子所需要的資源(材料, 工人)提供給建造的過程，建造過程就可以更加靈活且可擴展，例如一樓是水泥房，二樓想要充滿自然元素，就可以將注入的原料改為木材，也就是隨時可以根據需求替換要注入的項目。

Go裡面strcut用來定義物件，interface用來定義物件的行為，再搭配struct和interface舉一個更生活化的例子 - 買咖啡

介面（Interface）：「咖啡」。它是一個通用的概念，代表了一切可以被當作咖啡的飲品。它有一些通用特性，比如含有咖啡因、熱的、無糖、放在杯子裡。

結構體（Struct）：拿鐵、美式、卡布奇諾這接自己定義的struct，都可以看成是「咖啡」介面的具體實現。它們都是咖啡，但每一種都有其獨特的製作方法(method)。

到了星巴克，你不會說要哪一種咖啡，因為實做”咖啡”這個介面的有很多種飲料。這時會說「我要一杯拿鐵」，其實是將拿鐵這個struct注入到訂單，店員就會依照你的需求執行定義好的行為來實現正確的咖啡種類。

### 實踐方式
1. Constructor function injection

    CoffeeService 依賴 CoffeeOrder ，也就是消費者點什麼咖啡，店員才會提供相對應的咖啡服務。透過constructor function NewCoffeeService，將 CoffeeOrder 這個依賴作為參數注入，並在 CoffeeService 中使用。
    所以當我們需要建立一個CoffeeService 實例時，首先需要根據消費者的選擇建立一個
    CoffeeOrder 實例，並將其作為參數傳遞给 NewCoffeeService 函數。CoffeeService 不需要知道 CoffeeOrder 是如何建立的，只需要知道它可以使用 CoffeeOrder，藉此增加靈活性和可擴展性。
    ```go
    type CoffeeOrder struct {
      Type  string
      Size  string
    }

    type CoffeeService struct {
        order *CoffeeOrder
    }

    func NewCoffeeService (order *CoffeeOrder) *CoffeeService{
      return &CoffeeService{order : order}
    }
    ```

<br />

2. Setter injection

    Setter 注入允許在物件建立後的任何時間點注入依賴，以下面 setter 方法為例。做咖啡的服務會依賴於訂單，可以將 CoffeeOrder 這個依賴作為參數注入作為咖啡服務的屬性。

    ```go
    func (c *CoffeeService) SetCoffeeOrder(oder *CoffeeOrder) {
	    c.oder = oder
    }
    ```

<br />

3. Method injecion

    另一種常見的依賴注入方式是直接在函數參數中傳遞依賴項CoffeeOrder，透過新建立的方法直接使用依賴，例如下面的PrepareCoffee函數，使用注入的CoffeeOrder來製作咖啡，但不將其存在CoffeeService 實例中。

    ```go
    func (c *CoffeeService) PrepareCoffee(order *CoffeeOrder) {
      // 可以在此增加製作咖啡的其他邏輯。
    }
    ```

## Manual Dependency Injection - Example
在書中依賴注入這個單元是要測試Greet函數，但這個函數具有side effect，使用fmt.Printf會將資訊寫到標準輸出，難以抓到Hello字串進行測試。

```go
func Greet(name string) {
    fmt.Printf("Hello, %s", name)
}
```
這時候其實就可以用到DI的概念，修改Greet函數，注入一個print的依賴，不需要關心在哪裡印出，以及如何印出。要如何做到呢 ?  可以繼續看Printf的原始程式碼。Printf實際回傳的是Fprintf函數，第傳入的參數是os.Stdout，而在Fprintf函數裡面定義的第一個參數是**io.Writer** ，哇，是一個介面。原來Printf後面是透過Writer介面把問候送到某處，然後透過實現Writer介面的os.Stdout把問候字串送到標準輸出。
```go
//fmt package
// It returns the number of bytes written and any write error encountered.
func Printf(format string, a ...interface{}) (n int, err error) {
    return Fprintf(os.Stdout, format, a...)
}

// Fprintf formats according to a format specifier and writes to w.
// It returns the number of bytes written and any write error encountered.
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
    p := newPrinter()
    p.doPrintf(format, a)
    n, err = w.Write(p.buf)
    p.free()
    return
}

//io package
// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

因此可以修改Greet函數如下，讓其可以接受注入io.Writer讓參數，然後將fmt.Printf改為fmt.Fprintf ，因為fmt.Printf默認是標準輸出，fmt.Printf則可以接收一個io.Writer參數用於接受字串傳遞。在main函數裡面就可以傳入實現透過實現Writer介面的os.Stdout把問候字串送到標準輸出。

```go
// main.go
func Greet(writer io.Writer, name string) {
    fmt.Fprintf(writer, "Hello, %s", name)
}

func main() {
    Greet(os.Stdout, "Elodie")
}
```

那要如何測試呢，因為實際上難以抓到標準輸出的字串，我們就把字串送到buffer，bytes package裡面的Buffer也有實現了Writer介面，因此在測試中可以用其做為Writer介面，然後傳入Greet函數就可以做測試了。

```go
//main_test.go
func TestGreet(t *testing.T) {
	buffer := bytes.Buffer{}
	Greet(&buffer, "Chris")

	got := buffer.String()
	want := "Hello, Chris"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```


## Manual Mocking - Example

在上面介紹了如何通過 dependency injection 的方式來增加程式碼靈活性和提高測試性。這使我們更容易對各個部件進行隔離和模擬。而mock 就是利用這種依賴注入機制來進行單元測試的一種重要方式，使用mock可以在寫測試的時候模擬各種場景，而不就外部環境影響。

在此可以先參考[Clean Coder Blog](https://blog.cleancoder.com/uncle-bob/2014/05/10/WhenToMock.html)一文，作者建議何時使用mock，分別如下

+ 測試需要使用很多外部資源，如外部的api呼叫, 資料庫，使用模擬對象可以讓實際測試在不跟外部資源互動下進行，避免不必要的資源消耗提高測試的穩定性。

+ 當測試結果可能受到網路延遲、資料庫狀態等外部因素影響時。在這些情況下，使用模擬可以建立一個受控的測試環境。透過模擬外部系統，可以確保測試結果的一致性，而不會受到外部環境變化的影響。

+ 對於難以觸發的錯誤情況，例如特定的錯誤處理邏輯，或執行某些風險較高的操作（如刪除檔案或資料庫表）時，模擬也是一個很好的解決方法。透過模擬，可以建立各種錯誤和異常情況，確保這些情況在測試中得到充分處理，有助於提高測試覆蓋率。


在書中例子是要針對一個Countdown()的函數進行測試，其會在間隔一秒的情況下依序輸出3、2、1、Go!。
```go
// main.go
const finalWord = "Go!"
const countdownStart = 3

func Countdown(out io.Writer) {
    for i := countdownStart; i > 0; i-- {
        time.Sleep(1 * time.Second)
        fmt.Fprintln(out, i)
    }

    time.Sleep(1 * time.Second)
    fmt.Fprint(out, finalWord)
}

func main() {
    Countdown(os.Stdout)
}
```
接著寫測試，一樣有用到DI的技巧，注入Buffer實例去抓取輸出的字串。

```go
//main_test.go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}

	Countdown(buffer)

	got := buffer.String()
	want := `3
2
1
Go!`

	if got != want {
		t.Errorf("got '%s' want '%s'", got, want)
	}
}
```

實際執行go test可以通過但發現花了4秒的測試時間，這在實務上除了會降低開發人員的生產力，且當測試邏輯更複雜時也可能有更多因素，讓開發受外在環境影響。此時會希望可以mock time.sleep的行為，用依賴注入的方式去替代「真正的」time.Sleep。
![gotest_1](/img/DI-Mocking/gotest_1.png)

<br />


在原本的Countdown函數會依賴操作時間的time.Sleep，根據之前DI的觀念可以將依賴關係定義為一個接口，Countdown依賴一個自己package內定義的interface，在測試的時候去實做這個介面模擬時間的操作。

如下先定義Sleeper介面，在main.go建立 ConfigurableSleeper結構體，其會實現Sleeper介面，並在呼叫Sleep函數時使用time.Sleep方法對時間做操作。
然而撰寫測試的時候，可以定義SpySleeper做為mock，其有一個屬性較做Calls，這個結構體可以像間諜一樣去追蹤紀錄Sleep函數被呼叫了幾次。所以在TestCountdown函數注入對spySleeper並希望Sleep函數實際被使用了4次，如果不等於4就會出現錯誤。

```go
// main.go
type Sleeper interface {
	Sleep()
}

type ConfigurableSleeper struct {
	duration time.Duration
}

func (o *ConfigurableSleeper) Sleep() {
	time.Sleep(o.duration)
}


func Countdown(out io.Writer, sleeper Sleeper) {
	for i := 3; i > 0; i-- {
		sleeper.Sleep()
		fmt.Fprintln(out, i)
	}

	sleeper.Sleep()
	fmt.Fprint(out, "Go!")
}

func main() {
	sleeper := &ConfigurableSleeper{1 * time.Second}
	Countdown(os.Stdout, sleeper)
}

//main_test.go
type SpySleeper struct {
    Calls int
}

func (s *SpySleeper) Sleep() {
    s.Calls++
}

func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}
	spySleeper := &SpySleeper{}

	Countdown(buffer, spySleeper)

	got := buffer.String()
	want := `3
2
1
Go!`

	if got != want {
		t.Errorf("got '%s' want '%s'", got, want)
	}

	if spySleeper.Calls != 4 {
		t.Errorf("not enough calls to sleeper, want 4 got %d", spySleeper.Calls)
	}
}
```
<br />

修改完程式碼再執行go test，不僅可以通過，且會發現所使用的時間會大幅減少。
![gotest_2](/img/DI-Mocking/gotest_2.png)

## 小結
通過這些上面說明,我們了解什麼是依賴注入以及它如何幫助編寫可測試和可維護的程式碼。但值得注意的是，過度使用依賴注入和mocking可能也會造成程式碼太複雜難以理解，因此還是需要和團隊成員溝通權衡依賴注入的利弊並適度使用。

最後分享一些依賴注入和mock的框架，由於我現在都只有直接手動寫測試，對於框架沒有太多使用經驗，之後如果較為熟練可再分享。

依賴注入 :
+ [uber-go/dig](https://github.com/uber-go/dig)
+ [google/wire](https://github.com/google/wire)
+ [facebookarchive/inject](https://github.com/facebookarchive/inject)

Mock :
+ [golang/mock](https://github.com/golang/mock)
+ [agiledragon/gomonkey](https://github.com/agiledragon/gomonkey)
+ [stretchr/testify - mock模組](https://github.com/stretchr/testify)
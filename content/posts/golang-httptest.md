+++
title = 'Golang httptest'
date = 2024-03-15T07:06:31+08:00
draft = false
tags = ["Go"]
categories = ["Tech"]
+++

## 簡介
用go開發API通常會用到 **net/http** pacakge，其提供基本的 http 客戶端和服務器功，允許使用者建立 http server、處理 http 請求以及發送 http 請求到其他server。
然而之前沒有寫測試的習慣，在學TDD的時候發現一個有趣的package - **net/http/httptest**

**net/http/httptest** 套件主要用於測試 http 服務器，可以模擬 http 請求和回應，讓測試者可以在沒有實際server的情況下測試 http 操作相關的函數，而不用依賴網路和實際的http server。

## Key function in net/http/httptest
在進入實際操作前先說明package裡面重要的函數和屬性。

```go
// net/http/httptest
func NewServer(handler http.Handler) *Server {
	ts := NewUnstartedServer(handler)
	ts.Start()
	return ts
}

// net/http
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers. If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```
NewServer()接受一個 http.Handler 作為參數，其會先建立一個模擬的server， 並自動啟動該server，並返回一個新的 httptest.Server 實例。

帶入的參數http.Handler是http package中的一個介面，其定義了一個方法 ServeHTTP(ResponseWriter, *Request)。這個方法接受兩個參數：一個 ResponseWriter 和一個指向 *Request 的指針。ResponseWriter 是一個介面，它提供了寫入 HTTP 回應的方法，而 *Request 是一個結構，它包含了 http 請求的所有信息。http.Handler 這介面的目的是讓能夠實現自己的 http server，允許使用者根據請求的內容來決定如何回應。

最後再看原始碼會發現HandlerFunc有提供ServeHTTP()方法，有實現 http.Handler interface，因此可以當作NewServer()的參數

```go
type Server struct {
	URL      string // base URL of form http://ipaddr:port with no trailing slash
	Listener net.Listener
    // ...
}
```
httptest.Server.URL提供了伺服器的 URL

```go
// Close shuts down the server and blocks until all outstanding
// requests on this server have completed.
func (s *Server) Close() {
	s.mu.Lock()
	if !s.closed {
    }
}
```
httptest.Server.Close用來手動關閉由 NewServer 或 NewTLSServer 創建的伺服器，用於在測試結束後清理資源。

## 實做

```go
// racer.go
func Racer(urlA, urlB string) (winner string) {

	aDuration := measureResponseTime(urlA)
	bDuration := measureResponseTime(urlB)

	if aDuration < bDuration {
		return urlA
	}

	return urlB
}

func measureResponseTime(url string) time.Duration {
	start := time.Now()
	http.Get(url)
	duration := time.Since(start)
	return duration
}

// racer_test.go
func TestRacer(t *testing.T) {
	slowURL := "http://www.facebook.com"
	fastURL := "http://www.quii.co.uk"

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got '%s', want '%s'", got, want)
	}
}
```
這裡的範例一樣為參考[learn-go-with-tests](https://quii.gitbook.io/learn-go-with-tests/go-fundamentals/select)一書，作者的目標是撰寫一個Race函數，裡面有兩個URL，透過送出http Get請求，看哪個網址先取得回應。
但這樣的測試方法會有速度慢和不可靠的問題，寫單元測試也要盡量避免這些side effect，這時候就可以使用到httptest建立模擬的http server解決這個問題


```go
func TestRacer(t *testing.T) {

	slowServer := makeDelayedServer(20 * time.Millisecond)
	fastServer := makeDelayedServer(10 * time.Millisecond)

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got '%s', want '%s'", got, want)
	}

	defer slowServer.Close()
	defer fastServer.Close()
}


func makeDelayedServer(d time.Duration) *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(d)
		w.WriteHeader(http.StatusOK)
	}))
}
```

makeDelayedServer函數用來建立一個新的httptest.Server 實例，並用http.HandlerFunc 來處理 HTTP 請求，其我們指定收到請求要等待一定時間，最後再將http 狀態碼設定為200並返回
如此實際做測試的時候，可以將參數代為20 * time.Millisecond和10 * time.Millisecond，實際測試會比呼叫http請求還快速。


## goroutine + select
最後作者提到為什麼上面的Race函數裡面的url要一個一個測試呢，go的優勢就是在併發，難道不可以同時測兩個嗎可以同時測兩個。

```go
func RacerWithSelect(urlA, urlB string,) (winner string, err error) {
	select {
	case <-ping(urlA):
		return urlA, nil
	case <-ping(urlB):
		return urlB, nil
	case <-time.After(10 * time.Second):
		return "", fmt.Errorf("timed out waiting for %s and %s", urlA, urlB)
	}
}

func ping(url string) chan bool {
	ch := make(chan bool)
	go func() {
		http.Get(url)
		ch <- true
	}()
	return ch
}

```
channel用來在不同goroutine之間傳遞資訊，現在修改原本的Racer函數，透過使用select，允許等待多個channel的回應，只有第一個回傳的case程式碼就會被執行
接下來順便來用BenchmarkRacer來看出彼此的差異

```go
func BenchmarkRacer(b *testing.B) {
	slowServer := makeDelayedServer(10 * time.Millisecond)
	fastServer := makeDelayedServer(0 * time.Millisecond)

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	for i := 0; i < b.N; i++ {
		// Racer(slowURL, fastURL)
		RacerWithSelect(slowURL, fastURL)
	}
}
```
執行 **Racer(slowURL, fastURL)** 和 **RacerWithSelect(slowURL, fastURL)** 時候，benchmark分別為
![racer](/img/httptest/racer.png)

![racerwithtest](/img/httptest/racerwithtest.png)

有使用goroutine的話，每個operation會消耗的時間降地非常多。

## 結尾
httptest 是一個非常有用的工具，可以幫助我們模擬 http 請求和回應，從而方便地對 API 進行測試，最後總結一些看完這章節的注意事项
1. httptest 不應該在生產環境的 http 或 https 伺服器，僅用在測試刺目的。
2. 測試伺服器最後要關閉 關閉一個server，才不會測試套件消耗過多或是持續監聽port，消耗資源
3. 使用goroutine時如果搭配select，可以在多個channel等待消息。


## Ref.
1. [learn-go-with-tests](https://studygolang.gitbook.io/learn-go-with-tests/go-ji-chu/select)
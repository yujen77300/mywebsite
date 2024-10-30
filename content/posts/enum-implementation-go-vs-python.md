+++
title = 'Enum Implementation Go vs Python'
date = 2024-10-30T08:37:40+08:00
draft = false
tags = ["Go","Python"]
categories = ["Programming"]
+++

## 前言
enum其實是enumeration的縮寫，讓使用者自己定義data type而避免定義一大堆固定的常數，通常用來定義一群相關的屬性，上禮拜在公司資深工程師提到golang如何實現enum的寫法，之前只有透過使用python的enum套件，沒看過用Go來實作覺得很有趣，因此想寫篇文章做個簡單的紀錄，此文會先用python說明沒有使用enum的缺點，再透過enum套件實作enum，最後再用Go實踐。

## Python簡單範例
在公司開發的應用程式需要針對k8s的資源做操作，這裡用一個比較簡單沒有使用enum的寫法
```python
# 定義常數
DEPLOYMENT = "Deployment"
STATEFULSET = "Statefulset"
DAEMONSET = "Daemonset"

def handle_resource(resource: str) -> None:
    if resource == DEPLOYMENT:
        print("處理資源：Deployment")
    elif resource == STATEFULSET:
        print("處理資源：Statefulset")
    elif resource == DAEMONSET:
        print("處理資源：Daemonset")
    else:
        print("未知的資源類型")

resource = "statefulset"
handle_resource(resource)
```

可以看到如果沒使用enum直接定義常數，主要可能會有三個問題
+ 缺乏類型安全，當傳入的變數是`resource = "statefulset"`，因為第一個字沒有大寫我就會印出未知的資源，要在執行程式碼的時候才會發現錯誤
+ IDE不支援自動補齊的功能，當在`resource=`要輸入值時，IDE不會知道可能要填什麼
+ 常數散落在global空間，如果其他地方需要已定義的常數可能會發生衝突

```python
from enum import Enum

class ResourceType(Enum):
    DEPLOYMENT = "Deployment"
    STATEFULSET = "Statefulset"
    DAEMONSET = "Daemonset"


def handle_resource(resource: ResourceType) -> None:
    if resource == ResourceType.DEPLOYMENT:
        print("處理資源：Deployment")
    elif resource == ResourceType.STATEFULSET:
        print("處理資源：Statefulset")
    elif resource == ResourceType.DAEMONSET:
        print("處理資源：Daemonset")
    else:
        print("未知的資源類型")

resource = ResourceType.STATEFULSET
handle_resource(resource)
```

在使用enum這個函式庫後，可以更輕鬆建立枚舉，並解決第一個範例沒有使用enum，因為打字錯誤而造成的未預期結果。
此外也分別解決上述遇到的問題
+ 不會接受沒有定義過的值，達到類型安全
+ IDE可以直接列出所有可用的選項，或是直接用tab鍵自動補齊
+ 所有使用的常數都會在Enum類別，不會汙染到其他命名空間

最後補充一個很實用的auto()函數，自動賦予遞增的整數值，從 1 開始類似Go的`iota`可以自動賦值，在做一些狀態管理或是錯誤碼定義也很常會用到
```python
from enum import Enum, auto
class Status(Enum):
    PENDING = auto()    # 1
    RUNNING = auto()    # 2
    COMPLETED = auto()  # 3
    FAILED = auto()     # 4

print(Status.PENDING.value)    # 輸出: 1
print(Status.RUNNING.value)    # 輸出: 2
print(Status.COMPLETED.value)  # 輸出: 3
print(Status.FAILED.value)     # 輸出: 4

```


## Go實現

在Go裡面可以透過自定義型別`ResourceType`實現enum

```go
package main

import "fmt"

type ResourceType string

const (
	Deployment  ResourceType = "Deployment"
	Statefulset ResourceType = "Statefulset"
	Daemonset   ResourceType = "Daemonset"
)

func (r ResourceType) IsValid() bool {
	switch r {
	case Deployment, Statefulset, Daemonset:
		return true
	}
	return false
}

func handleResourceEnum(resource ResourceType) {
	if !resource.IsValid() {
		fmt.Println("無效的資源類型")
		return
	}

	switch resource {
	case Deployment:
		fmt.Println("處理資源：Deployment")
	case Statefulset:
		fmt.Println("處理資源：Statefulset")
	case Daemonset:
		fmt.Println("處理資源：Daemonset")
	}
}

func main() {
	handleResourceEnum(Statefulset)
}
```

這樣在呼叫handleResourceEnum函式傳入參數的時候，可以用到IDE自動補齊的功能，添加新的資源類型只需要在const裡面增加，更容易管理。此外也提供更好的封裝性，可以對自定義的型別增加`IsValidz`方法


最後也補上面提到的golang iota用法
```go
package main

import "fmt"

type Status int

const (
	Pending   Status = iota // 0
	Running                 // 1
	Completed               // 2
	Failed                  // 3
)

func (s Status) String() string {
	switch s {
	case Pending:
		return "Pending"
	case Running:
		return "Running"
	case Completed:
		return "Completed"
	case Failed:
		return "Failed"
	default:
		return "Unknown"
	}
}

func main() {
	fmt.Printf(int(Pending))      // 輸出: 0
	fmt.Printf(int(Running))      // 輸出: 1
	fmt.Printf(int(Completed))    // 輸出: 2
	fmt.Printf(int(Failed))       // 輸出: 3
}
```

## 結論
透過這次比較 Python 和 Go 實現 enum 的方式，可以發現兩種語言各有特色。
Python透過內建的enum函式庫提供完整的列舉功能，而 Go 則是利用型別定義(Type Definition)和 iota 來實現類似功能。
在這兩種語言透過enum，都能實現型別安全、IDE 自動補齊和避免命名空間污染等優點，希望透過這個小技巧，再之後開發時的產品可以更容易維護和擴展~
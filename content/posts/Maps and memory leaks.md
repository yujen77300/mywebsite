+++
title = 'Maps and memory leaks(#28)'
date = 2024-01-18T22:20:06+08:00
draft = false
categories = ["Tech"]
tags = ["Go","100 Go Mistakes"]
+++

## 前言
此為跟\<learn go with tests\>讀書會成員在分享[Maps章節](https://quii.gitbook.io/learn-go-with-tests/go-fundamentals/maps)，有人分享的文章，原文見於[100 Go Mistakes and How to Avoid Them](https://github.com/teivah/100-go-mistakes)，此篇針對作者提到的第28個常見錯誤，做一些讀後記錄。

## Map in Go
此篇不會介紹在go裡面如何操作map, 而是希望更了解map設計原理，並藉由文章來探討為什麼可能發生memory leaks.

在Go的[官方部落格](https://go.dev/blog/maps)關於map是以此做為開頭
> One of the most useful data structures in computer science is the hash table. Many hash table implementations exist with varying properties, but in general they offer fast lookups, adds, and deletes. Go provides a built-in map type that implements a hash table.

在計算機科學的世界裡面，hash table是一種很常使用的資料結構。hash table會用hash function將key產出獨特的hash value，然後分配到不同的bucket。在Python裡面有dictionary, 在Go裡面則是透過Map實現hash table

Map可看成由一個陣列構成，而陣列的每個元素都是指向一個bucket的pointer。每個bucket包含了一系列的key-value pairs。當要尋找或是新增Map中的元素時，會使用key的hash valuse來確定其在陣列中的位置，然後在對應的桶中進行操作。

而在Go裡面，每個bucket是一個固定大小的陣列，通常可以容納八個元素。當一個bucket已經滿了(bucket overflow)，這時又有新的元素要增加，Go會創建另一個新的bucket，並將原來的bucket與新建立的bucket連接起來，以解決雜湊碰撞（hash collision）問題。

![bucket-overflow](/img/100-Go-mistakes/bucket-overflow.png)

## 程式碼實作
此處作者透過新增、刪除100萬個元素來觀察記憶體的使用狀況。

```golang
func main() {
    // step 1
    n := 1_000_000
    m := make(map[int][128]byte)
    printAlloc() // 0 MB

    // step 2
    for i := 0; i < n; i++ {
        m[i] = [128]byte{}
    }
    printAlloc()

    for i := 0; i < n; i++ {
        delete(m, i)
    }

    // step 3
    runtime.GC()
    printAlloc()
    runtime.KeepAlive(m) //Go 的GC在 KeepAlive 被調用之前不要釋放m，如果註解調會變成printAlloc()結果為0

}

func printAlloc() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("%d MB\n", m.Alloc/(1024*1024))
}
```

+ Step 1
  +  建立空的map，key的型別為整數型別int，而value是一個有128 bytes的陣列。
  +  m是空的，尚未有任何元素被添加，分配給它的記憶體非常小，未達1MB，故顯示為0
+ Step 2
  +  新增一百萬個元素
  +  使用了 461 MB
+ Step 3
  +  刪除一百萬個元素
  +  使用了 293 MB



## load factor & map擴容
在進入最後說明之前，先研究一下什麼是load factor。
> load factor = 元素數量 / bucket數量

代表每個bucket 中平均含有的元素數量，例如一個map有100個元素和10個bucket，那麼load factor就是 100/10 = 10。如果load factor太大，代表每個bucket中的元素太多，就會影響效率 ; load facotr太小，則會浪費空間。

在[runtime package 的map.go](https://github.com/golang/go/blob/0262ea1ff9ac3b9fd268a48fcaaa6811c20cbea2/src/runtime/map.go)裡面有overLoadFactor(),hashGrow()和overLoadFactor()函數，提到map在什麼元素下會進行擴容。而這裡比較重要的是關於下面load factor的定義，大概是80%，這常數的定義代表當bucket的平均負載達到大約80%時，會觸發map的擴容，這對下一小節解析hmap的B值是很重要的概念。

```golang
	// Maximum average load of a bucket that triggers growth is bucketCnt*13/16 (about 80% full)
	// Because of minimum alignment rules, bucketCnt is known to be at least 8.
	// Represent as loadFactorNum/loadFactorDen, to allow integer math.
	loadFactorDen = 2
	loadFactorNum = loadFactorDen * bucketCnt * 13 / 16
```

## 解析
這時候會發現一件有趣的事，即使已透過`runtime.GC()`手動執行GC，最後使用到的記憶體並非0。為什麼會這樣呢 ?

在這一樣引入在 [runtime package](https://github.com/golang/go/blob/0262ea1ff9ac3b9fd268a48fcaaa6811c20cbea2/src/runtime/map.go#L117-L131) 所定義的結構體hmap，裡面包含了實現map所需的所有資訊，包括buckets的數量、大小等。
```golang
type hmap struct {
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}

```

當使用map時，實際上是在操作一個指向runtime.hmap 結構體的指針。這是因為 Go 的映射是由 runtime.hmap 結構體實現的。當你創建一個映射，如 make(map[int]int)，編譯器會將其替換為對 runtime.makemap 的調用，該函數返回一個指向 runtime.hmap 結構體的指針，參閱[If a map isn’t a reference variable, what is it?](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it)

當我們在map中新增元素時，B 的值（即bucket的對數）可能會增加來適應更多的元素。根據作者所述，當增加 100 萬個元素後，B會等於18，意味著有 2¹⁸ = 262,144 個buckets。然而有趣的是，當這些元素被刪除後，即使map變為空，B的值仍然保持為 18，代表map仍然保留著相同數量的buckets，這是因為map在Go裡面的是設計是只能擴展而不會自動縮減的。

>  A map can only grow and have more buckets; it never shrinks.


所以我們透過`runtime.GC()`手動執行GC後，即使回收了map中的元素，讓記憶體使用從 461MB降至293MB，但這並沒有影響map自身的結構，map操作過能中額外新增的buckets的數量仍然保持不變，B仍會保持18。

在日常實際上可能遇到的問題就像當建立一個map來存儲每個客戶的資料。如果在特殊活動（如雙11、過年）期間客戶量暴增，map會因應大量數據而擴大。但活動結束後，即使存儲的數據量減少，bucket的數量仍然保持在之前高峰期間的值，就會導致記憶體消耗長時間居高不下。

最後作者提供一個比較有效的解決辦法是，儲存指針。

```golang
func main() {
	// 其中鍵（key）是整數型別（int），而值（value）是an array of 128 bytes
	m := make(map[int]*[128]byte)
	n := 1_000_000
	printAlloc()

	for i := 0; i < n; i++ {
		m[i] = &[128]byte{}
	}
	printAlloc() // 182MB

	for i := 0; i < n; i++ {
		delete(m, i)
	}

	runtime.GC()
	printAlloc() //38MB
	runtime.KeepAlive(m)

}

```

例如將map定義為`map[int]*[128]byte`。這樣只需儲存指針大小的記憶體，而不是128 bytes的array，每個指針通常佔用 4 或 8 bytes（取決於系統是 32 位還是 64 位），雖然不會減少bucket的數量，但可以節省記憶體空間，在新增一百個元素後，使用的記憶體是182MB，執行GC後則是剩下38MB。


## References
1. [100 Go Mistakes and How to Avoid Them](https://github.com/teivah/100-go-mistakes)
2. [The Go blog - Go maps in action](https://go.dev/blog/maps)
3. [Go語言設計實現 - hash table](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)
4. [If a map isn’t a reference variable, what is it?](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it)
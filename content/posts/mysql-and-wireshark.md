+++
title = 'MySQL protocol 與 Wireshark實作'
date = 2023-11-11T21:16:56+08:00
draft = false
categories = ["Tech"]
tags = ["MySQL","Wireshark"]
[cover]
image = 'img/mysql-and-wireshark/mysql-and-wireshark.png'
alt = 'MySQL-and-Wireshark'
caption = 'MySQL-and-Wireshark'
+++

## Client/Server protocol
> MySQL客戶端和MySQL伺服器端通訊的協議，分為Connection Phase & Command Phase，前者負責驗證資料的交換，後者為指令的接受和執行。

  

連線過程參考下圖，圖片取自[Turing blog](https://www.turing.com/blog/understanding-mysql-client-server-protocol-using-python-and-wireshark-part-1/)
![phases](/img/mysql-and-wireshark/phases.png)
1. three way handshake 建立 tcp 連線
2. Server initiate <span style="color:red">connection phase</span> with sending handshake packet => server greeting
3. Client => login request
4. Server => (auto switch request), response OK
5. clinet initiate <span style="color:red">command phase </span> => query
6. Server => response ok
7. client => prepare statment
8. Server =>　response to prepare(指定一個唯一的識別碼statement id)
9. client => excute statment (有statement id 去辨識屬於哪個prepare statement)
10. Server => response ok
11. client => request close statement
12. client 發送tcp fin flag

## Golang 程式碼參考

```golang
package main

import (
	"database/sql"
	"fmt"
	"time"

	_ "github.com/go-sql-driver/mysql"
)

const (
	UserName     string = "root"
	Password     string = "1qaz@WSX"
	Addr         string = "127.0.0.1"
	Port         int    = 3306
	Database     string = "demo"
	MaxLifetime  int    = 10
	MaxOpenConns int    = 10
	MaxIdleConns int    = 10
)

func main() {
	conn := fmt.Sprintf("%s:%s@tcp(%s:%d)/%s", UserName, Password, Addr, Port, Database)
	DB, err := sql.Open("mysql", conn)
	if err != nil {
		fmt.Println("connection to mysql failed:", err)
		return
	}
	DB.SetConnMaxLifetime(time.Duration(MaxLifetime) * time.Second)
	DB.SetMaxOpenConns(MaxOpenConns)
	DB.SetMaxIdleConns(MaxIdleConns)

	res, err := DB.Exec("INSERT INTO member values(?,?,?)",233, "klay", "dubnation@sfffff")
	checkErr(err)

	id, err := res.LastInsertId()
	checkErr(err)

	fmt.Println(id)
}

func checkErr(err error) {
	if err != nil {
		panic(err)
	}
}
```
以上為透過Golang操作MySQL的簡單指令，這次主要是了解MySQL通訊協定，connection pool的設定並沒有很重要，但為保持比較好的習慣來優化資料庫效能，分別做些簡單設定如下
+ SetConnMaxLifetime => 最長的連線時間，一但連接的時間超過的限制，將會從connection pool移除。
+ SetMaxOpenConns => 此connection pool最大的連線數量，避免對資料庫伺服器造成太大的負擔。
+ SetMaxIdleConns => 允許的最大閒置連線數量，避免每次需要使用資料庫都要重新連線，可以提升效能。




## Wireshark 操作
1. 在監聽的interface選擇MySQL安裝的位置和使用的port，在這因為是安裝在遠端主機，所以選擇SSH remote capture。

    ![listen-interface](/img/mysql-and-wireshark/wireshark1.png)

2. 如果監聽封包的數量很多，可以在最上面的display filter篩選tcp.port == 3306，藉此只顯示跟MySQL連線相關的封包。
3. 附圖為此次連線完整的封包，其包含TCP連線和MySQL Protocol。
![all](/img/mysql-and-wireshark/all.png)
4. TCP 3-Way Handshake
    1. 第一個封包為SYN flag，用來同步Seq Number，代表目前的序列號為初始序列號。透過觀察左下角的Packet Details Pane，發現有兩個序列號，原始序列號才是真正的Seq Number，另一個0是相對的序列號，主要是讓我們方便觀察使用。
    ![first](/img/mysql-and-wireshark/first.png)
    2. 第二步為Server回覆確認的封包，其ACK 號碼為前一個Seq號碼+1，並回傳SYN告訴client自己的Seq 號碼為3363418260。
    ![second](/img/mysql-and-wireshark/second.png)
    3. 最後Client回傳ACK，號碼為Server回傳的Seq號碼+1，此時完成三項交握。
    ![third](/img/mysql-and-wireshark/third.png)
5. Connection Phase
    1. TCP連線完成後會由Server端送出handshake packet 
    2. 接下來可能會建立SSL連線和交換一些認證需要的封包
    3. Server端送出OK Packet，完成Connection phase
    ![connection](/img/mysql-and-wireshark/connection.png)
6. Command Phase
    1. Client送出prepare statement
    2. Server回傳response to PREPARE,並帶有Statement ID
    3. Client送出execute statement
    4. Server回傳response OK並帶有Affected row資訊
    5. Client送出close statement與FIN封包結束連線
    ![command](/img/mysql-and-wireshark/command.png)


## Ref.
1. [MySQL documentation](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_PROTOCOL.html)
2. [TCP 序列號 (Sequence Number, SEQ)](https://notfalse.net/26/tcp-seq)
3. [Understanding MySQL Client/Server Protocol Using Python Wireshark](https://www.turing.com/blog/understanding-mysql-client-server-protocol-using-python-and-wireshark-part-1/)
4. [解读 MySQL Client/Server Protocol: Connection & Replication](https://cloud.tencent.com/developer/article/1768901)
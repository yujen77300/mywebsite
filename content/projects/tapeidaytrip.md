+++
title = 'Tapei Day Trip'
date = 2024-03-03T19:00:03+08:00
draft = false
[cover]
image = 'img/projects/taipeidaytrip.png'
alt = 'TaipeiDayTrip'
caption = 'TaipeiDayTrip'
+++


## 專案概述

### 前言
台北一日遊為參加WeHelp訓練營的guided project，也是自己第一個網頁作品。雖然是指導作品，但彭彭老師只會在每周的說明需要的功能，不管是前端的畫面呈現或是後端的API開發，老師都沒有撰寫任何程式碼供我們參考，如下圖。

![guide sample](/img/projects/sample_guide.png)

### 架構簡介
由於是第一次做網頁開發，相較於Django有許多middleware和定義好的資料庫模型，Flask是一個更輕量級的Web框架，且也提供路由和模板引擎等基本的網站功能,不會因過多額外的概念和功能而感到煩惱。再者電網網站有會儲存許多結構化資訊，如景點、用戶、訂單資訊等等，為確保資料的完整性選擇關聯式資料庫，並選擇最常見的開源資料庫MySQL。

### 專案主要功能
+ 會員系統，可上傳大頭貼、修改密碼、手機等資訊
+ 透過Infinite scroll可以一頁式的形式瀏覽所有景點
+ 使用搜尋功能瀏覽景點
+ 串接第三方金流平台TapPay進行付款
+ 預定行程頁面可看到已預定但尚未付款的景點，類似購物車系統
+ 查看歷史購買清單

## 技術架構
![architecture](/img/projects/trip-architecture.png)

專案使用前後端分離的架構設計，除了python的flask框架、MySQL外，部屬的雲端服務為Azure，使用blueprint做模組管理，Nginx做反向代理，並用Uvicorn啟動ASGI伺服器。

### Blueprint & MethodView

#### Blueprint

Blueprint是Flask中的一個概念，可將應用程式分解成多個模組。每個模組可以有自己的路由、模板和靜態文件，讓開發者更容易維護和擴展。

```python
# attraction.py
from flask import Flask, Blueprint, request
attractionApi = Blueprint('attraction', __name__)

#app.py
from routes.attraction import attractionApi
app.register_blueprint(attractionApi, url_prefix='/api')
```

在其中一個路由，attraction.py import Blueprint後，接著建立了一個名為attractionApi的Blueprint對象。
接著在app.py將名為attractionApi的Blueprint註冊到Flask應用程式app中，url_prefix='/api'參數指定了這個Blueprint的所有路由都將以/api為前綴。這代表如果attractionApi Blueprint中定義了一個路由/example，那麼在應用程式中訪問這個路由的URL將會是/api/example。

#### MethodView


```python
#attraction.py
from flask import Flask, Blueprint, request
from flask.views import MethodView

attractionApi = Blueprint('attraction', __name__)


class attractionCategory(MethodView):
    def get(self):
        categories = []
        try:
            mydb = taipeiPool.get_connection()
            cur = mydb.cursor()
            cur.execute("SELECT CAT FROM spot_info GROUP BY CAT")
            info = cur.fetchall()
            for category in info:
                string = ''.join(category)
                string.replace('\u3000', ' ')
                categories.append(string)
            return ({'data': categories})
        except Exception as e:
            return {'error': True, "message": str(e)}
        finally:
            cur.close()
            mydb.close()

attractionApi.add_url_rule(
    '/categories', view_func=attractionCategory.as_view('Categories of all tourist spots'))

```

> For RESTful APIs it’s especially helpful to execute a different function for each HTTP method. 

flask.views.MethodView是[Flask官方文件](https://flask.palletsprojects.com/en/1.1.x/views/#method-based-dispatching)推薦用來設計後端RESTful APIs的方式，可以根據不同的 http 請求方法（如 GET, POST, PUT, DELETE 等）定義不同的處理函數。

在上面程式碼中透過 <font style="background: #e9e9e9;color: black;padding:0 5px 0 5px; border-radius:5px">class attractionCategory(MethodView):</font> 定義了一個名為attractionCategory的MethodView子類，也就是繼承MethodView，定義了一個 get 方法，專門用於處理 http GET 請求。這意味著當 Flask 檢測到一個指向該類別路由的 GET 請求時，它會自動調用這個 get 方法。在此 get 方法用來從資料庫中獲取景點類別，並將這些類別返回給客戶端

最後一行 <font style="background: #e9e9e9;color: black;padding:0 5px 0 5px; border-radius:5px">attractionApi.add_url_rule(
    '/categories', view_func=attractionCategory.as_view('Categories of all tourist spots'))</font> 則是將 /categories 路由與 attractionCategory 綁定，如果客戶端發送 GET 請求到 /categories 時，將調用 attractionCategory 類別的 get 方法。


### Nginx

Nginx是一個開源的網頁伺服器，主要可以用來做反向代理、load balancing、http快取等功能，在台北一日遊的專案裡主要是透過Nginx實現反向代理，並配置<a href="https://support.unethost.com/index.php?rp=/knowledgebase/82/SSLSSL-certificate.html" target="_blank">ssl證書</a>讓網站可以使用 https 協定進行通訊。


### 反向代理(Reverse proxy)

為了瞭解反向代理，先說明什麼是正向代理，附圖皆出自<a href="https://www.cloudflare.com/zh-tw/learning/cdn/glossary/reverse-proxy/" target="_blank">cloudfare</a> 一文。

正向代理即代理我們使用者(client端)，當使用者對internet送出請求的時候，proxy會代理我們跟目的地的網頁伺服器做通訊，這麼做的好處是可以保護我們在網際網路上的ip或是藉由proxy送出的請求，繞過防火牆的限制
![forward_proxy](/img/projects/forward_proxy.png) 


反向代理位於服務器端，不同於代理客戶端的正向代理，代表一個或多個服務器與外界溝通。客戶端發出的請求首先被送達到反向代理伺服器，然後由它轉發到實際的後端伺服器。這種方式不僅可以對外隱藏後端伺服器的存在和細節以增強安全性，還能提供負載平衡、緩存靜態內容等功能，從而提升網站的性能和可靠性。

![reverse_proxy](/img/projects/reverse_proxy.png)

此外專案除了用Nginx作為反向代理，也在其設定檔中配置了SLL憑證，允許在與客戶端的通訊中實現加密連接，從而確保所有經過 Nginx 的數據傳輸都是加密和安全，將網址變成<font style="background: #e9e9e9;color: black;padding:0 5px 0 5px; border-radius:5px">https://</font>
開頭也可增強使用者信任。

```
ssl on;
ssl_certificate /my-file-system-path/my-domain-name.crt;
ssl_certificate_key /my-file-system-path/my-domain-name.key;
```


### Uvicorn
最後每次在執行Flask應用程式的時候，就會出現這提醒
<font style="background: #e9e9e9;color: red;padding:0 5px 0 5px; border-radius:5px">WARNING: This is a development server. Do not use it in a production deployment.</font>。
後來閱讀相關文章 ([Flask想上線? 你還需要一些酷東西](https://minglunwu.com/notes/2021/flask_plus_wsgi.html/)) 才瞭解原來用Flask run的應用程式屬於application server，負責接受請求，將其route到對應的程式碼進行處理、運算，最後回傳客製化的response。

WSGI（Web Server Gateway Interface）是Python語言開發網頁應用程式的一個interface，任何實做這個interface的方式都用來定義http request 如何與 application server 互動，request進來都會經過WSGI封裝成特定python的函數供application server使用，而Flask本身是一個輕量級的框架，內建是較為陽春的WSGI Server (Werkzeug)，負責處理HTTP Request及Flask間資料的轉換，然而其在處理「短時間多個Request」時的負載能力不佳，因此都會建議使用其他的WSGI Server。

![WSGI](/img/projects/WSGI.png)

而ASGI（Asynchronous Server Gateway Interface）相對於WSGI，又有更好的性能，可以同時處裡多個請求和支援Websocket，所以選擇將Flask應用程式將通過Uvicorn作為ASGI伺服器運行，這樣就不會再出現類似的警告訊息了

```python
from asgiref.wsgi import WsgiToAsgi
import uvicorn

asgi_app = WsgiToAsgi(app)
if __name__ == '__main__':
    uvicorn.run(asgi_app, host='0.0.0.0', port=3000)
```
執行asgi app
> uvicorn app:asgi_app --host 0.0.0.0 --port 3000 --reload


## 資料庫設計
MySQL Connector + 正規畫

CONNECTION POOL


## 系統流程
詳細說明系統的主要流程
深入介紹一些開發過能遇到的困難
心得
測試 PYTEST

## 部署與測試
說明如何在 AWS 平台上部署和運行應用程式

## 心得與結語
開發這個專案的過程中的體會
感謝同學老師一些資源


## Demo
{{< youtube hpSVgh3kCLo >}}
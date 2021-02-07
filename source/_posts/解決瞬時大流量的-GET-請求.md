---
title: 解決瞬時大流量的 GET 請求
date: 2021-01-12 09:47:55
tags:
- expressjs
- nodejs
- javascript
- postgresql
---

最近 Funliday 常發一些精選旅遊回憶的 App 通知給使用者，在去年十一二月的時候發通知 Server 還能撐的了瞬時大流量的 request。

但今年開始發這類通知，總共發了三次，三次都造成 Server 被打掛，而且重開 AP 還緩解不了，瞬間手足無措。大概都要等過了十分鐘左右，Server 才將這些 request 消化完。

這裡就來簡單整理一下時間軸，順便分享一下 Funliday 是如何解決這個問題。

---

* 1/6 1900：系統排程發送精選旅遊回憶的 App 通知
* 1/6 1900+10s 開始：Server 收到極大量的 request
* 1/6 1900+20s：Nginx 出現錯誤訊息 1024 worker not enough，並回傳 http status code 503
* 1/6 1900+25s：PostgreSQL 出現錯誤訊息 could not fork new process for connection (cannot allocate memory)
* 1/6 1900+38s：Node.js 收到 PostgreSQL 的 exception。There was an error establishing an SSL connection error
* 1/6 1900+69s：PostgreSQL 出現錯誤訊息 database system is shut down
* 1/6 1900+546s：PostgreSQL 出現錯誤訊息 the database system is starting up

---

看了時間軸就覺得奇怪，先不論 10s 的時候發了極大量 request，造成 20s 在 Nginx 出現 worker not enough 的錯誤訊息。而是要關注 25s 時的 PostgreSQL 出現 could not fork new process for connection 的錯誤訊息。

Funliday 用了同時可承載 n 個 connection 的資料庫，而且程式碼又有加上 connection pool，理論上根本不該出現這個錯誤訊息。但整個時間軸看下來感覺就是 PostgreSQL 的 capacity 問題，造成系統無法運作。

因為就算將 Nginx 的 worker connection size 再加大 10 倍，只是造成 PostgreSQL 要接受的 request 也跟著被加大 10 倍，但 PostgreSQL 那裡因為 request 變多，原本在 69s 直接關機的時間點只會提早，而無法真正緩解這個狀況。

基於以上狀況，小編就開始回去看自己的程式碼是不是哪裡寫錯了。會這樣想也是覺得 PostgreSQL 應該沒這麼弱，一下就被打掛，一定是自己程式碼的問題 Orz

---

這邊來分享一下自己程式碼的寫法，圖一是原始寫法，在每個 API 都 create 一個 db client instance 來處理該 API 層的所有 db request。這是蠻單純的做法，也是 day 1 開始的處理方式。但有個小問題，就是每個 API 層都要自己 create instance，不好管理，且浪費資源。

後來因為想要做 graceful shutdown 的關係，所以調整了一下 db client instance 的建立方式，用 inject 將 instance 綁在 request 上面，如圖二。這樣只要在 middleware 建立 db client instance 就好，好管理，而且只要有 req 就可以取得 instance，非常方便。而這也是 1/6 時的程式碼，就從這裡開始研究吧。

---

直接切入 node-postgres 的文件，認真讀了一下 pool 有下面兩種使用方式：

1. pool.connect, pool.release：文件寫著 checkout, use, and return，光看描述就應該用這個沒錯。
2. pool.query：適用於不需要 pool 的連線方式，文件上也清楚寫著內部實作是直接 call client.query，所以用了這個方式是完全跟 pool 扯不上邊。

但偏偏小編從 day 1 用的就是第 2 種方式 Orz，雖然看起來應該是寫錯，但也是要修改後實測，才知道是不是真的可以解決問題。

---

如圖三，這是修改後的程式碼。想了一下子，覺得目前在 API 層使用 req.pool.query 還不錯，不想用官方的建議做法：先 create client，然後 query 之後，再使用 release。

如果照官方建議做法，API 層的程式碼會多一堆與商業邏輯無關的程式碼，也不好維護。所以在不想動到 API 層的程式碼，只能使用 monkey patch 的方式來達到這個需求。

monkey patch 可以將原方法利用類似 override 的方式，將整個方法改掉，而不改變 caller 的程式碼，這也是 JavaScript, Ruby, Python 這類動態語言的特性之一，但真的要慎用，一不小心就會把原方法改成完全不同意義的方法了。

所以原本應該要在 API 層實作 connect, query, release 一大堆程式碼，可以用 monkey patch 完美解決這一大堆程式碼。

---

在 dev 壓測後至少 capacity 可以達到原本的 4 倍以上，隔天實際上 production 之後也確實如壓測般的數據，可以承載目前的流量。

其實這篇分享的重點只有一點，文件看仔細才是最重要的事啦！如果沒把文件看仔細，然後開發經驗也不足的話，什麼 RCA、monkey patch 都幫不上忙啦！

---

後記：有夠丟臉，其實完全用不到圖三，只要把圖二的 pool creation 放到最外層就好了，因為 pool.query 的內部實作已經有做 connect, query, release 了。

感謝下面的 Mark T. W. Lin 及 Rui An Huang 的幫忙，實在是太搞笑了 Orz

* Pool 的文件：https://node-postgres.com/features/pooling
* 官方建議寫法：https://node-postgres.com/guides/project-structure
* pool.query 的內部實作：https://github.com/…/node-postgres/blob/master/packages/pg-…
---
title: 如何用 PostgreSQL 的 advisory lock 實作推薦景點
date: 2019-02-20 09:37:53
tags:
- postgresql
- lock
- advisorylock
---

Funliday 最近做了大改版，其中一項功能就是把去年中因為 Google 要開始收費而暫時拿掉的「推薦景點」加回來，這個功能就是提供使用者指定區域附近評價較高的景點。

大家現在在使用景點瀏覽時，應該有時候會發現資料出來的速度不一致，慢的時候 (超過 5 秒，有時候會落在 10 秒以上) 代表可能有其他使用者正在查詢這個區域的「推薦景點」，思考邏輯就是在同一區域只要有一個使用者查詢過這個區域的「推薦景點」，其他同區域的使用者就會因為已經查詢過這份資料 (如熱門景點：台北 101、東京晴空塔...等) 而受惠。

在目前小編能力還無法將查詢效能大幅改善的狀況之下 Orz，若同一個區域有多個使用者同時查詢，每個查詢都要花費 10 秒的話，這樣子資料庫會浪費超多時間在不必要的查詢上面。

像這類無法避免的使用者行為，但又要減少資料庫無謂的同樣操作時，小編突然想到之前 Triton Ho 在上課時提到的 exclusive lock。

Advisory lock 是 PostgreSQL 的一種 lock 機制，可以在 application 層操作 exclusive lock (以下稱為 X lock) 或是 shared lock (以下稱為 S lock)。當使用 X lock 時，若非同一個 session 是無法將它 unlock，而 S lock 則是任何一個 session 都可以用 S lock 拿取 (acquire) 相同的 lock，但不可以用 X lock 拿取 S lock。

依小編的這個使用情境，可以在查詢「推薦景點」之前先問一次 Redis 有沒有這個區域的資料，有的話直接從 Redis 取得，沒有的話就先到 PostgreSQL acquire 一次 X lock，如果可以成功的話，則表示還沒有其他 session 正在查詢這個區域的資料，所以可以在 acquire 之後直接查詢，接著再 unlock 這個 X lock；但如果無法 acquire 的話，則表示目前有其他 session 正在查，所以直接回 client 查詢中的訊息，可以等一下再查。

所以小編使用 PostgreSQL 的 advisory lock 及 Redis 就可以達成不會有同一時間查詢相同耗時的查詢了。

其實再好一點的作法可能是把這類耗時的查詢丟到 MQ 裡面，讓 response 先回 client，等到 MQ 做完這個 job 之後，再用 push notification 通知 client 到 Redis 取得已經計算完的資料。或是用 polling 固定 n 秒鐘查詢一次，這都要視你的基礎設施而定。小編現在是先用 polling 的方式簡單處理，雖然方法有點笨，但在沒多餘心力 (Elasticsearch 太博大精深了 Orz) 的狀態之下就先這樣做了。

* How do PostgreSQL advisory locks work: https://vladmihalcea.com/how-do-postgresql-advisory-locks-work/
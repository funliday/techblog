---
title: 用 Redis 來處理 City 的 autocomplete 功能 - 1
date: 2019-01-08 14:00:01
tags:
---

Autocomplete 在現在的應用程式已經是個不可或缺的功能，但這個功能因為要一直發 request 到 server 上，簡直就是 DDoS 了 XDDD，對 server 是個不小的負擔。一方面要讓功能正常快速的運作，一方面又要讓 server 不會被打掛，是個不容易做的好的功能。這篇就來分享一下 Redis 的作者 antirez 是如何運用 Redis 來達到這個功能。

Redis 是一個 in-memory database，讀寫的效率自然不在話下，做 Autocomplete 用這類資料庫是再正常不過了。Redis 裡面有個資料結構叫做 Sorted Set，只要塞資料 (ZADD) 進去，它就會幫你按照字母順序排列好。所以我們只要把要搜尋的資料 (這裡我們稱為 candidate) 分割成獨立的字元 (這裡我們稱為 keyword)，存進 Sorted Set 就可以完成初步的資料處理。

比如想要找到 Taiwan 這個 candidate 的話就先把 Taiwan 拆成 t, ta, tai, taiw, taiwa, taiwan，把這六個 keyword 都存入 Sorted Set 裡面。最後再存入 taiwan*，表示這個 candidate 的結尾。所以 Sorted Set 的內容會變成下面這樣：

---

1. t
2. ta
3. tai
4. taip
5. taipe
6. taipei
7. taipei*
8. taiw
9. taiwa
10. taiwan
11. taiwan*
12. tal
13. tall
14. tall*

---

當使用者輸入 t，發送請求到 server 的時候，server 用 Redis 的搜尋指令 (ZRANK) 找出 index 為 1，然後再從 1 開始，將資料取回來 (ZRANGE) 50 筆，所以上面的 14 筆資料都會取回來。最後 server 再把這 14 筆結尾有 * 的資料過濾出來，剩下 7, 11, 14 這三筆，再把 * 濾掉回給使用者就完成這個功能了。

所以使用者輸入了 t，server 就會回給使用者 taipei, taiwan, tall 這三個 candidate，這就完成最簡單的 Autocomplete 功能。但 Autocomplete 可不只有這樣而已，剩下的細節等下次有空再來分享一下好了。
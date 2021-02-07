---
title: 用 Redis 來處理 City 的 autocomplete 功能 - 2
date: 2019-01-09 09:40:23
tags:
- autocomplete
- redis
- javascript
- nodejs
---

[前一篇](https://techblog.funliday.com/2019/01/08/%E7%94%A8-Redis-%E4%BE%86%E8%99%95%E7%90%86-City-%E7%9A%84-autocomplete-%E5%8A%9F%E8%83%BD-1/) 提到了 Autocomplete 的實作方式，但仍然有許多可以調整的地方，像是如何加大 throughput、帶額外資料...等，下面就來分享一下我們的作法。

---

## 1. 減少傳輸量

因為 Autocomplete 的操作行為是使用者每打一個字，就要傳給 server，server 再回傳使用者一些 candidate。所以減少傳輸量是最先要處理的事情，要不然資料量太大傳輸慢會影響前端使用體驗。最簡單的作法就是改變原本回傳的 JSON 格式，如下所示：

### 調整前

```json
[
  {"id": 123, "candidate": "taipei"},
  {"id": 456, "candidate": "taiwan"},
  {"id": 789, "candidate": "tall"}
]
```

### 調整後

```json
["123%taipei","456%taiwan","789%tall"]
```

前端拿到資料後自己再用 split 的方式分割字串，實測下來大概可以減少 40% 的資料量。

## 2. 減少傳輸量

沒錯！第二點也是減少傳輸量，將準備要回傳的資料用 gzip 壓縮後再回傳。

以 expressjs 本身建議的 compression 套件來說，實測下來發揮不了什麼作用。因為 compression 套件預設為資料量大於 1kb 才會做壓縮，而目前的資料已經是小於 1kb 了，所以沒做任何壓縮就直接回傳。

另外還發現加了 compression 套件之後，以目前開的 heroku 機器來說，回應時間會加上 5-10ms 左右。不過現在服務還沒上線，沒有使用量都不準，等上線之後再來觀察看看好了。

## 3. 減少使用者打 server 的次數

前端可以在輸入一個字元的時候不要送 request 給 server，因為經驗法則，使用者應該至少會打兩個字元之後，Autocomplete 回應給使用者 candidate，這樣對 UX 上應該會比較好吧 (我們不專業分析 XD)。不止可以降低 server 的 loading，也可以減少存入 Redis 的資料量。

但這會牽涉到 CJK 與 non-CJK 的處理方式，這就還要再看看如何處理比較好。

## 4. 減少使用者打 server 的次數

沒錯！又是減少次數。client 可以在 server 回傳資料的時候，將資料暫存在 client 的記憶體內。因為常會有輸入相同文字的時候，這時就可以直接從 client 的記憶體取出資料，就不用打到 server 了。

但這個使用方式比較不好處理，需視情境而定。若是 Redis 的資料常常在變動，那這個方式會造成取不回最新的資料。或許可以在 client 放個 LRU cache 來做處理。

## 5. 減少使用者打 server 的次數

又是我 XDDD！這次是要 server 幫忙，當 client 重複輸入相同 keyword 時，client 會帶 If-None-Match 的 header 給 server，server 會檢查這串值是否已經有打過了，如果打過就回 client 304，表示資料沒變動，可以直接用 client 本身的資料。

這在之前的 JCConf 有分享 (https://www.facebook.com/kewang.information/posts/2192127034396992) 過，大家可以回去翻一下。

## 6. 減少 Redis 的資料量

西方國家所用的拉丁字母除了大家常用的 26 個英文字母外，也常會有一些包括重音之類的字母。像是 a 及 á 之類的，這個在搜尋的時候不會太影響，JavaScript 可以利用 String.normalize('NFD') 把 á 轉換成 aˊ，最後再將 ˊ 取代為空字串 (https://stackoverflow.com/a/37511463/939212)，Redis 裡面只要存 a 就好，這樣可以節省不少資料量。

當然還有將大寫轉為小寫、trim 掉頭尾空白這幾種做法，也都可以省下不少資料量。

至於 CJK 的話，再說吧 XDDD

## 7. 存入 metadata

如果這個 Autocomplete 只是單純選擇 candidate 之後做搜尋，那可以不用存 metadata 進去。但有些功能其實是要把 candidate 回傳給 client 時，也帶一些 metadata 給 client 做其他運用，最常見的應該就是帶 id 這類 metadata 了。

最簡單的作法就是在存入 candidate 的時候，直接把要存的 metadata 帶在字尾，如下所示：

1. t
2. ta
3. tai
4. taiw
5. taiwa
6. taiwan
7. taiwan*123

把 123 放在 taiwan 後面，在取出 candidate 的時候再利用 split 的方式把 taiwan 跟 123 分別取出就可以了。

## 總結

總結上面的幾種方式，目前我們這裡用到了 1, 2, 5, 6, 7 共五種，效果還不錯，就等上線再來看看實戰結果囉。
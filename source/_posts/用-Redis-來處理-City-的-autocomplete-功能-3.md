---
title: 用 Redis 來處理 City 的 autocomplete 功能 - 3
date: 2019-01-10 14:00:33
tags:
- autocomplete
- msgpack
- debounce
- nginx
---

前兩篇分享了 Autocomplete 的實作方式及開發細節，算是少數大家迴響比較多的文章 XDD，下面就來整理一下大家的迴響好了。

---

## 1. 減少傳輸量可以使用 msgpack

小編有聽過 msgpack 但還沒實際了解這是如何運作的。剛查了一下[資料](https://msgpack.org)，說是比 JSON 更省資料大小，基本上聽過的語言都有支援。

在前公司也用過 Avro 這類的格式，主打的也是省資料大小。但現在應該還不會考慮改用這類要另外做 serialize 的格式。

主要是基於後端是以 Node.js 為主開發，JSON 已經是原生支援，再引入一種資料格式會增加前後端維護的複雜度。另外就是開發人力，新創小公司要儘量減少工作，目前可以順暢運作就好，還有其他更重要的事要做，等之後用量大了再改也不遲。

## 2. 減少傳輸量可以使用 HTTP server 的壓縮機制

這真的是忽略了，忘了 expressjs 只是一套 web framework，在上面對資料做壓縮其實會影響到效率。讓如 nginx 之類的 HTTP server 做壓縮應該才是更好的作法。

不過因為現在的 infra 是建在 heroku 上面，heroku 並沒有原生 nginx 的支援。等量大撐不住的時候，倒是可以優先考慮使用 heroku 的 buildpack 把 nginx 架上去[試試](https://github.com/heroku/heroku-buildpack-nginx)。

另外也有提到用 CDN 做動態壓縮，這就真的沒做過了，也是可以研究的方向之一。

## 3. 減少使用者打 server 的次數，加上 debounce time

這大家都主推使用 debounce 方式，前端沒玩很深的小編第一次碰到這個名詞是高職的時候。記得那時上課在教 8051，老師說按按鈕時要加上 15 - 20ms 的 debounce time，避免重複送外部中斷。小編對單晶片實在不在行，但大概記得是這個意思。

剛查了一下[資料](https://css-tricks.com/debouncing-throttling-explained-examples/)，前端的 debounce time 大概也是類似的意思。在輸入文字後，會 delay n 秒再送出，若是在 n 秒內又有打其他內容的時候，就把之前的 request 從 queue 裡面丟棄，只關注最後一次的 request 就好。

這個應該也是有效減少 request 量的作法了。

## 4. 減少使用者打 request 的次數，將已經送出的 request 取消掉

這也是一個不錯的作法，若 A request 已經送出去，但還沒回 response 時又送了 B request 的話，此時可以把 A request 取消。

但要注意就是 A request 目前正在執行的步驟是去 DB 拿資料，或是在 server 本身處理一些基本計算。之前在使用 Java (grizzly + jersey) 開發的時候，若有這種情況發生會常在 log 裡面看到 IOException。

原因是 server 已經準備好資料要回傳給 client，但發現 A request 已經取消，不知道要怎麼回傳時就會發生這個狀況。但也有可能是小編自己沒控制好收發的關係啦 XD

---

關於 Autocomplete 的三篇大概就到這篇為止啦，等上線之後做了哪些調整再來分享給大家知道一下。
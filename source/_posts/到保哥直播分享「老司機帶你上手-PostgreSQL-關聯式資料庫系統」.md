---
title: 到保哥直播分享「老司機帶你上手 PostgreSQL 關聯式資料庫系統」
ogimage: ogimage.png
date: 2021-10-13 22:41:15
tags:
- postgresql
- flyway
- nodejs
- postgis
- blurhash
---

<iframe src="//www.slideshare.net/slideshow/embed_code/key/AIC2t5nj9WvSx3" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/kewang/postgresql-250366908" title="老司機帶你上手 PostgreSQL 關聯式資料庫系統" target="_blank">老司機帶你上手 PostgreSQL 關聯式資料庫系統</a> </strong> from <strong><a href="https://www.slideshare.net/kewang" target="_blank">Mu Chun Wang</a></strong> </div>

一個月前上了保哥的直播，主要是分享一下 Funliday 如何使用 PostgreSQL。

其實一開始會選擇 PostgreSQL 主要有兩點，第一點是 Funliday 是放在 Heroku 上面，而 Heroku 配置的資料庫就是 PostgreSQL，第二點是 POI Bank 儲存的內容是一堆的景點資料，用 PostgreSQL 內附的 PostGIS 操作當然是更適合，所以當時也沒考慮用其他的資料庫。

這次接到保哥邀請的時候，小編真的想了很久，到底要跟大家講什麼才好。講語法？CRUD 大家都會，這沒什麼好講的；講效能評估？這也不是小編的專長，主要都是看各位神人的文章來做調校；講如何安裝？這直接上網找教學文章就有了，在直播講這個實在太浪費時間。所以後來想了一下，還是以 Funliday 如何使用 PostgreSQL 為主，然後分享了五個部分，這裡簡單整理一下

## 1. Migration 資料庫版本管理

在程式開發過程中，如果有做任何變更資料庫結構動作 (也就是 DDL) 的話，記得要做 migration。小編第一次看到這個概念是 RoR 正紅的時候學到的，任何的資料庫結構變更都要有 tear down 以及 set up，後來學的幾套 web framework 好像也都有類似的機制，所以在使用 PostgreSQL 的時候就直接拿來用。

但 Express.js 這個 web framework 其實沒有內建 migration 的功能，找了一些 Node.js 的套件也沒看到比較好的。所以後來就找了這套 Java 開發的 Flyway，完全可以達到我們的需求，所以後來就一直沿用下去了。

## 2. Node.js 整合

這部分是在分享如何把 Node.js 與 node-postgres 套件整合在一起，其實寫久已經習慣了，但一開始在用這個套件的時候，光是為了要找如何把 array 塞到 SQL 語法裡面就花了好多時間。這個在影片裡面有介紹到，大家可以直接去看看。

## 3. PostGIS 實務應用

這是將之前在 PostgreSQL.TW 以及 ModernWeb 2019 分享的內容再整理了一下，簡單舉了幾個 PostGIS 使用的例子，也分享如何在 pgadmin 上面用 OpenStreetMap 直接顯示結果，還蠻好用的！

## 4. DB 儲存資料用

這其實也是把 PostgreSQL.TW 的內容整理了一下，然後再加上最近實作的方式分享出來。其實就是想強調一點，資料庫主要還是存資料為主，有做正規化當然很好，但也是要看當時的資源跟時間是否允許。像影片中有提到因為 Redis 比較貴，所以把一部分的 cache 負擔丟給 PostgreSQL，這樣也比較省錢。

## 5. Blurhash 顯示

Blurhash 好像是這次迴響比較大的部分，其實在去年的 COSCUP 2020 就有提到這個套件了。就是把一張 client 要顯示的圖片，運用演算法計算出 20 個 byte 的字串，因為這個字串短，所以可以直接存在資料庫裡面。而 client 將這個字串 decode 之後，就可以顯示一個具有象徵意義的色塊，而不是傳統的 image placeholder。

這個 Blurhash 在 Funliday 上面已經運用在 Web 跟 Android，大家可以實際去使用看看效果，小編是覺得還不錯。

---

* [保哥的直播影片](https://www.facebook.com/will.fans/videos/145266924463322)
* [Homepage - Flyway](https://flywaydb.org/)
* [Welcome | node-postgres](https://node-postgres.com/)
* {% post_link 在-PostgreSQL-TW-分享的「大解密！用-PostgreSQL-提升-350-倍的-Funliday-推薦景點計算速度」 "在 PostgreSQL.TW 分享的「大解密！用 PostgreSQL 提升 350 倍的 Funliday 推薦景點計算速度」" %}
* {% post_link 在-ModernWeb-2019-分享的「Google-Maps-開始收費了該怎麼辦？」 "在 ModernWeb 2019 分享的「Google Maps 開始收費了該怎麼辦？」" %}
* {% post_link 在-COSCUP-2020-分享的「模糊也是一種美-從-BlurHash-探討前後端上傳圖片架構」 "在 COSCUP 2020 分享的「模糊也是一種美 - 從 BlurHash 探討前後端上傳圖片架構」" %}
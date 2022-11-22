---
title: Elasticsearch 的多語言 index 到底該如何設計？
date: 2022-11-07 18:44:26
tags:
- elasticsearch
- poibank
---

新版的 POI Bank 除了要解決多欄位搜尋的不便之外，還要解決底層 Elasticsearch index 不易維護的問題。

## 1. 原本的索引設計

其實原本設計的索引結構就如同下面這樣，同一個 document 存入所有的語言，再加上經緯度和觀看次數這些額外欄位。但實際在運作時，會發現因為 CJKV 這類語言要定期更新詞庫的關係，造成**每次更新詞庫時都有 downtime 產生** (index 要先 close 再 open)。雖然 Funliday 還不是一個非常大的服務，但有 downtime 對於使用者的體驗總是不好。

```json
{ // idx-poi
  "id": 999,
  "location": {
    "lon": -0.1425249,
    "lat": 51.5007972
  },
  "country_id": 12345678,
  "name_zh_cn": "白金汉宮",
  "name_zh_tw": "白金漢宮",
  "name_de": "Buckingham Palace",
  "name_en": "Buckingham Palace",
  "view_count": 98765
}
```

## 2. 將語言從欄位層級提昇到索引層級

所以為了解決 downtime 的問題，將語言從欄位層級 (field level)，提昇到索引層級 (index level)，這樣子就算中文索引因為定期更新詞庫需要 reopen 造成 downtime，但其他語言的索引還是可以正常運作。

於是改成了下面這種結構，但實際運作時卻發現了另一個問題 Orz

```json
{ // idx-poi-zh_tw
  "id": 999,
  "location": {
    "lon": -0.1425249,
    "lat": 51.5007972
  },
  "country_id": 12345678,
  "name": "白金漢宮",
  "view_count": 98765
}
{ // idx-poi-zh_cn
  "id": 999,
  "location": {
    "lon": -0.1425249,
    "lat": 51.5007972
  },
  "country_id": 12345678,
  "name": "白金汉宮",
  "view_count": 98765
}
```

程式改寫完之後，一執行下去發現回傳的資料量怎麼這麼多重複的內容，只差在是從不同語言的索引回傳。後來仔細想了一下才發現，因為景點名稱本來在各語言就是不同名稱，像是「東京巨蛋」、「tokyo dome」，如果使用者關鍵字為「tokyo 巨蛋」，這樣子同個景點就會出現兩筆資料，這真的很困擾。

所以為了要解決 downtime 改寫結構，就目前來說其實是個不太划算的方式，所以又換了另種方式，將索引分割成子索引。

## 3. 將索引分割為子索引

其實分割成子索引主要也是避免 downtime 的發生，做法就是**每 n 個 document 為一個索引，要 reopen 索引的時候，就只會影響到 n 個 document 而已**。

```json
{ // idx-poi-0001
  "id": 999,
  "location": {
    "lon": -0.1425249,
    "lat": 51.5007972
  },
  "country_id": 12345678,
  "name_zh_cn": "白金汉宮",
  "name_zh_tw": "白金漢宮",
  "name_de": "Buckingham Palace",
  "name_en": "Buckingham Palace",
  "view_count": 98765
}
{ // idx-poi-0002
  "id": 1000,
  "location": {
    "lon": 121.123456,
    "lat": 24.5007972
  },
  "country_id": 87654321,
  "name_zh_cn": "國立台灣大學",
  "name_zh_tw": "国立台湾大学",
  "name_de": "Nationaluniversität Taiwan",
  "name_en": "National Taiwan University",
  "view_count": 5566
}
```

這裡假設 n 為 20 萬，所以如果有 600 萬個 document 的話，只針對單一個索引 reopen，不會一次所有 600 萬個 document 都無法搜尋，只會有大約 3% 左右 (20/600) 的 document 會搜不到，所以絕對不會有 downtime。聽起來是個權衡之計，但開發完後測試遇到另個問題。

現在突然忘了確切的問題是什麼，但小編記得當初好像是 index 太多個，所以產生 too many shard 還是 too many index 的 exception。這其實要對 Elasticsearch 底層 (lucene) 有較深的理解，才會知道如何設計比較好。

總之，後來又改回原本的單一個 index 了，不過之後勢必還是會分成子索引，讓 downtime 儘量減少。

---

整理一下總共實作的方式及會遇到的問題：

1. 將所有資料放在同一個索引：在更新詞庫時會有 downtime 的問題
2. 依照語言分別放在不同索引：在搜尋時會有重複資料的問題
3. 將一個大索引分割為子索引：會有 too many shard 或 too many index 的問題

這篇其實是分享小編在重構 Elasticsearch 索引時的思路，**因為 Elasticsearch 有 index alias 功能，所以在設計索引時可以非常靈活**，雖然試了兩種方式都不能滿足現在的場景，但也希望能讓大家知道各自的適用場景。接下來就是解決熱門景點的問題了，大家期待下一篇文章吧！

* [How many shards should I have in my Elasticsearch cluster?](https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster)
* [Multilingual search using language identification in Elasticsearch](https://www.elastic.co/blog/multilingual-search-using-language-identification-in-elasticsearch)
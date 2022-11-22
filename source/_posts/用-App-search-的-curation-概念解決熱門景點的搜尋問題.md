---
title: 用 App search 的 curation 概念解決熱門景點的搜尋問題
date: 2022-11-22 16:22:59
tags:
- elasticsearch
- appsearch
- poibank
- autocomplete
---

{% asset_img autocomplete.png 400 %}

無論在哪個領域，「搜尋」這個議題，一直都是以讓使用者找到最適合的內容為最高原則。而 LBS 這類服務，如果要做搜尋通常都會再加上「經緯度」為變因，但常會遇到在「**台北市**」找不到「**東京鐵塔**」，這類我們想要讓使用者能夠直接搜尋到的景點，這篇文章就是要來分享 Funliday 如何解決這個問題。

小編今年買了一本喬叔的 Elasticsearch 實戰書籍 (非業配，但覺得真的不錯)，看到裡面介紹了 App search，這其實就是把 Elasticsearch 再封裝之後的服務，把搜尋功能的開發難度降低許多。而其中的 curation 更是啟發小編，讓小編不會糾結在搜尋只能在一個索引上完成。

在接觸 curation 這個概念之前，小編一直認為搜尋就應該要在同樣一個索引上完成才對，沒有想到其實可以做兩次索引的操作。而 curation 其實就是將想要讓使用者特別找到的內容，放在另一個索引，搜尋時先搜這個索引，然後再搜原本的索引，就能達成想要的內容先出現，剩下的依序排在後面，這樣子也就達成依照熱門程度的搜尋結果。

---

## 一開始只是想做 autocomplete 熱門景點

在還沒開發熱門景點搜尋的時候，一開始其實只是想在畫面上用 autocomplete 讓使用者選擇熱門景點，而當初也不想大改原本的資料結構，所以就用 Redis 來做熱門景點的 POI autocomplete。

開發完後在 dev 環境測試沒發生問題，但是 production 環境在建立資料時，Redis 無論加到多少記憶體都不夠用，這是始料未及的。主要是 production 資料比較完整，所以要寫入到資料庫的內容相比就多很多。另外因為 Redis 是 in-memory database，跟硬碟相比記憶體又特別貴，所以只好捨棄 Redis，改用 Elasticsearch 來做。

雖然用 Elasticsearch 來做 autocomplete 會稍慢於 Redis，但在成本跟效率相比，這是目前最平衡的做法。

## 把 curation 概念套入 search 熱門景點

用 Elasticsearch 實作完 autocomplete 之後，又想到書上寫的 curation 這個概念，看了看現在的這個索引，好像可以用來做熱門景點的搜尋。

無論輸入的關鍵字是什麼，都先在熱門景點這個 index 搜尋，然後再進原本的 index 搜尋，這樣可以達成在任何地方，都保證能找到東京的「東京鐵塔」，然後才是該地區的「東京鐵塔」。

```json
{
  "_id": "poi-12345",
  "name": ["日本電波塔", "japan radio tower"],
  "alias": ["東京鐵塔"],
  "view_count": 99999,
  "location": {
    "lon": 139.74537080,
    "lat": 35.65855880,
  }
}
```

## 也可以拿來做別名功能

大家可以看到這個 index 把所有語言都放在一起，然後再用 edge n-gram 分詞，所以無論是哪種語言都可以搜出相同的景點。更重要的是，當這個結構建立之後，小編發現也可以拿來做別名 (alias name) 了！

像「**東京鐵塔**」其實只是別名，正式名稱叫做「**日本電波塔**」，在沒有這個 index 之前，雖然 Elasticsearch 有同義詞 synonym 的功能，但維護不是很方便 (每次都要 reopen index，而且也無法指定 document)，所以一直沒有拿來使用。有了這個 index，做別名搜尋可以更得心應手！

---

到此為止，search v2 的整體架構算是大致成形，當然還有超多細節，像是 autocomplete 及 search 這兩支 API，如何讓前端串接更容易 (因為太多邏輯了)，又或是如何更精準搜尋到想要的內容等，由於涉及商業邏輯過多，之後有機會再來分享給大家看看吧。

* [喬叔帶你上手 Elastic Stack：Elasticsearch 的最佳實踐與最佳化技巧（iT邦幫忙鐵人賽系列書）](https://www.tenlong.com.tw/products/9789864348572)
* [在 MOPCON 2022 分享的「深入淺出 autocomplete」](http://localhost:4000/2022/10/17/%E5%9C%A8-MOPCON-2022-%E5%88%86%E4%BA%AB%E7%9A%84%E3%80%8C%E6%B7%B1%E5%85%A5%E6%B7%BA%E5%87%BA-autocomplete%E3%80%8D/)
---
title: 在 MOPCON 2022 分享的「深入淺出 autocomplete」
ogimage: ogimage.png
date: 2022-10-17 11:47:54
tags:
- redis
- elasticsearch
- autocomplete
- poibank
---

<iframe src="//www.slideshare.net/slideshow/embed_code/key/dqYXDq5Cdt0LOK" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/kewang/autocomplete-253634452" title="深入淺出 autocomplete" target="_blank">深入淺出 autocomplete</a> </strong> from <strong><a href="//www.slideshare.net/kewang" target="_blank">Mu Chun Wang</a></strong> </div>

這次在 MOPCON 分享的「深入淺出 autocomplete」，算是歷次在 blog 分享的 [autocomplete](https://techblog.funliday.com/tags/autocomplete/) 相關內容以來，首次在研討會有完整的分享。

就如同大綱所提的，這次分享了如何使用 Redis 實作 autocomplete 之外，也首次分享了 Funliday 如何改用 Elasticsearch 實作這項功能。Redis 的實作之前提過蠻多次的，今天就多講一些關於 Elasticsearch 的實作。

---

其實 Elasticsearch 的 edge n-gram tokenizer 非常適合拿來做 autocomplete 的底層，除了不用自己寫 normalize 和 tokenize 外，也有限制長度的參數可以使用。

再來就是跟 Redis 相比，Elasticsearch 天生有 inverted index 的資料結構，所以比較不會有資料量過度膨脹的問題。而且不像 Redis 的 sorted set，資料寫入時就已經決定好格式，Elasticsearch 的 document 更新非常方便。

另外，雖然 Elasticsearch 的資料是儲存在硬碟裡，跟 Redis 原生就儲存在記憶體的速度一定有差異。但因為 Elasticsearch 實作 autocomplete 時，用的是 filter，所以只要這個關鍵字用的夠頻繁，其實最後 Elasticsearch 還是從 cache 裡面取資料出來，速度影響不會這麼大。

---

現在 POIBank 的 search v2 (還沒上線 囧) 的一部分 autocomplete 就是用 Elasticsearch 來實作，先來下個 flag，希望明年過年前後可以把所有的 autocomplete 都從 Redis 移植到 Elasticsearch！

---

* [用 Redis 來處理 City 的 autocomplete 功能 - 1](https://techblog.funliday.com/2019/01/08/%E7%94%A8-Redis-%E4%BE%86%E8%99%95%E7%90%86-City-%E7%9A%84-autocomplete-%E5%8A%9F%E8%83%BD-1/)
* [用 Redis 來處理 City 的 autocomplete 功能 - 2](https://techblog.funliday.com/2019/01/09/%E7%94%A8-Redis-%E4%BE%86%E8%99%95%E7%90%86-City-%E7%9A%84-autocomplete-%E5%8A%9F%E8%83%BD-2/)
* [用 Redis 來處理 City 的 autocomplete 功能 - 3](https://techblog.funliday.com/2019/01/10/%E7%94%A8-Redis-%E4%BE%86%E8%99%95%E7%90%86-City-%E7%9A%84-autocoete-%E5%8A%9F%E8%83%BD-3/)
* [用 Redis 來處理 City 的 autocomplete 功能 - 4](https://techblog.funliday.com/2021/07/14/%E7%94%A8-Redis-%E4%BE%86%E8%99%95%E7%90%86-City-%E7%9A%84-autocomplete-%E5%8A%9F%E8%83%BD-4/)
* [用 Redis 來處理熱門景點的 autocomplete 功能](https://techblog.funliday.com/2022/05/01/%E7%94%A8-Redis-%E4%BE%86%E8%99%95%E7%90%86%E7%86%B1%E9%96%80%E6%99%AF%E9%BB%9E%E7%9A%84-autocomplete-%E5%8A%9F%E8%83%BD/)
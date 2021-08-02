---
title: 在 COSCUP 2021 分享的「使用 PostgreSQL 及 MongoDB 從零開始建置社群必備的按讚追蹤功能」
ogimage: ogimage.png
date: 2021-08-02 12:47:39
tags:
- postgresql
- mongodb
- messagequeue
- redlock
- bullmq
---

<iframe src="//www.slideshare.net/slideshow/embed_code/key/lxG1x0Au6S8pfU" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/kewang/postgresql-mongodb" title="使用 PostgreSQL 及 MongoDB 從零開始建置社群必備的按讚追蹤功能" target="_blank">使用 PostgreSQL 及 MongoDB 從零開始建置社群必備的按讚追蹤功能</a> </strong> from <strong><a href="https://www.slideshare.net/kewang" target="_blank">Mu Chun Wang</a></strong> </div>

這是今年小編在 COSCUP 上分享的內容，「使用 PostgreSQL 及 MongoDB 從零開始建置社群必備的按讚追蹤功能」，分享了 Funliday 在建置「按讚」、「追蹤」的實作經驗。

其中包括了 PostgreSQL table、Redlock key、MongoDB document 的設計思路，以及如何使用 BullMQ 來串接不同的服務，以提昇效能。

---

其實這場裡面分享的一些做法，不一定適合每個情境，像是反正規化因為要同步許多欄位，在需要精準控制 table 狀態的情境就要花很大功夫處理，但這也是在使用 NoSQL 時的必學課題。

另外 lock key 也是在處理同步時一個很重要的設計，先搞懂要解決的問題是哪一種，如果想解決的是同一篇文章不能在 lock 期間給不同的使用者按讚，那 key 就要設計為 `journalid_like` 才對。

---

因為疫情關係，所以這次的 COSCUP 改在線上進行，使用 Gather town 也是一個蠻有趣的嘗試。但還是希望疫情能趕快過去，全世界回到正常生活吧！

## References

* [Distributed locks with Redis](https://redis.io/topics/distlock)
* [The Architecture Twitter Uses To Deal With 150M Active Users, 300K QPS, A 22 MB/S Firehose, And Send Tweets In Under 5 Seconds](http://highscalability.com/blog/2013/7/8/the-architecture-twitter-uses-to-deal-with-150m-active-users.html)
* [BullMQ - Premium Message Queue for NodeJS based on Redis](https://github.com/taskforcesh/bullmq)
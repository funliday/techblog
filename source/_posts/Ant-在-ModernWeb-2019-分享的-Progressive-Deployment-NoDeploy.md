---
title: Ant 在 ModernWeb 2019 分享的 Progressive Deployment & NoDeploy
date: 2019-08-28 10:25:55
tags:
- modernweb
- nodeploy
- netflix
---

Ant 分享的內容總是很有意思，這場學到許多東西。連 Netflix 這麼大的公司都這樣講了，要在 production 做測試！

小編最近在 Funliday 切環境的時候非常有感。新創錢不夠多，總不可能建置一模一樣的環境來做測試吧。像是現在為了省錢，所以 Elasticsearch、Redis 和 Worker 還是有部分只在 production 上運作。這樣子在做測試時還是有部分怕怕的，尤其是目前主機大都在 Heroku 上面，也還是有部分的 env 需要做調整。

無論如何，還是要有錢才能把環境完全切開才行啊 Orz

* Progressive Deployment & NoDeploy: https://s.itho.me/events/2019/modernweb/2019-08-28_ModernWeb19-Ant.pdf
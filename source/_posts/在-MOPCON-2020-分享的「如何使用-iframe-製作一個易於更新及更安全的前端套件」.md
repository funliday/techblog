---
title: 在 MOPCON 2020 分享的「如何使用 iframe 製作一個易於更新及更安全的前端套件」
date: 2020-10-26 09:58:38
tags:
- cors
- hsts
- iframe
---

<iframe src="//www.slideshare.net/slideshow/embed_code/key/2gXeRhdmwtkpFi" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/kewang/iframe-238967650" title="如何使用 iframe 製作一個易於更新及更安全的前端套件" target="_blank">如何使用 iframe 製作一個易於更新及更安全的前端套件</a> </strong> from <strong><a href="https://www.slideshare.net/kewang" target="_blank">Mu Chun Wang</a></strong> </div>

這是小編在這次 MOPCON 分享的內容「如何使用 iframe 製作一個易於更新及更安全的前端套件」，來聽的人跟預期的差不多，應該都被吸去 R1 聽游舒帆大大的分享了 XD。

不過要反省一下，這個題目已經改過一次了，但還是取的不好，這個確實要好好檢討。雖然改名後的題目名為「前端套件」，但內容主要是在講後端的安全性，要對前端做些保護，主要有下面幾點：

* CORS：要加上 Access-Control-Allow-Origin
* api key 外流：加上 referer 做允許名單的驗證
* 網域偽造：使用 HTTPS 及 HSTS

最後分享了如何把網域加到 HSTS preload list 以及目前還在草案中的 SVCB/HTTPS 如何解決首次 HTTP request 的問題 (又是看 DK 大神的文章學知識 XD)

另外，這個套件還沒公開出來，關心這個套件的開發狀況，歡迎隨時關注我們喔！

如果十一月的 PostgreSQL 小講堂沒有入選的話，這應該就是今年的最後一個公開演講了，今年一次投了三個分享都有上，真的要感謝各大研討會的議程委員會。但真的是累死，之後一年最多還是投兩個就好，要不然光行前準備就花很多時間，累死了 XDDD
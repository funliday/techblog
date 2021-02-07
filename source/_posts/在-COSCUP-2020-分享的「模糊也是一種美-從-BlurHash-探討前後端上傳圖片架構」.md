---
title: 在 COSCUP 2020 分享的「模糊也是一種美 - 從 BlurHash 探討前後端上傳圖片架構」
date: 2020-08-03 10:00:07
tags:
- blurhash
- coscup
---

<iframe src="//www.slideshare.net/slideshow/embed_code/key/55C8If84Cyr4CV" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/kewang/blurhash" title="模糊也是一種美 - 從 BlurHash 探討前後端上傳圖片架構" target="_blank">模糊也是一種美 - 從 BlurHash 探討前後端上傳圖片架構</a> </strong> from <strong><a href="https://www.slideshare.net/kewang" target="_blank">Mu Chun Wang</a></strong> </div>

上星期大家都有來聽小編在 COSCUP 分享的「模糊也是一種美 - 從 BlurHash 探討前後端上傳圖片架構」嗎？這個技術已經有實作在 Funliday 的 Web 跟 Android 囉。這個議題結束後，有朋友問了一些問題，這裡順便來統整回答一下。

## 1. client side upload 方式從 server 產出的 signed URL 是個什麼樣的東西？

signed URL 是為了讓沒有 access key 及 secret key 的 sender 也能在有權限管理的保護下做 S3 的檔案處理，而 Funliday 在這裡的實作是不用原檔名做 key，改用 UUID 產生避免檔名衝突。

## 2. client BlurHash decode 的效率如何？

在做 BlurHash decode 的時候因為用到的是 CPU 運算，而且 JavaScript 又是 single thread 的關係，所以在 decode 同時移動畫面的話，可能會造成 CPU 不夠力的 client 會有極短暫的延遲時間。這時候可以考慮把 decode 丟到 web worker 處理，避免卡到 UI thread 的順暢度。用 Android 的術語來說就是開 AsyncTask 啦！

## 3. 你們後來是使用哪種方式做上傳呢？

原先是使用 server side (2) 的方式，但在處理 MQ 上傳後的 notify 花了不少時間，而且上傳到 S3 也沒這麼快，所以後來改用 client side 的方式做上傳功能，運作上也比原先的方式順暢。

## 4. 為什麼不用 medium 的方式處理 blur？

因為 medium 檔案比 BlurHash 的字串大很多，而且要多發一次 request，成本比 BlurHash 高出不少，所以我們認為用 BlurHash 會是比較好的。

---

當然也是要感謝 gslin 大大，他在 4/26 (日) 簡單介紹了 BlurHash。小編下午看到這篇文章，馬上丟去 slack 問我們的設計師大大，看她覺得這效果如何？她過沒多久回覺得不錯，小編就在星期日的下午開始處理 server side 的實作。隔天星期一有了初步的成果，然後給我們的安卓五星上將看，星期二就完成 Android 實作並上線了！

也是因為 CDN 那塊後來把原先的 lambda 改用自己寫的 server 處理，所以實作 BlurHash 才能這麼快。lambda 這塊也是血淚史，下略 10000 字，之後有機會再跟大家分享。歡迎大家對這塊有興趣的也來交流一下喔！

## 影片

<iframe width="560" height="315" src="https://www.youtube.com/embed/l1JVBX3tsMY" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
---
title: Funliday 外站文章的 App 與 Web 不同實作
date: 2020-05-04 10:00:49
tags:
- 著作權
- iframe
---

<iframe src="https://www.facebook.com/plugins/video.php?href=https%3A%2F%2Fwww.facebook.com%2F1671274149815619%2Fvideos%2F240193833737580&width=500&show_text=false&appId=481027911976374&height=893" width="500" height="893" style="border:none;overflow:hidden" scrolling="no" frameborder="0" allowfullscreen="true" allow="autoplay; clipboard-write; encrypted-media; picture-in-picture; web-share" allowFullScreen="true"></iframe>

Funliday 身處武漢肺炎疫情最慘重的觀光業中心，雖然大家都不出去旅遊，但我們也趁著這個時間增強自己的核心功能，小編今天來聊一下其中一個功能的技術議題。

Funliday 有個功能是把外部文章直接顯示在 Funliday 的 App 跟 Web 上，但遇到了一些技術性及著作權的問題，相信應該也有朋友遇到過類似的狀況，今天就來分享一下吧。

在 Funliday App 上的顯示還算好處理，直接用 WebView 呈現就好，但在 Funliday Web 上就很難處理，這邊整理一下技術上可以實作的幾種方式。

## 1. iframe + original url

最暴力的方式，直接用 iframe 嵌入對方網址，但會有一些問題。像是無法讓 Google 大神爬內容、HTTP 網址無法嵌入、如果有設定 x-frame-options 為 SAMEORIGIN 的話就無法嵌入、CSP 的設定也有可能造成無法嵌入

## 2. iframe + proxy + funliday url

改善了第 1 種方式，直接在 Funliday server 這裡做 proxy，但還是會有無法讓 Google 大神爬內容以及內容網址如果是相對路徑時的導頁問題 (這應該好解決)

## 3. 寫爬蟲抓內容

比如 A 站就固定抓 `<div class="content">`，B 站就固定抓 `<article>` 的內容，然後直接顯示在 Funliday Web 上，但畫面可能會亂掉，所以要想辦法把 A 站跟 B 站的 CSS 也拿來用，在 CSS 前面也要想辦法加上 namespace 避免衝突

## 4. remote render

類似 2+3 的方式，就是把要顯示的網頁用 headless chrome render 完之後，再跟原本的內容一起顯示，但畫面應該是會亂掉。

---

技術面可以的解法都確認了之後，再來就是適法性的問題了，因為 234 會把對方的資料落地到 Funliday 上，所以可能會有著作權的問題。對科技及法律這塊當然要問有研究的 ant 啦，請教了 ant 之後也得到了一些結論。

234 都會有著作權法的問題，所以基本上是不可行的，但只要著作權人有同意的話，則不在此限。

最後 Funliday Web 的實作方式跟 1234 都無關，而是改用類似預覽頁的方式在 Funliday Web 顯示原連結的 og:title 及 og:image，應該會再加上簡單如「以上內容未經重製與改作，來源均援引來源網頁內容」的聲明。

對於技術這部分也不複雜，在後台上稿時先取得原網頁的 og 資料，跟原本的 234 相比簡單太多了 XD

---

有經過 ant 同意，認為這個問題應該蠻多人都會遇到，所以分享給大家看看啦！
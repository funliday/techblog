---
title: 目前 Funliday 打過的怪
date: 2019-07-29 10:08:05
tags:
---

這是篇很棒的[大型系統演進史](https://lukajojo.medium.com/%E9%87%91%E9%AD%9A%E8%85%A6%E5%BE%8C%E7%AB%AF%E5%B7%A5%E7%A8%8B%E5%B8%AB%E5%AF%AB%E7%9A%84%E5%A4%A7%E5%9E%8B%E7%B3%BB%E7%B5%B1%E5%85%A5%E9%96%80-329a0f51b51c)，是後端工程師一定要看的文章。

其實台灣很多技術文章，但屬於這類偏大型系統演進史的文章，依比例來說正體中文是少之又少，看到的大都是簡體中文的內容。最近看到的另一篇是 PressPlay 從 AWS 轉到 GCP 的分享。

後端是一個看到什麼怪就打什麼怪的開發模式，很難一次把架構做到位。

* 熱門排行讀取速度太慢？就加 local cache
* 熱門排行內容不一致？把 cache 改成 redis
* 圖片讀取速度太慢？就加 CDN
* 圖片檔案太大？就加 resize server
* 圖片上傳太慢？MQ+redis+notification
* 一般搜尋太慢？開 explain 加 index
* 一般搜尋不精確？加 ElasticSearch

隨便列幾個就是目前 Funliday 打過的怪，而且還遠遠不止咧 Orz

有時候需求端也要懂得妥協，什麼功能都要的話就是錢要夠，時間要夠。天下真的沒有白吃的午餐啊！

PS. 如果明年有機會的話，也希望可以在研討會分享一下 Funliday 這兩年來的技術演進史

* PressPlay從AWS搬家到GCP一年的心得: https://bit.ly/2YpUx7O
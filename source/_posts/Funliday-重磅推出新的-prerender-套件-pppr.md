---
title: Funliday 重磅推出新的 prerender 套件 pppr
date: 2020-05-25 10:00:10
tags:
- pppr
- prerender
---

這一系列文總共有三篇，這是最後一篇。

Funliday 重磅推出新的 prerender 套件 pppr！這是一個 zero config 的 express middleware，只要 `npm install pppr`，然後在 `app.js` 裡面加一行 `app.use(pppr())` 就可以直接拿來用了。

---

原本在使用 prerender.io 這個套件有時候會出現 504 timeout 的問題，後來發現這個套件用的是比較底層的 API (Chrome DevTools Protocol, CDP)，研究它的原始碼後發現 render HTML 的 timeout 判斷上有些怪怪的，本來想試著去改這塊，但對 CDP 不熟，所以用 puppeteer 重寫一套 prerender service，pppr 也就應運而生。

簡單先解釋一下，puppeteer 是基於 CDP 封裝後成為比較容易使用的 API。因為 client side rendering (CSR) 的流行，所以現在要做網路爬蟲的話，愈來愈多會選擇用 puppeteer 來處理。這裡來分享一下在開發 pppr 的時候，有哪些東西要注意的。

## 1. 把 URL 放到像是 50 人的 LINE 熱門群組，prerender 會遭到大量的 request

因為每個使用者接收到這個訊息之後，因為要顯示 og data，所以就會去打一次 prerender。這裡姑且先稱之為 OG-DDoS 好了 XD，所以一定要做 cache，讓第一個 request 把 HTML 產生出來之後就放到 cache 裡面。然後可以用 LRU cache 來處理，因為這類 URL 都是短時間會被大量使用，之後就很少被用了，用 LRU cache 剛剛好。

### 其實這一段實作還有一些問題

如果在第一個 request 還沒產生出 HTML 之前，第二個同樣 request 就進來了，這樣子 cache 可以說是根本沒作用，還要再找時間來處理 lock 機制才行。

## 2. 每一個 request 要新開一個 page 才行

如果沒有每個 request 都開新 page 的話，會造成 A request 還沒處理完，B request 就用同一個 page 做 render，這樣子 A request 就會 504 timeout 了。所以一定要記得每個 request 都要新開 page。

## 3. 調整 User Agent

因為 headless chrome 的 user agent 就叫做 HeadlessChrome，為了避免在 render 的時候會出現意料外的狀況，保險一點還是把 HeadlessChrome 改成 Chrome 會比較好。

## 4. 注意 redirect

因為 expressjs 跟 puppeter 是兩個不同的 context，對於 redirection 來說，expressjs 會回傳 3xx 系列的狀態碼，但 puppeteer 則會直接執行完成。所以把 puppeteer 放在 expressjs 裡面執行的話，必須要處理 redirect chain，讓 expressjs 能回給 client 正確的狀態碼才行。

## 5. 取得完整的 URL

pppr 因為是發想自 prerender.io，所以 interface 也一樣是 `/render?url=https://example.com`。但有時候原始的 url 後面會包含 query string，所以 expressjs 要用 URLSearchParams 另外做些處理，才能取得完整的 url。

開發 pppr 基本要注意的事項大概就這樣，總之記得給星，有任何問題歡迎發 issue 跟 pr 喔！

* pppr: https://github.com/funliday/pppr
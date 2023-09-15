---
title: 利用 Puppeteer 把行程轉換成 PDF 的實務經驗
ogimage: pdf.png
date: 2023-09-15 14:18:15
tags:
  - puppeteer
  - pdf
  - playwright
---

Funliday 最近功能萬箭齊發，其中有幾項比較值得一提的，今天先來分享第一個，大家敲碗已久的行程轉成 PDF 功能是如何煉成的。

{% asset_img pdf.png %}

十年前在前公司有做過一個產險的案子，其中有一個功能是將保戶在 App 上填寫完的申請資料，轉成 PDF 存下來。雖然這個 PDF 功能不是我開發的，但記得當初的做法好像是用 iText 之類的 PDF library，把保戶填的資料塞進已經預先定義好的欄位裡面。

這種做法可以保證輸出時的版面不會受到影響，但對於一般的網頁開發者，門檻高了不少。因為網頁開發者還要了解從原始文件轉成 PDF 的過程，再加上預定義的欄位，完全沒使用網頁技術，總是覺得麻煩不少。

畢竟網頁開發者比較熟悉的還是 HTML+CSS，直接用網頁產生 PDF 的話，應該會更受到網頁開發者歡迎。過了這麼多年，直接用網頁產生 PDF 的開發工具也愈來愈多，現在當紅的應該就是 Puppeteer 跟 Playwright 了！總算進入正題，來分享一下我們是如何用 Puppeteer 產生 PDF 的。

## 最簡單使用 Puppeteer 的方式

Puppeteer 是一套 Headless Chrome 的 toolkit，Funliday 目前還在運作的 pppr (prerender engine) 也是使用 Puppeteer 開發的喔！而 Puppeteer 本身就有一行程式碼可以將目前讀取到的網頁，直接輸出成 PDF 內容

```javascript
await page.goto("https://www.funliday.com");

const output = await page.pdf(); // 就是這行啦

return res.set("Content-Type", "application/pdf").send(output);
```

只要三行程式碼就可以完成工作的話，那今天寫這篇真的是灌水灌大了！

## 減少 Google Chrome 記憶體使用量

Google Chrome 是一個非常吃記憶體的怪獸，我們也不想開太大的機器來服侍 Google Chrome，於是 CDN 就變的非常重要了。

使用者點擊 URL 的時候，會先去 CDN 問這個 URL 有沒有資料，有的話就不經過 origin server 直接回傳結果，減少 origin server 的負擔，沒有的話就先到 origin server 執行業務邏輯，再透過 CDN 回傳給使用者，以這裡的例子就是把網頁 render 成 PDF。

所以如果同一個 URL 丟到 LINE 的大群組之類的，短時間就不用擔心一堆人點 URL 造成 Google Chrome 吃一大堆記憶體重複 render。

但這裡有個額外的小細節，如果丟到社群媒體上，記得要多判斷 user agent，判斷如果是社群媒體的話，就顯示 PDF 的縮圖或想要呈現的內容，使用者體驗會更好。(但我們還沒做 XD)

## 加上驗證功能

另外這個功能目前限制只有自己跟同群組的夥伴可以使用，所以在 URL 上有做了一點小處理。就是 URL 有帶了 member id 以及 auth token，如果有人隨意更動這兩個值的話，會造成驗證失敗，算是非常基本的驗證，目前需求也用不到嚴謹的驗證。如果要再嚴謹驗證的話，目前的想法應該是會再加上 cookie 驗證。

另外帶了 member id 之後，也可以拿來做統計，算是目前有限資源的最快開發方式。

```javascript
async function CheckAuthMember(req, res, next) {
  req.memberId = extractMemberId(req);

  if (!req.memberId) {
    return res.redirect(302, "https://www.funliday.com");
  }

  return next();
}

const extractMemberId = req => {
  if (!req.query.member_id || !req.query.token) {
    return "";
  }

  let decoded;

  try {
    decoded = jwt.verify(req.query.token, tokenSecret);

    return req.query.member_id === decoded.member_id ? req.query.member_id : "";
  } catch (err) {
    console.error(err);

    return "";
  }
};
```

## 提升 render 速度

再來就是關於 render 的速度了，在 local 端做 render 一定是比 remote 端要快上許多，而且專案複雜度 local 也遠比 remote 要少許多，所以我們先使用 pug 在 local 做完 render 之後 (其實就是 server side render)，直接開啟 `file:///tmp/the-best-pdf.html`，減少 remote 的資料傳輸，當 render 結束後再輸出成 PDF。最後記得要把 local 的 HTML 檔刪除喔。

```javascript
const page = await browser.newPage();

const content = pug.renderFile("pdf.pug", {
  trip
});

const tmpFilename = `/tmp/${fileName}.html`;

await fs.writeFile(tmpFilename, content);

await page.goto(`file://${tmpFilename}`, {
  waitUntil: ["load", "domcontentloaded", "networkidle2"]
});

const output = await page.pdf();

await page.close();

await fs.unlink(tmpFilename);

return res.set("Content-Type", "application/pdf").send(output);
```

## 提升開發效率

因為 PDF 的內容設計主要是依靠前端，所以可以與後端大致脫勾，這樣的開發方式大幅提升了我們 delivery 的效率，除非畫面上的元素有特殊要判斷的內容，否則前端設計完 PDF 之後就可以直接 push 到 master 上線使用。

---

這個功能在沒宣傳的狀態之下，每日使用者人數比預期的多，算是蠻開心的。另外往日本遊玩的台灣使用者，PDF 也會放入各種 coupon，最重要的當然就是 **BIC CAMERA** 和**唐吉訶德**啦，幫大家省錢之餘，也希望大家多多愛用這個功能喔！

* [Funliday 重磅推出新的 prerender 套件 pppr](https://techblog.funliday.com/2020/05/25/Funliday-%E9%87%8D%E7%A3%85%E6%8E%A8%E5%87%BA%E6%96%B0%E7%9A%84-prerender-%E5%A5%97%E4%BB%B6-pppr/)
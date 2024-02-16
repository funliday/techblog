---
title: 把 Web 底層重構，從前後端分離變成 SSR
ogimage: cover.webp
date: 2024-02-16 16:36:02
tags:
- ssr
- seo
- cookie
- view state
- expressjs
---

{% asset_img cover.webp %}

Web 底層重構完成過了半年多，目前效果還算不錯，來分享一下我們是如何重構的。

## 前後端分離

Funliday 的初期開發模式，一直都是 backend 撰寫 API，然後 Web 及 App 都直接串接 API，達到真正前後端分離。這樣的開發模式無論對於前後端都非常單純，串接的 API 都是一模一樣，所以溝通成本也不高，那為什麼要重構？

## 為什麼要重構？

對於 App 來說，因為是原生系統，所以還是維持串接 API 為主。但 Web 如果還是用純 API 串接的話，其實會失去原本 Web 就有的優勢。

Web 原本純粹只使用 API 做前後端的溝通，所以每一次的初始畫面呈現都要經過好多次的 API call。像是我們的首頁，因為 API 是基於前後端分離所開發的，所以要分別取得熱門行程、熱門遊記、熱門城市...等，大概近 10 支 API。

然後每個畫面在第一次使用的時候，因為要取得初始參數，所以又要多一次 API call，這樣子來來回回，就算各個 API 互不相關，可以用 `Promise.all` 同時取得資料，但只要一支 API 因效能卡死。又或是有相關，前面死了就無法取得後面的內容，都會讓開發上變得更複雜。當然這些都能透過各種 design pattern 解決，但還是解決不了速度慢的問題。

而且 API call 是從使用者的瀏覽器發出，中間經過各種 latency 才到達，這個速度延遲可見一斑。更何況 Funliday 在上升期，SEO 真的是非常非常重要，雖然已經有 pppr 可以幫忙處理 prerender 的事情，但底層的 puppeteer 三不五時要不就是記憶體吃太多，要不就是暫存檔太肥，所以改成 SSR 是更重要的事。

## SSR 開始動工

動架構是一件很重大的事，而且在人數不足的狀況之下，工作項更要謹慎評估。與前端工程師討論後，決定前端只做一個 adapter，把新流程用 adapter 接回舊流程解決。

如同上面所提的，在 web 執行第一次 React 動作的時候，會先去打 init API 取得所有參數後，再繼續後面步驟。而改成新流程之後，最主要的想法是解決 API 的 round-trip 問題，而且 web 也是使用 expressjs 做為後端框架，所以把 init API 取得所有參數的流程，直接在每一個 route 加上一個 middleware 處理就好。

### init middleware

```js
async function initMiddleware(req, res, next) {
  let webToken = getCookie(req, "webToken");

  console.log(`webToken: ${webToken}`);

  if (!verifyWebToken(webToken)) {
    webToken = generateWebToken();

    setCookie(res, "webToken", webToken);
  }

  return next();
}

const getCookie = (req, name) => req.cookies[`fld-${name}`];

const setCookie = (res, name, value) => {
  res.cookie(`fld-${name}`, value);
};

app.get("*", initMiddleware, appRouter);
```

把後端提供的 init API 改成在 web server 的 init middleware 處理，但前端要如何取得 middleware 的所有參數？那當然就是要找可以前後端共同存取的資料結構啦。

一開始想到的當然是 cookie，後端的 init middleware 寫入 cookie，前端的 init adapter 取得 cookie，然後再執行原本的工作流程。後端把 middleware 開發完後，塞了測試資料，前端也可以順利取得測試資料。但開發到後期，要真正把所有參數塞到 cookie 之後，卻發現了一個重大的問題，那就是每一個 cookie 的大小只能有 4096bytes 而已，這對於還要經過 base64 編碼後的參數大小根本不夠。

### 把 cookie 分割

```js
const setCookie = (res, name, value) => {
  const stringifyValue = JSON.stringify(value);

  // split value every 1000 chars if value's length > 2000
  if (stringifyValue.length > 2000) {
    console.log(`Use array cookie for ${name}`);

    const valueArr = [];

    for (let i = 0; i < stringifyValue.length; i += 1000) {
      valueArr.push(stringifyValue.slice(i, i + 1000));
    }

    for (let i = 0; i < valueArr.length; i++) {
      res.cookie(`fld-${name}-${i}`, valueArr[i]);
    }

    res.cookie(`fld-${name}-$`, valueArr.length);
  } else {
    res.cookie(`fld-${name}`, stringifyValue);
  }
};
```

所以改良後的寫法變成把 cookie 分割，比如每 2000 bytes 切一刀，總共切成 n 個 chunk，然後前端取得資料的時候再合併起來後解碼。這個改良後的方式最後沒上線，因為原本前端只要關注取得實際的內容就好，結果現在卻因為 cookie 分割的關係，還要處理「合併」這個步驟。怎麼想都覺得奇怪，所以後來就把這個方案捨棄，不使用 cookie 了。

### 想到了 View State

```html
<input
  type="hidden"
  name="__VIEWSTATE"
  id="__VIEWSTATE"
  value="QxHX4IaM9Z+otkbxCcwK...lNymmMdHoN+iO3PnA06vqcbm+JiQGvJNiqJTDNK918Tfnylm7Bdw1f83/GVw=="
/>
```

View State 是 ASP.net 在 WebForm 在保留控制項狀態時的解決方案，就是把狀態存在 `<input type="hidden" />` 裡面，讓狀態可以帶到下一次的 request 裡面，我想到這方式剛好可以讓 HTML 做前後端共同存取。所以利用這方式，把原本在 init middeware 寫到 cookie 的內容，改成寫到 res.locals 裡面，然後再利用 res.render 把內容寫到 HTML 裡面

#### 後端的 middleware

```js
async function initMiddleware(req, res, next) {
  const initParams = await getInit({
    memberId,
    language,
  });

  writeConfig(res, "initParams", initParams);

  combineInitEnv(res);

  return next();
}

const writeConfig = (res, name, value) => {
  if (!Array.isArray(res.locals.xxxConfig)) {
    res.locals.xxxConfig = [];
  }

  res.locals.xxxConfig.push({
    name,
    value,
  });
};

const combineInitEnv = (res) => {
  res.locals.xxxConfig = Buffer.from(
    JSON.stringify(res.locals.xxxConfig)
  ).toString("base64");
};
```

#### 後端的 view template

```pug
meta(charset="UTF-8")
meta(name='funliday-env' content=xxxConfig)
```

而前端的 adapter 就單純使用 `document.querySelector` 取得初始參數就可以了。adapter 剩下的工作就是把所取得的初始參數，一個一個接回去原本的流程就能結束工作。但這牽涉到的業務邏輯太多，這裡就不多提了。

#### 前端的 adapter

```js
const funlidayEnv = document.querySelector('meta[name="funliday-env"]');

initAdapter.parse(funlidayEnv.content);
```

## 結論

從規劃到真正實作完成，大約經歷了半年左右，雖然只完成部分的頁面，但成功從前後端分離轉型為 SSR，顯著提升了使用者體驗與 SEO 效能。這次重構不僅加速了頁面載入速度，還大幅提高了我們在搜尋引擎的排名，吸引更多訪客。雖然最終的解決方案沒有什麼了不起，只要有一點經驗的 Node.js 後端工程師都應該要做的出來，但穿著衣服改衣服，突破各種舊架構上的限制，成功上線後還是覺得蠻值得拿來說嘴的 XD

* [91之ASP.NET由淺入深 不負責講座 Day3 - ViewState](https://ithelp.ithome.com.tw/articles/10051189)
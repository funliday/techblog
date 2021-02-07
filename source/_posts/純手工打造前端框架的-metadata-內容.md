---
title: 純手工打造前端框架的 metadata 內容
date: 2020-05-15 15:50:27
tags:
- prerender
---

這一系列文總共有三篇，這是第一篇。

對於用 Vue, Angular, React 這類前端框架來講，如果沒有對 search engine 或 social network 特別處理的話，出來的結果一定不會如你所想的一樣。

因為 client side rendering (CSR) 的作用域通常是在 body 裡面的 container，所以對於 search engine 或 social network 來說，container 外的 head tag 只是一層皮，裡面的資料基本上沒有什麼用，因為解析出來的內容都一模一樣，沒有獨特性。

```js
const { fstat } = require("fs");

app.get("*", async (req, res) => {
  const urlPath = req.url;

  let loadMetaDataFunction = null;

  // load data function
  for (const obj of mathRoutes(Routes, req.path)) {
    if (obj.route.loadData) {
      loadMetaDataFunction = obj.route.loadData;

      const keys = [];
      const regexp = pathToRegexp(obj.route.path, keys);
      const result = regexp.exec(req.path);

      if (result) {
        req.params = {};

        keys.forEach((key, index) => {
          req.params[key.name] = result[index + 1];
        });
      }
    }
  }

  let content;

  if (!templateHtml) {
    templateHtml = await fs.readFile("public/index.htm", "utf-8");
  }

  if (loadMetaDataFunction) {
    const metaDataResult = await loadMetaDataFunction(
      urlPath,
      req.qpery,
      req.params,
      req.headers
    );

    content = buildMeta(templateHtml, metaDataResult.metaData);
  } else {
    content = buildMeta(templateHtml, defaultMetaTagsString(urlPath));
  }

  res.set("Cache-Control", "no-cache");

  return res.send(content);
});
```

為了要讓每一頁的 head tag 都不一樣，最簡單的方式就是在 web 這一層的 expressjs server 多加一個特殊的 route，去 catch 所有 web 的 get request，類似上面的程式碼。

然後搭配 `path-to-regexp` 來設定特定 url pattern 做不同的 head tag 設計，比如 `kewang/trips/123` 的 title 叫做 "kewang 的台北二天一夜行程"，`kewang/journals/456` 的 title 叫做 "kewang 的宜蘭三日遊記"，當然這裡的例子舉的很簡單，正規的做法應該要搭配資料庫來取得對應的 title 跟 og data 才對，也就是附圖程式裡面的 `loadMetaDataFunction`。

只要做好這類的處理，在 social network 上面應該就可以所向無敵了。但這世界可不只有 social network，更恐怖的谷歌大神還在後面等著你咧！

下一篇就來講一下如何對付搜尋引擎吧。
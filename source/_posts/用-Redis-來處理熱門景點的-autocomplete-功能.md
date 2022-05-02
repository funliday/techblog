---
title: 用 Redis 來處理熱門景點的 autocomplete 功能
date: 2022-05-01 22:40:49
tags:
- autocomplete
- redis
- sorted set
---

{% asset_img screenshot.png 400 %}

又是一篇關於 Autocomplete 的文章，不過這次我們來講如何用 Redis 來處理熱門景點的 autocomplete。Funliday 是一個排行程的服務，當然會有使用者在安排時的熱門景點，所以當使用者輸入「北門」的時候，如果在 autocomplete 可以出現下面這些 candidates 的話，對使用者一定更方便。

<table>
  <thead>
    <tr>
      <th>
        Name
      </th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>北門口肉圓</td>
    </tr>
    <tr>
      <td>北門綠豆沙</td>
    </tr>
    <tr>
      <td>北門肉羹</td>
    </tr>
  </tbody>
</table>

即使如此，這些 candidate 也一定有先後順序，所以加上熱門程度 (可以用加入或觀看的次數做為基礎) 之後就會變成下面這樣。

<table>
  <thead>
    <tr>
      <th>
        Name
      </th>
      <th>
        Hot
      </th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>北門口肉圓</td>
      <td>79</td>
    </tr>
    <tr>
      <td>北門綠豆沙</td>
      <td>84</td>
    </tr>
    <tr>
      <td>北門肉羹</td>
      <td>82</td>
    </tr>
  </tbody>
</table>

## 寫入階段

以前在 {% post_link 用-Redis-來處理-City-的-autocomplete-功能-1 %} 的時候有提過，在做 Autocomplete 的時候，我們可以利用 sorted set 來處理字母順序的問題，而 sorted set 在新增元素的時候，其實也可以加分數 (score) 上去，以上面為例，我們可以使用 `ZADD` 這樣處理。

```
ZADD ac_poi:北 79 北門口肉圓
ZADD ac_poi:北門 79 北門口肉圓
ZADD ac_poi:北門口 79 北門口肉圓
ZADD ac_poi:北門口肉 79 北門口肉圓
ZADD ac_poi:北門口肉圓 79 北門口肉圓
ZADD ac_poi:北 84 北門綠豆沙
ZADD ac_poi:北門 84 北門綠豆沙
ZADD ac_poi:北門綠 84 北門綠豆沙
ZADD ac_poi:北門綠豆 84 北門綠豆沙
ZADD ac_poi:北門綠豆沙 84 北門綠豆沙
ZADD ac_poi:北 82 北門肉羹
ZADD ac_poi:北門 82 北門肉羹
ZADD ac_poi:北門肉 82 北門肉羹
ZADD ac_poi:北門肉羹 82 北門肉羹
```

### 兩種不同的寫入方式

大家應該有發現到這裡的儲存方式和以前不同，之前在儲存的時候是固定的 key 搭配不同的 member prefix，以及固定為 0 的 score。

```sh
# 以前
ZADD ac_city 0 台
ZADD ac_city 0 台北
ZADD ac_city 0 台北市
ZADD ac_city 0 台北市*

# 現在
ZADD ac_poi:北 79 北門口肉圓
ZADD ac_poi:北門 79 北門口肉圓
ZADD ac_poi:北門口 79 北門口肉圓
ZADD ac_poi:北門口肉 79 北門口肉圓
ZADD ac_poi:北門口肉圓 79 北門口肉圓
```

主要是為了可以讓 score 做排序，所以將 prefix 移往 key。雖然 key 看起來會有點奇怪，但這是一個還蠻巧妙的設計。

### 寫入後的結果

內容排不下，我們就以「北門口」為例，存到 redis 之後會變成下面這樣子。

<table>
  <thead>
    <tr>
      <th colspan="2">
        ac_poi:北
      </th>
      <th colspan="2">
        ac_poi:北門
      </th>
      <th colspan="2">
        ac_poi:北門口
      </th>
    </tr>
    <tr>
      <th>
        score
      </th>
      <th>
        member
      </th>
      <th>
        score
      </th>
      <th>
        member
      </th>
      <th>
        score
      </th>
      <th>
        member
      </th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        79
      </td>
      <td>
        北門口肉圓
      </td>
      <td>
        79
      </td>
      <td>
        北門口肉圓
      </td>
      <td>
        79
      </td>
      <td>
        北門口肉圓
      </td>
    </tr>
    <tr>
      <td>
        82
      </td>
      <td>
        北門肉羹
      </td>
      <td>
        82
      </td>
      <td>
        北門肉羹
      </td>
    </tr>
    <tr>
      <td>
        84
      </td>
      <td>
        北門綠豆沙
      </td>
      <td>
        84
      </td>
      <td>
        北門綠豆沙
      </td>
    </tr>
  </tbody>
</table>

所以 sorted set 在同一個 key 存入不同 member 之後，會以 score 由小排到大。到這邊為止，算是寫入階段完成。

## 查詢階段

在搜尋的時候就是仿照 City 那邊的做法，只不過會將原本需要用 `ZRANK` 將 cursor 定位至特定的 member 再往後用 `ZRANGE` 取得資料，現在只要直接用 `ZRANGE` 取得就可以了。雖然 City 也可以用 Lua 來達成 atomic operation，但為了讓系統單純一點，這裡就先不這樣處理了。

```sh
# 以前
pos = ZRANK ac_city 北門
results = ZRANGE ac_city pos + 1 pos + 1 + 50

# 現在
results = ZRANGE ac_poi:北門 +INF -INF BYSCORE REV LIMIT 0 50
```

取回來的 `results` 會變下面這樣子 score 由大排到小，然後再做一些處理，就可以輸出回前端了。

* 84 北門綠豆沙
* 82 北門肉羹
* 79 北門口肉圓

## 結論

原文章還有提到可以用 `ZINCRBY` 來更新 score，但 Funliday 的使用情境其實不用到這麼即時，所以我們的做法是每天重新產生新的 autocomplete 資料。

現在這個功能也已經出現在最新版的 App 了，大家可以在首頁點進「住宿」做搜尋，就會看到這個功能囉。至於原本的「景點瀏覽」要再給我們一些時間，會儘快將這個功能上線！

## References

* [用 Redis 來處理 City 的 autocomplete 功能 - 1](https://techblog.funliday.com/2019/01/08/%E7%94%A8-Redis-%E4%BE%86%E8%99%95%E7%90%86-City-%E7%9A%84-autocomplete-%E5%8A%9F%E8%83%BD-1/)
* [Auto Complete with Redis](http://oldblog.antirez.com/post/autocomplete-with-redis.html)
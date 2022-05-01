---
title: 用 Redis 來處理熱門景點的 autocomplete 功能
date: 2022-05-01 22:40:49
tags:
- autocomplete
- redis
- sorted set
---

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

但即使如此，這些 candidate 也一定有先後順序，所以加上熱門程度 (可以用加入的次數或觀看的次數做為基礎) 之後就會變成下面這樣。

<table>
  <thead>
    <tr>
      <th>
        ID
      </th>
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
      <td>123</td>
      <td>北門口肉圓</td>
      <td>79</td>
    </tr>
    <tr>
      <td>456</td>
      <td>北門綠豆沙</td>
      <td>84</td>
    </tr>
    <tr>
      <td>789</td>
      <td>北門肉羹</td>
      <td>82</td>
    </tr>
  </tbody>
</table>

## 寫入階段

以前在 {% post_link 用-Redis-來處理-City-的-autocomplete-功能-1 %} 的時候有提過，在做 Autocomplete 的時候，我們可以利用 sorted set 來處理字母順序的問題，而 sorted set 在新增元素的時候，其實也可以加分數 (score) 上去，以上面為例，我們可以使用 ZADD 這樣處理。

```
ZADD ac_poi:北 79 123
ZADD ac_poi:北門 79 123
ZADD ac_poi:北門口 79 123
ZADD ac_poi:北門口肉 79 123
ZADD ac_poi:北門口肉圓 79 123
ZADD ac_poi:北 84 456
ZADD ac_poi:北門 84 456
ZADD ac_poi:北門綠 84 456
ZADD ac_poi:北門綠豆 84 456
ZADD ac_poi:北門綠豆沙 84 456
ZADD ac_poi:北 82 789
ZADD ac_poi:北門 82 789
ZADD ac_poi:北門肉 82 789
ZADD ac_poi:北門肉羹 82 789
```

### 兩種不同的寫入方式

大家應該有發現到這裡的儲存方式和以前不同，之前在儲存的時候是固定的 key 搭配不同的 member prefix，以及固定為 0 的 score。

```sh
# 以前
ZADD ac_city 0 t
ZADD ac_city 0 ta
ZADD ac_city 0 tai
ZADD ac_city 0 taip
ZADD ac_city 0 taipe
ZADD ac_city 0 taipei
ZADD ac_city 0 taipei*

# 現在
ZADD ac_poi:北 79 123
ZADD ac_poi:北門 79 123
ZADD ac_poi:北門口 79 123
ZADD ac_poi:北門口肉 79 123
ZADD ac_poi:北門口肉圓 79 123
```

主要是為了可以用 score 做排序所以將 prefix 移往 key，雖然 key 看起來會有點奇怪，但這是一個巧妙的設計。

### 寫入後的結果

內容排不下，我們就以「北門口」為例，存到 redis 之後會變成下面這樣子

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
        123
      </td>
      <td>
        79
      </td>
      <td>
        123
      </td>
      <td>
        79
      </td>
      <td>
        123
      </td>
    </tr>
    <tr>
      <td>
        82
      </td>
      <td>
        789
      </td>
      <td>
        82
      </td>
      <td>
        789
      </td>
    </tr>
    <tr>
      <td>
        84
      </td>
      <td>
        456
      </td>
      <td>
        84
      </td>
      <td>
        456
      </td>
    </tr>
  </tbody>
</table>

所以 sorted set 在同一個 key 存入不同 member 之後，會以 score 由小排到大。到這邊為止，算是寫入階段完成。

## 查詢階段

在搜尋的時候就是仿照 City 那邊的做法，只不過會調整
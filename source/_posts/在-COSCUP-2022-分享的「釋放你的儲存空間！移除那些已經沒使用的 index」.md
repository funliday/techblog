
---
title: 在 COSCUP-2022-分享的「釋放你的儲存空間！移除那些已經沒使用的 index」
ogimage: ogimage.png
date: 2022-09-06 12:47:39
tags:
- postgresql
---

<iframe class="speakerdeck-iframe" frameborder="0" src="https://speakerdeck.com/player/e8398770b7084acabbfc6258adcd0be3" title="釋放你的儲存空間！移除那些已經沒使用的 index" allowfullscreen="true" mozallowfullscreen="true" webkitallowfullscreen="true" style="border: 0px; background: padding-box padding-box rgba(0, 0, 0, 0.1); margin: 0px; padding: 0px; border-radius: 6px; box-shadow: rgba(0, 0, 0, 0.2) 0px 5px 40px; width: 560px; height: 314px;" data-ratio="1.78343949044586"></iframe>

今年小編ZHIH在 COSCUP 上分享的內容，「釋放你的儲存空間！移除那些已經沒使用的 index」，主要是針對這個主題所做的survey。

內容主要分作兩部分 
1. 移除未使用的index 
2. 針對已存在的index 進行空間上的優化

會以大略講重點的方式提及內容

---

## 移除未使用的index 

### 為何要移除這些沒有在使用的index 
+ 預算考量，本身系統用不到下一個量級，卻因為這些不在使用的資料佔據空間，導致必須升級提高成本，備份時間也拉長
+ 當初為了特定的feature設置的index，隨時間推移，原本的feature不再使用，而當初的設置index也不再需要，而佔據空間 
+ delete / update / insert 導致空間碎片化，造成效率降低

### 該怎麼確認這些沒有使用的index 
可以透過pg_stat_all_indexes 利用語句將 idx_scan & idx_tup_read & idx_tup_read 設為 0 (或是一個很小的數字)來找出候選人

### 刪除之前 
+ 看到idx_scan & idx_tup_read & idx_tup_read 的 0，並不代表真的沒使用到，背後的統計邏輯其實是可能會用到但不算記載統計表內， 舉例來說： optimizer 其實會讀取index 來做後續query planer的決策，但讀取的動作不會算到討既資料內
+ 目標的index 沒有使用到的原因，是否為語句上的失誤導致沒有正常使用？
+ 目標的index 在當前的情況下，刪除後是否有刪除的價值，比方說 佔據空間很大，卻又都沒在用

### 刪除之後
+ 監控目標index 刪除前後對效能的影響，如果有嚴重影響要補回去
+ 養成定期檢查習慣，重置index統計表，方便確認每一個區間是否有一些待觀察名單，養成好習慣，日後好相伴～

## 針對已存在的index 進行空間上的優化

### 為何要做針對已存在的index 進行空間上的優化？
Pg有實作MVCC去保存舊有的資料，而這個實作雖然可以方便roll back，卻造成了update，並不是直接針對column修改成指定的value，而是會先標記就資料是過期的再去insert新的資料上去，照樣的特性下，會保留舊有的資料一段時間，進而造成空間上的佔據浪費空間，而這個狀況也增作bloat。

### 要怎麼解決bloat 的情形呢？
主要分兩種語法
+ vacuum
1. vacuum 
可以把位置上的舊資料清掉，重複利用，但這個動作本身不會把空間還給OS
2. vacuum full
可以把位置上的舊資料清掉，且會把空間還給OS，缺點是進行時整張table 會lock不能使用，有不間斷得使用需求是硬傷
+ reindex 
1. reindex 
建立新的index替換舊的index，但是會write lock，有write的需求時會有影響
2. reindex concurrently
可以在不write lock & read lock情況下持續運作，但會造成IO loading負擔及記憶體上升，完成動作的時間拉長，執行效率變差，但好處是service 可以保持運作不間斷

--- 

綜合上述：
+ reindex concurrently 適合production運作，好處是不會中斷，但可接受service 中斷選擇在離峰執行也是不錯的選擇
+ 至於development 則依照需求，找出適用的即可

### 避免在常變更的column設置index 
提到實際行為前要先知道幾樣東西的特性
+ 實際存放資料的地方叫做page (像是memory) or block(像是disk)，每一個page or block都有大小的限制
+ index 裡面會存放資料的實際位置 ex: (22,8) 代表第22個block第8個element，藉此快速拿到資料

Pg在有建立index的情況下進行update，除了在index 會需要多一個entry來放新的資料（前面提及的mvcc有關），也會針對實際的資料新增一筆entry，新的index會指向新的資料位置，這一來一往，除了page 會增加資料外，index的部分也額外需要多一個entry造成資料的浪費，為了優化index那邊空間浪費的部分，而有了HOT update
而HOT update的特性有以下：
+ 使用同一個index entry 指向實際的資料位置不做更換，代表不會額外新增entry
+ 新舊資料要在同一張page or block 

這樣的特性下，確實減少了index那邊的空間，但要滿足觸發條件除了新舊資料要在同一個page or block，也需要更新的 column value 並不屬於任何一個 index，因為對應的column value改變，勢必要產生新的entry來做對應，就跟一般update一樣了，當然實務上還是會有需求對有設index 的column value 做變更，但依然盡可能地去做避免


### 避免錯誤的設置index 
假設一個情境有一張table 專門儲存訂單資料，多數的情況下訂單都是成立的不太需要特別查看，但總會有少數幾張訂單是會取消的，在某些情況下還是需要特別拿出來去確認，假設這個是否取消的column叫"isCanceled" 是boolean value，這個table有1000筆資料有5筆是取消的，為了加快query速度對"isCanceled"這個column做index，但實際上需求真正需要的只是要知道被取消的那5筆資料，但操作上去卻1000筆資料都做index，造成其餘的995筆都是沒意義的，那為何不針對那5筆作index就好？1000 vs 5空間誰大誰小這顯而易見了吧～，而針對這五筆作index動作就是partial index，因此也可以知道適當使用partial index 的重要性

## 結論
+ 透過定期監看 pg_stat_all_indexes，確認沒用到的 index ，並移除
+ 定時去處理 index bloat 問題，節省空間同時也能維護效率

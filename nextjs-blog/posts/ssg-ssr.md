---
title: "When to Use Static Generation v.s. Server-side Rendering"
date: "2020-01-02"
---

### Before & After 對比

#### Before

![image](https://hackmd.io/_uploads/SJMZ17yDee.png)  
![image](https://hackmd.io/_uploads/rJQG1Qywxe.png)

#### After

![image](https://hackmd.io/_uploads/HyoD6slPle.png)  
![image](https://hackmd.io/_uploads/SkCqasgPgx.png)

- 效能提升：3.9 秒 → 2.91 秒（提升約 25.38%）  
- 檔案體積縮小：1903 KB → 1370 KB（減少約 28.01%）

| 項目         | Stat Size（stat）              | Parsed Size（parse）                                         |
|------------|------------------------------|------------------------------------------------------------|
| **定義**     | 原始打包後模組的大小（未壓縮） | 實際在瀏覽器執行時，解析後佔用的大小（包含語法結構、載入狀況） |
| **比較用處** | 方便追蹤模組來源與結構       | 反映真正在瀏覽器中執行時的大小（更準確）                     |
| **常見用途** | 了解各個模組的輸出量         | 最佳化效能與 lazy load 時機點                              |

```js
// 原本：一進頁面就載入（不論有沒有用到）
import EditModeWrapper from "components/EditModeWrapper";
import MonacoEditModeWrapper from "components/EditModeWrapper/monacoEditor";
import PageHeader from "models/micro/PageHeader";
import SearchBoxLayout from "models/micro/SearchBoxLayout";
import Table from "models/micro/Table";
```



| 表頭1 | 表頭2 | 表頭3 |
|------|:----:|-----:|
| 左1   |  中1  |   右1 |
| 左2   |  中2  |   右2 |
| 左3   |  中3  |   右3 |



<table>
  <tr>
    <td>項次</td>
    <td>品名</td>
    <td>描述</td>
  </tr>
  <tr>
    <td>100000</td>
    <td>iPhone 5</td>
    <td>iPhone 5是由蘋果公司開發的觸控式螢幕智慧型手機，是第六代的iPhone和繼承前一代的iPhone 4S。這款手機的設計比較以前產品更薄、更輕，及擁有更高解析度及更廣闊的4英寸觸控式螢幕，支援16:9寬螢幕。這款手機包括了一個自定義設計的ARMv7處理器的蘋果A6的更新、iOS 6操作系統，並且支援高速LTE網路。</td>
  </tr>
</table>
---
title: "Webpack研究:以乘車碼為例"
date: "2025-01-02"
---

## Webpack 研究:以乘車碼為例

### 痛點分析:

目前的問題在於說有些 bundle 過大，會造成載入速度很慢。  
那為了解決這個狀況，我們可以做的事情就是把檔案進行拆分(splitting)，也就是讓檔案資源再切分為更小 size 的 chunk。

### 為什麼要做拆包?

拆包的概念在於把原本一整包的 JavaScript 打包檔（bundle），切成多個較小的檔案（chunks），並且我們可以根據實際需求載入，不需要一次就把整個資源全部載入。這樣就可以改善載入速度慢的問題 (因為 bundle 變小啦~)

### 如何分析專案內的檔案大小?

我們可以使用一個工具叫做 [webpack-bundle-analyzer](https://www.npmjs.com/package/webpack-bundle-analyzer)。  
這個工具可以幫我們揪出檔案肥大造成效能瓶頸的元兇!! (簡單判別: 哪個體積最大)

![image](https://hackmd.io/_uploads/Sy4auOnIxg.png)

> 最需要整頓的對象:  
> entry modules (concatenated)，整包檔案目前有 1.82MB

剛剛這邊其實是 Treemap size : Parsed 顯示的結果，現在我們可以看一下如果 size 是 Stat 的話，圖表會有什麼樣的變化?

![image](https://hackmd.io/_uploads/BJMzYTT8xe.png)

#### 比較 Stat vs Parsed

| 項目         | Stat Size（stat）              | Parsed Size（parse）                                         |
|------------|------------------------------|------------------------------------------------------------|
| **定義**     | 原始打包後模組的大小（未壓縮） | 實際在瀏覽器執行時，解析後佔用的大小（包含語法結構、載入狀況） |
| **比較用處** | 方便追蹤模組來源與結構       | 反映真正在瀏覽器中執行時的大小（更準確）                     |
| **常見用途** | 了解各個模組的輸出量         | 最佳化效能與 lazy load 時機點                              |

### 為什麼 stat size 明明很分散但 parsed size 卻會過大?

即使 Webpack 做了切割（Stat Size 分散），只要程式碼有大量同步引用（如 import），Webpack 還是會把這些模組全都包進入口檔案，導致 Parsed Size 變得集中又巨大。

#### 可能造成的實作問題：

1. 用的是同步 import（靜態載入）: 會讓 chunk 在 build 時就合併，這些模組就會一股腦被 Webpack bundle 進 entry module。  
2. 沒有用 SplitChunksPlugin 配合正確設定: 即使切了 chunk，Webpack 也要看「使用方式」決定要不要分開載入，若沒有設定好，它會傾向全部打進主檔案。

---

## 如何把 entry modules 進行拆包

### ❶ 使用動態載入（Dynamic Import）

改為 lazy load：不是一開始就載入所有程式碼，而是「需要的時候才載入」。

#### 範例：

```js
// 原本：一進頁面就載入（不論有沒有用到）
import EditModeWrapper from "components/EditModeWrapper";
import MonacoEditModeWrapper from "components/EditModeWrapper/monacoEditor";
import PageHeader from "models/micro/PageHeader";
import SearchBoxLayout from "models/micro/SearchBoxLayout";
import Table from "models/micro/Table";
```

```js
// 改成：只有真的需要時才載入（lazy load）
const EditModeWrapper = lazy(() => import("components/EditModeWrapper"));
const MonacoEditModeWrapper = lazy(() =>
  import("components/EditModeWrapper/monacoEditor")
);
const PageHeader = lazy(() => import("models/micro/PageHeader"));
const SearchBoxLayout = lazy(() => import("models/micro/SearchBoxLayout"));
const Table = lazy(() => import("models/micro/Table"));
```

---

### ❷ 設定 SplitChunksPlugin 正確配置

```js
splitChunks: {
  chunks: 'async',
  minSize: 20000,
  minChunks: 1,
  maxAsyncRequests: 30,
  maxInitialRequests: 30,
  cacheGroups: {
    defaultVendors: {
      test: /[\/]node_modules[\/]/,
      priority: -10,
      reuseExistingChunk: true,
    },
    default: {
      minChunks: 2,
      priority: -20,
      reuseExistingChunk: true,
    },
  },
},
```

---

## 實作觀察紀錄：

### 初始狀態: entry modules (concatenated): 1.82MB

![image](https://hackmd.io/_uploads/BkhN1yRUlx.png)  
![image](https://hackmd.io/_uploads/BJkuJJ0Ule.png)

---

### 把 lodash 做模組導入: 1.74MB

| 寫法                                              | 打包體積              |
|---------------------------------------------------|-------------------|
| `import { camelCase, upperFirst } from 'lodash';` | 整包進來 (整包引入)   |
| `import upperFirst from 'lodash/upperFirst';`     | 僅載入單個 (按需引入) |
| `import camelCase from 'lodash/camelCase'`        | 僅載入單個 (按需引入) |

![image](https://hackmd.io/_uploads/HkYw2G1wex.png)  
![image](https://hackmd.io/_uploads/S1CkiG1wll.png)

---

### 試做 Dynamic Import + SplitChunks：1.3MB

![image](https://hackmd.io/_uploads/ryI1hsgvll.png)  
![image](https://hackmd.io/_uploads/ByDi9jxDgl.png)

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

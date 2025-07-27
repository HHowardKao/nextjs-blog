---
title: "Webpack研究:以乘車碼為例"
date: "2025-01-02"
---

## Webpack 研究:以乘車碼為例

### 痛點分析:

目前的問題在於說有些 bundle 過大，會造成載入速度很慢。
那為了解決這個狀況，我們可以做的事情就是把檔案進行拆分(splitting)，
也就是讓檔案資源再切分為更小 size 的 chunk。

### 為什麼要做拆包?

拆包的概念在於把原本一整包的 JavaScript 打包檔（bundle），切成多個較小的檔案（chunks），並且我們可以根據實際需求載入，不需要一次就把整個資源全部載入。這樣就可以改善載入速度慢的問題(因為 bundle 變小啦~)

### 如何分析專案內的檔案大小?

我們可以使用一個工具叫做 webpack-bundle-analyzer，
詳細安裝方式請參考:https://www.npmjs.com/package/webpack-bundle-analyzer
這個工具可以幫我們揪出檔案肥大造成效能瓶頸的元兇!!(簡單判別:哪個體積最大)
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

❶ 用的是同步 import（靜態載入）:會讓 chunk 在 build 時就合併，這些模組就會一股腦被 Webpack bundle 進 entry module。

❷ 沒有用 SplitChunksPlugin 配合正確設定:即使切了 chunk，Webpack 也要看「使用方式」決定要不要分開載入，若沒有設定好，它會傾向全部打進主檔案。

### 如何把 entry modules 進行拆包

#### ❶ 使用動態載入（Dynamic Import）

改為 lazy load：不是一開始就載入所有程式碼，而是「需要的時候才載入」，這樣可以避免把整個網站的 JavaScript 一次全部打包進主檔案（主 bundle），讓網站初次載入更快。

#### 舉例說明:

```js
// 一進頁面就載入（不論有沒有用到）
import EditModeWrapper from "components/EditModeWrapper";
import MonacoEditModeWrapper from "components/EditModeWrapper/monacoEditor";
import PageHeader from "models/micro/PageHeader";
import SearchBoxLayout from "models/micro/SearchBoxLayout";
import Table from "models/micro/Table";
```

> ➡️ 缺點：這樣 EditModeWrapper 和 MonacoEditModeWrapper 等等的東西會被一開始就打包進主 bundle，增加主檔案大小。

```js
// 只有真的需要時才載入
const EditModeWrapper = lazy(() => import("components/EditModeWrapper"));
const MonacoEditModeWrapper = lazy(() =>
  import("components/EditModeWrapper/monacoEditor")
);
const PageHeader = lazy(() => import("models/micro/PageHeader"));
const SearchBoxLayout = lazy(() => import("models/micro/SearchBoxLayout"));
const Table = lazy(() => import("models/micro/Table"));
```

> ➡️ 優點：Webpack 會幫你把動態載入的部分拆成一個的 chunk，只有執行到這段才去下載。

#### ❷ 確認 SplitChunks 設定是否正確

設定 webpack.config.js 的 optimization.splitChunks，確保 chunk 真正被「獨立出來」。
官方文件:https://webpack.docschina.org/plugins/split-chunks-plugin/

#### 舉例說明:

```js
 splitChunks: {
      chunks: 'async', // 僅對動態 import 拆分
      minSize: 20000, // 產生 chunk 的最小體積限制（20KB）
      minChunks: 1, // 模組至少被幾個 chunk 共用才拆分
      maxAsyncRequests: 30, // 同時可並發請求的最大 async chunk 數
      maxInitialRequests: 30, // 初始同步 chunk 最大並發數
      cacheGroups: {
        defaultVendors: {
          test: /[\\/]node_modules[\\/]/,
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


### 實作觀察紀錄:

#### 初始狀態:entry modules (concatenated):1.82MB

![image](https://hackmd.io/_uploads/BkhN1yRUlx.png)
![image](https://hackmd.io/_uploads/BJkuJJ0Ule.png)

#### 1.把 lodash 做模組導入:entry modules (concatenated):1.74MB

(表格僅舉一個例子，有把所有 lodash 都改成這樣按需引入的寫法)
| 寫法                                              | 打包體積             |
|---------------------------------------------------|------------------|
| `import { camelCase, upperFirst } from 'lodash';` | 整包進來(整包引入)   |
| `import upperFirst from 'lodash/upperFirst';`     | 僅載入單個(按需引入) |
| `import camelCase from 'lodash/camelCase'`        | 僅載入單個(按需引入) |

![image](https://hackmd.io/_uploads/HkYw2G1wex.png)
![image](https://hackmd.io/_uploads/S1CkiG1wll.png)

#### 2.試做 Dynamic Import + SplitChunks :1.3MB

> 目前是先沒有很嚴謹的測試(EditModeWrapper...等五個內容) ，只是先確保 lazy load + async 可以正常運作

![螢幕擷取畫面 2025-07-25 150033](https://hackmd.io/_uploads/ryI1hsgvll.png)

![image](https://hackmd.io/_uploads/ByDi9jxDgl.png)

### 來個 Before & After 對比

#### Before

![image](https://hackmd.io/_uploads/SJMZ17yDee.png)
![image](https://hackmd.io/_uploads/rJQG1Qywxe.png)

#### After

![image](https://hackmd.io/_uploads/HyoD6slPle.png)
![image](https://hackmd.io/_uploads/SkCqasgPgx.png)

效能提升幅度（從 3.9 秒 → 2.91 秒）
➡️ 效能提升約 25.38%
檔案體積縮小幅度（從 1903 KB → 1370 KB）
➡️ 體積減少約 28.01%

<!-- ---

1.把loadsh.js拆出來

```js
splitChunks: {
  cacheGroups: {
    lodash: {
      test: /[\\/]node_modules[\\/]lodash[\\/]/,
      name: 'lodash',
      chunks: 'all',
    }
  }
}
```
現在:1.67MB
![image](https://hackmd.io/_uploads/H1q2byCIge.png)
![image](https://hackmd.io/_uploads/HyAgGJCIel.png)

---

2.把@codemirror拆出來
```ts
 codemirror: {
      test: /[\\/]node_modules[\\/]@codemirror[\\/]/,
      name: 'codemirror',
      chunks: 'all',
    },
```
現在:1.35MB
![image](https://hackmd.io/_uploads/Hkro6e0Uxe.png)
![image](https://hackmd.io/_uploads/H1L6agRLxg.png)


---

做lazy loading:
先抓form/FormList.js

```ts
import FormList from 'models/form/FormList'
```

```ts
const FormList = lazy(() =>
  import('models/form/FormList')
)
```
現在:1.21MB
![image](https://hackmd.io/_uploads/S1pTN-CUxe.png)
![image](https://hackmd.io/_uploads/HyWGSWAUee.png)

進一步處理其他相對大的form:
```ts
const Uploader = lazy(() =>
  import('models/form/FormUpload')
)

const FormUploader = lazy(() =>
  import('models/form/FormUploader')
)
const Repeater = lazy(() =>
  import('models/form/FormRepeater')
)
const ShopPicker = lazy(() =>
  import('models/form/FormShopPicker')
)
```

現在:1.003MB
![image](https://hackmd.io/_uploads/SkMfFZ08el.png)
![image](https://hackmd.io/_uploads/rkjtYWAUgx.png) -->

<!-- ---

### 7/24測試

#### 1.原本的初始狀況
> 原本:1.81MB


![image](https://hackmd.io/_uploads/SkCKTfkDgg.png)
![image](https://hackmd.io/_uploads/SJEoaGyPlg.png) -->
<!-- ![image](https://hackmd.io/_uploads/SJMZ17yDee.png)
![image](https://hackmd.io/_uploads/rJQG1Qywxe.png) -->

<!-- #### 2.把lodash的import方式改變


> 現在:1.74MB

![image](https://hackmd.io/_uploads/HkYw2G1wex.png)
![image](https://hackmd.io/_uploads/S1CkiG1wll.png)
![image](https://hackmd.io/_uploads/B1tFymkPxl.png)
![image](https://hackmd.io/_uploads/r1GsJQJvex.png)
 -->

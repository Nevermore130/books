CJS 是 CommonJS 的模塊系統，最初是為了在伺服器端使用的 Node.js 開發而設計的，CJS 使用 require 函數來導入模塊，並使用 module.exports 或 exports 對象來定義導出的內容

```js
// 定義模塊
// math.js
exports.add = function(a, b) {
  return a + b;
};

// 導入模塊
// main.js
var math = require('./math.js');
console.log(math.add(2, 3)); // 輸出: 5
```

ESM 是 ECMAScript 的模塊系統，從 ECMAScript 6（ES6）開始引入並成為 JavaScript 的一部分。ESM 使用 **import** 和 **export** 關鍵字來定義和導入模塊。例如：

```js
// 定義模塊
// math.js
export function add(a, b) {
  return a + b;
}

// 導入模塊
// main.js
import { add } from './math.js';
console.log(add(2, 3)); // 輸出: 5
```

#### 兩者差異之處

1. 語法：ESM 使用 **import** 和 **export**，而 CJS 使用 **require** 和 **module.exports** 或 **exports**。
2. 加載時間：ESM 是靜態加載，即在編譯時就可以確定模塊的依賴關係；而 CJS 是動態加載，即在運行時根據需要動態加載模塊
3. 運行環境：ESM 可以在現代瀏覽器中使用，但需要在 **<script>** 標籤上使用 **type="module"** 屬性；而 CJS 主要用於 Node.js 環境。

ESM 是 JavaScript 的官方模塊系統，自 ECMAScript 6（ES6）開始引入並成為語言的一部分。

它在現代瀏覽器中得到廣泛支援，同時也可以在 Node.js 環境中使用（從 Node.js 12 版本開始原生支援）

相較於CJS之下有以下的優勢:

1. 靜態加載：ESM 在編譯時就可以確定模塊的依賴關係，這使得瀏覽器可以更有效地進行模塊的加載和緩存，提高應用程序的性能。
2. 非阻塞加載：ESM 的加載是非阻塞的，這意味著當瀏覽器遇到 **<script type="module">** 標籤時，它可以繼續解析後面的 HTML，而不需要等待模塊加載完成。
3. 預設導出：ESM 支援預設導出，可以使用 **export default** 導出模塊的預設內容，這使得導入模塊時可以更簡潔。






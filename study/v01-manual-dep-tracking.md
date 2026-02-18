# V01 - 手動依賴追蹤：Dep + effect 的觀察者模式

## 設計故事

### 從一個具體問題開始

想像你正在用原生 JavaScript 寫一個購物車頁面。頁面上有三個地方需要顯示資訊：

```ts
// 這是你的「狀態」——應用程式的資料
let price = 100
let quantity = 3

// 頁面上有三個地方需要根據這些資料來顯示
// 1. 商品總價
document.getElementById('total').textContent = `總價：${price * quantity}`
// 2. 折扣後價格（滿 200 打八折）
const discounted = price * quantity >= 200 ? price * quantity * 0.8 : price * quantity
document.getElementById('discounted').textContent = `折扣價：${discounted}`
// 3. 數量提示
document.getElementById('hint').textContent = quantity > 5 ? '大量訂購' : '一般訂購'
```

到這裡一切正常。但當使用者改變數量時：

```ts
quantity = 5
// 現在你必須手動更新每個依賴 quantity 的地方：
document.getElementById('total').textContent = `總價：${price * quantity}`
const discounted = price * quantity >= 200 ? price * quantity * 0.8 : price * quantity
document.getElementById('discounted').textContent = `折扣價：${discounted}`
document.getElementById('hint').textContent = quantity > 5 ? '大量訂購' : '一般訂購'
```

當使用者改變價格時：

```ts
price = 120
// 又要手動更新所有依賴 price 的地方：
document.getElementById('total').textContent = `總價：${price * quantity}`
const discounted = price * quantity >= 200 ? price * quantity * 0.8 : price * quantity
document.getElementById('discounted').textContent = `折扣價：${discounted}`
// hint 不依賴 price，不需更新——但你得記住這件事
```

問題已經浮現：

1. **你必須記住誰依賴誰**：total 和 discounted 依賴 price 和 quantity，hint 只依賴 quantity。一個真實應用有幾百個這樣的關係，人腦記不住
2. **更新邏輯散落各處**：每次修改狀態後，你都要複製貼上一堆更新程式碼
3. **容易遺漏**：忘了更新某個地方 → 頁面顯示錯誤的資料，而且這種 bug 很難被發現

### 什麼是「副作用（Side Effect）」？

讓我們把上面的更新邏輯整理成函數：

```ts
// 這三個函數就是「副作用」
function updateTotal() {
  document.getElementById('total').textContent = `總價：${price * quantity}`
}

function updateDiscounted() {
  const discounted = price * quantity >= 200 ? price * quantity * 0.8 : price * quantity
  document.getElementById('discounted').textContent = `折扣價：${discounted}`
}

function updateHint() {
  document.getElementById('hint').textContent = quantity > 5 ? '大量訂購' : '一般訂購'
}
```

為什麼叫「副作用」？因為這些函數的目的不是「計算並回傳一個值」，而是「對外部世界產生影響」——修改 DOM、發送網路請求、寫入 localStorage 等。它們「讀取」了某些狀態，然後「做了某些事」。

**沒有副作用會怎樣？** 那你的狀態就只是 JavaScript 變數——改了也沒人知道，頁面永遠不會更新。副作用是「狀態」和「使用者看到的畫面」之間的橋樑。

### 什麼是「依賴關係（Dependency）」？

回頭看那三個副作用函數：

```
updateTotal      讀取了 price 和 quantity  → 它「依賴」price 和 quantity
updateDiscounted 讀取了 price 和 quantity  → 它「依賴」price 和 quantity
updateHint       只讀取了 quantity         → 它「只依賴」quantity
```

「依賴」就是這麼簡單：**一個副作用函數在執行時讀取了某個狀態，它就「依賴」那個狀態。** 當那個狀態改變時，這個副作用需要重新執行。

如果用圖表示：

```
          狀態                    副作用
      ┌─────────┐
      │  price  │──────────→ updateTotal
      │         │──────────→ updateDiscounted
      └─────────┘
      ┌─────────┐
      │quantity │──────────→ updateTotal
      │         │──────────→ updateDiscounted
      │         │──────────→ updateHint
      └─────────┘
```

現在問題變成：**誰來維護這張圖？誰來確保 price 改變時，只重新執行 updateTotal 和 updateDiscounted，而不是所有副作用？**

### 方案一：手動維護（不可行）

```ts
function setPrice(newPrice) {
  price = newPrice
  updateTotal()       // 記得呼叫
  updateDiscounted()  // 記得呼叫
  // updateHint 不需要——記得不呼叫
}

function setQuantity(newQuantity) {
  quantity = newQuantity
  updateTotal()       // 記得呼叫
  updateDiscounted()  // 記得呼叫
  updateHint()        // 記得呼叫
}
```

每新增一個副作用，你都要回去修改所有相關的 set 函數。每刪除一個副作用，也要記得從所有 set 函數中移除。這在小專案中勉強可行，但在有幾十個狀態和幾百個副作用的真實應用中完全不可行。

### 方案二：讓資料自己知道「誰在用我」

換一個角度思考：如果每個「狀態」自己維護一份名單——「誰依賴我」，那狀態改變時，它就能自動通知名單上的每個副作用：

```
price 的名單：[updateTotal, updateDiscounted]
quantity 的名單：[updateTotal, updateDiscounted, updateHint]

price 改變 → 遍歷 price 的名單 → 執行 updateTotal, updateDiscounted
quantity 改變 → 遍歷 quantity 的名單 → 執行所有三個
```

這就是**觀察者模式（Observer Pattern）**，也是 Vue 響應式系統的根基。

但名單怎麼建立？手動建立的話又回到方案一的問題。有沒有辦法**自動建立**這份名單？

### 核心洞察：activeSub 技巧

觀察一個事實：**副作用函數在執行時，一定會讀取它依賴的狀態。** updateTotal 在執行時一定會讀取 price 和 quantity——否則它怎麼計算總價？

所以，如果我們能在副作用函數執行期間「攔截」每次狀態讀取，就能自動知道依賴關係。

具體做法：

1. 設一個全域變數 `activeSub`，指向「當前正在執行的副作用」
2. 副作用開始執行前，把自己放進 `activeSub`
3. 狀態被讀取時，查看 `activeSub` → 「噢，是 updateTotal 在讀我」→ 把 updateTotal 加入名單
4. 副作用執行完，清除 `activeSub`

```
        activeSub 的時間線
        ─────────────────────────────────
時間 →  undefined → updateTotal → undefined → updateHint → undefined
                     ↑ 在這段期間                ↑ 在這段期間
                     price.track() 看到           quantity.track() 看到
                     activeSub = updateTotal      activeSub = updateHint
                     → 加入名單                    → 加入名單
```

這就是 Vue 響應式系統最核心、最底層的思想。後續所有版本的複雜度——Proxy、Link、computed、watch——都是在這個基礎上解決不同層面的問題。

## 核心概念

### 心智模型：三層抽象

在讀程式碼之前，先建立整體的心智模型。Vue 的響應式系統可以用三層來理解：

```
┌──────────────────────────────────────────────────────────┐
│                    第三層：使用者的世界                      │
│                                                          │
│   const state = reactive({ price: 100, quantity: 3 })    │
│   effect(() => { total = state.price * state.quantity })  │
│   state.quantity = 5  // total 自動更新                   │
│                                                          │
│   使用者不需要知道任何底層細節，只管讀寫資料                   │
├──────────────────────────────────────────────────────────┤
│                  第二層：攔截機制                           │
│                                                          │
│   Proxy 攔截 state.price 的讀取 → 呼叫 track()            │
│   Proxy 攔截 state.quantity = 5 的寫入 → 呼叫 trigger()    │
│                                                          │
│   讓「手動呼叫 track/trigger」變成「自動攔截」              │
│   （V03 才會實作，V01 先跳過這層）                          │
├──────────────────────────────────────────────────────────┤
│               第一層：依賴追蹤引擎（V01 的重點）             │
│                                                          │
│   Dep：「我的名單上有誰？」                                │
│   activeSub：「現在是誰在執行？」                           │
│   effect(fn)：「在 fn 執行期間追蹤依賴」                    │
│                                                          │
│   這一層不關心怎麼攔截、不關心 DOM、不關心 Proxy             │
│   它只做一件事：管理「狀態」和「副作用」之間的多對多關係        │
└──────────────────────────────────────────────────────────┘
```

V01 只實作第一層。你可以把它想成一個「引擎」——它不知道汽車長什麼樣，它只知道「油進來，動力出去」。

### Dep 和 Effect 的關係

```
       Dep（狀態端）                Effect（副作用端）

    「誰依賴我？」                 「我依賴誰？」
    ┌───────────┐                ┌───────────────┐
    │   subs    │  ←── 註冊 ───  │    effect fn  │
    │ {eff1,    │                │               │
    │  eff2}    │  ─── 通知 ──→  │  重新執行 fn   │
    └───────────┘                └───────────────┘

    dep.track()                  effect 執行中
    → 把 activeSub               → activeSub = 自己
      加入 subs                  → 執行 fn()
                                 → fn() 中讀到 dep
    dep.trigger()                  → dep.track() 被呼叫
    → 遍歷 subs                    → dep 記住「eff 依賴我」
    → 重新執行每個 fn
```

它們之間是**多對多**的關係：
- 一個 Dep 可以被多個 Effect 依賴（price 被 updateTotal 和 updateDiscounted 依賴）
- 一個 Effect 可以依賴多個 Dep（updateTotal 依賴 price 和 quantity）

### activeSub 的角色

`activeSub` 是這個系統中唯一的「全域狀態」，它扮演的角色像是一個**傳話筒**：

```
沒有 activeSub 的世界：
  dep.track(effect)   // 每次都要手動告訴 dep「請追蹤 effect」
  dep.track(effect)   // 每個 dep 都要手動呼叫

有 activeSub 的世界：
  activeSub = effect  // 設定一次
  fn()                // fn 內部所有 dep.track() 自動知道要追蹤 effect
  activeSub = undefined
```

它消除了「手動告訴每個 dep 要追蹤誰」的繁瑣工作。副作用函數只管執行自己的邏輯，依賴關係在執行過程中自動建立。

## 注意事項

### 這個版本故意留下的問題

V01 是最小可行版本，它故意省略了很多東西，每個省略都是後續版本的動機：

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| `fn()` 拋錯後 `activeSub` 不會被清除 | 後續所有 dep.track() 都會錯誤地追蹤到已死的 effect | V02（try/finally） |
| track/trigger 需要手動呼叫 | 使用者要自己記得在正確的時機呼叫 | V03（Proxy 自動攔截） |
| 依賴只增不減 | 條件分支改變後，舊的依賴不會被移除，造成不必要的重新執行 | V05（版本標記清理） |
| subs 用 Set，操作是 O(n) | 依賴數量多時效能下降 | V06（雙向鏈結串列） |
| 每次 trigger 立即執行所有 effect | 連續修改多個狀態會重複執行 effect | V07（批次更新） |

### 為什麼不一開始就解決所有問題？

因為這些問題是**逐步浮現**的。在只有 3 個 dep 和 2 個 effect 的簡單場景中，你不會感受到 O(n) 的問題，也不需要批次更新。只有當系統變複雜後，這些問題才會冒出來。

Vue 源碼中的每個「複雜設計」都是在回答一個具體問題。理解問題比理解解法重要。

## 程式碼實作

```ts
// ==================== V01：手動依賴追蹤 ====================
//
// 這份程式碼實作了響應式系統的最底層：
// 1. Dep —— 代表一個「可被觀察的狀態」，維護訂閱者名單
// 2. activeSub —— 全域指標，指向「當前正在執行的副作用」
// 3. effect(fn) —— 在 fn 執行期間追蹤依賴，之後依賴變更時自動重新執行 fn
//
// 對應源碼：effect.ts 中的 activeSub、dep.ts 中的 Dep class

// -------------------------------------------------------
// 全域指標：指向當前正在執行的副作用函數
// 當值為 undefined 時，代表「現在沒有副作用在執行」，dep.track() 會跳過
//
// 對應源碼：effect.ts:39 — export let activeSub
// -------------------------------------------------------
let activeSub: (() => void) | undefined = undefined

/**
 * Dep（Dependency）：依賴容器
 *
 * 每個 Dep 代表一個「可觀察的狀態」。
 * 在真實的 Vue 中，每個響應式物件的每個屬性都有一個對應的 Dep。
 * 例如 reactive({ price: 100, quantity: 3 }) 會產生兩個 Dep：
 * 一個給 price，一個給 quantity。
 *
 * 對應源碼：dep.ts:67 — export class Dep
 * 源碼中的 Dep 用雙向鏈結串列取代 Set，V06 會演進到那個設計。
 */
class Dep {
  // 訂閱者名單：所有依賴此狀態的副作用函數
  // 用 Set 確保同一個 effect 不會被重複加入
  subs: Set<() => void> = new Set()

  /**
   * track()：追蹤依賴
   *
   * 當某個狀態被「讀取」時呼叫。
   * 如果此時有正在執行的副作用（activeSub 不為空），
   * 就把那個副作用加入訂閱者名單。
   *
   * 為什麼不直接傳參數 track(effect)？
   * 因為一個副作用函數可能讀取幾十個狀態，
   * 如果每個都要手動傳，就太繁瑣了。
   * activeSub 讓 track() 能「自動知道」該追蹤誰。
   *
   * 對應源碼：dep.ts:108 — track() 方法
   */
  track(): void {
    if (activeSub) {
      this.subs.add(activeSub)
    }
  }

  /**
   * trigger()：觸發更新
   *
   * 當某個狀態被「修改」時呼叫。
   * 遍歷訂閱者名單，重新執行每個副作用。
   *
   * 這裡直接同步執行所有副作用——
   * V07 會改成「加入批次佇列，延遲到所有修改完成後再統一執行」。
   *
   * 對應源碼：dep.ts:167 — trigger() 方法
   */
  trigger(): void {
    this.subs.forEach(effect => effect())
  }
}

/**
 * effect()：建立一個響應式副作用
 *
 * 這個函數做三件事：
 * 1. 把 fn 記錄到 activeSub（告訴系統「接下來是 fn 在執行」）
 * 2. 執行 fn（fn 內部讀取狀態時，dep.track() 會自動建立依賴）
 * 3. 清除 activeSub（告訴系統「fn 執行結束了」）
 *
 * 執行完後，fn 讀取過的每個 dep 都會記住 fn。
 * 之後任何一個 dep 呼叫 trigger()，fn 就會被重新執行。
 *
 * 對應源碼：effect.ts:473 — export function effect()
 * 源碼中 effect 是一個 ReactiveEffect 類別實例，V02 會演進到那個設計。
 */
function effect(fn: () => void): void {
  activeSub = fn
  fn()
  activeSub = undefined
}
```

## 使用範例 / 測試

```ts
// ==================== 範例 1：購物車場景 ====================
//
// 回到設計故事中的購物車問題。
// 現在用 Dep + effect 來解決「誰依賴誰」的管理問題。

// 為每個狀態建立一個 Dep
const priceDep = new Dep()
const quantityDep = new Dep()

// 狀態值（目前還是普通變數，V03 會用 Proxy 包裝）
let price = 100
let quantity = 3

// 副作用 1：計算總價
let total = 0
effect(() => {
  priceDep.track()       // 告訴 priceDep：「我依賴你」
  quantityDep.track()    // 告訴 quantityDep：「我依賴你」
  total = price * quantity
})
console.log(total)       // 300

// 副作用 2：數量提示（只依賴 quantity）
let hint = ''
effect(() => {
  quantityDep.track()    // 只告訴 quantityDep
  hint = quantity > 5 ? '大量訂購' : '一般訂購'
})
console.log(hint)        // '一般訂購'

// ---- 修改 quantity ----
quantity = 10
quantityDep.trigger()
// quantityDep 的名單上有兩個副作用：updateTotal 和 updateHint
// 兩者都被重新執行
console.log(total)       // 1000（price * 10）
console.log(hint)        // '大量訂購'

// ---- 修改 price ----
price = 50
priceDep.trigger()
// priceDep 的名單上只有 updateTotal
// updateHint 不會被執行（因為它沒有追蹤 priceDep）
console.log(total)       // 500（50 * 10）
console.log(hint)        // '大量訂購'（沒變——正確！）


// ==================== 範例 2：驗證自動建立依賴 ====================
//
// 驗證 dep.track() 在 effect 之外呼叫時不會做任何事

const isolatedDep = new Dep()
isolatedDep.track()  // activeSub 是 undefined → 什麼都不做
console.log(isolatedDep.subs.size) // 0

// 只有在 effect 執行期間，track 才會建立依賴
effect(() => {
  isolatedDep.track()
})
console.log(isolatedDep.subs.size) // 1


// ==================== 範例 3：多個 effect 依賴同一個 dep ====================
//
// 一個狀態可以被多個副作用依賴

const sharedDep = new Dep()
let result1 = 0
let result2 = 0

effect(() => {
  sharedDep.track()
  result1 = 1
})
effect(() => {
  sharedDep.track()
  result2 = 2
})

console.log(sharedDep.subs.size) // 2 — 兩個 effect 都在名單上

// trigger 時兩個都會執行
result1 = 0
result2 = 0
sharedDep.trigger()
console.log(result1) // 1
console.log(result2) // 2
```

## 與源碼對照

| 概念 | V01 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| 全域指標 | `let activeSub` （裸函數） | `effect.ts:39` — `export let activeSub: Subscriber` | 源碼中是 `Subscriber` 介面，不是裸函數 |
| 依賴容器 | `class Dep`（用 `Set`） | `dep.ts:67` — `export class Dep` | 源碼用雙向鏈結串列（`Link`）取代 `Set` |
| 追蹤方法 | `dep.track()` | `dep.ts:108` — `track()` 方法 | 源碼中還處理 activeLink 快取、版本同步 |
| 觸發方法 | `dep.trigger()` | `dep.ts:167` — `trigger()` 方法 | 源碼中先遞增 version 和 globalVersion |
| 建立副作用 | `effect(fn)` 直接執行 | `effect.ts:473` — `export function effect()` | 源碼建立 `ReactiveEffect` 類別實例 |

> **V01 的價值**：它不是一個「能用的系統」——它是一個「能理解的模型」。真正的 Vue 源碼有 600 行的 effect.ts 和 400 行的 dep.ts，但它們做的事情和這 30 行程式碼在本質上是一樣的：**讓狀態知道誰在用它，狀態改變時通知那些使用者。**

# V02 - ReactiveEffect 類別

## 設計故事

### 從 V01 遺留的問題開始

V01 建立了基本的依賴追蹤機制——Dep、activeSub、effect(fn)。但它有一個致命缺陷：`activeSub` 的管理太脆弱了。

讓我們用一個具體場景來看這件事。

### 場景：一個 effect 拋錯，全盤崩壞

假設你正在寫一個儀表板，頁面上有三個區塊，各自用一個 effect 來更新：

```ts
// V01 的 effect 實作（回顧）
function effect(fn: () => void): void {
  activeSub = fn
  fn()
  activeSub = undefined
}

const depData = new Dep()
const depChart = new Dep()
let apiData: any = null
let chartResult = ''
let summaryResult = ''

// 副作用 1：抓取 API 資料並渲染表格
effect(() => {
  depData.track()
  apiData = fetchData()        // 如果這裡拋錯呢？
  renderTable(apiData)
})

// 副作用 2：根據資料畫圖
effect(() => {
  depChart.track()
  chartResult = renderChart()
})

// 副作用 3：摘要資訊
effect(() => {
  depData.track()
  summaryResult = summarize()
})
```

現在想像 `fetchData()` 因為網路錯誤拋出了異常。在 V01 的 `effect` 中：

```ts
activeSub = fn       // activeSub 被設為副作用 1
fn()                 // fn() 拋錯！直接跳出！
activeSub = undefined // 這行永遠不會執行
```

結果是什麼？`activeSub` 永遠指向那個已經死掉的副作用 1。接下來副作用 2 和副作用 3 的 `dep.track()` 都會把依賴加到副作用 1 身上——一個已經炸掉的函數。整個儀表板的響應式系統靜靜地壞掉了，而且你很難發現，因為程式不會再報任何錯。

### 場景：巢狀 effect 互相踩踏

還有另一個問題。假設外層 effect 在執行過程中需要啟動一個內層 effect：

```ts
effect(() => {
  depX.track()         // depX 訂閱了外層 effect -- OK

  // 巢狀 effect
  effect(() => {
    depY.track()       // depY 訂閱了內層 effect -- OK
  })
  // 內層 effect 結束時：activeSub = undefined

  depZ.track()         // depZ 想訂閱外層 effect...
                       // 但 activeSub 已經是 undefined 了！
                       // 這個依賴靜靜地丟失了
})
```

外層 effect 在巢狀 effect 之後的所有 `dep.track()` 都會失效。如果你的 effect 裡面用到了一個 `computed`（它本質上也是一個 effect），這個問題立刻就會出現。

### 第三個問題：無法停止

V01 的 effect 就是一個裸函數，一旦建立就永遠活著。你無法告訴系統「我不再需要這個 effect 了」。在元件卸載時，那些 effect 仍然會在 trigger 時被執行——浪費效能，甚至可能操作到已經不存在的 DOM。

### 解法：把裸函數提升為類別

這三個問題的根源是一樣的：**裸函數沒有狀態、沒有身份、沒有保護機制**。解法是把 effect 包裝成一個 `ReactiveEffect` 類別，讓它擁有：

- **`run()` 方法中的 `try/finally`**：無論 fn() 是否拋錯，`activeSub` 一定被還原
- **`prevEffect` 保存機制**：巢狀 effect 結束後，還原到外層 effect 而非 undefined
- **`flags` 狀態旗標**：標記 effect 是否活躍（ACTIVE）、是否正在執行（RUNNING）
- **`stop()` 方法**：讓外部可以終止一個 effect 的生命週期

核心改進是 `run()` 方法中的 `try/finally`：

```
run():
  prevEffect = activeSub     // 記住之前的 effect
  activeSub = this            // 設定自己為當前 effect
  try:
    return this.fn()          // 執行副作用
  finally:
    activeSub = prevEffect    // 無論成功或失敗，都還原
```

這個 `try/finally` 同時解決了：
- **錯誤恢復**：`finally` 保證 `activeSub` 一定被還原
- **巢狀 effect**：透過 `prevEffect` 形成一個隱式的堆疊

## 核心概念

### 從裸函數到類別

```
V01：裸函數                    V02：類別
                              ┌─────────────────┐
fn ──────────────────>        │ ReactiveEffect   │
                              │  fn: () => T     │
activeSub = fn                │  flags: number   │
fn()                          │                  │
activeSub = undefined         │  run(): T        │
                              │  stop(): void    │
                              └─────────────────┘
```

### 在整體系統中的位置

```
┌──────────────────────────────────────────────────────────┐
│                    第三層：使用者的世界                      │
│   const state = reactive({ count: 0 })                   │
│   effect(() => console.log(state.count))                 │
├──────────────────────────────────────────────────────────┤
│                  第二層：攔截機制（V03）                     │
│   Proxy 攔截 get/set → 自動呼叫 track/trigger             │
├──────────────────────────────────────────────────────────┤
│              第一層：依賴追蹤引擎                            │
│                                                          │
│  ┌─── V01 ───┐   ┌────── V02 新增 ──────┐               │
│  │ Dep        │   │ ReactiveEffect 類別   │               │
│  │ activeSub  │   │ try/finally 保護      │               │
│  │ track()    │   │ flags 狀態管理        │               │
│  │ trigger()  │   │ stop() 生命週期       │               │
│  └────────────┘   └──────────────────────┘               │
└──────────────────────────────────────────────────────────┘
```

### 巢狀 effect 的堆疊行為

```
effect(() => {           // activeSub = effectA
  depX.track()           // depX 訂閱 effectA ✓

  effect(() => {         // prevEffect = effectA, activeSub = effectB
    depY.track()         // depY 訂閱 effectB ✓
  })                     // activeSub = effectA（還原）

  depZ.track()           // depZ 訂閱 effectA ✓（正確！）
})
```

### EffectFlags

```ts
enum EffectFlags {
  ACTIVE  = 1 << 0,  // 0b001 - effect 是否活躍
  RUNNING = 1 << 1,  // 0b010 - effect 是否正在執行中
}
```

用位元旗標（bitwise flags）而非多個布林值，是因為：
- 記憶體更緊湊（一個數字 vs 多個布林）
- 可以用位元運算快速檢查和組合狀態

## 注意事項

### 設計取捨

- `run()` 中先設定 `RUNNING` 旗標，結束後清除。這可以用來偵測循環依賴（effect 觸發自己）
- `stop()` 清除 `ACTIVE` 旗標。被停止的 effect 即使被觸發也不會執行
- 在 V01 中 `activeSub` 是函數型別，這裡改為 `Subscriber` 介面——為後續 computed（也是一種 subscriber）做準備
- 源碼中的 `Subscriber` 介面還有 `deps`、`depsTail` 等欄位，這裡先省略，V05/V06 會加入

### 這個版本故意留下的問題

V02 解決了 effect 的錯誤恢復和生命週期問題，但仍然有很多東西故意沒處理：

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| track/trigger 需要手動呼叫 | 使用者必須記得在讀取時呼叫 track、修改時呼叫 trigger，容易遺漏 | V03（Proxy 自動攔截） |
| 依賴只增不減 | 條件分支改變後，舊的依賴不會被移除，造成不必要的重新執行 | V05（版本標記清理） |
| subs 用 Set，操作是 O(n) | 當依賴數量很多時，新增/刪除訂閱者的效能下降 | V06（雙向鏈結串列） |
| 每次 trigger 立即執行所有 effect | 連續修改多個狀態會重複執行同一個 effect | V07（批次更新） |
| 沒有 computed | 無法表達「由其他狀態衍生的值」，每次都要手動用 effect + 變數模擬 | V08（ComputedRefImpl） |

## 程式碼實作

```ts
// ==================== V02：ReactiveEffect 類別 ====================

// 效果旗標——用位元運算管理狀態
enum EffectFlags {
  ACTIVE  = 1 << 0,  // effect 是否活躍（未被 stop）
  RUNNING = 1 << 1,  // effect 是否正在執行中
}

/**
 * Subscriber 介面
 * 任何需要訂閱 Dep 的東西都實作此介面
 * （目前只有 ReactiveEffect，V08 會加入 ComputedRefImpl）
 */
interface Subscriber {
  flags: EffectFlags
  notify(): void
}

// 全域指標：指向當前正在執行的 subscriber
let activeSub: Subscriber | undefined = undefined

/**
 * Dep：依賴容器
 * 與 V01 相同，但 subs 改為存放 Subscriber 而非裸函數
 */
class Dep {
  subs: Set<Subscriber> = new Set()

  track(): void {
    if (activeSub) {
      this.subs.add(activeSub)
    }
  }

  trigger(): void {
    this.subs.forEach(sub => sub.notify())
  }
}

/**
 * ReactiveEffect：響應式副作用類別
 *
 * 核心改進：
 * 1. run() 用 try/finally 確保 activeSub 正確還原
 * 2. 用 flags 管理生命週期（ACTIVE / RUNNING）
 * 3. 提供 stop() 方法終止 effect
 */
class ReactiveEffect<T = any> implements Subscriber {
  // 初始狀態：活躍 + 未執行
  flags: EffectFlags = EffectFlags.ACTIVE

  constructor(public fn: () => T) {}

  /**
   * 執行副作用函數
   * 這是整個類別最關鍵的方法
   */
  run(): T {
    // 如果已被停止，仍然執行函數但不追蹤依賴
    if (!(this.flags & EffectFlags.ACTIVE)) {
      return this.fn()
    }

    // 標記為「正在執行」
    this.flags |= EffectFlags.RUNNING

    // ★ 核心：保存前一個 activeSub，設定自己
    const prevEffect = activeSub
    activeSub = this

    try {
      return this.fn()
    } finally {
      // ★ 無論 fn() 成功或拋錯，都會執行 finally
      // 還原前一個 activeSub——支援巢狀 effect
      activeSub = prevEffect
      // 清除「正在執行」旗標
      this.flags &= ~EffectFlags.RUNNING
    }
  }

  /**
   * 接收到依賴變更通知時的回調
   * 目前直接重新執行，V07 會改為加入批次佇列
   */
  notify(): void {
    // 防止 effect 執行中又觸發自己（無限迴圈）
    if (this.flags & EffectFlags.RUNNING) {
      return
    }
    if (this.flags & EffectFlags.ACTIVE) {
      this.run()
    }
  }

  /**
   * 停止 effect
   * 被停止的 effect 不再響應依賴變更
   */
  stop(): void {
    if (this.flags & EffectFlags.ACTIVE) {
      // 清除 ACTIVE 旗標
      this.flags &= ~EffectFlags.ACTIVE
    }
  }
}

/**
 * 建立並立即執行一個響應式副作用
 */
function effect<T>(fn: () => T): ReactiveEffect<T> {
  const e = new ReactiveEffect(fn)
  e.run()
  return e
}
```

## 使用範例 / 測試

```ts
// ==================== 測試 1：基本的 try/finally 保護 ====================

const dep = new Dep()
let value = 1
let result = 0

const e = effect(() => {
  dep.track()
  result = value * 2
})

console.log(result) // 2

value = 5
dep.trigger()
console.log(result) // 10

// ==================== 測試 2：拋錯後 activeSub 正確還原 ====================

const depSafe = new Dep()
let safeResult = 0

// 第一個 effect 會拋錯
try {
  effect(() => {
    throw new Error('boom!')
  })
} catch (e) {
  // activeSub 應該已經被還原為 undefined
}

// 第二個 effect 不受影響
effect(() => {
  depSafe.track()
  safeResult = 42
})

console.log(safeResult)          // 42
console.log(activeSub)           // undefined — activeSub 已正確還原

// ==================== 測試 3：巢狀 effect ====================

const depOuter = new Dep()
const depInner = new Dep()
let outerValue = 'outer'
let innerValue = 'inner'
let outerResult = ''
let innerResult = ''

effect(() => {
  depOuter.track()
  outerResult = outerValue

  // 巢狀 effect
  effect(() => {
    depInner.track()
    innerResult = innerValue
  })
})

console.log(outerResult)  // 'outer'
console.log(innerResult)  // 'inner'

// 觸發外層
outerValue = 'OUTER'
depOuter.trigger()
console.log(outerResult)  // 'OUTER'

// 觸發內層
innerValue = 'INNER'
depInner.trigger()
console.log(innerResult)  // 'INNER'

// ==================== 測試 4：stop() ====================

const depStop = new Dep()
let stopValue = 0
let stopResult = 0

const stoppable = effect(() => {
  depStop.track()
  stopResult = stopValue
})

stopValue = 100
depStop.trigger()
console.log(stopResult) // 100

// 停止 effect
stoppable.stop()
stopValue = 200
depStop.trigger()
console.log(stopResult) // 100 — 不再響應變更
```

## 與源碼對照

| 概念 | V02 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| ReactiveEffect 類別 | `class ReactiveEffect` | `effect.ts:87-217` — 完整的 `ReactiveEffect` 類別 | 源碼還包含 scheduler、deps 鏈結串列、cleanup 等機制 |
| EffectFlags | 簡化的 `ACTIVE`, `RUNNING` | `effect.ts:41-53` — 完整的 8 個旗標 | 源碼還有 NOTIFIED、DIRTY、TRACKING 等旗標用於批次和 computed |
| Subscriber 介面 | 簡化版 | `effect.ts:58-83` — 包含 `deps`, `depsTail`, `next` 等 | 源碼的 Subscriber 用雙向鏈結串列管理依賴，支援高效清理 |
| run() 方法 | 基本的 try/finally | `effect.ts:151-181` — 含 `prepareDeps`, `cleanupDeps` | 源碼在 run 前後還會進行依賴版本比對和過期依賴清理 |
| stop() 方法 | 只清除旗標 | `effect.ts:183-193` — 還會清理所有依賴連結 | 源碼會遍歷並斷開所有 Link 節點，避免記憶體洩漏 |
| notify() 方法 | 直接重新執行 | `effect.ts:139-149` — 會加入批次佇列 | 源碼不會立即執行，而是標記 NOTIFIED 並推入排程器 |
| effect() 函數 | 簡化版 | `effect.ts:473-494` — 回傳 runner 函數 | 源碼回傳一個綁定了 effect.run() 的 runner，並掛載 effect 引用 |

> **關鍵洞察**：ReactiveEffect 不只是對 V01 的 bug fix——它定義了整個系統的「身份模型」。在 V01 中，副作用是匿名的裸函數；在 V02 中，它變成了有狀態、有旗標、可控制的實體。後續的 computed（V08）和 watch（V13）都建立在這個類別之上。try/finally 建立的「可安全巢狀」執行模型，是整條依賴鏈能正確運作的前提——沒有它，任何一個 effect 拋錯都會讓整個追蹤系統崩潰。

> **下一步**：目前 `dep.track()` 和 `dep.trigger()` 仍然需要手動呼叫。V03 將引入 Proxy，讓屬性存取自動觸發追蹤和觸發。

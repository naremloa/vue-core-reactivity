# V09 - 陣列方法攔截

## 設計故事

V03 用 Proxy 攔截了物件的屬性存取，陣列在 JavaScript 中也是物件，所以 Proxy 也能攔截陣列操作。但陣列有一些特殊行為會導致嚴重問題：

### 問題 1：push 的無限迴圈

```ts
const arr = reactive([])

effect(() => {
  arr.push(1)
})
```

為什麼會無限迴圈？讓我們追蹤 `push` 的執行過程：

```
arr.push(1)
  1. 讀取 arr.length → 觸發 track(arr, 'length')
  2. 設定 arr[0] = 1  → 觸發 trigger(arr, '0')
  3. 設定 arr.length = 1 → 觸發 trigger(arr, 'length')
  4. step 3 觸發了 step 1 建立的依賴 → effect 重新執行
  5. effect 再次呼叫 arr.push(2)
  6. 讀取 arr.length → ...
  7. 無限迴圈！
```

核心問題：`push` 既讀取 `length`（觸發 track）又寫入 `length`（觸發 trigger），形成了一個讀-寫迴圈。

### 解決方案：暫停追蹤

`push`、`pop`、`shift`、`unshift`、`splice` 這些**變異方法**需要在執行時**暫停追蹤**（`pauseTracking`），這樣它們內部的讀取操作不會建立依賴。

### 問題 2：Proxy 身份查找

```ts
const raw = {}
const arr = reactive([raw])

arr.indexOf(raw)  // -1 ❌ 預期是 0
```

因為 `arr[0]` 返回的是 `reactive(raw)`（Proxy），而 `indexOf` 用嚴格相等比較：
```
reactive(raw) === raw  // false
```

解決方案：先用原始參數搜尋，如果找不到且參數是 Proxy，用 `toRaw` 後再搜尋一次。

## 核心概念

### 陣列方法的分類

```
┌──────────────────────────────────────────────┐
│            陣列方法攔截策略                     │
├─────────────────┬────────────────────────────┤
│ 變異方法         │ push, pop, shift,          │
│ (noTracking)    │ unshift, splice             │
│                 │ → pauseTracking + startBatch│
├─────────────────┼────────────────────────────┤
│ 搜尋方法         │ includes, indexOf,         │
│ (searchProxy)   │ lastIndexOf                 │
│                 │ → 先用原始值搜尋，           │
│                 │   再用 toRaw 重試            │
├─────────────────┼────────────────────────────┤
│ 迭代方法         │ forEach, map, filter,      │
│ (apply)         │ find, every, some, etc.     │
│                 │ → 追蹤 ARRAY_ITERATE_KEY    │
│                 │ → 回調中的 item 包裝為       │
│                 │   reactive                   │
├─────────────────┼────────────────────────────┤
│ 迭代器方法       │ [Symbol.iterator],         │
│ (iterator)      │ values, entries             │
│                 │ → 追蹤 ARRAY_ITERATE_KEY    │
│                 │ → next() 時包裝值            │
└─────────────────┴────────────────────────────┘
```

### pauseTracking / resetTracking

```ts
const trackStack: boolean[] = []
let shouldTrack = true

function pauseTracking(): void {
  trackStack.push(shouldTrack)
  shouldTrack = false
}

function resetTracking(): void {
  const last = trackStack.pop()
  shouldTrack = last === undefined ? true : last
}
```

用堆疊而非簡單的布林值，是為了支援巢狀暫停。

### ARRAY_ITERATE_KEY

迭代相關的方法（`forEach`、`map` 等）使用一個特殊的 key `ARRAY_ITERATE_KEY` 做追蹤，而非具體的索引。這樣：
- 新增元素時只需要觸發 `ARRAY_ITERATE_KEY`，而非每個索引
- 修改特定索引時也觸發 `ARRAY_ITERATE_KEY`（因為迭代結果可能改變）

## 注意事項

- `noTracking` 函數同時使用 `pauseTracking` 和 `startBatch/endBatch`——暫停追蹤是為了防止讀取觸發依賴，批次是為了合併寫入觸發的更新
- `searchProxy` 先用原始參數搜尋（可能已經是 Proxy），再用 `toRaw` 後搜尋——這確保 `reactive(raw)` 和 `raw` 都能找到
- 迭代方法中，回調接收到的 `item` 會被 `toReactive` 包裝，確保一致性
- `reactiveReadArray` 用於非變異的讀取操作（`concat`、`join` 等），它追蹤 `ARRAY_ITERATE_KEY` 並返回 raw 陣列的響應式副本

### 這個版本故意留下的問題

V09 處理了陣列方法的攔截，但響應式系統仍有幾類資料結構和管理問題尚未解決：

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| `Map`、`Set` 等集合型別未處理 | 集合使用 `.get()`、`.set()`、`.has()` 等方法呼叫操作資料，Proxy 的 get/set trap 只能攔截屬性存取，無法正確攔截集合的語意操作 | V10（集合型別處理） |
| 巢狀物件不會自動轉為響應式 | `reactive({ list: [{ name: 'A' }] })` 中，修改 `list[0].name` 不會觸發追蹤，必須手動對每層巢狀物件呼叫 `reactive` | V11（深層響應式） |
| 沒有 effect 作用域管理 | 元件卸載時無法批量清理該元件內建立的所有 effect、computed 和 watch，必須逐一手動 `stop()`，容易遺漏造成記憶體洩漏 | V12（EffectScope） |

## 程式碼實作

```ts
// ==================== V09：陣列方法攔截 ====================

// （假設 V01-V08 的基礎設施已存在）

// 追蹤堆疊
const trackStack: boolean[] = []

function pauseTracking(): void {
  trackStack.push(shouldTrack)
  shouldTrack = false
}

function resetTracking(): void {
  const last = trackStack.pop()
  shouldTrack = last === undefined ? true : last
}

// 陣列迭代專用 key
const ARRAY_ITERATE_KEY: unique symbol = Symbol('Array iterate')

/**
 * noTracking：暫停追蹤 + 批次更新
 * 用於 push、pop、shift、unshift、splice
 *
 * 對應源碼：arrayInstrumentations.ts:353-364
 */
function noTracking(
  self: unknown[],
  method: string,
  args: unknown[] = [],
): any {
  pauseTracking()
  startBatch()
  const res = (toRaw(self) as any)[method].apply(self, args)
  endBatch()
  resetTracking()
  return res
}

/**
 * searchProxy：支援 Proxy 身份查找
 * 先用原始參數搜尋，失敗後用 toRaw 再試
 *
 * 對應源碼：arrayInstrumentations.ts:332-349
 */
function searchProxy(
  self: unknown[],
  method: string,
  args: unknown[],
): any {
  const arr = toRaw(self) as any
  track(arr, 'iterate', ARRAY_ITERATE_KEY)

  // 先用原始參數搜尋
  const res = arr[method](...args)

  // 如果找不到且參數是 Proxy，用 raw 值重試
  if ((res === -1 || res === false) && isProxy(args[0])) {
    args[0] = toRaw(args[0])
    return arr[method](...args)
  }

  return res
}

/**
 * reactiveReadArray：追蹤迭代並返回 raw 陣列
 *
 * 對應源碼：arrayInstrumentations.ts:20-25
 */
function reactiveReadArray<T>(array: T[]): T[] {
  const raw = toRaw(array)
  if (raw === array) return raw
  track(raw, 'iterate', ARRAY_ITERATE_KEY)
  return raw.map(toReactive)
}

/**
 * arrayInstrumentations：陣列方法的攔截器
 * 這些方法會取代 Proxy get trap 中返回的原始方法
 *
 * 對應源碼：arrayInstrumentations.ts:42-230
 */
const arrayInstrumentations: Record<string | symbol, Function> = {
  // ===== 變異方法：暫停追蹤 =====

  push(...args: unknown[]) {
    return noTracking(this, 'push', args)
  },

  pop() {
    return noTracking(this, 'pop')
  },

  shift() {
    return noTracking(this, 'shift')
  },

  unshift(...args: unknown[]) {
    return noTracking(this, 'unshift', args)
  },

  splice(...args: unknown[]) {
    return noTracking(this, 'splice', args)
  },

  // ===== 搜尋方法：支援 Proxy 身份查找 =====

  includes(...args: unknown[]) {
    return searchProxy(this, 'includes', args)
  },

  indexOf(...args: unknown[]) {
    return searchProxy(this, 'indexOf', args)
  },

  lastIndexOf(...args: unknown[]) {
    return searchProxy(this, 'lastIndexOf', args)
  },

  // ===== 迭代方法：追蹤 ARRAY_ITERATE_KEY =====

  forEach(
    fn: (item: unknown, index: number, array: unknown[]) => unknown,
    thisArg?: unknown,
  ) {
    return apply(this, 'forEach', fn, thisArg)
  },

  map(
    fn: (item: unknown, index: number, array: unknown[]) => unknown,
    thisArg?: unknown,
  ) {
    return apply(this, 'map', fn, thisArg)
  },

  filter(
    fn: (item: unknown, index: number, array: unknown[]) => unknown,
    thisArg?: unknown,
  ) {
    return apply(this, 'filter', fn, thisArg)
  },

  every(
    fn: (item: unknown, index: number, array: unknown[]) => unknown,
    thisArg?: unknown,
  ) {
    return apply(this, 'every', fn, thisArg)
  },

  some(
    fn: (item: unknown, index: number, array: unknown[]) => unknown,
    thisArg?: unknown,
  ) {
    return apply(this, 'some', fn, thisArg)
  },

  find(
    fn: (item: unknown, index: number, array: unknown[]) => boolean,
    thisArg?: unknown,
  ) {
    return apply(this, 'find', fn, thisArg)
  },

  findIndex(
    fn: (item: unknown, index: number, array: unknown[]) => boolean,
    thisArg?: unknown,
  ) {
    return apply(this, 'findIndex', fn, thisArg)
  },

  // ===== 其他讀取方法 =====

  concat(...args: unknown[]) {
    return reactiveReadArray(this).concat(
      ...args.map(x => (Array.isArray(x) ? reactiveReadArray(x) : x)),
    )
  },

  join(separator?: string) {
    return reactiveReadArray(this).join(separator)
  },
}

/**
 * apply：通用的迭代方法攔截
 * 追蹤 ARRAY_ITERATE_KEY，並確保回調中的 item 是響應式的
 *
 * 對應源碼：arrayInstrumentations.ts:270-306
 */
function apply(
  self: unknown[],
  method: string,
  fn: (item: unknown, index: number, array: unknown[]) => unknown,
  thisArg?: unknown,
): any {
  const arr = toRaw(self)
  track(arr, 'iterate', ARRAY_ITERATE_KEY)

  // 包裝回調：讓 item 是響應式的
  const wrappedFn = function (this: unknown, item: unknown, index: number) {
    return fn.call(this, toReactive(item), index, self)
  }

  return (arr as any)[method](wrappedFn, thisArg)
}

/**
 * 修改 baseHandlers 的 get trap：
 * 當存取的 key 是陣列方法時，返回攔截版本
 *
 * 對應源碼：baseHandlers.ts:89-97
 */
function createArrayAwareGetTrap() {
  return function get(target: any, key: string | symbol, receiver: any): any {
    // 如果是陣列且 key 是被攔截的方法
    if (Array.isArray(target)) {
      const fn = arrayInstrumentations[key]
      if (fn) {
        return fn
      }
    }

    const res = Reflect.get(target, key, receiver)
    track(target, 'get', key)
    return res
  }
}
```

## 使用範例 / 測試

```ts
// ==================== 測試 1：push 不再無限迴圈 ====================

const arr = reactive<number[]>([])
let callCount = 0

effect(() => {
  callCount++
  arr.push(callCount)
})

console.log(callCount)       // 1（只執行一次，沒有無限迴圈）
console.log(toRaw(arr))      // [1]

// ==================== 測試 2：Proxy 身份查找 ====================

const raw = { id: 1 }
const list = reactive([raw])

// 用原始物件搜尋——應該找到
console.log(list.includes(raw))   // true
console.log(list.indexOf(raw))    // 0

// 用 reactive 版本搜尋——也應該找到
const reactiveRaw = reactive(raw)
console.log(list.includes(reactiveRaw))  // true

// ==================== 測試 3：forEach 的響應式追蹤 ====================

const items = reactive([1, 2, 3])
let sum = 0

effect(() => {
  sum = 0
  items.forEach(item => {
    sum += item as number
  })
})

console.log(sum)             // 6

items.push(4)                // 新增元素——觸發 ARRAY_ITERATE_KEY
console.log(sum)             // 10

// ==================== 測試 4：pauseTracking 的堆疊行為 ====================

// 模擬巢狀的追蹤暫停
console.log(shouldTrack)     // true

pauseTracking()
console.log(shouldTrack)     // false

pauseTracking()              // 巢狀暫停
console.log(shouldTrack)     // false

resetTracking()
console.log(shouldTrack)     // false（還在第一層暫停中）

resetTracking()
console.log(shouldTrack)     // true（完全恢復）
```

## 與源碼對照

| 概念 | V09 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| noTracking | pauseTracking + startBatch | `arrayInstrumentations.ts:353-364` | 邏輯相同，源碼中函數名為 `noTracking` |
| searchProxy | 先搜原始值再搜 raw | `arrayInstrumentations.ts:332-349` | 邏輯相同，源碼中還處理 `fromIndex` 參數 |
| apply（迭代攔截） | 通用迭代方法攔截 | `arrayInstrumentations.ts:270-306` | 源碼還處理自訂 Array 子類和 `isShallow` 判斷 |
| reactiveReadArray | 追蹤 ARRAY_ITERATE 並返回 raw | `arrayInstrumentations.ts:20-25` | 邏輯相同，源碼中 `isShallow` 時不呼叫 `toReactive` |
| arrayInstrumentations | 方法攔截器物件 | `arrayInstrumentations.ts:42-230` | 簡化版省略了 `reduce`、`reduceRight`、`flat`、`flatMap` 等方法 |
| pauseTracking | 堆疊式暫停 | `effect.ts:514-517` | 相同實作 |
| resetTracking | 堆疊式恢復 | `effect.ts:530-533` | 相同實作 |
| ARRAY_ITERATE_KEY | `Symbol('Array iterate')` | `dep.ts:248-250` | 相同，源碼中還有 `MAP_KEY_ITERATE_KEY` 用於 Map |
| baseHandlers get trap | 陣列方法攔截 | `baseHandlers.ts:89-97` | 源碼中用 `targetIsArray && (fn = arrayInstrumentations[key])` 判斷，還處理 `__v_isReactive` 等內部 key |

> **關鍵洞察**：不是所有操作都應該被追蹤。`push` 讀取 `length` 是內部實作細節，不是使用者意圖。`pauseTracking` 讓我們能精確控制「哪些操作應該建立依賴」。

> **下一步**：V10 將處理 `Map`/`Set` 等集合型別——它們使用方法呼叫（`.get()`、`.set()`）而非屬性存取，需要完全不同的攔截策略。

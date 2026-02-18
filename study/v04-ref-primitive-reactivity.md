# V04 - Ref：原始值的響應式包裝

## 設計故事

### 從一個具體問題開始

你正在做一個購物車結帳頁面。商品資料用 `reactive()` 包裝得很順利：

```ts
const product = reactive({ price: 100, quantity: 3 })

effect(() => {
  console.log(`總價：${product.price * product.quantity}`)
})

product.quantity = 5  // 自動更新 → 總價：500
```

一切美好。但接下來你需要加一個折扣百分比——它就是一個數字 `0.8`：

```ts
let discount = 0.8

effect(() => {
  console.log(`折扣價：${product.price * product.quantity * discount}`)
})

discount = 0.5  // 什麼都沒發生。頁面還是顯示舊的折扣價。
```

為什麼？因為 `discount` 只是一個普通變數。V03 的 `reactive()` 靠的是 Proxy，而 Proxy 只能代理物件：

```ts
// 這是不可能的
const proxy = new Proxy(42, { ... })  // TypeError: Cannot create proxy with a non-object
```

更根本的原因是，JavaScript 沒有「變數賦值攔截」：

```ts
let count = 0
count = 1  // 沒有任何方法能攔截這個操作
```

那怎麼讓原始值也能響應式？答案是用一個**物件包裝**它，把原始值放在 `.value` 屬性中：

```ts
const discount = ref(0.8)
discount.value    // 讀取 → 觸發 track
discount.value = 0.5  // 寫入 → 觸發 trigger → 頁面自動更新
```

`.value` 的存在是 JavaScript 語言限制下的最小可行方案。它不是設計缺陷，而是在「不能攔截變數賦值」的約束下，最優雅的解決方式。

### RefImpl 的設計

與 `reactive` 不同，ref 不需要 `targetMap`。因為 ref 只有一個 `.value` 屬性，所以 Dep 直接內嵌在 RefImpl 實例上：

```
reactive:  targetMap → depsMap → dep    （間接查找）
ref:       refImpl.dep                   （直接持有）
```

這讓 ref 的追蹤/觸發比 reactive 更快——少了 WeakMap 和 Map 的查找開銷。

## 核心概念

### RefImpl 結構

```
┌─────────────────────┐
│     RefImpl         │
│                     │
│  _value: T          │  ← 存放當前值（如果是物件，會被 reactive 包裝）
│  _rawValue: T       │  ← 存放原始值（未被 Proxy 包裝的）
│  dep: Dep           │  ← 自帶的依賴容器
│                     │
│  get value() ────── │ ──→ dep.track() + return _value
│  set value(v) ───── │ ──→ 比較新舊值 → dep.trigger()
└─────────────────────┘
```

### 為什麼是 `.value`？

考慮幾種替代方案：

| 方案 | 問題 |
|------|------|
| `ref(0)` 返回 `Proxy(Number(0))` | Number 物件行為怪異，`typeof` 結果不同 |
| 編譯器魔法（去掉 .value） | Vue 3.5 實際有 Reactivity Transform，但已廢棄。增加心智負擔 |
| `ref(0)` 返回 getter/setter | 無法作為參數傳遞，會丟失響應性 |
| `.value` 屬性 | 簡單、可預測、可傳遞 |

`.value` 是唯一能同時滿足「可攔截」和「可傳遞」的方案。

## 注意事項

- `ref` 如果接收到物件，會自動用 `reactive()` 包裝它（`toReactive`）
- `_rawValue` 儲存未經 Proxy 包裝的值，用於 set 時的比較（避免 Proxy 和原始值比較的問題）
- `hasChanged` 使用 `Object.is()` 進行比較——所以 `NaN === NaN` 不會重複觸發
- 源碼中的 `RefImpl` 還有 `__v_isRef` 標記，用於 `isRef()` 判斷
- `shallowRef` 在這個版本省略——它只是不做 `toReactive` 包裝的 ref

### 這個版本故意留下的問題

V04 完成了原始值的響應式包裝，但仍有幾個已知問題等待後續版本解決：

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| 依賴只增不減 | 條件分支改變後，舊的依賴不會被移除，造成不必要的重新執行 | V05（版本標記清理） |
| subs 用 Set，操作是 O(n) | 依賴數量多時，查找和刪除效能下降 | V06（雙向鏈結串列） |
| 每次 trigger 立即執行所有 effect | 連續修改多個狀態會重複執行 effect，造成不必要的計算 | V07（批次更新） |
| 沒有 computed | 無法定義衍生值（如 total = price * quantity），只能用 effect 手動同步 | V08（computed） |

## 程式碼實作

```ts
// ==================== V04：Ref ====================

// ---------- 從 V03 沿用（核心部分）----------

enum EffectFlags {
  ACTIVE  = 1 << 0,
  RUNNING = 1 << 1,
}

interface Subscriber {
  flags: EffectFlags
  notify(): void
}

let activeSub: Subscriber | undefined = undefined
let shouldTrack = true

class Dep {
  subs: Set<Subscriber> = new Set()

  track(): void {
    if (activeSub && shouldTrack) {
      this.subs.add(activeSub)
    }
  }

  trigger(): void {
    this.subs.forEach(sub => sub.notify())
  }
}

class ReactiveEffect<T = any> implements Subscriber {
  flags: EffectFlags = EffectFlags.ACTIVE

  constructor(public fn: () => T) {}

  run(): T {
    if (!(this.flags & EffectFlags.ACTIVE)) return this.fn()
    this.flags |= EffectFlags.RUNNING
    const prevEffect = activeSub
    const prevShouldTrack = shouldTrack
    activeSub = this
    shouldTrack = true
    try {
      return this.fn()
    } finally {
      activeSub = prevEffect
      shouldTrack = prevShouldTrack
      this.flags &= ~EffectFlags.RUNNING
    }
  }

  notify(): void {
    if (this.flags & EffectFlags.RUNNING) return
    if (this.flags & EffectFlags.ACTIVE) this.run()
  }

  stop(): void {
    if (this.flags & EffectFlags.ACTIVE) {
      this.flags &= ~EffectFlags.ACTIVE
    }
  }
}

function effect<T>(fn: () => T): ReactiveEffect<T> {
  const e = new ReactiveEffect(fn)
  e.run()
  return e
}

// toReactive：如果值是物件就用 reactive 包裝，否則原樣返回
// （假設 reactive 已從 V03 實作）
function isObject(val: unknown): val is Record<any, any> {
  return val !== null && typeof val === 'object'
}
function toReactive<T>(value: T): T {
  return isObject(value) ? reactive(value) : value
}
function toRaw<T>(observed: T): T {
  return observed // 簡化版，V11 會實作完整的 toRaw
}

// ---------- V04 新增：RefImpl ----------

/**
 * hasChanged：判斷值是否真正改變
 * 使用 Object.is 比較，所以 NaN === NaN 為 true
 *
 * 對應源碼：@vue/shared 的 hasChanged
 */
function hasChanged(value: any, oldValue: any): boolean {
  return !Object.is(value, oldValue)
}

/**
 * RefImpl：原始值的響應式包裝
 *
 * 核心設計：
 * - 用 getter/setter 攔截 .value 的存取
 * - 自帶一個 Dep 實例（不需要 targetMap）
 * - 物件值會被自動 reactive 包裝
 *
 * 對應源碼：ref.ts:113-164
 */
class RefImpl<T = any> {
  _value: T          // 當前值（可能是 Proxy 包裝過的）
  _rawValue: T       // 原始值（未被 Proxy 包裝）
  dep: Dep = new Dep()

  // 標記：讓 isRef() 能判斷
  readonly __v_isRef = true

  constructor(value: T, public readonly __v_isShallow: boolean = false) {
    // 淺層 ref 不做 reactive 包裝
    this._rawValue = __v_isShallow ? value : (toRaw(value) as T)
    this._value = __v_isShallow ? value : (toReactive(value) as T)
  }

  /**
   * get value()：讀取 .value 時追蹤依賴
   *
   * 對應源碼：ref.ts:128-139
   */
  get value(): T {
    this.dep.track()
    return this._value
  }

  /**
   * set value()：寫入 .value 時觸發更新
   * 只有在值真正改變時才觸發
   *
   * 對應源碼：ref.ts:141-163
   */
  set value(newValue: T) {
    const useDirectValue = this.__v_isShallow
    // 比較用的值：如果不是 shallow，就用 raw 值比較
    newValue = (useDirectValue ? newValue : toRaw(newValue)) as T
    if (hasChanged(newValue, this._rawValue)) {
      this._rawValue = newValue
      this._value = useDirectValue ? newValue : (toReactive(newValue) as T)
      this.dep.trigger()
    }
  }
}

/**
 * ref：建立一個響應式引用
 *
 * 對應源碼：ref.ts:58-65
 */
function ref<T>(value: T): RefImpl<T> {
  // 如果已經是 ref，直接返回
  if (isRef(value)) {
    return value as any
  }
  return new RefImpl(value)
}

/**
 * isRef：判斷值是否為 ref
 *
 * 對應源碼：ref.ts:47-49
 */
function isRef(r: any): r is RefImpl {
  return r ? r.__v_isRef === true : false
}

/**
 * unref：如果是 ref 就解包，否則原樣返回
 *
 * 對應源碼：ref.ts:231-233
 */
function unref<T>(ref: T | RefImpl<T>): T {
  return isRef(ref) ? ref.value : ref
}
```

## 使用範例 / 測試

```ts
// ==================== 測試 1：基本的原始值響應式 ====================
//
// 回到設計故事中的問題：讓一個數字也能響應式。

const count = ref(0)
let doubled = 0

effect(() => {
  doubled = count.value * 2
})

console.log(doubled)     // 0
count.value = 5
console.log(doubled)     // 10

// ==================== 測試 2：字串 ref ====================

const name = ref('hello')
let greeting = ''

effect(() => {
  greeting = `${name.value}, world!`
})

console.log(greeting)    // "hello, world!"
name.value = 'hi'
console.log(greeting)    // "hi, world!"

// ==================== 測試 3：物件 ref（自動 reactive 包裝）====================

const data = ref({ count: 0 })
let result = 0

effect(() => {
  // data.value 是一個 reactive proxy
  result = data.value.count
})

console.log(result)      // 0

// 修改內部屬性——因為 data.value 是 reactive 的，所以會觸發
data.value.count = 10
console.log(result)      // 10

// 替換整個物件——觸發 ref 的 set
data.value = { count: 99 }
console.log(result)      // 99

// ==================== 測試 4：相同值不觸發 ====================

const stable = ref(42)
let callCount = 0

effect(() => {
  callCount++
  void stable.value  // 讀取但不使用
})

console.log(callCount)   // 1

stable.value = 42        // 設定相同的值
console.log(callCount)   // 1（不觸發，因為值沒變）

stable.value = 43
console.log(callCount)   // 2

// ==================== 測試 5：isRef / unref ====================

const r = ref(10)
console.log(isRef(r))          // true
console.log(isRef(10))         // false
console.log(unref(r))          // 10
console.log(unref(10))         // 10
```

## 與源碼對照

| 概念 | V04 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| RefImpl 類別 | 基本的 getter/setter | `ref.ts:113-164` — 還有 ReactiveFlags 標記 | 源碼有更完整的 flag 系統和 SSR 支援 |
| ref() 函數 | 簡化版 | `ref.ts:58-65` — 呼叫 `createRef` | 源碼透過 `createRef` 統一處理 ref 和 shallowRef |
| isRef() | 檢查 `__v_isRef` | `ref.ts:47-49` — 檢查 `ReactiveFlags.IS_REF` | 源碼使用列舉常數而非字串 |
| unref() | 簡化版 | `ref.ts:231-233` | 行為一致，無顯著差異 |
| Dep 在 ref 上 | `dep: Dep = new Dep()` | `ref.ts:117` — 相同，Dep 直接內嵌 | 源碼的 Dep 使用雙向鏈結串列而非 Set |
| shallowRef | 省略 | `ref.ts:90-101` — 透過 `createRef(value, true)` | 簡化版用 `__v_isShallow` 參數保留了基本支援 |
| customRef | 省略 | `ref.ts:295-330` — 使用者自訂 track/trigger 時機 | 本版完全省略，需要時可獨立學習 |
| toRefs / toRef | 省略 | `ref.ts:345-504` — 將 reactive 物件屬性轉為 ref | 本版完全省略，屬於 API 層面的便利工具 |

> **關鍵洞察**：ref 和 reactive 的追蹤機制本質相同——都是 `dep.track()` / `dep.trigger()`。差別只在於攔截方式：ref 用 getter/setter（因為只有一個 `.value`），reactive 用 Proxy（因為有多個屬性）。ref 的 `_rawValue` 設計也值得注意：因為 ref 接收物件時會自動 `reactive()` 包裝，比較新舊值時必須用原始值（`_rawValue`）而非 Proxy 包裝後的值——否則 `Object.is(proxy, raw)` 永遠是 `false`，導致每次賦值都觸發不必要的更新。

> **下一步**：目前 effect 的依賴是累加的——如果條件分支改變，舊的依賴不會被清理。V05 將引入版本標記的依賴清理機制。

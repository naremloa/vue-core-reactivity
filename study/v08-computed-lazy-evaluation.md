# V08 - Computed：惰性求值的雙重身份

## 設計故事

假設你有一個「衍生值」——它從其他響應式資料計算而來：

```ts
const firstName = ref('John')
const lastName = ref('Doe')
// fullName 是衍生值
const fullName = firstName.value + ' ' + lastName.value
```

你可能想用 effect 來自動更新它，但 effect 有個問題——它是**主動的**（eager）。每次依賴變更都會立即重新計算，即使沒有人在讀取結果。

如果有 100 個 computed，但頁面只顯示其中 3 個，剩下的 97 個都白算了。

考慮一個更具體的場景：

```ts
// 一個管理後台的儀表板頁面
const users = ref<User[]>([/* ... 10,000 筆使用者資料 */])

// 各種統計指標——都用 effect 實作
let activeCount = 0
let premiumCount = 0
let avgAge = 0
let regionBreakdown: Record<string, number> = {}
// ... 還有 96 個類似的統計

effect(() => { activeCount = users.value.filter(u => u.active).length })
effect(() => { premiumCount = users.value.filter(u => u.premium).length })
effect(() => { avgAge = users.value.reduce((s, u) => s + u.age, 0) / users.value.length })
// ... 其餘 97 個 effect

// 但目前頁面只顯示「概覽」分頁——只用到 activeCount、premiumCount、avgAge 這 3 個
// 其餘 97 個統計都在「詳細分析」分頁，使用者根本還沒切過去
// 每次 users 變更，100 個 effect 全部重新執行——97 個都是白算
```

我們需要一個**惰性**的機制：只有在被讀取時才重新計算。

### Computed 的雙重身份

computed 的獨特之處在於：它**同時是消費者和生產者**。

```
         consumer                    producer
  ┌─────────────────┐         ┌─────────────────┐
  │    Subscriber    │         │       Dep        │
  │                  │         │                  │
  │  deps ──→ dep1   │         │  subs ──→ effect │
  │         → dep2   │         │                  │
  │  flags: DIRTY    │         │  version: 3      │
  └─────────────────┘         └─────────────────┘
          ↑                            ↑
          └── ComputedRefImpl ─────────┘
              fn: () => T
              _value: T
```

作為 **Subscriber**：它訂閱 `firstName` 和 `lastName`，當它們變更時收到通知
作為 **Dep**：它被 effect 訂閱，當自己的值改變時通知 effect

### DIRTY flag 和惰性求值

computed 不會在依賴變更時立即重算。而是：
1. 收到通知 → 設定 `DIRTY` flag
2. 被讀取（`.value`）→ 檢查 DIRTY → 如果 dirty 才重算
3. 重算後比較新舊值 → 只有真正改變才通知訂閱者

### globalVersion 快速路徑

每次任何 dep 觸發時，`globalVersion` 都會遞增。computed 記錄上次計算時的 `globalVersion`：

```
如果 computed.globalVersion === globalVersion
  → 整個系統沒有任何響應式變更 → 直接返回快取值
```

這讓連續讀取同一個 computed 幾乎是零成本的。

## 核心概念

### 求值流程

```
computed.value 被讀取
  │
  ├── this.dep.track()          // 作為 Dep：追蹤讀取者
  │
  ├── refreshComputed(this)     // 嘗試重新計算
  │     │
  │     ├── globalVersion 沒變？ → 直接返回
  │     │
  │     ├── 不是 DIRTY 且依賴沒變？ → 直接返回
  │     │
  │     └── 重新執行 fn()
  │           │
  │           ├── 值沒變 → 不遞增 dep.version
  │           └── 值變了 → dep.version++
  │
  └── return this._value
```

### 通知流程

```
依賴 dep 觸發
  │
  └── computed.notify()
        │
        ├── 設定 DIRTY flag
        │
        └── 如果有訂閱者 → 向上傳遞通知
              │
              └── computed.dep.notify()
                    │
                    └── effect 收到通知 → 加入批次佇列
```

### isDirty 檢查

effect 在執行前會檢查「是否真正 dirty」——這會觸發 computed 鏈上的 `refreshComputed`：

```
effect 被觸發
  → isDirty(effect)
    → 遍歷 effect 的所有 dep
      → dep 是 computed？
        → refreshComputed(computed)
          → computed 重算
          → dep.version 是否變了？
            → 是 → effect 確實 dirty
            → 否 → effect 不 dirty，跳過執行
```

## 注意事項

- `refreshComputed` 在重新求值前會呼叫 `prepareDeps` / `cleanupDeps`——computed 的依賴也需要清理
- computed 的 `notify()` 返回 `true` 表示它是 computed——需要級聯通知 `computed.dep.notify()`
- `dep.version === 0` 時表示第一次求值，不需要比較新舊值
- computed 如果沒有任何訂閱者，不會追蹤自己的依賴（透過 `addSub` 中的懶訂閱邏輯）——這是為了讓未使用的 computed 能被 GC

### 這個版本故意留下的問題

V08 實作了 computed 的惰性求值與雙重身份，但仍有多個面向尚未處理：

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| 陣列的 `push`、`indexOf` 等方法未特殊處理 | `push` 在 effect 中會因同時讀寫 `length` 造成無限迴圈；`indexOf` 無法以原始物件找到 Proxy 包裝的元素 | V09（陣列方法攔截） |
| `Map`、`Set` 等集合型別未處理 | 集合使用 `.get()`、`.set()` 等方法呼叫，而非屬性存取，Proxy 的 get/set trap 無法正確攔截其語意 | V10（集合型別處理） |
| 巢狀物件不會自動轉為響應式 | `reactive({ nested: { count: 0 } })` 中，`nested` 內部的變更不會被追蹤 | V11（深層響應式） |
| 沒有 effect 作用域管理 | 元件卸載時無法批量清理該元件建立的所有 effect 和 computed，造成記憶體洩漏 | V12（EffectScope） |

## 程式碼實作

```ts
// ==================== V08：Computed ====================

// 全域版本號：任何 dep.trigger() 都會遞增
let globalVersion = 0

enum EffectFlags {
  ACTIVE    = 1 << 0,
  RUNNING   = 1 << 1,
  TRACKING  = 1 << 2,
  NOTIFIED  = 1 << 3,
  DIRTY     = 1 << 4,
  EVALUATED = 1 << 7,  // computed 是否已至少求值過一次
}

interface Subscriber {
  deps?: Link
  depsTail?: Link
  flags: EffectFlags
  next?: Subscriber
  notify(): true | void  // computed 返回 true 以觸發級聯通知
}

let activeSub: Subscriber | undefined = undefined
let shouldTrack = true
let batchDepth = 0
let batchedSub: Subscriber | undefined

// --- Link / Dep / batch 系統（從 V07 沿用）---
// 省略重複定義，只列出有變更的部分

class Link {
  version: number
  nextDep?: Link; prevDep?: Link
  nextSub?: Link; prevSub?: Link
  prevActiveLink?: Link
  constructor(public sub: Subscriber, public dep: Dep) {
    this.version = dep.version
  }
}

function batch(sub: Subscriber, isComputed = false): void {
  sub.flags |= EffectFlags.NOTIFIED
  sub.next = batchedSub
  batchedSub = sub
}
function startBatch(): void { batchDepth++ }
function endBatch(): void {
  if (--batchDepth > 0) return
  let error: unknown
  while (batchedSub) {
    let e: Subscriber | undefined = batchedSub
    batchedSub = undefined
    while (e) {
      const next = e.next
      e.next = undefined
      e.flags &= ~EffectFlags.NOTIFIED
      if (e.flags & EffectFlags.ACTIVE) {
        try { (e as ReactiveEffect).trigger() } catch (err) { if (!error) error = err }
      }
      e = next
    }
  }
  if (error) throw error
}

function addSub(link: Link): void {
  link.dep.sc++
  if (link.sub.flags & EffectFlags.TRACKING) {
    const computed = link.dep.computed
    // ★ computed 得到第一個訂閱者時，才開始追蹤自己的依賴
    if (computed && !link.dep.subs) {
      computed.flags |= EffectFlags.TRACKING | EffectFlags.DIRTY
      for (let l = computed.deps; l; l = l.nextDep) {
        addSub(l)
      }
    }
    const currentTail = link.dep.subs
    if (currentTail !== link) {
      link.prevSub = currentTail
      if (currentTail) currentTail.nextSub = link
    }
    link.dep.subs = link
  }
}

function removeSub(link: Link): void {
  const { dep, prevSub, nextSub } = link
  if (prevSub) { prevSub.nextSub = nextSub; link.prevSub = undefined }
  if (nextSub) { nextSub.prevSub = prevSub; link.nextSub = undefined }
  if (dep.subs === link) {
    dep.subs = prevSub
    // ★ computed 失去所有訂閱者時，取消追蹤自己的依賴
    if (!prevSub && dep.computed) {
      dep.computed.flags &= ~EffectFlags.TRACKING
      for (let l = dep.computed.deps; l; l = l.nextDep) {
        removeSub(l, true)
      }
    }
  }
  dep.sc--
}

function removeDep(link: Link): void {
  const { prevDep, nextDep } = link
  if (prevDep) { prevDep.nextDep = nextDep; link.prevDep = undefined }
  if (nextDep) { nextDep.prevDep = prevDep; link.nextDep = undefined }
}

function prepareDeps(sub: Subscriber): void {
  for (let link = sub.deps; link; link = link.nextDep) {
    link.version = -1
    link.prevActiveLink = link.dep.activeLink
    link.dep.activeLink = link
  }
}
function cleanupDeps(sub: Subscriber): void {
  let head: Link | undefined; let tail = sub.depsTail; let link = tail
  while (link) {
    const prev = link.prevDep
    if (link.version === -1) {
      if (link === tail) tail = prev
      removeSub(link); removeDep(link)
    } else { head = link }
    link.dep.activeLink = link.prevActiveLink
    link.prevActiveLink = undefined
    link = prev
  }
  sub.deps = head; sub.depsTail = tail
}

class Dep {
  version = 0
  activeLink?: Link = undefined
  subs?: Link = undefined
  sc: number = 0

  constructor(public computed?: ComputedRefImpl) {}

  track(): Link | undefined {
    if (!activeSub || !shouldTrack || activeSub === this.computed) return
    let link = this.activeLink
    if (link === undefined || link.sub !== activeSub) {
      link = this.activeLink = new Link(activeSub, this)
      if (!activeSub.deps) { activeSub.deps = activeSub.depsTail = link }
      else { link.prevDep = activeSub.depsTail; activeSub.depsTail!.nextDep = link; activeSub.depsTail = link }
      addSub(link)
    } else if (link.version === -1) {
      link.version = this.version
      if (link.nextDep) {
        const next = link.nextDep
        next.prevDep = link.prevDep
        if (link.prevDep) link.prevDep.nextDep = next
        link.prevDep = activeSub.depsTail
        link.nextDep = undefined
        activeSub.depsTail!.nextDep = link
        activeSub.depsTail = link
        if (activeSub.deps === link) activeSub.deps = next
      }
    }
    return link
  }

  trigger(): void {
    this.version++
    globalVersion++
    this.notify()
  }

  notify(): void {
    startBatch()
    try {
      for (let link = this.subs; link; link = link.prevSub) {
        if (link.sub.notify()) {
          // ★ 返回 true 表示是 computed → 級聯通知
          ;(link.sub as ComputedRefImpl).dep.notify()
        }
      }
    } finally {
      endBatch()
    }
  }
}

// ---------- isDirty / refreshComputed ----------

/**
 * isDirty：檢查 subscriber 的依賴是否真正改變
 *
 * 對應源碼：effect.ts:342-359
 */
function isDirty(sub: Subscriber): boolean {
  for (let link = sub.deps; link; link = link.nextDep) {
    if (
      link.dep.version !== link.version ||
      (link.dep.computed &&
        (refreshComputed(link.dep.computed) ||
          link.dep.version !== link.version))
    ) {
      return true
    }
  }
  return false
}

/**
 * refreshComputed：嘗試重新計算 computed 的值
 *
 * 對應源碼：effect.ts:365-419
 */
function refreshComputed(computed: ComputedRefImpl): undefined {
  if (
    computed.flags & EffectFlags.TRACKING &&
    !(computed.flags & EffectFlags.DIRTY)
  ) {
    return
  }
  computed.flags &= ~EffectFlags.DIRTY

  // globalVersion 快速路徑
  if (computed.globalVersion === globalVersion) {
    return
  }
  computed.globalVersion = globalVersion

  // 如果已求值過且依賴沒變，跳過
  if (computed.flags & EffectFlags.EVALUATED && !isDirty(computed)) {
    return
  }

  computed.flags |= EffectFlags.RUNNING
  const dep = computed.dep
  const prevSub = activeSub
  const prevShouldTrack = shouldTrack
  activeSub = computed
  shouldTrack = true

  try {
    prepareDeps(computed)
    const value = computed.fn(computed._value)
    if (dep.version === 0 || !Object.is(value, computed._value)) {
      computed.flags |= EffectFlags.EVALUATED
      computed._value = value
      dep.version++
    }
  } catch (err) {
    dep.version++
    throw err
  } finally {
    activeSub = prevSub
    shouldTrack = prevShouldTrack
    cleanupDeps(computed)
    computed.flags &= ~EffectFlags.RUNNING
  }
}

// ---------- ComputedRefImpl ----------

/**
 * ComputedRefImpl：同時實作 Subscriber 和持有 Dep
 *
 * 對應源碼：computed.ts:47-154
 */
class ComputedRefImpl<T = any> implements Subscriber {
  _value: any = undefined
  readonly dep: Dep = new Dep(this)

  deps?: Link = undefined
  depsTail?: Link = undefined
  flags: EffectFlags = EffectFlags.DIRTY
  globalVersion: number = globalVersion - 1
  next?: Subscriber = undefined

  constructor(
    public fn: (oldValue?: T) => T,
    private readonly setter?: (newValue: T) => void,
  ) {}

  /**
   * notify：收到依賴變更通知
   * 設定 DIRTY flag 並向上傳遞
   *
   * 對應源碼：computed.ts:117-129
   */
  notify(): true | void {
    this.flags |= EffectFlags.DIRTY
    if (
      !(this.flags & EffectFlags.NOTIFIED) &&
      activeSub !== this
    ) {
      batch(this, true)
      return true  // ★ 返回 true 觸發 dep.notify() 級聯
    }
  }

  /**
   * get value()：惰性求值
   *
   * 對應源碼：computed.ts:131-145
   */
  get value(): T {
    const link = this.dep.track()
    refreshComputed(this)
    if (link) {
      link.version = this.dep.version
    }
    return this._value
  }

  set value(newValue: T) {
    if (this.setter) {
      this.setter(newValue)
    }
  }
}

/**
 * computed：建立一個惰性求值的響應式引用
 *
 * 對應源碼：computed.ts:198-221
 */
function computed<T>(
  getterOrOptions: (() => T) | { get: () => T; set: (v: T) => void },
): ComputedRefImpl<T> {
  let getter: () => T
  let setter: ((v: T) => void) | undefined
  if (typeof getterOrOptions === 'function') {
    getter = getterOrOptions
  } else {
    getter = getterOrOptions.get
    setter = getterOrOptions.set
  }
  return new ComputedRefImpl(getter, setter)
}

// ---------- ReactiveEffect（isDirty 整合）----------

class ReactiveEffect<T = any> implements Subscriber {
  deps?: Link = undefined
  depsTail?: Link = undefined
  flags: EffectFlags = EffectFlags.ACTIVE | EffectFlags.TRACKING
  next?: Subscriber = undefined
  scheduler?: () => void = undefined

  constructor(public fn: () => T) {}

  run(): T {
    if (!(this.flags & EffectFlags.ACTIVE)) return this.fn()
    this.flags |= EffectFlags.RUNNING
    prepareDeps(this)
    const prevEffect = activeSub
    const prevShouldTrack = shouldTrack
    activeSub = this
    shouldTrack = true
    try { return this.fn() }
    finally {
      cleanupDeps(this)
      activeSub = prevEffect
      shouldTrack = prevShouldTrack
      this.flags &= ~EffectFlags.RUNNING
    }
  }

  notify(): void {
    if (this.flags & EffectFlags.RUNNING) return
    if (!(this.flags & EffectFlags.NOTIFIED)) batch(this)
  }

  trigger(): void {
    if (this.scheduler) { this.scheduler() }
    else { this.runIfDirty() }
  }

  /**
   * runIfDirty：只在真正 dirty 時才執行
   * 配合 computed 的惰性求值
   */
  runIfDirty(): void {
    if (isDirty(this)) {
      this.run()
    }
  }

  stop(): void {
    if (this.flags & EffectFlags.ACTIVE) {
      for (let link = this.deps; link; link = link.nextDep) removeSub(link)
      this.deps = this.depsTail = undefined
      this.flags &= ~EffectFlags.ACTIVE
    }
  }
}

function effect<T>(fn: () => T): ReactiveEffect<T> {
  const e = new ReactiveEffect(fn); e.run(); return e
}
```

## 使用範例 / 測試

```ts
// ==================== 測試 1：基本的惰性求值 ====================

const count = new Dep()
let countValue = 0
let computeCount = 0

const doubled = computed(() => {
  computeCount++
  count.track()
  return countValue * 2
})

// 尚未讀取——不應計算
console.log(computeCount)     // 0

// 第一次讀取——觸發計算
console.log(doubled.value)    // 0
console.log(computeCount)     // 1

// 再次讀取——不需重新計算（globalVersion 沒變）
console.log(doubled.value)    // 0
console.log(computeCount)     // 1

// 修改依賴
countValue = 5
count.trigger()

// 尚未讀取——仍不計算
console.log(computeCount)     // 1

// 讀取——觸發重新計算
console.log(doubled.value)    // 10
console.log(computeCount)     // 2

// ==================== 測試 2：computed 鏈 ====================

const base = new Dep()
let baseValue = 1

const a = computed(() => { base.track(); return baseValue * 2 })
const b = computed(() => a.value + 1)

let result = 0
effect(() => { result = b.value })

console.log(result)           // 3 (1*2 + 1)

baseValue = 10
base.trigger()
console.log(result)           // 21 (10*2 + 1)

// ==================== 測試 3：值沒變不觸發下游 ====================

const dep = new Dep()
let raw = 5

// computed 從 raw 計算，但結果可能沒變
const clamped = computed(() => {
  dep.track()
  return Math.min(raw, 10)  // 限制最大值為 10
})

let effectCount = 0
effect(() => {
  effectCount++
  void clamped.value
})

console.log(effectCount)      // 1

// raw 從 5 改為 8——clamped 仍然是 8（變了）
raw = 8
dep.trigger()
console.log(effectCount)      // 2

// raw 從 8 改為 15——clamped 仍然是 10
raw = 15
dep.trigger()
// raw 從 15 改為 20——clamped 仍然是 10（沒變！）
raw = 20
dep.trigger()
// effect 可能不需要重新執行，取決於 isDirty 的判斷
```

## 與源碼對照

| 概念 | V08 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| ComputedRefImpl | 基本版 | `computed.ts:47-154` | 源碼還有 `isSSR`、`effect` compat、`__v_isRef` 標記等 |
| globalVersion | `let globalVersion = 0` | `dep.ts:19` | 相同概念，源碼中同樣作為快速路徑判斷 |
| refreshComputed | 核心求值邏輯 | `effect.ts:365-419` | 源碼還處理 SSR 場景和 `_dirty` 向下相容 |
| isDirty | 遍歷依賴檢查版本 | `effect.ts:342-359` | 源碼還檢查 `_dirty` compat flag |
| computed.notify() | 設 DIRTY + 級聯 | `computed.ts:117-129` | 相同邏輯，源碼中額外處理 `ALLOW_RECURSE` |
| computed.value getter | track + refreshComputed | `computed.ts:131-145` | 源碼還有 `__DEV__` 警告和 `isSSR` 分支 |
| dep.notify() 的級聯 | `link.sub.notify()` 返回 true 時呼叫 dep.notify() | `dep.ts:193-199` | 在 notify 迴圈中處理，邏輯相同 |

> **關鍵洞察**：computed 的設計揭示了「通知」和「求值」的精妙分離。當依賴變更時，computed 只做最少的工作——設定 DIRTY flag 並向上傳遞通知。真正的重算被延遲到有人讀取 `.value` 時。更精妙的是 `isDirty` 檢查：即使 computed 重算了，如果結果沒變（例如 `Math.min(x, 10)` 中 x 從 15 變成 20，結果仍是 10），下游的 effect 完全不需要執行。DIRTY 避免不必要的計算，isDirty 避免不必要的通知——兩者缺一不可。

> **下一步**：V09 將處理陣列方法（`push`、`indexOf` 等）的特殊攔截，解決 `push` 在 effect 中的無限迴圈問題。

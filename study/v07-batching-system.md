# V07 - 批次更新：防止級聯效應

## 設計故事

到目前為止，每次 `dep.trigger()` 都會立即執行訂閱者。考慮這個場景：

```ts
const state = reactive({ firstName: '', lastName: '' })

effect(() => {
  console.log(`${state.firstName} ${state.lastName}`)
})

// 同時更新兩個屬性
state.firstName = 'John'   // → effect 執行：「John 」（lastName 還是空的！）
state.lastName = 'Doe'     // → effect 再次執行：「John Doe」
```

問題：
1. **中間狀態**：第一次執行時，只有 firstName 更新了——這是一個不一致的中間狀態
2. **效能浪費**：effect 執行了兩次，但使用者只關心最終結果
3. **DOM glitch**：如果 effect 更新 DOM，使用者會看到一瞬間的閃爍

解決方案：**批次更新（Batching）**。

核心思想出奇地簡單：
- 用一個 `batchDepth` 計數器表示「當前是否在批次中」
- `dep.trigger()` 不直接執行 subscriber，而是把它加入一個**佇列**
- 只有當 `batchDepth` 歸零時（所有批次都結束），才一次性執行佇列中的所有 subscriber

```
startBatch()                 // batchDepth = 1
  state.firstName = 'John'   // subscriber 加入佇列（不執行）
  state.lastName = 'Doe'     // subscriber 已在佇列中（NOTIFIED flag 防重複）
endBatch()                   // batchDepth = 0 → 執行佇列中的所有 subscriber
```

## 核心概念

### 批次的運作流程

```
batchDepth: 0
                    startBatch()
batchDepth: 1       ──────────────────
                    dep.trigger()
                      → sub.notify()
                        → sub 未 NOTIFIED → 加入 batchedSub 佇列
                    dep.trigger()
                      → sub.notify()
                        → sub 已 NOTIFIED → 跳過（不重複加入）
                    endBatch()
batchDepth: 0       ──────────────────
                    → 遍歷 batchedSub 佇列
                    → 對每個 sub 呼叫 trigger()
                    → 清除 NOTIFIED flag
```

### NOTIFIED flag 防重複

```ts
notify(): void {
  if (!(this.flags & EffectFlags.NOTIFIED)) {
    // 設定 NOTIFIED flag → 防止同一個 sub 被加入佇列兩次
    this.flags |= EffectFlags.NOTIFIED
    // 加入批次佇列
    batch(this)
  }
}
```

### 巢狀批次

批次可以巢狀——`batchDepth` 計數器確保只有最外層的 `endBatch` 才會真正執行佇列：

```
startBatch()          // batchDepth = 1
  startBatch()        // batchDepth = 2
    dep.trigger()     // 加入佇列
  endBatch()          // batchDepth = 1 → 不執行
endBatch()            // batchDepth = 0 → 執行佇列
```

這很重要，因為 `dep.notify()` 內部就會呼叫 `startBatch/endBatch`。

### 批次佇列的結構

佇列使用 subscriber 的 `next` 指標形成一個**單向鏈結串列**：

```
batchedSub → sub₁ → sub₂ → sub₃ → undefined
```

這比用陣列更省記憶體（不需要額外分配陣列空間），且不需要 `push` 操作。

## 注意事項

- 批次是**同步的**——不涉及 `setTimeout` 或 `Promise`。Vue 的非同步更新（`nextTick`）是在 runtime-core 層處理的，不在 reactivity 層
- `dep.notify()` 本身就包在 `startBatch/endBatch` 中，所以即使不手動呼叫 `startBatch()`，同一個 `dep.trigger()` 觸發的多個 subscriber 也會被批次處理
- 佇列中的 subscriber 按**加入順序**執行（先加入的先執行）
- 如果 subscriber 執行時又觸發了新的 trigger，新的 subscriber 會被加入佇列尾部，在同一個 `endBatch` 中處理

### 這個版本故意留下的問題

V07 解決了批次更新，但仍有幾個重要的功能尚未實現：

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| 沒有 computed 惰性求值 | 無法實現「只在被讀取時才重新計算」的衍生值，所有計算都是即時的，無法延遲到真正需要時才執行 | V08（computed） |
| 沒有陣列方法攔截 | `push`、`pop`、`splice` 等陣列變異方法無法被正確追蹤，使用者操作陣列時不會觸發更新 | V09（陣列方法處理） |
| 沒有 collection handlers | `Map`、`Set`、`WeakMap`、`WeakSet` 等集合型別無法被響應式代理，無法追蹤 `map.get()`、`set.has()` 等操作 | V10（集合型別處理） |

## 程式碼實作

```ts
// ==================== V07：批次更新 ====================

enum EffectFlags {
  ACTIVE   = 1 << 0,
  RUNNING  = 1 << 1,
  TRACKING = 1 << 2,
  NOTIFIED = 1 << 3,  // 已加入批次佇列
  DIRTY    = 1 << 4,  // V08 會用到
}

interface Subscriber {
  deps?: Link
  depsTail?: Link
  flags: EffectFlags
  next?: Subscriber      // 批次佇列的 next 指標
  notify(): void
}

let activeSub: Subscriber | undefined = undefined
let shouldTrack = true

// ---------- 批次系統 ----------

let batchDepth = 0
let batchedSub: Subscriber | undefined

/**
 * batch：將 subscriber 加入批次佇列
 *
 * 對應源碼：effect.ts:240-249
 */
function batch(sub: Subscriber): void {
  sub.flags |= EffectFlags.NOTIFIED
  // 加入佇列頭部（單向鏈結串列）
  sub.next = batchedSub
  batchedSub = sub
}

/**
 * startBatch：開始批次
 *
 * 對應源碼：effect.ts:254-256
 */
function startBatch(): void {
  batchDepth++
}

/**
 * endBatch：結束批次
 * 當 batchDepth 歸零時，執行佇列中所有 subscriber
 *
 * 對應源碼：effect.ts:262-299
 */
function endBatch(): void {
  if (--batchDepth > 0) {
    return
  }

  let error: unknown
  while (batchedSub) {
    let e: Subscriber | undefined = batchedSub
    batchedSub = undefined  // 清空佇列（如果執行中有新的 trigger，會重新填充）

    while (e) {
      const next: Subscriber | undefined = e.next
      e.next = undefined
      e.flags &= ~EffectFlags.NOTIFIED  // 清除 NOTIFIED flag

      if (e.flags & EffectFlags.ACTIVE) {
        try {
          ;(e as ReactiveEffect).trigger()
        } catch (err) {
          if (!error) error = err
        }
      }
      e = next
    }
  }

  if (error) throw error
}

// ---------- Link / Dep（從 V06 沿用，trigger 改為批次）----------

class Link {
  version: number
  nextDep?: Link
  prevDep?: Link
  nextSub?: Link
  prevSub?: Link
  prevActiveLink?: Link

  constructor(public sub: Subscriber, public dep: Dep) {
    this.version = dep.version
    this.nextDep = this.prevDep =
      this.nextSub = this.prevSub =
      this.prevActiveLink = undefined
  }
}

class Dep {
  version = 0
  activeLink?: Link = undefined
  subs?: Link = undefined
  sc: number = 0

  track(): Link | undefined {
    if (!activeSub || !shouldTrack) return

    let link = this.activeLink
    if (link === undefined || link.sub !== activeSub) {
      link = this.activeLink = new Link(activeSub, this)
      if (!activeSub.deps) {
        activeSub.deps = activeSub.depsTail = link
      } else {
        link.prevDep = activeSub.depsTail
        activeSub.depsTail!.nextDep = link
        activeSub.depsTail = link
      }
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

  /**
   * trigger：遞增版本號，然後通知
   * 注意：globalVersion 也需要遞增（V08 computed 用到）
   */
  trigger(): void {
    this.version++
    this.notify()
  }

  /**
   * notify：★ 改為批次模式
   * 不直接執行 subscriber，而是加入批次佇列
   *
   * 對應源碼：dep.ts:173-204
   */
  notify(): void {
    startBatch()
    try {
      for (let link = this.subs; link; link = link.prevSub) {
        link.sub.notify()
      }
    } finally {
      endBatch()
    }
  }
}

function addSub(link: Link): void {
  link.dep.sc++
  if (link.sub.flags & EffectFlags.TRACKING) {
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
  if (prevSub) {
    prevSub.nextSub = nextSub
    link.prevSub = undefined
  }
  if (nextSub) {
    nextSub.prevSub = prevSub
    link.nextSub = undefined
  }
  if (dep.subs === link) dep.subs = prevSub
  dep.sc--
}

function removeDep(link: Link): void {
  const { prevDep, nextDep } = link
  if (prevDep) {
    prevDep.nextDep = nextDep
    link.prevDep = undefined
  }
  if (nextDep) {
    nextDep.prevDep = prevDep
    link.nextDep = undefined
  }
}

function prepareDeps(sub: Subscriber): void {
  for (let link = sub.deps; link; link = link.nextDep) {
    link.version = -1
    link.prevActiveLink = link.dep.activeLink
    link.dep.activeLink = link
  }
}

function cleanupDeps(sub: Subscriber): void {
  let head: Link | undefined
  let tail = sub.depsTail
  let link = tail
  while (link) {
    const prev = link.prevDep
    if (link.version === -1) {
      if (link === tail) tail = prev
      removeSub(link)
      removeDep(link)
    } else {
      head = link
    }
    link.dep.activeLink = link.prevActiveLink
    link.prevActiveLink = undefined
    link = prev
  }
  sub.deps = head
  sub.depsTail = tail
}

// ---------- ReactiveEffect（加入批次支援）----------

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
    try {
      return this.fn()
    } finally {
      cleanupDeps(this)
      activeSub = prevEffect
      shouldTrack = prevShouldTrack
      this.flags &= ~EffectFlags.RUNNING
    }
  }

  /**
   * notify：★ 改為加入批次佇列
   *
   * 對應源碼：effect.ts:139-149
   */
  notify(): void {
    if (this.flags & EffectFlags.RUNNING) return
    if (!(this.flags & EffectFlags.NOTIFIED)) {
      batch(this)
    }
  }

  /**
   * trigger：實際執行 effect
   * 由 endBatch 呼叫，支援 scheduler
   *
   * 對應源碼：effect.ts:195-203
   */
  trigger(): void {
    if (this.scheduler) {
      this.scheduler()
    } else {
      this.run()
    }
  }

  stop(): void {
    if (this.flags & EffectFlags.ACTIVE) {
      for (let link = this.deps; link; link = link.nextDep) {
        removeSub(link)
      }
      this.deps = this.depsTail = undefined
      this.flags &= ~EffectFlags.ACTIVE
    }
  }
}

function effect<T>(fn: () => T): ReactiveEffect<T> {
  const e = new ReactiveEffect(fn)
  e.run()
  return e
}
```

## 使用範例 / 測試

```ts
// ==================== 測試 1：同步批次——避免多次執行 ====================

const depA = new Dep()
const depB = new Dep()
let valueA = 1
let valueB = 2
let result = 0
let callCount = 0

effect(() => {
  depA.track()
  depB.track()
  result = valueA + valueB
  callCount++
})

console.log(result)      // 3
console.log(callCount)   // 1

// 不使用批次——會執行兩次
callCount = 0
valueA = 10
depA.trigger()  // effect 被通知，加入佇列，endBatch 執行
valueB = 20
depB.trigger()  // effect 被通知，加入佇列，endBatch 執行
// 因為 dep.notify() 內部已有 startBatch/endBatch，
// 但兩次 trigger 是獨立的批次

// 使用手動批次——只執行一次
callCount = 0
startBatch()
valueA = 100
depA.trigger()   // effect 加入佇列（NOTIFIED）
valueB = 200
depB.trigger()   // effect 已在佇列中（NOTIFIED），跳過
endBatch()       // 佇列中只有一個 effect，執行一次

console.log(result)      // 300
console.log(callCount)   // 1  ← 只執行了一次！

// ==================== 測試 2：巢狀批次 ====================

const dep = new Dep()
let v = 0
let count = 0

effect(() => {
  dep.track()
  v  // 讀取
  count++
})

count = 0
startBatch()              // depth = 1
  v = 1
  dep.trigger()           // 加入佇列
  startBatch()            // depth = 2
    v = 2
    dep.trigger()         // 已在佇列中，跳過
  endBatch()              // depth = 1，不執行
endBatch()                // depth = 0，執行佇列

console.log(count)        // 1（只執行了一次）

// ==================== 測試 3：scheduler ====================

const depS = new Dep()
let sValue = 0
const jobs: (() => void)[] = []

const e = new ReactiveEffect(() => {
  depS.track()
  return sValue
})
e.scheduler = () => {
  // 不立即執行，而是加入非同步佇列
  jobs.push(() => e.run())
}
e.run()  // 初始執行

sValue = 42
depS.trigger()

console.log(jobs.length)  // 1 — scheduler 被呼叫，job 加入佇列
// 手動執行 job
jobs[0]()
```

## 與源碼對照

| 概念 | V07 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| batchDepth | 計數器 | `effect.ts:236` | 邏輯完全相同 |
| batchedSub | 單向鏈結串列 | `effect.ts:237` | 邏輯完全相同 |
| batchedComputed | 省略 | `effect.ts:238` | 源碼中 computed 有獨立的批次佇列，在 effect 佇列之前執行，確保 computed 值先更新 |
| batch() | 加入佇列 | `effect.ts:240-249` | 源碼還區分 computed 和一般 subscriber，分別放入不同佇列 |
| startBatch() | `batchDepth++` | `effect.ts:254-256` | 邏輯完全相同 |
| endBatch() | 遍歷佇列執行 | `effect.ts:262-299` | 源碼先處理 batchedComputed 佇列，再處理 batchedSub 佇列 |
| NOTIFIED flag | 防重複加入 | `effect.ts:46` | 邏輯完全相同 |
| dep.notify() | startBatch + 遍歷 subs + endBatch | `dep.ts:173-204` | 源碼還處理 computed 的 dep.notify 級聯通知 |
| scheduler | 可選的排程器 | `effect.ts:111` | Vue runtime 用 scheduler 實作非同步更新（nextTick），reactivity 層本身是同步批次 |

> **關鍵洞察**：Vue 的批次是完全同步的——它不使用 microtask 或 setTimeout。這一點經常被誤解。reactivity 層只負責「收集需要執行的 subscriber，延遲到所有修改完成後再統一執行」。非同步更新（如 `nextTick`、`flush: 'post'`）是 runtime-core 層透過 `scheduler` 選項疊加上去的。這種分層設計讓 reactivity 層保持簡單和可預測——它永遠是同步的，不會引入時序上的不確定性。

> **下一步**：V08 將引入 `computed`——一個同時是 Subscriber 和 Dep 的「中繼節點」，使用 DIRTY flag 實現惰性求值。

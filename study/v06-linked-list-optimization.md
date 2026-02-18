# V06 - 雙向鏈結串列：Link 節點的 O(1) 操作

## 設計故事

### 從一個具體問題開始

想像你正在開發一個類似 Google Sheets 的試算表元件。這個元件有 1000 個儲存格，每個儲存格都是一個 effect，並且依賴多個共享的資料來源：

```ts
// 假設有 1000 個儲存格 effect，每個都依賴 sharedData
for (let i = 0; i < 1000; i++) {
  effect(() => {
    sharedDataDep.track()
    cells[i].textContent = computeCell(i, sharedData)
  })
}
```

V05 用 `Set` 管理 `dep.subs`，用陣列管理 `sub.deps`。當 sharedData 改變時：

```
sharedDataDep.trigger()
→ 遍歷 subs（Set），逐一呼叫每個 effect
→ 每個 effect 重新執行前要清理舊依賴
→ 清理時 dep.subs.delete(effect) → Set 的 delete 操作
→ 清理時 sub.deps.splice(index, 1) → 陣列的 splice 操作
```

這裡的瓶頸在哪？

1. **`dep.subs` 使用 `Set`**：`delete` 操作在最壞情況下是 O(n)
2. **`sub.deps` 使用陣列**：`splice` 操作是 O(n)，因為要移動後面所有元素
3. **每次清理都要遍歷整個依賴列表**

當你有 1000 個儲存格 effect，每個依賴 5 個 dep 時，一次資料變更可能觸發數千次 O(n) 操作。在 60fps 的動畫場景下，這個開銷足以造成明顯的卡頓。

### 解決方案：雙向鏈結串列

雙向鏈結串列的核心優勢：**添加和移除都是 O(1)**——只需要調整前後節點的指標，不需要搬移任何元素。

把 dep 和 sub 之間的連結（Link）同時放在兩條雙向鏈結串列中：

1. **dep-list**：一個 subscriber 的所有依賴 → 每個 sub 維護一條鏈，透過 `nextDep` / `prevDep` 連結
2. **sub-list**：一個 dep 的所有訂閱者 → 每個 dep 維護一條鏈，透過 `nextSub` / `prevSub` 連結

```
同一個 Link 同時存在於兩條鏈中：

Sub 的 dep-list:
  Link(sub,depA) ←→ Link(sub,depB) ←→ Link(sub,depC)
       ↕                  ↕                  ↕
Dep 的 sub-list:     Dep 的 sub-list:    Dep 的 sub-list:
  Link(sub1,dep)     Link(sub1,dep)      Link(sub1,dep)
       ↕                  ↕                  ↕
  Link(sub2,dep)     Link(sub2,dep)      Link(sub2,dep)
```

回到試算表的例子：當某個儲存格 effect 被銷毀時，從 `sharedDataDep.subs` 中移除它只需要 O(1)——不管有 100 個還是 10000 個訂閱者，操作時間都是固定的。

## 核心概念

### Link 的結構

```
                    ┌───────────────────┐
                    │       Link        │
                    │                   │
  dep-list ←──────  │  prevDep  nextDep │ ──────→ dep-list
  (同一個 sub       │                   │   (同一個 sub
   的上一個 dep)    │  prevSub  nextSub │    的下一個 dep)
                    │                   │
  sub-list ←──────  │  sub      dep     │ ──────→ sub-list
  (同一個 dep       │  version          │   (同一個 dep
   的上一個 sub)    └───────────────────┘    的下一個 sub)
```

### Subscriber 和 Dep 的指標

```
Subscriber:
  deps ────→ Link₁ ←→ Link₂ ←→ Link₃    (head)
  depsTail ────────────────────→ Link₃    (tail)

Dep:
  subs ────────────────────────→ Link_c   (tail，從尾部開始遍歷)
```

### O(1) 操作

**addSub（添加訂閱者）**：
```
之前：  ... ←→ [tail]
之後：  ... ←→ [tail] ←→ [new]
        dep.subs = new
```

**removeSub（移除訂閱者）**：
```
之前：  [prev] ←→ [link] ←→ [next]
之後：  [prev] ←→ [next]
```

兩者都是指標操作，O(1)。

## 注意事項

- `dep.subs` 指向鏈的**尾部**（tail），`dep.notify()` 從尾部往前遍歷。這是為了在批次模式中以正確的順序觸發
- `sub.deps` 指向鏈的**頭部**（head），`sub.depsTail` 指向尾部。新依賴從尾部添加，保持依賴的存取順序
- `dep.activeLink` 是一個快取：指向「當前 activeSub 對應的 Link」，讓 `dep.track()` 能 O(1) 找到已有的連結
- `link.prevActiveLink` 用於巢狀 effect：保存外層 effect 的 activeLink，effect 結束後還原
- `dep.sc`（subscriber count）追蹤訂閱者數量，用於 computed 的懶訂閱和屬性 dep 的自動清理

### 這個版本故意留下的問題

V06 解決了資料結構的效能問題，但仍有幾個重要的行為尚未處理：

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| 沒有批次更新——trigger 立即執行所有 subscriber | 連續修改 `state.a = 1; state.b = 2` 會讓 effect 執行兩次，產生不一致的中間狀態 | V07（批次系統） |
| 沒有 computed 惰性求值 | 無法實現「只在被讀取時才重新計算」的衍生值，每次依賴變更都會立即重算 | V08（computed） |
| 沒有陣列方法攔截 | `push`、`pop`、`splice` 等陣列方法無法被正確追蹤，使用者操作陣列時不會觸發更新 | V09（陣列方法處理） |

## 程式碼實作

```ts
// ==================== V06：雙向鏈結串列 ====================

enum EffectFlags {
  ACTIVE   = 1 << 0,
  RUNNING  = 1 << 1,
  TRACKING = 1 << 2,
  NOTIFIED = 1 << 3,
}

interface Subscriber {
  deps?: Link
  depsTail?: Link
  flags: EffectFlags
  notify(): void
}

let activeSub: Subscriber | undefined = undefined
let shouldTrack = true

/**
 * Link：連結節點
 * 同時存在於 dep 的 sub-list 和 sub 的 dep-list 中
 *
 * 對應源碼：dep.ts:32-62
 */
class Link {
  version: number

  // dep-list 的指標（同一個 sub 的所有 dep）
  nextDep?: Link
  prevDep?: Link

  // sub-list 的指標（同一個 dep 的所有 sub）
  nextSub?: Link
  prevSub?: Link

  // 巢狀 effect 時保存前一個 activeLink
  prevActiveLink?: Link

  constructor(
    public sub: Subscriber,
    public dep: Dep,
  ) {
    this.version = dep.version
    this.nextDep = this.prevDep =
      this.nextSub = this.prevSub =
      this.prevActiveLink = undefined
  }
}

/**
 * Dep：使用雙向鏈結串列管理訂閱者
 *
 * 對應源碼：dep.ts:67-205
 */
class Dep {
  version = 0

  // 指向「當前 activeSub 對應的 Link」的快取
  activeLink?: Link = undefined

  // sub-list 的尾節點
  subs?: Link = undefined

  // 訂閱者計數
  sc: number = 0

  /**
   * track：追蹤依賴
   *
   * 兩種情況：
   * 1. 新的 subscriber → 建立 Link，加入兩條鏈
   * 2. 已存在的 subscriber → 同步 version（mark-and-sweep 的「mark」）
   *
   * 對應源碼：dep.ts:108-165
   */
  track(): Link | undefined {
    if (!activeSub || !shouldTrack) return

    let link = this.activeLink
    // 檢查快取：activeLink 是否指向當前 activeSub
    if (link === undefined || link.sub !== activeSub) {
      // 新的連結
      link = this.activeLink = new Link(activeSub, this)

      // 加入 sub 的 dep-list（尾部添加）
      if (!activeSub.deps) {
        activeSub.deps = activeSub.depsTail = link
      } else {
        link.prevDep = activeSub.depsTail
        activeSub.depsTail!.nextDep = link
        activeSub.depsTail = link
      }

      // 加入 dep 的 sub-list
      addSub(link)
    } else if (link.version === -1) {
      // 已存在的連結，被 prepareDeps 標記為 -1
      // 同步 version → 標記為「仍在使用」
      link.version = this.version

      // 如果這個 link 不在 dep-list 尾部，移到尾部
      // 確保 dep-list 按存取順序排列
      if (link.nextDep) {
        const next = link.nextDep
        next.prevDep = link.prevDep
        if (link.prevDep) {
          link.prevDep.nextDep = next
        }

        link.prevDep = activeSub.depsTail
        link.nextDep = undefined
        activeSub.depsTail!.nextDep = link
        activeSub.depsTail = link

        if (activeSub.deps === link) {
          activeSub.deps = next
        }
      }
    }

    return link
  }

  trigger(): void {
    this.version++
    this.notify()
  }

  /**
   * notify：通知所有訂閱者
   * 從 subs（尾部）往前遍歷
   *
   * 對應源碼：dep.ts:173-204
   */
  notify(): void {
    for (let link = this.subs; link; link = link.prevSub) {
      link.sub.notify()
    }
  }
}

/**
 * addSub：將 Link 加入 dep 的 sub-list（尾部）
 * O(1) 操作
 *
 * 對應源碼：dep.ts:207-232
 */
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

/**
 * removeSub：從 dep 的 sub-list 中移除 Link
 * O(1) 操作——只需調整前後指標
 *
 * 對應源碼：effect.ts:421-459
 */
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
  // 如果是尾節點，更新 dep.subs
  if (dep.subs === link) {
    dep.subs = prevSub
  }
  dep.sc--
}

/**
 * removeDep：從 sub 的 dep-list 中移除 Link
 * O(1) 操作
 *
 * 對應源碼：effect.ts:461-471
 */
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

/**
 * prepareDeps：標記所有依賴為 -1，並設定 activeLink
 *
 * 對應源碼：effect.ts:301-311
 */
function prepareDeps(sub: Subscriber): void {
  for (let link = sub.deps; link; link = link.nextDep) {
    link.version = -1
    link.prevActiveLink = link.dep.activeLink
    link.dep.activeLink = link
  }
}

/**
 * cleanupDeps：清理版本為 -1 的依賴，還原 activeLink
 *
 * 對應源碼：effect.ts:313-340
 */
function cleanupDeps(sub: Subscriber): void {
  let head: Link | undefined
  let tail = sub.depsTail
  let link = tail

  while (link) {
    const prev = link.prevDep
    if (link.version === -1) {
      // 未使用——從兩條鏈中移除
      if (link === tail) tail = prev
      removeSub(link)
      removeDep(link)
    } else {
      // 仍在使用——記錄為新的 head
      head = link
    }
    // 還原 activeLink
    link.dep.activeLink = link.prevActiveLink
    link.prevActiveLink = undefined
    link = prev
  }
  sub.deps = head
  sub.depsTail = tail
}

/**
 * ReactiveEffect：使用 Link 鏈結串列的版本
 */
class ReactiveEffect<T = any> implements Subscriber {
  deps?: Link = undefined
  depsTail?: Link = undefined
  flags: EffectFlags = EffectFlags.ACTIVE | EffectFlags.TRACKING

  constructor(public fn: () => T) {}

  run(): T {
    if (!(this.flags & EffectFlags.ACTIVE)) {
      return this.fn()
    }
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

  notify(): void {
    if (this.flags & EffectFlags.RUNNING) return
    if (this.flags & EffectFlags.ACTIVE) {
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
// ==================== 測試 1：Link 的 O(1) 操作驗證 ====================

const dep1 = new Dep()
const dep2 = new Dep()
const dep3 = new Dep()

let result = 0
let condition = true

const e = effect(() => {
  dep1.track()
  if (condition) {
    dep2.track()
  } else {
    dep3.track()
  }
  result++
})

// 驗證 dep-list：e.deps → dep1 → dep2
console.log(e.deps?.dep === dep1)          // true
console.log(e.deps?.nextDep?.dep === dep2) // true
console.log(e.depsTail?.dep === dep2)      // true

// 驗證 sub-list
console.log(dep1.subs?.sub === e)          // true
console.log(dep2.subs?.sub === e)          // true
console.log(dep3.subs)                     // undefined（未被追蹤）

// 切換條件
condition = false
dep1.trigger()

// dep2 被清理，dep3 被添加
console.log(dep2.subs)                     // undefined（已清理）
console.log(dep3.subs?.sub === e)          // true

// ==================== 測試 2：多個 subscriber 共享 dep ====================

const sharedDep = new Dep()
let r1 = 0, r2 = 0

const e1 = effect(() => {
  sharedDep.track()
  r1 = 1
})

const e2 = effect(() => {
  sharedDep.track()
  r2 = 2
})

// sharedDep.subs 鏈：e2 ←→ e1（e2 是尾部）
console.log(sharedDep.subs?.sub === e2)          // true
console.log(sharedDep.subs?.prevSub?.sub === e1) // true
console.log(sharedDep.sc)                         // 2

// 停止 e1——O(1) 移除
e1.stop()
console.log(sharedDep.sc)                         // 1
console.log(sharedDep.subs?.sub === e2)           // true
```

## 與源碼對照

| 概念 | V06 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| Link 類別 | 基本的雙向鏈結指標 | `dep.ts:32-62` | 結構完全相同。一個 Link 同時活在兩條鏈結串列中，這是整個系統最精巧的資料結構設計 |
| Dep 類別 | subs + activeLink + sc | `dep.ts:67-205` | 源碼還有 map、key、computed 等欄位，用於 computed 的懶訂閱和 dep.map 的自動清理 |
| addSub | 尾部添加，O(1) | `dep.ts:207-232` | 源碼還處理 computed 的懶訂閱——首次被訂閱時才開始追蹤自身依賴 |
| removeSub | 指標調整，O(1) | `effect.ts:421-459` | 源碼還處理 computed 的取消訂閱和 dep.map 清理 |
| removeDep | 指標調整，O(1) | `effect.ts:461-471` | 邏輯完全相同 |
| prepareDeps | version=-1 + activeLink 設定 | `effect.ts:301-311` | 邏輯完全相同 |
| cleanupDeps | 遍歷 + removeSub/removeDep | `effect.ts:313-340` | 邏輯完全相同 |

> **關鍵洞察**：Link 的雙鏈設計不只是一個資料結構優化——它是後續所有高級功能的基礎。computed 的懶訂閱（V08）依賴 addSub/removeSub 的 O(1) 操作；批次更新（V07）從 subs 尾部遍歷以保證正確的通知順序；依賴清理從 O(n) splice 變成 O(1) 指標調整。一個 Link 節點同時活在兩條鏈結串列中——這是整個系統最精巧的資料結構設計，也是 Vue 3.4 響應式重寫的核心成果。

> **下一步**：目前每次 dep.trigger() 都會立即執行所有 subscriber。如果連續修改多個屬性（`state.a = 1; state.b = 2`），effect 會執行兩次。V07 將引入批次更新來解決這個問題。

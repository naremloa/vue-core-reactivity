# V05 - 依賴清理：版本標記的 mark-and-sweep

## 設計故事

### 從一個具體問題開始

延續購物車的場景。你做了一個「顯示模式切換」功能：使用者可以選擇看「詳細資訊」或「精簡資訊」。詳細模式會顯示商品名稱，精簡模式只顯示數量：

```ts
const showDetail = ref(true)
const name = ref('Alice')
const age = ref(25)

effect(() => {
  if (showDetail.value) {
    console.log(name.value)
  } else {
    console.log(age.value)
  }
})
```

到 V04 為止，每次 effect 執行時，依賴只會**累加**，永遠不會被移除。這在這種條件分支場景下會出問題：

第一次執行：effect 依賴 `showDetail` 和 `name`。

現在把 `showDetail` 設為 `false`：effect 重新執行，這次依賴 `showDetail` 和 `age`。

但 `name` 的依賴**沒有被清理**！如果之後修改 `name.value`，effect 會不必要地重新執行——即使它已經不關心 `name` 了。

更糟的是，想像一個 tab 切換場景：切換 100 次後，effect 會訂閱**所有曾經觸碰過的依賴**。這是一個嚴重的效能問題。

### 解決方案：版本標記法（Mark-and-Sweep）

靈感來自垃圾回收的 mark-and-sweep 演算法：

1. **Mark 階段**（`prepareDeps`）：effect 執行前，把所有現有依賴的 `version` 標記為 `-1`
2. **Sweep 階段**（`cleanupDeps`）：effect 執行後，仍然是 `-1` 的依賴就是「未被使用的」，移除它們

```
執行前：    dep-A(v=0)  dep-B(v=0)  dep-C(v=0)
            ↓            ↓            ↓
Mark:       dep-A(v=-1) dep-B(v=-1) dep-C(v=-1)

執行中：    讀取 A → A(v=同步)    讀取 C → C(v=同步)
                                   B 沒被讀取

Sweep:      dep-A(保留)  dep-B(移除!) dep-C(保留)
```

## 核心概念

### 為什麼不是每次都重建依賴？

一個更簡單的做法是：每次 effect 執行前，清除所有依賴，然後重新收集。但這有問題：

1. **效能**：如果大部分依賴不變，清除再重建是浪費的
2. **在移除的同時修改**：正在遍歷的集合被修改會出問題

版本標記法只清理「不再需要的」依賴，保留大部分不變的依賴——這在依賴穩定的常見場景下更高效。

### Link 的 version 欄位

在 V06 才會引入 Link 資料結構，但版本標記的邏輯需要先理解。目前我們用簡化的方式：每個 dep 和 effect 之間的關係用 version 來追蹤。

```
Dep                        Subscriber
┌──────────┐              ┌──────────────────┐
│ version  │  ←── 比對 ── │ 記錄的 version    │
│ (每次    │              │ （-1 = 待清理）   │
│  trigger │              │                  │
│  時遞增) │              └──────────────────┘
└──────────┘
```

### 流程圖

```
effect.run()
  │
  ├── prepareDeps(this)          ← Step 1: 標記所有現有依賴 version = -1
  │
  ├── activeSub = this
  │
  ├── this.fn()                  ← Step 2: 執行副作用函數
  │     │
  │     ├── dep.track()          ← 被使用的 dep：version 同步為 dep.version
  │     └── dep.track()
  │
  ├── cleanupDeps(this)          ← Step 3: 遍歷依賴，移除 version === -1 的
  │
  └── activeSub = prevEffect
```

## 注意事項

- 這個版本為了清晰，仍使用 `Set` 和陣列來管理依賴。V06 會改用雙向鏈結串列
- `version` 的設計在 Vue 源碼中不只用於清理——它還用於判斷「dep 的值是否改變」（computed 的 dirty check）
- `prepareDeps` 還會保存和設定 `dep.activeLink`，這是為了讓 `dep.track()` 能快速找到已存在的連結。簡化版省略了這個優化
- 每次 `dep.trigger()` 時 `version++`，所以 version 同時扮演「變更計數器」的角色

### 這個版本故意留下的問題

V05 解決了依賴清理，但仍有幾個已知問題等待後續版本處理：

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| subs 用 Set，deps 用陣列，操作是 O(n) | 依賴數量多時，查找現有連結需要遍歷，cleanupDeps 的 splice 也是 O(n) | V06（雙向鏈結串列） |
| 每次 trigger 立即執行所有 effect | 連續修改多個狀態會重複執行 effect，造成不必要的計算 | V07（批次更新） |
| 沒有 computed | 無法定義衍生值（如 total = price * quantity），只能用 effect 手動同步 | V08（computed） |

## 程式碼實作

```ts
// ==================== V05：依賴清理 ====================

enum EffectFlags {
  ACTIVE   = 1 << 0,
  RUNNING  = 1 << 1,
  TRACKING = 1 << 2,  // 新增：標記是否正在追蹤依賴
}

interface Subscriber {
  flags: EffectFlags
  deps: DepLink[]       // 此 subscriber 的所有依賴連結
  notify(): void
}

/**
 * DepLink：代表一個 dep 和 subscriber 之間的連結
 * 記錄了連結建立時的 dep.version
 */
interface DepLink {
  dep: Dep
  sub: Subscriber
  version: number       // 追蹤版本——-1 表示待清理
}

let activeSub: Subscriber | undefined = undefined
let shouldTrack = true

/**
 * Dep：帶版本號的依賴容器
 *
 * version 每次 trigger 時遞增，用途：
 * 1. 依賴清理的標記
 * 2. computed 的 dirty check（V08）
 */
class Dep {
  version = 0           // 依賴的版本號
  subs: Set<DepLink> = new Set()

  track(): DepLink | undefined {
    if (!activeSub || !shouldTrack) return

    // 檢查是否已經有這個 subscriber 的連結
    let existingLink: DepLink | undefined
    for (const link of this.subs) {
      if (link.sub === activeSub) {
        existingLink = link
        break
      }
    }

    if (existingLink) {
      // 已存在的連結——同步版本號（標記為「仍在使用」）
      if (existingLink.version === -1) {
        existingLink.version = this.version
      }
      return existingLink
    }

    // 新的連結
    const link: DepLink = {
      dep: this,
      sub: activeSub,
      version: this.version,
    }
    this.subs.add(link)
    activeSub.deps.push(link)
    return link
  }

  trigger(): void {
    this.version++
    this.notify()
  }

  notify(): void {
    for (const link of this.subs) {
      link.sub.notify()
    }
  }
}

/**
 * prepareDeps：標記所有現有依賴為 -1（mark 階段）
 *
 * 對應源碼：effect.ts:301-311
 */
function prepareDeps(sub: Subscriber): void {
  for (const link of sub.deps) {
    // 把所有依賴的版本標記為 -1
    // 如果 fn() 執行中這個 dep 被再次存取，version 會被更新
    // 如果沒被存取，version 會維持 -1 → cleanupDeps 中被移除
    link.version = -1
  }
}

/**
 * cleanupDeps：移除版本仍為 -1 的依賴（sweep 階段）
 *
 * 對應源碼：effect.ts:313-340
 */
function cleanupDeps(sub: Subscriber): void {
  // 從後往前遍歷，這樣刪除不會影響索引
  let i = sub.deps.length
  while (i--) {
    const link = sub.deps[i]
    if (link.version === -1) {
      // 這個依賴在這次執行中沒有被存取——移除
      link.dep.subs.delete(link)
      sub.deps.splice(i, 1)
    }
  }
}

/**
 * ReactiveEffect：加入依賴清理的版本
 */
class ReactiveEffect<T = any> implements Subscriber {
  flags: EffectFlags = EffectFlags.ACTIVE | EffectFlags.TRACKING
  deps: DepLink[] = []

  constructor(public fn: () => T) {}

  run(): T {
    if (!(this.flags & EffectFlags.ACTIVE)) {
      return this.fn()
    }

    this.flags |= EffectFlags.RUNNING
    // ★ 新增：執行前標記所有現有依賴
    prepareDeps(this)

    const prevEffect = activeSub
    const prevShouldTrack = shouldTrack
    activeSub = this
    shouldTrack = true

    try {
      return this.fn()
    } finally {
      // ★ 新增：執行後清理未使用的依賴
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
      // 停止時移除所有依賴
      for (const link of this.deps) {
        link.dep.subs.delete(link)
      }
      this.deps.length = 0
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
// ==================== 測試 1：條件分支的依賴清理 ====================

const toggle = new Dep()
const depA = new Dep()
const depB = new Dep()

let showA = true
let valueA = 'A'
let valueB = 'B'
let result = ''

effect(() => {
  toggle.track()
  if (showA) {
    depA.track()
    result = valueA
  } else {
    depB.track()
    result = valueB
  }
})

console.log(result)  // 'A'

// 切換到 showB
showA = false
toggle.trigger()
console.log(result)  // 'B'

// 現在修改 valueA——不應觸發 effect（depA 已被清理）
let callCount = 0
const origNotify = ReactiveEffect.prototype.notify
// 驗證：depA 的 subs 應該是空的
console.log(depA.subs.size)  // 0 — depA 的訂閱者已被清理！

// depB 的 subs 應該有 1 個
console.log(depB.subs.size)  // 1

// ==================== 測試 2：依賴穩定時不會重複添加 ====================

const stableDep = new Dep()
let stableValue = 0
let stableCallCount = 0

const stableEffect = effect(() => {
  stableCallCount++
  stableDep.track()
  return stableValue
})

console.log(stableCallCount)  // 1

// 第一次觸發
stableValue = 1
stableDep.trigger()
console.log(stableCallCount)  // 2
// subs 中仍然只有一個訂閱者（不會重複）
console.log(stableDep.subs.size)  // 1

// ==================== 測試 3：tab 切換場景 ====================

const tabIndex = new Dep()
const tabDeps = [new Dep(), new Dep(), new Dep()]
let currentTab = 0

effect(() => {
  tabIndex.track()
  tabDeps[currentTab].track()
})

// 切換 tab 多次
for (let i = 0; i < 3; i++) {
  currentTab = (currentTab + 1) % 3
  tabIndex.trigger()
}

// 驗證：只有當前 tab 的 dep 有訂閱者
tabDeps.forEach((dep, i) => {
  const hasSub = dep.subs.size > 0
  console.log(`tab${i} subscribed: ${hasSub}`)
})
// 只有 currentTab 對應的 dep 有訂閱者
```

## 與源碼對照

| 概念 | V05 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| prepareDeps | 遍歷 deps 陣列設 version=-1 | `effect.ts:301-311` — 還設定 `prevActiveLink` 和 `dep.activeLink` | 源碼額外管理 `activeLink` 快取，讓 `dep.track()` 能 O(1) 判斷是否已有連結 |
| cleanupDeps | 遍歷刪除 version=-1 的連結 | `effect.ts:313-340` — 用雙向鏈結串列的 O(1) 操作 | 源碼用鏈結串列指標操作取代陣列 splice，效能從 O(n) 變為 O(1) |
| DepLink | 簡單的 `{dep, sub, version}` | `dep.ts:32-62` — `Link` 類別，包含 4 個鏈結指標 | 源碼的 Link 同時串在 dep 和 sub 兩條鏈結串列上，支援雙向遍歷 |
| Dep.version | `version = 0`，trigger 時遞增 | `dep.ts:68` — 相同設計 | 行為一致，源碼的 version 還用於 computed 的 dirty check |
| TRACKING flag | `EffectFlags.TRACKING` | `effect.ts:44` — 用於判斷是否應該被加入訂閱 | 行為一致 |

> **關鍵洞察**：版本標記法的效率建立在一個重要的觀察上：**大多數依賴是穩定的**。在典型的 Vue 應用中，一個 effect 每次執行時讀取的依賴幾乎不變——只有條件分支切換時才會改變。mark-and-sweep 利用這個特性，只清理「不再需要的」依賴，保留穩定的大多數——這比「全部清除再重建」高效得多。`version` 欄位同時扮演「變更計數器」和「清理標記」兩個角色，這個雙重用途的設計會在 V08 的 computed dirty check 中再次展現其價值。

> **下一步**：目前使用 `Set` 和陣列管理依賴，`cleanupDeps` 需要遍歷和 splice，這是 O(n) 操作。V06 將引入雙向鏈結串列，讓添加和移除都變成 O(1)。

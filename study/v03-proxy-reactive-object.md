# V03 - Proxy 響應式物件：reactive + baseHandlers

## 設計故事

### 從 V02 遺留的問題開始

V02 解決了 effect 的錯誤恢復和生命週期管理。但還有一個每天都會讓你痛苦的問題：**手動呼叫 track/trigger**。

### 場景：一個表單，十個欄位

想像你正在開發一個使用者註冊表單，有姓名、Email、密碼、地址、電話等欄位。用 V02 來寫的話：

```ts
// V02：每個欄位都需要一個 Dep，每次讀寫都要手動呼叫
const nameDep = new Dep()
const emailDep = new Dep()
const passwordDep = new Dep()
const addressDep = new Dep()
const phoneDep = new Dep()
// ... 還有更多欄位

let name = ''
let email = ''
let password = ''
let address = ''
let phone = ''

// 副作用：驗證表單
effect(() => {
  nameDep.track()       // 手動追蹤 name
  emailDep.track()      // 手動追蹤 email
  passwordDep.track()   // 手動追蹤 password
  addressDep.track()    // 手動追蹤 address
  phoneDep.track()      // 手動追蹤 phone
  validateForm(name, email, password, address, phone)
})

// 副作用：即時預覽
effect(() => {
  nameDep.track()       // 又要追蹤一次
  emailDep.track()      // 又要追蹤一次
  renderPreview(name, email)
})

// 使用者修改 email
email = 'user@example.com'
emailDep.trigger()      // 別忘了手動觸發！
```

痛點一目了然：

1. **重複繁瑣**：每個欄位一個 Dep，每個 effect 裡要手動 track 每個用到的欄位
2. **容易遺漏**：新增了一個欄位卻忘了在某個 effect 裡 track？依賴靜靜地丟失，bug 很難追蹤
3. **修改值和觸發分離**：`email = 'xxx'` 和 `emailDep.trigger()` 是兩行獨立的程式碼。少寫一行就壞掉了

理想的 API 應該是：

```ts
// 理想：存取屬性自動追蹤，修改屬性自動觸發
const form = reactive({ name: '', email: '', password: '', address: '', phone: '' })

effect(() => {
  // 讀取 form.name 就自動追蹤 name
  // 讀取 form.email 就自動追蹤 email
  validateForm(form.name, form.email, form.password, form.address, form.phone)
})

form.email = 'user@example.com'  // 自動觸發！不需要手動呼叫任何東西
```

### 關鍵洞察：Proxy 可以攔截屬性存取

JavaScript 的 `Proxy` 讓這成為可能——它能攔截物件的**屬性存取**（get trap）和**屬性賦值**（set trap）。只要在 get 裡呼叫 `track()`、在 set 裡呼叫 `trigger()`，使用者就完全不需要知道 Dep 的存在。

但一個物件有多個屬性，每個屬性需要獨立追蹤。所以我們需要一個映射結構：

```
targetMap: WeakMap<target, Map<key, Dep>>

target → key → Dep
  obj  → 'a' → Dep (存放所有讀取 obj.a 的 effect)
       → 'b' → Dep (存放所有讀取 obj.b 的 effect)
```

用 `WeakMap` 是因為當物件被垃圾回收時，對應的依賴也應該被清理。

## 核心概念

### 資料結構

```
                    WeakMap              Map           Dep
┌──────────┐     ┌─────────┐      ┌──────────┐    ┌─────────┐
│ target   │ ──> │ depsMap  │ ──>  │ key:'a'  │ ──>│  subs   │
│ (原始物件)│     │          │      │ key:'b'  │    │  {eff1} │
└──────────┘     └─────────┘      └──────────┘    └─────────┘
     │
     │  Proxy
     v
┌──────────┐
│ reactive │  ← 使用者操作的是這個 Proxy
│  proxy   │
└──────────┘
```

### 在整體系統中的位置

```
┌──────────────────────────────────────────────────────────┐
│                    第三層：使用者的世界                      │
│   const state = reactive({ count: 0 })                   │
│   effect(() => console.log(state.count))                 │
│   state.count++  // 自動更新                              │
├──────────────────────────────────────────────────────────┤
│           第二層：攔截機制 ★ V03 的重點 ★                   │
│                                                          │
│  ┌────── V03 新增 ──────────────────────────┐            │
│  │ Proxy get trap → 自動呼叫 track()         │            │
│  │ Proxy set trap → 自動呼叫 trigger()       │            │
│  │ targetMap: target → key → Dep             │            │
│  │ reactiveMap: 快取同一物件的 Proxy          │            │
│  └───────────────────────────────────────────┘            │
├──────────────────────────────────────────────────────────┤
│              第一層：依賴追蹤引擎（從 V01/V02 沿用）         │
│   Dep / activeSub / ReactiveEffect / try/finally         │
└──────────────────────────────────────────────────────────┘
```

### Proxy 的 get/set trap

```
proxy.count
  → get trap 攔截
  → track(target, 'count')    // 查找或建立 dep，呼叫 dep.track()
  → return target.count

proxy.count = 1
  → set trap 攔截
  → target.count = 1           // 先修改原始物件
  → trigger(target, 'count')   // 查找 dep，呼叫 dep.trigger()
```

### reactive() 的快取

同一個物件呼叫 `reactive()` 兩次應該返回相同的 Proxy。所以需要一個 `reactiveMap` 做快取：

```ts
reactive(obj) === reactive(obj)  // true
```

## 注意事項

### 設計取捨

- `targetMap` 用 `WeakMap` 而非 `Map`，讓原始物件被 GC 時依賴也能被清理
- Proxy 的 `set` trap 只在值真正改變時才觸發（用 `Object.is()` 比較）
- 區分 `ADD`（新增屬性）和 `SET`（修改屬性）—— `ADD` 還需要觸發迭代相關的依賴
- 這個版本只處理普通物件和陣列，不處理 `Map`/`Set`（V10 處理）
- 深層響應是 V11 的主題——這裡只做一層攔截

### 這個版本故意留下的問題

V03 讓 track/trigger 自動化了，但還有很多問題故意沒處理：

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| 原始型別（number、string）無法被 Proxy 包裝 | 無法讓 `let count = 0` 變成響應式，只能包裝物件 | V04（ref） |
| 依賴只增不減 | 條件分支改變後，舊的依賴不會被移除，造成不必要的 effect 重新執行 | V05（版本標記清理） |
| subs 用 Set，操作是 O(n) | 依賴數量多時，新增/刪除訂閱者的效能下降 | V06（雙向鏈結串列） |
| 每次 trigger 立即執行所有 effect | 連續修改 `state.a = 1; state.b = 2` 會觸發兩次 effect，其實只需要一次 | V07（批次更新） |
| 只做一層攔截（shallow） | `reactive({ nested: { x: 1 } })` 的 `nested.x` 修改不會被追蹤 | V11（深層響應式） |

## 程式碼實作

```ts
// ==================== V03：Proxy 響應式物件 ====================

// ---------- 從 V02 沿用 ----------

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
    if (!(this.flags & EffectFlags.ACTIVE)) {
      return this.fn()
    }
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
    if (this.flags & EffectFlags.ACTIVE) {
      this.run()
    }
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

// ---------- V03 新增：Proxy 響應式 ----------

/**
 * targetMap：全域依賴映射
 * target → key → Dep
 *
 * 對應源碼：dep.ts:240
 */
type KeyToDepMap = Map<any, Dep>
const targetMap: WeakMap<object, KeyToDepMap> = new WeakMap()

/**
 * track：追蹤屬性存取
 * 在 targetMap 中查找（或建立）對應的 Dep，然後呼叫 dep.track()
 *
 * 對應源碼：dep.ts:262-284
 */
function track(target: object, key: unknown): void {
  if (!shouldTrack || !activeSub) return

  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (!dep) {
    depsMap.set(key, (dep = new Dep()))
  }
  dep.track()
}

/**
 * trigger：觸發屬性變更
 * 在 targetMap 中查找對應的 Dep，然後呼叫 dep.trigger()
 *
 * 對應源碼：dep.ts:294-389（完整版處理陣列、集合等情況）
 */
function trigger(target: object, key: unknown): void {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  const dep = depsMap.get(key)
  if (dep) {
    dep.trigger()
  }
}

/**
 * reactiveMap：快取，確保同一個原始物件只建立一個 Proxy
 *
 * 對應源碼：reactive.ts:26
 */
const reactiveMap: WeakMap<object, any> = new WeakMap()

/**
 * reactive：將普通物件轉為響應式 Proxy
 *
 * 對應源碼：reactive.ts:93-105
 */
function reactive<T extends object>(target: T): T {
  // 已經有快取的 Proxy，直接返回
  const existingProxy = reactiveMap.get(target)
  if (existingProxy) {
    return existingProxy
  }

  const proxy = new Proxy(target, {
    /**
     * get trap：攔截屬性讀取
     * 讀取屬性時自動呼叫 track()
     *
     * 對應源碼：baseHandlers.ts:55-134（BaseReactiveHandler.get）
     */
    get(target: object, key: string | symbol, receiver: object): any {
      const res = Reflect.get(target, key, receiver)
      // 追蹤依賴
      track(target, key)
      return res
    },

    /**
     * set trap：攔截屬性賦值
     * 修改屬性時自動呼叫 trigger()
     *
     * 對應源碼：baseHandlers.ts:142-192（MutableReactiveHandler.set）
     */
    set(
      target: Record<string | symbol, unknown>,
      key: string | symbol,
      value: unknown,
      receiver: object,
    ): boolean {
      const oldValue = target[key]
      const result = Reflect.set(target, key, value, receiver)
      // 只有值真正改變才觸發
      if (!Object.is(value, oldValue)) {
        trigger(target, key)
      }
      return result
    },

    /**
     * has trap：攔截 `key in obj` 操作
     *
     * 對應源碼：baseHandlers.ts:207-213
     */
    has(target: Record<string | symbol, unknown>, key: string | symbol): boolean {
      const result = Reflect.has(target, key)
      track(target, key)
      return result
    },

    /**
     * deleteProperty trap：攔截 `delete obj.key` 操作
     *
     * 對應源碼：baseHandlers.ts:194-205
     */
    deleteProperty(
      target: Record<string | symbol, unknown>,
      key: string | symbol,
    ): boolean {
      const hadKey = Object.prototype.hasOwnProperty.call(target, key)
      const result = Reflect.deleteProperty(target, key)
      if (result && hadKey) {
        trigger(target, key)
      }
      return result
    },
  })

  reactiveMap.set(target, proxy)
  return proxy as T
}
```

## 使用範例 / 測試

```ts
// ==================== 測試 1：基本的自動追蹤和觸發 ====================

const state = reactive({ count: 0, name: 'Vue' })
let renderResult = ''

effect(() => {
  // 讀取 state.count 和 state.name → 自動追蹤
  renderResult = `${state.name}: ${state.count}`
})

console.log(renderResult)  // "Vue: 0"

// 修改 state.count → 自動觸發 effect 重新執行
state.count++
console.log(renderResult)  // "Vue: 1"

// 修改 state.name → 同樣自動觸發
state.name = 'Vue 3'
console.log(renderResult)  // "Vue 3: 1"

// ==================== 測試 2：多個 effect 訂閱同一屬性 ====================

const shared = reactive({ value: 0 })
let result1 = 0
let result2 = 0

effect(() => { result1 = shared.value * 2 })
effect(() => { result2 = shared.value + 10 })

console.log(result1, result2)  // 0, 10

shared.value = 5
console.log(result1, result2)  // 10, 15

// ==================== 測試 3：同一物件 reactive 兩次返回相同 Proxy ====================

const raw = { x: 1 }
const p1 = reactive(raw)
const p2 = reactive(raw)
console.log(p1 === p2)  // true

// ==================== 測試 4：不相關的屬性不會觸發 effect ====================

const obj = reactive({ a: 1, b: 2 })
let onlyA = 0
let callCount = 0

effect(() => {
  callCount++
  onlyA = obj.a  // 只讀取 a
})

console.log(callCount)  // 1

obj.b = 100  // 修改 b——不應觸發
console.log(callCount)  // 1（不變）

obj.a = 10   // 修改 a——應觸發
console.log(callCount)  // 2
console.log(onlyA)       // 10
```

## 與源碼對照

| 概念 | V03 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| targetMap | `WeakMap<object, Map<key, Dep>>` | `dep.ts:240` — 相同結構 | 結構一致，源碼額外處理 debug 事件 |
| track() | 查找/建立 Dep 並呼叫 track | `dep.ts:262-284` — 還處理 debug 資訊 | 源碼會觸發 onTrack debug hook，並處理 activeLink 快取 |
| trigger() | 查找 Dep 並呼叫 trigger | `dep.ts:294-389` — 還處理陣列 length、集合迭代等 | 源碼區分 SET/ADD/DELETE/CLEAR 操作類型，對陣列 length 有特殊邏輯 |
| reactive() | 建立 Proxy | `reactive.ts:93-105` — 呼叫 `createReactiveObject` | 源碼透過 createReactiveObject 統一處理四種響應式類型（reactive/readonly/shallow 組合） |
| get trap | 簡單的 track + Reflect.get | `baseHandlers.ts:55-134` — 還處理 ReactiveFlags、陣列方法、ref 解包、深層響應 | 源碼在 get 中會遞迴呼叫 reactive() 實現深層響應，並對陣列方法做特殊攔截 |
| set trap | 比較後 trigger | `baseHandlers.ts:142-192` — 還處理 ref 賦值、ADD vs SET 區分 | 源碼區分新增屬性（ADD）和修改屬性（SET），ADD 還會觸發 ITERATE_KEY 依賴 |
| reactiveMap | WeakMap 快取 | `reactive.ts:26` — 四種 map（reactive/readonly/shallow 組合） | 源碼有 reactiveMap、readonlyMap、shallowReactiveMap、shallowReadonlyMap 四個快取 |
| createReactiveObject | 內聯在 reactive() 中 | `reactive.ts:261-302` — 獨立函數，處理型別判斷和集合型別 | 源碼會檢查是否為集合型別（Map/Set）並選用不同的 handler |

> **關鍵洞察**：Proxy 是使用者體驗和引擎機制之間的橋樑。在使用者的世界裡，只有普通的屬性讀寫；在引擎的世界裡，每次讀寫都在精確地維護依賴圖。Vue 3 選擇 Proxy 取代 Vue 2 的 `Object.defineProperty`，正是因為 Proxy 能攔截更多操作（新增屬性、刪除屬性、`in` 運算子），而 `defineProperty` 只能攔截已知屬性的讀寫。`targetMap` 使用 `WeakMap` 也是一個關鍵的設計選擇——當原始物件被垃圾回收時，對應的依賴會自動清理，不需要手動管理生命週期。

> **下一步**：Proxy 只能代理物件，不能代理原始值（number、string、boolean）。V04 將引入 `ref`——用 `.value` 包裝原始值，讓它們也能參與響應式系統。

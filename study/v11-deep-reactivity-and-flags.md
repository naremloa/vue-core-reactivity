# V11 - 深層響應與 ReactiveFlags

## 設計故事

### 從一個具體問題開始

到目前為止，`reactive` 只處理物件的第一層屬性。巢狀物件不是響應式的：

```ts
const state = reactive({
  user: { name: 'Alice' }
})

effect(() => {
  console.log(state.user.name)  // 只追蹤了 state.user，沒追蹤 user.name
})

state.user.name = 'Bob'  // ❌ effect 不會重新執行
```

我們需要「深層響應式」——讓巢狀物件也自動成為 Proxy。

### 預先包裝 vs 惰性包裝

一種做法是在 `reactive()` 時遞迴遍歷所有巢狀物件，全部包裝成 Proxy。但這有嚴重問題：

1. **效能浪費**：如果物件有 1000 個巢狀屬性，但只使用 3 個，其餘 997 個 Proxy 就白建了
2. **循環引用**：`obj.self = obj` 會導致無限遞迴
3. **啟動成本**：大物件的初始化會很慢

Vue 的解法是**惰性深層包裝（lazy deep wrapping）**——只有在屬性被讀取時，才把返回值包裝為 Proxy：

```ts
// baseHandlers.ts 的 get trap 中
get(target, key, receiver) {
  const res = Reflect.get(target, key, receiver)
  track(target, key)
  // ★ 惰性包裝：只有被存取的路徑才被 Proxy 包裝
  if (isObject(res)) {
    return isReadonly ? readonly(res) : reactive(res)
  }
  return res
}
```

### ReactiveFlags：虛擬屬性

Proxy 需要回答一些元資料問題：「這個物件是不是 reactive？」「它的原始值是什麼？」

Vue 使用「虛擬屬性」（實際上不存在於物件上的屬性）來實現：

```ts
proxy.__v_isReactive  // true（由 get trap 攔截返回）
proxy.__v_isReadonly  // false
proxy.__v_raw         // 原始物件（跳過 Proxy）
proxy.__v_isShallow   // false
proxy.__v_skip        // 是否跳過響應式包裝
```

這些屬性不真正存在於物件上——它們被 Proxy 的 get trap 攔截，根據 handler 的配置返回對應值。

## 核心概念

### 惰性深層包裝的流程

```
state = reactive({
  user: { name: 'Alice', address: { city: 'Taipei' } }
})

初始化：只有 state 是 Proxy
state.user       → get trap → track → reactive({ name: 'Alice', ... }) → 返回 Proxy
state.user.name  → get trap → track → 'Alice'（原始值，不需要包裝）
state.user.address → 尚未存取 → 不是 Proxy

→ 只有被存取的路徑才被包裝，未存取的路徑維持原始物件
```

### ReactiveFlags 的虛擬屬性機制

```
proxy.__v_isReactive
  → Proxy get trap 攔截
  → key === ReactiveFlags.IS_REACTIVE
  → return !this._isReadonly   (由 handler 的建構參數決定)

proxy.__v_raw
  → Proxy get trap 攔截
  → key === ReactiveFlags.RAW
  → return target             (返回原始物件)
```

### Handler 繼承體系

```
BaseReactiveHandler                    // get trap 的共用邏輯
  ├── MutableReactiveHandler           // set/delete/has/ownKeys trap
  │     ├── mutableHandlers            // reactive()
  │     └── shallowReactiveHandlers    // shallowReactive()
  └── ReadonlyReactiveHandler          // set/delete 拋警告
        ├── readonlyHandlers           // readonly()
        └── shallowReadonlyHandlers    // shallowReadonly()
```

### isReadonly x isShallow 的四種組合

| | deep | shallow |
|---|---|---|
| **mutable** | `reactive()` | `shallowReactive()` |
| **readonly** | `readonly()` | `shallowReadonly()` |

## 注意事項

- `reactive()` 對同一物件重複呼叫會返回相同的 Proxy（透過 `reactiveMap` 快取）
- `readonly(reactive(obj))` 是合法的——外層 readonly，內層 reactive 的追蹤仍然生效
- `isRef(res)` 的檢查在陣列的整數 key 時不做 ref 解包——`arr[0]` 返回 ref 本身而非 `.value`
- `toRaw()` 是遞迴的——`toRaw(readonly(reactive(obj)))` 返回原始 `obj`
- `markRaw()` 透過在物件上設定 `__v_skip = true` 來阻止包裝

### 這個版本故意留下的問題

V11 完成了深層響應式與 ReactiveFlags 虛擬屬性機制，但以下問題尚未處理：

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| 沒有 scope 管理機制來管理 effect 的生命週期 | 元件卸載時無法批次停止所有 effect，每個 effect 必須手動清理，容易造成記憶體洩漏 | V12（EffectScope） |
| 沒有 watch API | 使用者無法精確監聽特定狀態的變化、取得新舊值、或使用 deep/immediate 等選項 | V13（watch） |
| readonly/shallow 變體與集合型別的整合尚未完善 | `shallowReadonly(new Map())` 等邊界情境的行為可能不一致，缺乏統一的測試覆蓋 | V13（完整整合） |

## 程式碼實作

```ts
// ==================== V11：深層響應與 ReactiveFlags ====================

/**
 * ReactiveFlags：虛擬屬性的 key
 *
 * 對應源碼：constants.ts:17-24
 */
enum ReactiveFlags {
  SKIP       = '__v_skip',
  IS_REACTIVE = '__v_isReactive',
  IS_READONLY = '__v_isReadonly',
  IS_SHALLOW = '__v_isShallow',
  RAW        = '__v_raw',
  IS_REF     = '__v_isRef',
}

/**
 * BaseReactiveHandler：get trap 的基礎實作
 * 處理 ReactiveFlags 和深層包裝
 *
 * 對應源碼：baseHandlers.ts:49-135
 */
class BaseReactiveHandler {
  constructor(
    protected readonly _isReadonly = false,
    protected readonly _isShallow = false,
  ) {}

  get(target: any, key: string | symbol, receiver: object): any {
    // ===== ReactiveFlags 虛擬屬性 =====
    if (key === ReactiveFlags.SKIP) return target[ReactiveFlags.SKIP]
    if (key === ReactiveFlags.IS_REACTIVE) return !this._isReadonly
    if (key === ReactiveFlags.IS_READONLY) return this._isReadonly
    if (key === ReactiveFlags.IS_SHALLOW) return this._isShallow
    if (key === ReactiveFlags.RAW) {
      // 返回原始物件（需要驗證 receiver 是正確的 Proxy）
      return target
    }

    const targetIsArray = Array.isArray(target)

    // 陣列方法攔截（V09）
    if (!this._isReadonly && targetIsArray) {
      const fn = arrayInstrumentations[key]
      if (fn) return fn
    }

    const res = Reflect.get(target, key, receiver)

    // 不追蹤 Symbol 內建屬性和特殊 key
    if (typeof key === 'symbol') return res

    // readonly 不追蹤（因為值不會變）
    if (!this._isReadonly) {
      track(target, 'get', key)
    }

    // ★ shallow 模式：不做深層包裝
    if (this._isShallow) {
      return res
    }

    // ref 解包（陣列的整數 key 除外）
    if (isRef(res)) {
      return targetIsArray && isIntegerKey(key) ? res : res.value
    }

    // ★★★ 深層響應的核心：如果結果是物件，惰性包裝 ★★★
    if (isObject(res)) {
      return this._isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}

/**
 * MutableReactiveHandler：可變物件的 handler
 * 繼承 BaseReactiveHandler，加入 set/delete/has/ownKeys
 *
 * 對應源碼：baseHandlers.ts:137-223
 */
class MutableReactiveHandler extends BaseReactiveHandler {
  constructor(isShallow = false) {
    super(false, isShallow)
  }

  set(target: any, key: string | symbol, value: unknown, receiver: object): boolean {
    let oldValue = target[key]

    if (!this._isShallow) {
      // 深層模式：如果舊值是 ref 且新值不是 ref，直接修改 ref.value
      if (isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
      // 用 raw 值比較
      oldValue = toRaw(oldValue)
      value = toRaw(value)
    }

    const hadKey = Array.isArray(target)
      ? Number(key) < target.length
      : Object.prototype.hasOwnProperty.call(target, key)

    const result = Reflect.set(target, key, value, receiver)

    // 只在 receiver 是原始 Proxy 時觸發（避免原型鏈上的觸發）
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        trigger(target, 'add', key)
      } else if (!Object.is(value, oldValue)) {
        trigger(target, 'set', key)
      }
    }
    return result
  }

  deleteProperty(target: any, key: string | symbol): boolean {
    const hadKey = Object.prototype.hasOwnProperty.call(target, key)
    const result = Reflect.deleteProperty(target, key)
    if (result && hadKey) {
      trigger(target, 'delete', key)
    }
    return result
  }

  has(target: any, key: string | symbol): boolean {
    const result = Reflect.has(target, key)
    track(target, 'has', key)
    return result
  }

  ownKeys(target: any): (string | symbol)[] {
    track(target, 'iterate', Array.isArray(target) ? 'length' : ITERATE_KEY)
    return Reflect.ownKeys(target)
  }
}

/**
 * ReadonlyReactiveHandler：唯讀物件的 handler
 * set 和 deleteProperty 都拋出警告
 *
 * 對應源碼：baseHandlers.ts:225-249
 */
class ReadonlyReactiveHandler extends BaseReactiveHandler {
  constructor(isShallow = false) {
    super(true, isShallow)
  }

  set(target: object, key: string | symbol): boolean {
    console.warn(`Set operation on key "${String(key)}" failed: target is readonly.`)
    return true
  }

  deleteProperty(target: object, key: string | symbol): boolean {
    console.warn(`Delete operation on key "${String(key)}" failed: target is readonly.`)
    return true
  }
}

// 四種 handler 實例
const mutableHandlers = new MutableReactiveHandler()
const readonlyHandlers = new ReadonlyReactiveHandler()
const shallowReactiveHandlers = new MutableReactiveHandler(true)
const shallowReadonlyHandlers = new ReadonlyReactiveHandler(true)

// 四種 Proxy 快取
const reactiveMap = new WeakMap()
const readonlyMap = new WeakMap()
const shallowReactiveMap = new WeakMap()
const shallowReadonlyMap = new WeakMap()

/**
 * createReactiveObject：統一建立 Proxy 的工廠函數
 *
 * 對應源碼：reactive.ts:261-302
 */
function createReactiveObject(
  target: any,
  isReadonly: boolean,
  baseHandlers: any,
  collectionHandlers: any,
  proxyMap: WeakMap<any, any>,
) {
  if (!isObject(target)) return target

  // 已經是 Proxy → 直接返回（readonly(reactive(obj)) 除外）
  if (target[ReactiveFlags.RAW] && !(isReadonly && target[ReactiveFlags.IS_REACTIVE])) {
    return target
  }

  // 已有快取
  const existingProxy = proxyMap.get(target)
  if (existingProxy) return existingProxy

  // 判斷型別：普通物件/陣列 vs 集合
  const rawType = Object.prototype.toString.call(target).slice(8, -1)
  const isCollection = ['Map', 'Set', 'WeakMap', 'WeakSet'].includes(rawType)

  // 不可擴展的物件或標記為 skip 的物件不做代理
  if (target[ReactiveFlags.SKIP] || !Object.isExtensible(target)) {
    return target
  }

  const proxy = new Proxy(
    target,
    isCollection ? collectionHandlers : baseHandlers,
  )
  proxyMap.set(target, proxy)
  return proxy
}

function reactive(target: any) {
  if (isReadonly(target)) return target
  return createReactiveObject(target, false, mutableHandlers, mutableCollectionHandlers, reactiveMap)
}

function readonly(target: any) {
  return createReactiveObject(target, true, readonlyHandlers, readonlyCollectionHandlers, readonlyMap)
}

function shallowReactive(target: any) {
  return createReactiveObject(target, false, shallowReactiveHandlers, shallowCollectionHandlers, shallowReactiveMap)
}

function shallowReadonly(target: any) {
  return createReactiveObject(target, true, shallowReadonlyHandlers, shallowReadonlyCollectionHandlers, shallowReadonlyMap)
}

// ===== 工具函數 =====

function isReactive(value: unknown): boolean {
  if (isReadonly(value)) {
    return isReactive((value as any)[ReactiveFlags.RAW])
  }
  return !!(value && (value as any)[ReactiveFlags.IS_REACTIVE])
}

function isReadonly(value: unknown): boolean {
  return !!(value && (value as any)[ReactiveFlags.IS_READONLY])
}

function isShallow(value: unknown): boolean {
  return !!(value && (value as any)[ReactiveFlags.IS_SHALLOW])
}

function isProxy(value: any): boolean {
  return value ? !!value[ReactiveFlags.RAW] : false
}

function toRaw<T>(observed: T): T {
  const raw = observed && (observed as any)[ReactiveFlags.RAW]
  return raw ? toRaw(raw) : observed
}

function markRaw<T extends object>(value: T): T {
  if (Object.isExtensible(value)) {
    Object.defineProperty(value, ReactiveFlags.SKIP, { value: true })
  }
  return value
}
```

## 使用範例 / 測試

```ts
// ==================== 測試 1：深層響應 ====================

const state = reactive({
  user: {
    name: 'Alice',
    address: { city: 'Taipei' }
  }
})

let result = ''
effect(() => {
  result = `${state.user.name} in ${state.user.address.city}`
})

console.log(result)                // 'Alice in Taipei'

state.user.name = 'Bob'
console.log(result)                // 'Bob in Taipei'

state.user.address.city = 'Tokyo'
console.log(result)                // 'Bob in Tokyo'

// ==================== 測試 2：ReactiveFlags ====================

const obj = reactive({ x: 1 })
console.log(obj.__v_isReactive)    // true — 虛擬屬性
console.log(obj.__v_isReadonly)    // false
console.log(obj.__v_raw)          // { x: 1 } — 原始物件

const ro = readonly(obj)
console.log(ro.__v_isReadonly)     // true
console.log(ro.__v_isReactive)    // false

// ==================== 測試 3：shallowReactive ====================

const shallow = shallowReactive({
  nested: { count: 0 }
})

let shallowResult = 0
let callCount = 0

effect(() => {
  callCount++
  shallowResult = shallow.nested.count
})

// shallow.nested 的修改不追蹤
shallow.nested.count = 10
console.log(callCount)             // 1（沒觸發）

// 替換整個 nested 會觸發
shallow.nested = { count: 99 }
console.log(callCount)             // 2
console.log(shallowResult)        // 99

// ==================== 測試 4：toRaw / markRaw ====================

const raw = { id: 1 }
const proxy = reactive(raw)
console.log(toRaw(proxy) === raw)  // true

const marked = markRaw({ skip: true })
const wrapped = reactive(marked)
console.log(wrapped === marked)    // true — 不會被代理

// ==================== 測試 5：惰性包裝驗證 ====================

const big = reactive({
  a: { value: 1 },
  b: { value: 2 },  // 從未存取
  c: { value: 3 },  // 從未存取
})

// 只有 big.a 被存取，b 和 c 的物件不會被包裝為 Proxy
effect(() => { void big.a.value })
// b 和 c 的值仍然是原始物件（直到被存取）
```

## 與源碼對照

| 概念 | V11 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| ReactiveFlags | enum 定義 | `constants.ts:17-24` — 相同 | 邏輯一致，無重大差異 |
| BaseReactiveHandler.get | 虛擬屬性 + 深層包裝 | `baseHandlers.ts:55-134` — 還處理 `builtInSymbols`、`isNonTrackableKeys` | 源碼過濾了 `Symbol.iterator` 等內建 Symbol 和 `__proto__` 等特殊 key 的追蹤，V11 簡化為僅跳過所有 Symbol |
| MutableReactiveHandler.set | ref 解包 + ADD/SET 區分 | `baseHandlers.ts:142-192` — 相同邏輯 | 源碼在開發模式下會對 shallow reactive 做額外的警告檢查，V11 省略開發模式邏輯 |
| ReadonlyReactiveHandler | set/delete 拋警告 | `baseHandlers.ts:225-249` — 相同 | 邏輯一致，無重大差異 |
| createReactiveObject | 工廠函數 | `reactive.ts:261-302` — 相同 | 源碼使用 `getTargetType()` 細分目標型別（INVALID/COMMON/COLLECTION），V11 簡化為字串比對 |
| 四種 handler | 4 個實例 | `baseHandlers.ts:251-264` — 相同 | 邏輯一致，無重大差異 |
| 四種 Map | WeakMap 快取 | `reactive.ts:26-35` — 相同 | 邏輯一致，無重大差異 |
| toRaw | 遞迴取 raw | `reactive.ts:387-390` — 相同 | 邏輯一致，無重大差異 |
| markRaw | 設定 __v_skip | `reactive.ts:416-421` — 相同 | 邏輯一致，無重大差異 |
| 深層包裝 | `isObject(res) ? reactive(res) : res` | `baseHandlers.ts:126-131` — 相同 | 源碼額外檢查 `target[ReactiveFlags.RAW]` 的 receiver 驗證，V11 省略此驗證 |

> **關鍵洞察**：深層響應是惰性的——只有被存取的路徑才會建立 Proxy。這是一個典型的「按需計算」設計，避免了大物件的啟動成本。惰性包裝和 ReactiveFlags 虛擬屬性背後是同一個設計哲學：**不污染原始物件**。Proxy 攔截層完全獨立於原始資料——原始物件上不會被添加任何額外屬性。`toRaw()` 能隨時取回乾淨的原始物件，`isReactive()` 透過虛擬屬性判斷而非物件上的標記。這讓 Vue 的響應式系統與外部資料（API 回傳、第三方庫物件）能無縫整合。

> **下一步**：V12 將介紹 EffectScope——如何把多個 effect 組織成一個群組，實現批次停止（如元件卸載時）。

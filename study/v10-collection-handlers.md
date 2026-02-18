# V10 - 集合型別處理（Map/Set/WeakMap/WeakSet）

## 設計故事

### 從一個具體問題開始

V03 的 Proxy `set` trap 能攔截屬性賦值：

```ts
obj.key = value  // → set trap 被觸發 ✓
```

但 `Map` 和 `Set` 的修改是透過**方法呼叫**：

```ts
map.set('key', value)  // → set trap 不會觸發！
                       // 因為 .set() 是方法呼叫，不是屬性賦值
```

Proxy 的 `set` trap 只在 `obj.prop = value` 時觸發。`map.set()` 在 Proxy 看來只是一次「讀取 `.set` 屬性」（觸發 get trap）+ 一次函數呼叫。

### 解決方案：方法攔截

既然 Proxy 只能攔截到「讀取 `.set` 屬性」，那我們就在 get trap 中攔截——返回一個**替代方法**，這個方法內部手動呼叫 `track` 和 `trigger`：

```ts
// Proxy 的 get trap
get(target, key) {
  if (key === 'set') {
    return function(k, v) {
      // 手動觸發
      target.set(k, v)
      trigger(target, 'set', k)
    }
  }
}
```

### this 指向問題

另一個問題：`Map` 的原生方法需要 `this` 指向真正的 `Map` 實例，而非 Proxy：

```ts
const map = reactive(new Map())
map.get('key')  // ❌ TypeError: Method Map.prototype.get called on incompatible receiver
```

因為 `this` 指向 Proxy，而 Map 的內部 slot `[[MapData]]` 存在於原始 Map 上。解決方法是攔截方法時，在原始 `target` 上呼叫。

### 雙重 key 查找

當 key 本身是 reactive 時，需要同時追蹤原始 key 和 raw key：

```ts
const key = reactive({ id: 1 })
map.set(key, 'hello')
map.get(key)     // 追蹤 key
map.get(toRaw(key))  // 也應該追蹤 rawKey
```

## 核心概念

### 攔截策略對比

```
普通物件（baseHandlers）          集合（collectionHandlers）
┌─────────────────────┐         ┌──────────────────────────┐
│ get trap → track     │         │ get trap → 返回攔截方法    │
│ set trap → trigger   │         │   .get()  → track + 原始方法│
│ has trap → track     │         │   .set()  → 原始方法 + trigger│
│ delete trap → trigger│         │   .has()  → track + 原始方法│
└─────────────────────┘         │   .delete()→ 原始方法 + trigger│
                                │   .size    → track(ITERATE_KEY)│
                                │   .forEach → track + 包裝回調│
                                └──────────────────────────┘
```

### 追蹤的 key 種類

```
操作                    追蹤/觸發的 key
─────────────────────────────────────
map.get(key)           track(GET, key) + track(GET, rawKey)
map.has(key)           track(HAS, key) + track(HAS, rawKey)
map.set(key, val)      trigger(ADD/SET, key)
map.delete(key)        trigger(DELETE, key)
map.size               track(ITERATE, ITERATE_KEY)
map.forEach(cb)        track(ITERATE, ITERATE_KEY)
map.keys()             track(ITERATE, MAP_KEY_ITERATE_KEY)
map.clear()            trigger(CLEAR)
set.add(val)           trigger(ADD, val)
```

注意 `map.keys()` 使用 `MAP_KEY_ITERATE_KEY` 而非 `ITERATE_KEY`——這是因為 `Map.set()` 修改 value 時應該觸發 `forEach`/`values`/`entries` 但不應該觸發 `keys`（key 沒變）。

## 注意事項

- 集合的 Proxy 只用 `get` trap——不需要 `set`/`has`/`deleteProperty` trap
- `toRaw(value)` 在 `set.add()` 和 `map.set()` 中用於存儲——確保內部不存儲 Proxy 引用
- `readonly` 版本的集合方法會阻止 `add`/`set`/`delete`/`clear` 操作
- `shallow` 版本不會對返回值做 `toReactive` 包裝

### 這個版本故意留下的問題

V10 完成了集合型別的攔截策略，但以下問題尚未處理：

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| 從 Map/Set 取出的巢狀物件不是響應式的 | `map.get('user').name = 'Bob'` 不會觸發 effect 重新執行，因為返回值沒有被深層包裝 | V11（惰性深層包裝） |
| 沒有 scope 管理機制 | 元件卸載時無法批次停止所有 effect，造成記憶體洩漏 | V12（EffectScope） |
| 沒有 watch API | 使用者無法精確監聽特定狀態的變化並取得新舊值 | V13（watch） |

## 程式碼實作

```ts
// ==================== V10：集合型別處理 ====================

// 特殊追蹤 key
const ITERATE_KEY = Symbol('iterate')
const MAP_KEY_ITERATE_KEY = Symbol('Map keys iterate')

/**
 * createInstrumentations：為集合建立攔截方法
 *
 * 對應源碼：collectionHandlers.ts:96-261
 */
function createInstrumentations(
  isReadonly: boolean,
  isShallow: boolean,
): Record<string | symbol, Function | number> {
  const wrap = isShallow
    ? (v: unknown) => v                    // shallow：不包裝
    : isReadonly
      ? (v: unknown) => toReadonly(v)      // readonly：包裝為 readonly
      : (v: unknown) => toReactive(v)     // 預設：包裝為 reactive

  const instrumentations: Record<string | symbol, any> = {
    /**
     * Map.get(key)：追蹤 key 和 rawKey
     *
     * 對應源碼：collectionHandlers.ts:101-124
     */
    get(this: any, key: unknown) {
      const target = this[ReactiveFlags.RAW]
      const rawTarget = toRaw(target)
      const rawKey = toRaw(key)

      if (!isReadonly) {
        // 追蹤兩個 key（可能不同）
        if (key !== rawKey) {
          track(rawTarget, 'get', key)
        }
        track(rawTarget, 'get', rawKey)
      }

      const { has } = Reflect.getPrototypeOf(rawTarget)
      if (has.call(rawTarget, key)) {
        return wrap(target.get(key))
      } else if (has.call(rawTarget, rawKey)) {
        return wrap(target.get(rawKey))
      } else if (target !== rawTarget) {
        // readonly(reactive(Map)) 的情況
        target.get(key)
      }
    },

    /**
     * size：追蹤 ITERATE_KEY
     */
    get size() {
      const target = (this as any)[ReactiveFlags.RAW]
      !isReadonly && track(toRaw(target), 'iterate', ITERATE_KEY)
      return target.size
    },

    /**
     * has(key)：追蹤 key 和 rawKey
     */
    has(this: any, key: unknown): boolean {
      const target = this[ReactiveFlags.RAW]
      const rawTarget = toRaw(target)
      const rawKey = toRaw(key)
      if (!isReadonly) {
        if (key !== rawKey) track(rawTarget, 'has', key)
        track(rawTarget, 'has', rawKey)
      }
      return key === rawKey
        ? target.has(key)
        : target.has(key) || target.has(rawKey)
    },

    /**
     * forEach(cb)：追蹤 ITERATE_KEY，包裝回調參數
     */
    forEach(this: any, callback: Function, thisArg?: unknown) {
      const observed = this
      const target = observed[ReactiveFlags.RAW]
      const rawTarget = toRaw(target)
      !isReadonly && track(rawTarget, 'iterate', ITERATE_KEY)
      return target.forEach((value: unknown, key: unknown) => {
        return callback.call(thisArg, wrap(value), wrap(key), observed)
      })
    },
  }

  // ===== 寫入方法 =====
  if (isReadonly) {
    // readonly：所有寫入操作都拋警告
    instrumentations.add = readonlyMethod('add')
    instrumentations.set = readonlyMethod('set')
    instrumentations.delete = readonlyMethod('delete')
    instrumentations.clear = readonlyMethod('clear')
  } else {
    /**
     * Set.add(value)
     *
     * 對應源碼：collectionHandlers.ts:169-181
     */
    instrumentations.add = function (this: any, value: unknown) {
      if (!isShallow) value = toRaw(value)
      const target = toRaw(this)
      const proto = Reflect.getPrototypeOf(target)
      const hadKey = proto.has.call(target, value)
      if (!hadKey) {
        target.add(value)
        trigger(target, 'add', value)
      }
      return this
    }

    /**
     * Map.set(key, value)
     *
     * 對應源碼：collectionHandlers.ts:182-205
     */
    instrumentations.set = function (this: any, key: unknown, value: unknown) {
      if (!isShallow) value = toRaw(value)
      const target = toRaw(this)
      const { has, get } = Reflect.getPrototypeOf(target)
      let hadKey = has.call(target, key)
      if (!hadKey) {
        key = toRaw(key)
        hadKey = has.call(target, key)
      }
      const oldValue = get.call(target, key)
      target.set(key, value)
      if (!hadKey) {
        trigger(target, 'add', key)
      } else if (!Object.is(value, oldValue)) {
        trigger(target, 'set', key)
      }
      return this
    }

    /**
     * delete(key)
     *
     * 對應源碼：collectionHandlers.ts:206-224
     */
    instrumentations.delete = function (this: any, key: unknown) {
      const target = toRaw(this)
      const { has } = Reflect.getPrototypeOf(target)
      let hadKey = has.call(target, key)
      if (!hadKey) {
        key = toRaw(key)
        hadKey = has.call(target, key)
      }
      const result = target.delete(key)
      if (hadKey) {
        trigger(target, 'delete', key)
      }
      return result
    }

    /**
     * clear()
     *
     * 對應源碼：collectionHandlers.ts:225-246
     */
    instrumentations.clear = function (this: any) {
      const target = toRaw(this)
      const hadItems = target.size !== 0
      const result = target.clear()
      if (hadItems) {
        trigger(target, 'clear')
      }
      return result
    }
  }

  // ===== 迭代器方法 =====
  const iteratorMethods = ['keys', 'values', 'entries', Symbol.iterator] as const
  iteratorMethods.forEach(method => {
    instrumentations[method] = createIterableMethod(method, isReadonly, isShallow)
  })

  return instrumentations
}

/**
 * createIterableMethod：為迭代器方法建立攔截
 * keys() 追蹤 MAP_KEY_ITERATE_KEY
 * 其他迭代方法追蹤 ITERATE_KEY
 *
 * 對應源碼：collectionHandlers.ts:33-75
 */
function createIterableMethod(
  method: string | symbol,
  isReadonly: boolean,
  isShallow: boolean,
) {
  return function (this: any, ...args: unknown[]) {
    const target = this[ReactiveFlags.RAW]
    const rawTarget = toRaw(target)
    const targetIsMap = rawTarget instanceof Map
    const isPair = method === 'entries' || (method === Symbol.iterator && targetIsMap)
    const isKeyOnly = method === 'keys' && targetIsMap
    const innerIterator = target[method](...args)
    const wrap = isShallow
      ? (v: unknown) => v
      : isReadonly
        ? toReadonly
        : toReactive

    if (!isReadonly) {
      track(rawTarget, 'iterate', isKeyOnly ? MAP_KEY_ITERATE_KEY : ITERATE_KEY)
    }

    return {
      next() {
        const { value, done } = innerIterator.next()
        return done
          ? { value, done }
          : {
              value: isPair ? [wrap(value[0]), wrap(value[1])] : wrap(value),
              done,
            }
      },
      [Symbol.iterator]() { return this },
    }
  }
}

function readonlyMethod(type: string): Function {
  return function () {
    console.warn(`${type} operation failed: target is readonly.`)
    return type === 'delete' ? false : this
  }
}

/**
 * collectionHandlers：集合的 Proxy handler
 * 只需要 get trap——所有攔截都透過方法替換實現
 *
 * 對應源碼：collectionHandlers.ts:263-304
 */
function createCollectionHandlers(isReadonly: boolean, isShallow: boolean) {
  const instrumentations = createInstrumentations(isReadonly, isShallow)

  return {
    get(target: any, key: string | symbol, receiver: any) {
      // ReactiveFlags 處理
      if (key === '__v_isReactive') return !isReadonly
      if (key === '__v_isReadonly') return isReadonly
      if (key === '__v_raw') return target

      // 如果 key 在 instrumentations 中且 target 上也有這個 key
      // → 返回攔截版本
      return Reflect.get(
        key in instrumentations && key in target
          ? instrumentations
          : target,
        key,
        receiver,
      )
    },
  }
}

const mutableCollectionHandlers = createCollectionHandlers(false, false)
const readonlyCollectionHandlers = createCollectionHandlers(true, false)
const shallowCollectionHandlers = createCollectionHandlers(false, true)
```

## 使用範例 / 測試

```ts
// ==================== 測試 1：基本的 reactive Map ====================

const map = reactive(new Map())
let result = ''

effect(() => {
  result = map.get('name') ?? 'unknown'
})

console.log(result)          // 'unknown'

map.set('name', 'Vue')
console.log(result)          // 'Vue'

map.set('name', 'Vue 3')
console.log(result)          // 'Vue 3'

// ==================== 測試 2：Map.size 追蹤 ====================

const sizedMap = reactive(new Map())
let size = 0

effect(() => {
  size = sizedMap.size
})

console.log(size)            // 0

sizedMap.set('a', 1)
console.log(size)            // 1

sizedMap.set('b', 2)
console.log(size)            // 2

sizedMap.delete('a')
console.log(size)            // 1

// ==================== 測試 3：reactive Set ====================

const set = reactive(new Set())
let hasItem = false

effect(() => {
  hasItem = set.has('hello')
})

console.log(hasItem)         // false

set.add('hello')
console.log(hasItem)         // true

set.delete('hello')
console.log(hasItem)         // false

// ==================== 測試 4：Map.keys() vs Map.values() ====================

const m = reactive(new Map([['a', 1]]))
let keysCount = 0
let valuesCount = 0

effect(() => {
  keysCount++
  for (const _ of m.keys()) { /* iterate */ }
})

effect(() => {
  valuesCount++
  for (const _ of m.values()) { /* iterate */ }
})

// 修改 value（不改 key）
m.set('a', 2)
// keys effect 不應觸發（key 沒變）
// values effect 應觸發
// 這依賴於 trigger 中區分 SET 觸發 ITERATE_KEY 但不觸發 MAP_KEY_ITERATE_KEY
```

## 與源碼對照

| 概念 | V10 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| createInstrumentations | 建立攔截方法 | `collectionHandlers.ts:96-261` — 相同結構 | 源碼中為四種組合（mutable/readonly/shallow/shallowReadonly）各建立一組，V10 透過參數動態產生 |
| get (Map) | 雙 key 追蹤 | `collectionHandlers.ts:101-124` — 還處理 `readonly(reactive(Map))` | 源碼額外處理 `readonly(reactive(Map))` 的 track 觸發情境，V10 僅簡化處理 |
| set (Map) | ADD/SET 區分 | `collectionHandlers.ts:182-205` — 還有 `checkIdentityKeys` | 源碼在開發模式下會檢查 reactive key 與 raw key 同時存在的衝突 |
| createIterableMethod | 迭代器攔截 | `collectionHandlers.ts:33-75` — 相同 | 邏輯一致，無重大差異 |
| collectionHandlers | 只有 get trap | `collectionHandlers.ts:263-304` — 4 種組合 | 源碼產生 4 個獨立的 handler 物件，V10 用工廠函數統一建立 |
| ITERATE_KEY | 迭代追蹤用 | `dep.ts:242-244` — 相同 | 邏輯一致，無重大差異 |
| MAP_KEY_ITERATE_KEY | keys() 專用 | `dep.ts:245-247` — 相同 | 邏輯一致，無重大差異 |
| trigger 中的集合邏輯 | 省略 | `dep.ts:359-384` | 源碼中 SET 觸發 ITERATE_KEY（Map），ADD/DELETE 觸發 ITERATE_KEY + MAP_KEY_ITERATE_KEY，V10 未實作此邏輯 |

> **關鍵洞察**：不同的資料結構需要不同的攔截策略。普通物件用 property interception（get/set/has/deleteProperty trap），集合用 method interception（只用 get trap 替換方法）。這是因為集合的內部 slot 必須由原生 `this` 操作。

> **下一步**：V11 將處理深層響應——為什麼 `reactive` 的巢狀物件也是響應式的？答案是惰性深層包裝 + ReactiveFlags 虛擬屬性。

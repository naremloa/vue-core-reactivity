# V13 - Readonly/Shallow 變體與 Watch API

## 設計故事

### Part 1：Readonly 和 Shallow 的四種組合

Vue 的響應式系統有四種模式，由兩個維度組合而成：

| | **deep** | **shallow** |
|---|---|---|
| **mutable** | `reactive()` | `shallowReactive()` |
| **readonly** | `readonly()` | `shallowReadonly()` |

為什麼需要這些變體？

- **reactive**：最常見。元件的 data/state。
- **readonly**：Props。子元件不應修改從父元件傳來的資料，但仍需追蹤以便在父元件更新時重新渲染。
- **shallowReactive**：大物件只需追蹤頂層屬性。例如一個巨大的 JSON 結構，只有根屬性需要響應式。
- **shallowReadonly**：Props 的 shallow 版本。不深層唯讀化——巢狀物件可以被修改。

這四種模式共用同一套 handler 體系，只是透過 `_isReadonly` 和 `_isShallow` 兩個旗標控制行為。

### Part 2：Watch 是什麼？

`watch` 看起來是一個獨立的 API，但它的核心其實就是一個**帶 scheduler 的 ReactiveEffect**：

```
watch(source, callback)

→ 建立 ReactiveEffect(getter)
→ effect.scheduler = () => job()
→ job() 中比較新舊值 → 呼叫 callback(newValue, oldValue)
```

`watch` 的複雜性不在於響應式機制（那已經在前面的版本中解決了），而在於：
1. **source 正規化**：source 可以是 ref、reactive、getter、或它們的陣列
2. **deep traverse**：深度監聽時需要遍歷所有屬性以建立依賴
3. **callback timing**：`immediate` / `once` 的控制
4. **cleanup**：上次 callback 的清理函數

## 核心概念

### Handler 體系的繼承

```
BaseReactiveHandler (get trap)
│   _isReadonly: boolean
│   _isShallow: boolean
│
├── MutableReactiveHandler (set/delete/has/ownKeys)
│     constructor(isShallow = false)
│     ├── mutableHandlers = new MutableReactiveHandler()
│     └── shallowReactiveHandlers = new MutableReactiveHandler(true)
│
└── ReadonlyReactiveHandler (set/delete → warn)
      constructor(isShallow = false)
      ├── readonlyHandlers = new ReadonlyReactiveHandler()
      └── shallowReadonlyHandlers = new ReadonlyReactiveHandler(true)
```

### Watch 的 Source 正規化

```ts
// watch 接受多種 source 形式，統一轉為 getter 函數：

watch(ref(1), cb)
  → getter = () => source.value

watch(reactive(obj), cb)
  → getter = () => traverse(source)

watch(() => state.count, cb)
  → getter = source  （已經是 getter）

watch([ref(1), () => state.count], cb)
  → getter = () => [source[0].value, source[1]()]
```

### traverse：深度遍歷

`watch` 的 `deep` 選項透過 `traverse` 函數實現——遞迴存取所有屬性以建立依賴：

```
traverse(obj, depth)
  ├── isRef → traverse(obj.value)
  ├── isArray → traverse(每個元素)
  ├── isSet/isMap → traverse(每個值)
  └── isPlainObject → traverse(每個屬性)
```

### Watch 的排程

```
依賴變更
  → effect.scheduler 被呼叫
    → scheduler(job, false)  // 外部提供的 scheduler
    或
    → job()                  // 直接呼叫

job():
  → effect.run()            // 重新執行 getter，取得新值
  → 比較新舊值
  → 如果變了 → callback(newValue, oldValue)
```

## 注意事項

- `readonly` 的 `get` trap **不追蹤依賴**——因為值不可能被修改，追蹤沒有意義。但 `readonly(reactive(obj))` 是特例——外層 readonly 但內層 reactive 仍然可以被修改
- `watch` 的 `scheduler` 選項讓使用者控制「何時執行 job」。Vue runtime 用它實作 `flush: 'post'`（DOM 更新後）和 `flush: 'pre'`（DOM 更新前）
- `watch` 的 cleanup 機制使用 `WeakMap` 管理——key 是 effect 實例，value 是清理函數陣列
- `once` 選項透過包裝 callback 實現——在第一次呼叫後自動 `stop()`

### 與真實源碼的差距

作為整個系列的最後一個版本，V13 涵蓋了 Vue 響應式系統的核心 API，但與真正的生產級源碼之間仍有以下差距：

| 問題 | 說明 |
|------|------|
| 沒有 SSR 支援 | Vue 源碼中有 `serverPrefetch`、`ssrContext` 等 SSR 相關邏輯，響應式系統在伺服器端需要不同的行為（例如不追蹤依賴） |
| 沒有 DevTools 整合 | 源碼中透過 `devtools.ts` 提供響應式追蹤的視覺化除錯，包含 `onTrack` / `onTrigger` 等 debug hook |
| 沒有 `__DEV__` 警告 | 源碼中大量使用 `__DEV__` 條件來提供開發環境的友善警告（例如修改 readonly、在 computed 中寫入等），生產環境會被 tree-shaking 移除 |
| 簡化的錯誤處理 | 源碼有完整的 `callWithErrorHandling` / `callWithAsyncErrorHandling` 機制，搭配 `app.config.errorHandler` 做統一錯誤管理 |
| 沒有 `triggerRef` | 源碼提供 `triggerRef()` 讓使用者手動觸發 ref 的更新，用於搭配 `shallowRef` 在深層修改後手動通知 |
| 沒有 `customRef` | 源碼提供 `customRef()` 讓使用者自訂 track/trigger 時機，用於防抖搜尋等進階場景 |
| 沒有 `effectScope` 的 `warn` 參數 | 源碼中 `onScopeDispose` 有 `failSilently` 選項，在沒有活躍 scope 時控制是否顯示警告 |
| 沒有 `watchEffect` / `watchPostEffect` / `watchSyncEffect` | 源碼提供這些便捷 API 作為 `watch` 的特化版本，分別對應不同的 flush timing |
| 沒有 `toRaw` / `markRaw` 的完整處理 | 簡化版的 `toRaw` / `markRaw` 缺少邊界情況的處理，例如巢狀 raw 物件的遞迴解包 |
| 沒有 `isProxy` / `isReadonly` / `isShallow` 的完整實作 | 源碼中這些工具函數會處理多層 proxy 包裝的判斷邏輯 |

## 程式碼實作

```ts
// ==================== V13：Readonly/Shallow 變體與 Watch ====================

// ===== Part 1：Handler 體系已在 V11 完整實作 =====
// 這裡聚焦於 Watch API

// ---------- traverse：深度遍歷 ----------

/**
 * traverse：遞迴存取物件的所有屬性
 * 用於 watch 的 deep 選項——透過存取建立依賴
 *
 * 對應源碼：watch.ts:331-367
 */
function traverse(
  value: unknown,
  depth: number = Infinity,
  seen?: Map<unknown, number>,
): unknown {
  if (depth <= 0 || !isObject(value) || (value as any)[ReactiveFlags.SKIP]) {
    return value
  }

  seen = seen || new Map()
  // 防止循環引用的無限遞迴
  if ((seen.get(value) || 0) >= depth) {
    return value
  }
  seen.set(value, depth)
  depth--

  if (isRef(value)) {
    traverse(value.value, depth, seen)
  } else if (Array.isArray(value)) {
    for (let i = 0; i < value.length; i++) {
      traverse(value[i], depth, seen)
    }
  } else if (value instanceof Set || value instanceof Map) {
    value.forEach((v: any) => {
      traverse(v, depth, seen)
    })
  } else if (isPlainObject(value)) {
    for (const key in value) {
      traverse((value as any)[key], depth, seen)
    }
    for (const key of Object.getOwnPropertySymbols(value)) {
      if (Object.prototype.propertyIsEnumerable.call(value, key)) {
        traverse((value as any)[key], depth, seen)
      }
    }
  }
  return value
}

function isPlainObject(val: unknown): val is object {
  return Object.prototype.toString.call(val) === '[object Object]'
}

// ---------- Watch API ----------

type WatchSource<T = any> = (() => T) | { value: T } // Ref 或 getter
type WatchCallback<V = any, OV = any> = (
  value: V,
  oldValue: OV,
  onCleanup: (fn: () => void) => void,
) => any

interface WatchOptions {
  immediate?: boolean
  deep?: boolean | number
  once?: boolean
  scheduler?: (job: () => void, isFirstRun: boolean) => void
}

interface WatchHandle {
  (): void           // 呼叫即停止
  pause: () => void
  resume: () => void
  stop: () => void
}

// 初始值標記
const INITIAL_WATCHER_VALUE = {}

// 清理函數管理
const cleanupMap: WeakMap<ReactiveEffect, (() => void)[]> = new WeakMap()
let activeWatcher: ReactiveEffect | undefined = undefined

/**
 * onWatcherCleanup：註冊 watcher 的清理函數
 *
 * 對應源碼：watch.ts:103-118
 */
function onWatcherCleanup(
  cleanupFn: () => void,
  owner: ReactiveEffect | undefined = activeWatcher,
): void {
  if (owner) {
    let cleanups = cleanupMap.get(owner)
    if (!cleanups) cleanupMap.set(owner, (cleanups = []))
    cleanups.push(cleanupFn)
  }
}

/**
 * watch：建立一個響應式監聽器
 *
 * 核心：watch 就是一個帶 scheduler 的 ReactiveEffect
 *
 * 對應源碼：watch.ts:120-329
 */
function watch(
  source: WatchSource | WatchSource[] | (() => void) | object,
  cb?: WatchCallback | null,
  options: WatchOptions = {},
): WatchHandle {
  const { immediate, deep, once, scheduler } = options

  // ===== Step 1：Source 正規化 → 統一為 getter 函數 =====

  let effect: ReactiveEffect
  let getter: () => any
  let cleanup: (() => void) | undefined
  let boundCleanup: (fn: () => void) => void
  let forceTrigger = false
  let isMultiSource = false

  if (isRef(source)) {
    // ref → getter 讀取 .value
    getter = () => (source as any).value
    forceTrigger = isShallow(source)
  } else if (isReactive(source)) {
    // reactive → getter 遍歷所有屬性
    getter = () => {
      if (deep) return source
      if (deep === false || deep === 0) return traverse(source, 1)
      return traverse(source)
    }
    forceTrigger = true
  } else if (Array.isArray(source)) {
    // 陣列 → 對每個 source 正規化
    isMultiSource = true
    forceTrigger = source.some(s => isReactive(s) || isShallow(s))
    getter = () =>
      source.map(s => {
        if (isRef(s)) return s.value
        if (isReactive(s)) return traverse(s)
        if (typeof s === 'function') return s()
      })
  } else if (typeof source === 'function') {
    if (cb) {
      // watch(getter, callback) → getter 直接使用
      getter = source as () => any
    } else {
      // watchEffect(fn) → 無 callback
      getter = () => {
        if (cleanup) {
          pauseTracking()
          try { cleanup() } finally { resetTracking() }
        }
        const currentEffect = activeWatcher
        activeWatcher = effect
        try {
          return (source as Function)(boundCleanup)
        } finally {
          activeWatcher = currentEffect
        }
      }
    }
  } else {
    getter = () => {}
  }

  // deep 選項：用 traverse 包裝 getter
  if (cb && deep) {
    const baseGetter = getter
    const depth = deep === true ? Infinity : (deep as number)
    getter = () => traverse(baseGetter(), depth)
  }

  // ===== Step 2：建立停止和清理機制 =====

  const scope = getCurrentScope()
  const watchHandle: WatchHandle = (() => {
    effect.stop()
    if (scope && scope.active) {
      const idx = scope.effects.indexOf(effect)
      if (idx > -1) scope.effects.splice(idx, 1)
    }
  }) as WatchHandle

  // once：只執行一次
  if (once && cb) {
    const _cb = cb
    cb = (...args) => {
      _cb(...args)
      watchHandle()
    }
  }

  // ===== Step 3：建立 job 和 effect =====

  let oldValue: any = isMultiSource
    ? new Array((source as []).length).fill(INITIAL_WATCHER_VALUE)
    : INITIAL_WATCHER_VALUE

  /**
   * job：watch 的核心排程函數
   * 比較新舊值，呼叫 callback
   */
  const job = (immediateFirstRun?: boolean) => {
    if (!(effect.flags & EffectFlags.ACTIVE) || (!effect.dirty && !immediateFirstRun)) {
      return
    }

    if (cb) {
      // watch(source, cb) 模式
      const newValue = effect.run()
      if (
        deep ||
        forceTrigger ||
        (isMultiSource
          ? (newValue as any[]).some((v, i) => !Object.is(v, oldValue[i]))
          : !Object.is(newValue, oldValue))
      ) {
        // 執行清理函數
        if (cleanup) cleanup()

        const currentWatcher = activeWatcher
        activeWatcher = effect
        try {
          const args = [
            newValue,
            oldValue === INITIAL_WATCHER_VALUE ? undefined : oldValue,
            boundCleanup,
          ]
          oldValue = newValue
          cb!(...(args as [any, any, any]))
        } finally {
          activeWatcher = currentWatcher
        }
      }
    } else {
      // watchEffect 模式
      effect.run()
    }
  }

  // ===== Step 4：建立 ReactiveEffect =====

  effect = new ReactiveEffect(getter)

  // scheduler：控制 job 何時執行
  effect.scheduler = scheduler
    ? () => scheduler(job, false)
    : job

  // 清理函數綁定
  boundCleanup = (fn: () => void) => onWatcherCleanup(fn, effect)

  cleanup = effect.onStop = () => {
    const cleanups = cleanupMap.get(effect)
    if (cleanups) {
      for (const fn of cleanups) fn()
      cleanupMap.delete(effect)
    }
  }

  // ===== Step 5：初始執行 =====

  if (cb) {
    if (immediate) {
      job(true)
    } else {
      oldValue = effect.run()
    }
  } else if (scheduler) {
    scheduler(job.bind(null, true), true)
  } else {
    effect.run()
  }

  // ===== Step 6：返回 handle =====

  watchHandle.pause = effect.pause.bind(effect)
  watchHandle.resume = effect.resume.bind(effect)
  watchHandle.stop = watchHandle

  return watchHandle
}
```

## 使用範例 / 測試

```ts
// ==================== 測試 1：watch ref ====================

const count = ref(0)
const changes: string[] = []

const stop = watch(count, (newVal, oldVal) => {
  changes.push(`${oldVal} → ${newVal}`)
})

count.value = 1
// 假設同步 scheduler
console.log(changes)         // ['0 → 1']

count.value = 2
console.log(changes)         // ['0 → 1', '1 → 2']

stop()
count.value = 3
console.log(changes)         // ['0 → 1', '1 → 2']（已停止）

// ==================== 測試 2：watch reactive（deep）====================

const state = reactive({ user: { name: 'Alice' } })
let lastChange = ''

watch(state, (newVal) => {
  lastChange = `name is ${newVal.user.name}`
}, { deep: true })

state.user.name = 'Bob'
console.log(lastChange)      // 'name is Bob'

// ==================== 測試 3：watch getter ====================

const a = ref(1)
const b = ref(2)

watch(
  () => a.value + b.value,
  (sum, oldSum) => {
    console.log(`sum: ${oldSum} → ${sum}`)
  }
)

a.value = 10                 // 'sum: 3 → 12'

// ==================== 測試 4：immediate ====================

const name = ref('Vue')
const logs: string[] = []

watch(name, (val) => {
  logs.push(val)
}, { immediate: true })

console.log(logs)            // ['Vue']（立即執行）

name.value = 'React'
console.log(logs)            // ['Vue', 'React']

// ==================== 測試 5：once ====================

const val = ref(0)
let onceResult = ''

watch(val, (v) => {
  onceResult = `value: ${v}`
}, { once: true })

val.value = 1
console.log(onceResult)      // 'value: 1'

val.value = 2
console.log(onceResult)      // 'value: 1'（只執行了一次）

// ==================== 測試 6：cleanup ====================

const id = ref(1)

watch(id, (newId, oldId, onCleanup) => {
  const controller = new AbortController()
  // 模擬 fetch
  console.log(`fetching id=${newId}`)

  onCleanup(() => {
    console.log(`cancelled fetch for id=${newId}`)
    controller.abort()
  })
})

id.value = 2    // 'cancelled fetch for id=1', 'fetching id=2'
id.value = 3    // 'cancelled fetch for id=2', 'fetching id=3'

// ==================== 測試 7：readonly 行為 ====================

const original = reactive({ count: 0 })
const ro = readonly(original)

console.log(isReadonly(ro))   // true
console.log(isReactive(ro))  // false

// 修改 readonly 會失敗
ro.count = 10  // 警告：Set operation on key "count" failed: target is readonly.

// 但修改 original 仍然有效
effect(() => {
  console.log(ro.count)
})
original.count = 5  // console: 5（readonly 的 get trap 會追蹤 reactive 的來源）
```

## 與源碼對照

| 概念 | V13 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| watch() | 核心邏輯 | `watch.ts:120-329` | 源碼還有 `call` 選項用於統一錯誤處理，以及 `flush` 選項控制執行時機 |
| source 正規化 | ref/reactive/getter/array | `watch.ts:153-205` | 相同結構，源碼額外處理了 `computed` 作為 source 的情況 |
| traverse() | 遞迴遍歷 | `watch.ts:331-367` | 相同，均使用 Map 防止循環引用 |
| job | 比較新舊值 + 呼叫 callback | `watch.ts:233-280` | 源碼的 job 會透過 `callWithAsyncErrorHandling` 包裝 callback 呼叫 |
| watchHandle | stop + pause + resume | `watch.ts:214-219, 324-328` | 相同介面，源碼額外處理了 scope 的自動註冊和移除 |
| onWatcherCleanup | WeakMap 管理清理函數 | `watch.ts:103-118` | 相同機制 |
| INITIAL_WATCHER_VALUE | `{}` 標記初始值 | `watch.ts:78` | 相同 |
| handler 體系 | 4 種 handler 實例 | `baseHandlers.ts:251-264` | 相同繼承結構，源碼有更多 `__DEV__` 下的警告邏輯 |
| readonly/shallow 組合 | 4 種 API | `reactive.ts:93-259` | 源碼使用獨立的 WeakMap 快取每種變體的 proxy 實例 |

> **最終洞察**：`watch` 就是一個帶 scheduler 的 `ReactiveEffect`。整個 Vue 的響應式系統，從最底層的 Dep/Link，到 ReactiveEffect，到 computed，到 watch，再到元件的 render effect——都建立在同一套觀察者模式之上。理解了 V01 的核心思想，你就理解了整個系統的根基。

---

## 全系列總結

```
V01  Dep + effect + activeSub         → 觀察者模式的核心
V02  ReactiveEffect class + try/finally → 可靠的生命週期管理
V03  Proxy + targetMap                → 自動化追蹤和觸發
V04  RefImpl + .value                 → 原始值的響應式
V05  version + prepareDeps/cleanupDeps → 依賴的 mark-and-sweep
V06  Link 雙向鏈結串列                 → O(1) 依賴操作
V07  batchDepth + batchedSub          → 防止級聯更新
V08  ComputedRefImpl + DIRTY/isDirty  → 惰性求值的中繼節點
V09  arrayInstrumentations            → 陣列方法的特殊處理
V10  collectionHandlers               → Map/Set 的方法攔截
V11  lazy deep wrapping + ReactiveFlags → 惰性深層響應
V12  EffectScope                      → 元件級的生命週期管理
V13  readonly/shallow + watch         → 完整的 API 體系
```

每一層都在回答一個設計問題。這些問題不是憑空產生的——它們是在解決前一層遺留的限制時自然浮現的。理解「為什麼」比理解「怎麼做」更重要。

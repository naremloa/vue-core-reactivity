# V12 - EffectScope 與生命週期管理

## 設計故事

到目前為止，每個 effect 都是獨立的。要停止它，需要持有它的引用：

```ts
const e1 = effect(() => { ... })
const e2 = effect(() => { ... })
const e3 = effect(() => { ... })

// 元件卸載時，需要逐一停止
e1.stop()
e2.stop()
e3.stop()
```

在 Vue 的元件系統中，一個元件可能建立了數十個 effect（computed、watcher、render effect）。元件卸載時，需要一次清理所有這些響應式行為。

逐一管理顯然不實際。我們需要一個「容器」來統一管理：

```ts
const scope = effectScope()

scope.run(() => {
  // 在 scope 中建立的所有 effect 都會自動註冊
  effect(() => { ... })
  effect(() => { ... })
  const c = computed(() => { ... })
})

// 一次停止所有
scope.stop()
```

### EffectScope 的樹狀結構

就像 DOM 樹一樣，scope 也形成一棵樹——它鏡像了 Vue 的元件樹：

```
Root Scope
  ├── App Scope
  │     ├── Header Scope
  │     └── Content Scope
  │           ├── Sidebar Scope
  │           └── Main Scope
  └── Modal Scope
```

停止一個 scope 會遞迴停止它的所有子 scope——就像卸載一個元件會卸載它的所有子元件。

## 核心概念

### EffectScope 的結構

```
┌────────────────────────────┐
│       EffectScope          │
│                            │
│  effects: ReactiveEffect[] │ ← 收集的所有 effect
│  cleanups: (() => void)[]  │ ← 清理函數（onScopeDispose）
│  scopes: EffectScope[]     │ ← 子 scope
│  parent: EffectScope       │ ← 父 scope
│  _active: boolean          │ ← 是否活躍
│  _isPaused: boolean        │ ← 是否暫停
│                            │
│  run(fn): T                │ ← 在此 scope 中執行函數
│  stop(): void              │ ← 停止所有 effect 和子 scope
│  pause(): void             │ ← 暫停所有 effect
│  resume(): void            │ ← 恢復所有 effect
└────────────────────────────┘
```

### 自動註冊機制

```
activeEffectScope = scopeA

new ReactiveEffect(fn)
  → constructor 中檢查 activeEffectScope
  → activeEffectScope.effects.push(this)
  → effect 自動註冊到 scopeA

scopeA.stop()
  → 遍歷 effects，逐一 stop()
  → 遍歷 scopes，逐一 stop()
```

### detached scope

預設情況下，scope 會自動成為 `activeEffectScope` 的子 scope。但有時你需要一個「獨立的」scope——不受父 scope 停止的影響：

```ts
const parent = effectScope()
parent.run(() => {
  const detached = effectScope(true)  // detached = true
  detached.run(() => {
    effect(() => { ... })  // 不受 parent.stop() 影響
  })
})

parent.stop()  // 不會停止 detached scope
```

### O(1) 子 scope 移除

停止子 scope 時，需要從父 scope 的 `scopes` 陣列中移除。Vue 用了一個巧妙的 O(1) 方法：

```ts
// 不是 splice（O(n)），而是用末尾元素替換被刪除的元素
const last = parent.scopes.pop()
if (last !== this) {
  parent.scopes[this.index] = last
  last.index = this.index
}
```

## 注意事項

- `scope.run(fn)` 在執行期間設定 `activeEffectScope = this`，結束後還原。這和 `effect.run()` 設定 `activeSub` 的模式完全一致
- `on()` / `off()` 是內部方法，用於元件的 setup 函數——setup 執行期間需要「暫時切換到元件的 scope」
- `on()` 使用引用計數（`_on`），支援多次呼叫
- 已停止的 scope 呼叫 `run()` 會靜默失敗（返回 `undefined`）
- `onScopeDispose` 註冊的清理函數在 `scope.stop()` 時執行，類似 React 的 cleanup

### 這個版本故意留下的問題

| 問題 | 後果 | 哪個版本解決 |
|------|------|------------|
| 沒有 watch API | 使用者只能用底層的 `effect` + 手動比較新舊值來監聽變化，缺少高階抽象來監聽特定的響應式來源並取得變更前後的值 | V13（watch） |
| 沒有 readonly/shallow 變體的完整體系 | 無法表達「只讀」或「淺層響應」的語意，例如 props 應該是 readonly 但目前沒有機制阻止子元件修改 | V13（readonly/shallow） |

## 程式碼實作

```ts
// ==================== V12：EffectScope ====================

// 全域指標：當前活躍的 scope
let activeEffectScope: EffectScope | undefined

/**
 * EffectScope：effect 的生命週期管理容器
 *
 * 對應源碼：effectScope.ts:6-165
 */
class EffectScope {
  private _active = true
  private _isPaused = false
  private _on = 0

  // 收集的 effect
  effects: ReactiveEffect[] = []
  // 清理函數
  cleanups: (() => void)[] = []
  // 子 scope
  scopes: EffectScope[] | undefined
  // 父 scope
  parent: EffectScope | undefined
  // 在父 scope.scopes 中的索引（用於 O(1) 移除）
  private index: number | undefined

  /**
   * @param detached - 是否脫離父 scope
   */
  constructor(public detached = false) {
    // 記錄父 scope
    this.parent = activeEffectScope
    // 非 detached → 自動註冊到父 scope
    if (!detached && activeEffectScope) {
      this.index =
        (activeEffectScope.scopes || (activeEffectScope.scopes = [])).push(this) - 1
    }
  }

  get active(): boolean {
    return this._active
  }

  /**
   * run：在此 scope 中執行函數
   * 執行期間 activeEffectScope 指向 this
   *
   * 對應源碼：effectScope.ts:95-107
   */
  run<T>(fn: () => T): T | undefined {
    if (this._active) {
      const currentEffectScope = activeEffectScope
      try {
        activeEffectScope = this
        return fn()
      } finally {
        activeEffectScope = currentEffectScope
      }
    }
  }

  /**
   * on：暫時切換到此 scope（元件 setup 用）
   * 支援多次呼叫（引用計數）
   *
   * 對應源碼：effectScope.ts:114-119
   */
  prevScope: EffectScope | undefined
  on(): void {
    if (++this._on === 1) {
      this.prevScope = activeEffectScope
      activeEffectScope = this
    }
  }

  /**
   * off：恢復前一個 scope
   *
   * 對應源碼：effectScope.ts:125-130
   */
  off(): void {
    if (this._on > 0 && --this._on === 0) {
      activeEffectScope = this.prevScope
      this.prevScope = undefined
    }
  }

  /**
   * pause：暫停此 scope 及所有子 scope 和 effect
   *
   * 對應源碼：effectScope.ts:60-73
   */
  pause(): void {
    if (this._active) {
      this._isPaused = true
      if (this.scopes) {
        for (let i = 0; i < this.scopes.length; i++) {
          this.scopes[i].pause()
        }
      }
      for (let i = 0; i < this.effects.length; i++) {
        this.effects[i].pause()
      }
    }
  }

  /**
   * resume：恢復此 scope 及所有子 scope 和 effect
   *
   * 對應源碼：effectScope.ts:78-93
   */
  resume(): void {
    if (this._active && this._isPaused) {
      this._isPaused = false
      if (this.scopes) {
        for (let i = 0; i < this.scopes.length; i++) {
          this.scopes[i].resume()
        }
      }
      for (let i = 0; i < this.effects.length; i++) {
        this.effects[i].resume()
      }
    }
  }

  /**
   * stop：停止此 scope 的所有 effect、子 scope、清理函數
   *
   * 對應源碼：effectScope.ts:132-164
   */
  stop(fromParent?: boolean): void {
    if (this._active) {
      this._active = false

      // 停止所有 effect
      for (let i = 0; i < this.effects.length; i++) {
        this.effects[i].stop()
      }
      this.effects.length = 0

      // 執行所有清理函數
      for (let i = 0; i < this.cleanups.length; i++) {
        this.cleanups[i]()
      }
      this.cleanups.length = 0

      // 遞迴停止子 scope
      if (this.scopes) {
        for (let i = 0; i < this.scopes.length; i++) {
          this.scopes[i].stop(true)
        }
        this.scopes.length = 0
      }

      // 從父 scope 中移除自己（O(1) 操作）
      if (!this.detached && this.parent && !fromParent) {
        const last = this.parent.scopes!.pop()!
        if (last !== this) {
          this.parent.scopes![this.index!] = last
          last.index = this.index!
        }
      }
      this.parent = undefined
    }
  }
}

/**
 * effectScope：建立一個 effect scope
 *
 * 對應源碼：effectScope.ts:176-178
 */
function effectScope(detached?: boolean): EffectScope {
  return new EffectScope(detached)
}

/**
 * getCurrentScope：取得當前活躍的 scope
 *
 * 對應源碼：effectScope.ts:185-187
 */
function getCurrentScope(): EffectScope | undefined {
  return activeEffectScope
}

/**
 * onScopeDispose：在當前 scope 註冊清理函數
 *
 * 對應源碼：effectScope.ts:196-205
 */
function onScopeDispose(fn: () => void): void {
  if (activeEffectScope) {
    activeEffectScope.cleanups.push(fn)
  }
}

// ★ ReactiveEffect 建構函數中的自動註冊邏輯
// 對應源碼：effect.ts:116-119
//
// constructor(public fn: () => T) {
//   if (activeEffectScope && activeEffectScope.active) {
//     activeEffectScope.effects.push(this)
//   }
// }
```

## 使用範例 / 測試

```ts
// ==================== 測試 1：基本的 scope 收集 ====================

const scope = effectScope()

let result1 = 0
let result2 = 0

const dep1 = new Dep()
const dep2 = new Dep()

scope.run(() => {
  effect(() => {
    dep1.track()
    result1 = 1
  })
  effect(() => {
    dep2.track()
    result2 = 2
  })
})

console.log(scope.effects.length)  // 2
console.log(result1, result2)      // 1, 2

// 一次停止所有
scope.stop()

dep1.trigger()
dep2.trigger()
console.log(result1, result2)      // 1, 2（不再響應）

// ==================== 測試 2：巢狀 scope ====================

const parent = effectScope()
let childStopped = false

parent.run(() => {
  const child = effectScope()
  child.run(() => {
    onScopeDispose(() => {
      childStopped = true
    })
    effect(() => { /* ... */ })
  })
})

// 停止父 scope → 子 scope 也被停止
parent.stop()
console.log(childStopped)          // true

// ==================== 測試 3：detached scope ====================

const parentScope = effectScope()
let detachedAlive = true

parentScope.run(() => {
  const detached = effectScope(true)  // detached!
  detached.run(() => {
    onScopeDispose(() => {
      detachedAlive = false
    })
  })
})

parentScope.stop()
console.log(detachedAlive)          // true — detached scope 不受影響

// ==================== 測試 4：onScopeDispose ====================

const s = effectScope()
const disposed: string[] = []

s.run(() => {
  onScopeDispose(() => disposed.push('cleanup 1'))
  onScopeDispose(() => disposed.push('cleanup 2'))
})

s.stop()
console.log(disposed)               // ['cleanup 1', 'cleanup 2']

// ==================== 測試 5：pause / resume ====================

const pausable = effectScope()
const dep = new Dep()
let val = 0

pausable.run(() => {
  effect(() => {
    dep.track()
    val++
  })
})

console.log(val)                    // 1

dep.trigger()
console.log(val)                    // 2

pausable.pause()
dep.trigger()
console.log(val)                    // 2（暫停中，不執行）

pausable.resume()
console.log(val)                    // 3（恢復時自動執行被暫停的更新）
```

## 與源碼對照

| 概念 | V12 簡化版 | Vue 源碼 | 差異說明 |
|------|-----------|---------|---------|
| EffectScope 類別 | 完整版 | `effectScope.ts:6-165` | 幾乎一致，簡化版省略了部分錯誤處理和 `__DEV__` 警告 |
| activeEffectScope | 全域指標 | `effectScope.ts:4` | 相同機制 |
| effectScope() | 工廠函數 | `effectScope.ts:176-178` | 相同 |
| getCurrentScope() | 取得當前 scope | `effectScope.ts:185-187` | 相同 |
| onScopeDispose() | 註冊清理函數 | `effectScope.ts:196-205` | 源碼有 `failSilently` 參數，簡化版省略 |
| scope.run() | try/finally 保護 | `effectScope.ts:95-107` | 相同 |
| scope.stop() | 遞迴停止 | `effectScope.ts:132-164` | 相同 |
| scope.on/off() | 元件 setup 用 | `effectScope.ts:114-130` | 引用計數機制相同 |
| O(1) 移除 | pop + swap | `effectScope.ts:156-161` | 相同技巧 |
| effect 自動註冊 | constructor 中 push | `effect.ts:116-119` | 相同 |

> **關鍵洞察**：EffectScope 鏡像了元件樹。停止一個 scope 等於「卸載一個元件的所有響應式行為」。它把「多對多」的管理問題簡化為「一對多」的樹狀結構。`activeEffectScope` 全域指標的設計和 V01 的 `activeSub` 如出一轍——都是利用「執行期間的全域狀態」來實現自動註冊。effect 在建構時自動加入當前 scope，就像 `dep.track()` 自動追蹤當前 `activeSub`。這個「隱式註冊」模式貫穿了整個響應式系統——理解了它，你就能預測任何新功能的註冊機制。

> **下一步**：V13 將介紹 readonly/shallow 變體的完整體系，以及 `watch` API 如何建立在 `ReactiveEffect` 之上。

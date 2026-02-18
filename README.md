# Vue Core Reactivity 源碼研讀

## 建議閱讀順序

1. 推薦先讀 [READING-GUIDE.md](READING-GUIDE.md)，了解這份材料的閱讀方式與每一版的結構
2. 進入 [study/](study/) 目錄，從 v01 開始依序閱讀；若對某主題已有概念，可直接跳到對應版本

## 源碼來源

本研讀材料基於 [vuejs/core/reactivity](https://github.com/vuejs/core/tree/09dec962ae9c2e2ad49e584cef5f416a0d4558ff/packages/reactivity)

## 目錄

以下為各版本涵蓋的主題：

| 版本 | 主題 | 說明 |
|------|------|------|
| [v01](study/v01-manual-dep-tracking.md) | 手動依賴追蹤 | Observer pattern 基礎：`Dep`、`activeSub`、`effect` |
| [v02](study/v02-reactive-effect-class.md) | ReactiveEffect 類別 | 從裸函式升級為 class，支援巢狀 effect 與錯誤恢復 |
| [v03](study/v03-proxy-reactive-object.md) | Proxy 響應式物件 | 透過 Proxy get/set 自動追蹤與觸發，`reactive()` 實作 |
| [v04](study/v04-ref-primitive-reactivity.md) | Ref：原始值響應式 | `.value` 包裝原始值，`RefImpl` 與自動深層響應 |
| [v05](study/v05-dependency-cleanup.md) | 依賴清理 | Mark-and-sweep 版本追蹤，解決條件分支的過期依賴 |
| [v06](study/v06-linked-list-optimization.md) | 雙向鏈結串列優化 | `Link` 節點取代 Set/Array，O(1) 新增與移除 |
| [v07](study/v07-batching-system.md) | 批次更新系統 | `batchDepth` 計數器與佇列，避免連鎖觸發 |
| [v08](study/v08-computed-lazy-evaluation.md) | Computed：惰性求值 | `ComputedRefImpl` 雙重身份、DIRTY flag、`globalVersion` 快速路徑 |
| [v09](study/v09-array-instrumentation.md) | 陣列方法攔截 | `pauseTracking()` 與 `arrayInstrumentations` |
| [v10](study/v10-collection-handlers.md) | Collection Handlers | Map/Set/WeakMap/WeakSet 的方法攔截與雙鍵查找 |
| [v11](study/v11-deep-reactivity-and-flags.md) | 深層響應與 ReactiveFlags | Handler 繼承體系、四種響應模式組合 |
| [v12](study/v12-effect-scope.md) | EffectScope 生命週期管理 | 作用域群組化、樹狀結構、`onScopeDispose()` |
| [v13](study/v13-readonly-shallow-and-watch.md) | Readonly/Shallow 與 Watch | 完整 handler 變體、`watch()` 實作與排程 |

## 其他源碼研讀

<!-- 未來新增其他 source-code study 的連結 -->

| 專案 | 說明 | 連結 |
|------|------|------|
| — | — | — |

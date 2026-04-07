# Architecture

## 全体構成

```
[Developer's Application]   ← 普通の React Web アプリ
        │
        │ import { TUIClient } from '@tangi/core'
        ▼
┌─────────────────────────────────────────┐
│  @tangi/core (Client SDK)               │
│  ├── TUIClient (public API)             │
│  ├── Event Emitter                      │
│  ├── State Store (Yjs Y.Map)            │
│  ├── Calibration Engine                 │
│  ├── Device Abstraction (getUserMedia)  │
│  └── Transport (ws / local)             │
├─────────────────────────────────────────┤
│  @tangi/detector-qr  @tangi/detector-aruco │ ← プラグイン
└─────────────────────────────────────────┘
        │ WebSocket（マルチデバイス時のみ）
        ▼
┌─────────────────────────────────────────┐
│  @tangi/server (Sync Server)            │
│  └── ws + y-websocket                   │
└─────────────────────────────────────────┘
```

## エッジとサーバーの分担

**エッジ（ブラウザ）で行うこと**

- カメラ映像取得（`getUserMedia`）
- マーカー検出（qr-scanner / js-aruco2）
- イベント生成（`enter` / `move` / `leave`）
- ローカル状態管理（Yjs `Y.Map`）

目標レイテンシ：1 フレームあたり **50ms 以下**

カメラ映像は毎フレーム数 MB と大きすぎてサーバーには送れない。検出をブラウザで完結させることで、物理オブジェクトを動かした瞬間にフィードバックが返せる。

**サーバー（Node.js）で行うこと**

- セッション（ルーム）管理
- デバイスレジストリ
- Yjs ドキュメントのグローバル状態マージ・ブロードキャスト

サーバーはマルチデバイス時のみ必要。シングルデバイスでは起動不要。

## 主要な設計判断

### 1. SDK のスコープをマーカー検出 + 同期に絞る

タッチパネルは OS 標準の HID として認識されブラウザの `PointerEvent` がそのまま動くため、SDK でラップする必要がない。SDK は「既存の Web アプリにタンジブル機能を足すレイヤー」と位置づける。

### 2. Yjs を状態同期に採用

CRDT による自動競合解決、軽量（~30KB）、エコシステム充実。Automerge は WASM バンドルが 800KB+ で重すぎる。`Y.Map` でマーカー ID をキーに位置・角度・状態を管理する構造が TUI シーングラフに自然に対応する。

### 3. ws を Socket.IO より優先

`y-websocket` が素の WebSocket プロトコル上で動くため、ws との統合がシームレス。Socket.IO は独自プロトコルで Yjs カスタムプロバイダの自作が必要になる。コードの美しさでは Socket.IO が勝るが、KISS 原則と Yjs 互換性を優先。

### 4. PartyKit を外してセルフホスト構成に

研究機関は閉域ネットワークやデータ主権の要件がある。クラウド依存ゼロが必須要件。

### 5. マーカー検出をプラグイン化

QR と ArUco を独立した npm パッケージとして分離。開発者が必要な検出器だけ選べる。

### 6. 4 点射影変換ベースのキャリブレーション

エンドユーザーでも実行可能な簡易さが要件。画面四隅にマーカーを置くだけで、3×3 の Homography 行列が計算でき、位置ずれ・スケール・回転・反転を一括補正できる。計算は純粋な TypeScript で実装可能（外部ライブラリ不要）。

## 技術スタック

| レイヤー       | 採用             | 理由                               |
| -------------- | ---------------- | ---------------------------------- |
| QR 検出        | qr-scanner       | 16KB、WebWorker、活発メンテ        |
| ArUco 検出     | js-aruco2        | 純粋 JS、複数辞書対応              |
| 状態同期       | Yjs              | CRDT、軽量、実績豊富               |
| WebSocket      | ws               | 最軽量、y-websocket と直接互換     |
| Yjs 同期       | y-websocket      | Yjs 公式プロバイダ、セルフホスト可 |
| モノレポ       | pnpm + Turborepo | 高速、ビルドキャッシュ             |
| バンドラー     | tsdown           | tsup 後継、Rolldown ベース         |
| バージョン管理 | Changesets       | モノレポ対応                       |
| テスト         | Vitest           | TypeScript ファースト              |
| Lint / Format  | Biome            | ESLint + Prettier の後継           |

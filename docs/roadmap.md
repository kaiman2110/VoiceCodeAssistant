# ロードマップ

## 現在のマイルストーン

### Phase 1: MVP (Milestone #1)
音声でClaude Codeに指示→結果表示+レイテンシ可視化

| Issue | タイトル | 優先度 | 依存 | 状態 |
|-------|---------|--------|------|------|
| #1 | プロジェクト初期化（モノレポ構成） | high | なし | open |
| #2 | 共有型定義（WebSocketプロトコル） | high | #1 | open |
| #3 | Node.js バックエンド基盤 | high | #1, #2 | open |
| #4 | Python STTサーバー | high | #1 | open |
| #5 | Node.js → STT接続 | high | #3, #4 | open |
| #6 | Agent SDK 統合 | high | #3 | open |
| #7 | パフォーマンス計測基盤 | medium | #2, #3 | open |
| #8 | WebSocket ハンドラ統合 | high | #5, #6, #7 | open |
| #9 | React 状態管理 | medium | #1, #2 | open |
| #10 | React WebSocket + 音声フック | medium | #9 | open |
| #11 | React UI コンポーネント | medium | #9, #10 | open |
| #12 | docker-compose | low | #8, #11 | open |

**クリティカルパス:** #1 → #2 → #3 → #5/#6 → #8
**並列実行可能:** #4 と #3 / #9 と #3

## 将来のマイルストーン候補
- **Phase 2: 実用化** — Diff表示、プロジェクト切替、会話履歴永続化
- **Phase 3: 拡張** — VOICEVOX連携、ウェイクワード、Godot連携
- **Phase 4: 発展** — マルチセッション、カスタム辞書、Electron化

## 先送り課題（Deferred）
<!-- スコープ外にした機能。理由・取り込み先候補MS・日付を記載 -->

## 更新履歴
- 2026-04-03: Phase 1 MVP マイルストーン作成、Issue #1-#12 登録

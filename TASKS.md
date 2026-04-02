# TASKS.md — Voice Code Assistant

## Phase 1: MVP

### Task 1: プロジェクト初期化
- [ ] ルート package.json 作成
- [ ] server/ 初期化（package.json, tsconfig.json, 依存インストール）
- [ ] client/ 初期化（Vite + React + TS + Tailwind v4）
- [ ] stt-server/ 初期化（requirements.txt, pip install）
- [ ] 各サービスの空エントリポイントで起動確認

### Task 2: 共有型定義
- [ ] server/src/types.ts — WebSocket プロトコル型
- [ ] client/src/types.ts — 同じ型をコピー

### Task 3: Node.js バックエンド基盤
- [ ] server/src/index.ts — Express + WebSocket 起動
- [ ] server/src/websocket.ts — メッセージハンドラ基盤
- [ ] CORS 設定、ヘルスチェック
- [ ] wscat で接続テスト

### Task 4: Python STT サーバー
- [ ] stt-server/main.py — FastAPI + WebSocket エンドポイント
- [ ] stt-server/transcriber.py — faster-whisper ラッパー
- [ ] stt-server/vad.py — Silero VAD ストリーミング判定
- [ ] stt-server/audio_buffer.py — PCM バッファ管理
- [ ] ヘルスチェック + モデルプリロード
- [ ] WebSocket 経由の文字起こしテスト

### Task 5: Node.js → STT 接続
- [ ] server/src/stt-client.ts — WebSocket クライアント
- [ ] audio_chunk 転送
- [ ] transcription / vad_status 受信コールバック
- [ ] 再接続ロジック

### Task 6: Agent SDK 統合
- [ ] server/src/agent.ts — executePrompt() 実装
- [ ] SDKMessage の型マッピング検証
- [ ] ストリーミングコールバック動作確認
- [ ] ツール許可レベル切替

### Task 7: パフォーマンス計測
- [ ] server/src/perf-tracker.ts — PerfTracker クラス
- [ ] 各区間タイムスタンプ記録
- [ ] JSONL ログ永続化
- [ ] finalize() → PerformanceMetrics 返却

### Task 8: WebSocket ハンドラ統合
- [ ] 全メッセージフローの結合
- [ ] recording_start → audio_chunk → transcription → Agent SDK → perf_metrics
- [ ] text_input からの直接実行パス
- [ ] 排他制御（Agent 実行中のリクエストキューイング）
- [ ] status メッセージ配信

### Task 9: React 状態管理
- [ ] client/src/stores/chatStore.ts — Zustand ストア
- [ ] client/src/stores/perfStore.ts — パフォーマンスストア

### Task 10: React WebSocket + 音声フック
- [ ] client/src/hooks/useWebSocket.ts — 接続管理 + メッセージディスパッチ
- [ ] client/src/hooks/useVoiceRecorder.ts — MediaRecorder + Push-to-Talk
- [ ] client/src/hooks/usePerfMetrics.ts — メトリクス統計計算

### Task 11: React UI コンポーネント
- [ ] App.tsx — 全体レイアウト（ダーク基調）
- [ ] ChatPanel.tsx — メッセージリスト + 入力エリア
- [ ] MessageList.tsx — user / assistant / tool メッセージ表示
- [ ] VoiceButton.tsx — Push-to-Talk（パルスアニメーション）
- [ ] TextInput.tsx — テキストフォールバック
- [ ] StatusBar.tsx — 接続状態 + レイテンシ表示 + 権限レベル切替

### Task 12: docker-compose（オプション）
- [ ] stt-server Dockerfile
- [ ] server Dockerfile
- [ ] client Dockerfile
- [ ] docker-compose.yml
- [ ] GPU パススルー設定

### E2E 動作確認
- [ ] 音声入力 → STT → Agent → ストリーミング応答 → メトリクス表示
- [ ] テキスト入力 → Agent → ストリーミング応答 → メトリクス表示
- [ ] 権限レベル切替が Agent の allowedTools に反映される
- [ ] STT サーバー未起動時のフォールバック（テキスト入力のみ）
- [ ] パフォーマンスログが JSONL に記録される

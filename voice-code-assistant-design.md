# Voice Code Assistant - 設計ドキュメント

**Claude Agent SDK × faster-whisper によるボイスドリブン開発ツール**

---

## 1. コンセプト

音声でClaude Codeに指示を出し、リアルタイムでコーディングタスクを実行するデスクトップアプリ。
「このファイルのバグ直して」「テスト書いて」と喋るだけでコードが変わる体験を目指す。

---

## 2. アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (React)                     │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ 音声入力  │  │ チャットログ  │  │ ファイル変更ビュー │  │
│  │ ボタン/   │  │ (ストリーミ   │  │ (diff表示)        │  │
│  │ ウェイク  │  │  ング表示)    │  │                   │  │
│  │ ワード    │  │              │  │                   │  │
│  └────┬─────┘  └──────▲───────┘  └────────▲──────────┘  │
│       │               │                   │              │
│       │ WebSocket      │ WebSocket          │ WebSocket    │
└───────┼───────────────┼───────────────────┼──────────────┘
        │               │                   │
┌───────▼───────────────┴───────────────────┴──────────────┐
│                 Backend (Node.js + Express)               │
│                                                          │
│  ┌──────────────┐  ┌──────────────────────────────────┐  │
│  │ WebSocket     │  │ Claude Agent SDK                 │  │
│  │ Server        │  │                                  │  │
│  │               │  │  - query() / session管理         │  │
│  │ 音声受信 ─────┼──│  - allowedTools制御              │  │
│  │ 結果配信 ◄────┼──│  - cwd（作業ディレクトリ）切替   │  │
│  │               │  │  - ストリーミングレスポンス       │  │
│  └───────┬──────┘  └──────────────────────────────────┘  │
│          │                                               │
│  ┌───────▼──────┐                                        │
│  │ STT Service  │                                        │
│  │ Connector    │                                        │
│  └───────┬──────┘                                        │
└──────────┼───────────────────────────────────────────────┘
           │ HTTP / gRPC
┌──────────▼───────────────────────────────────────────────┐
│              STT Server (Python + FastAPI)                │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ faster-whisper (large-v3)                        │    │
│  │  - GPU推論 (CUDA / cuDNN)                        │    │
│  │  - VAD (Silero) による発話区間検出               │    │
│  │  - 日本語 + 英語 バイリンガル対応                │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ VOICEVOX（オプション・Phase 3）                   │    │
│  │  - レスポンスの音声読み上げ                      │    │
│  │  - ずんだもん / 四国めたん等                      │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

---

## 3. 技術スタック

### Backend（メインサーバー）
| 要素 | 技術 | 理由 |
|------|------|------|
| ランタイム | Node.js 20+ | Agent SDK がNode.js必須 |
| フレームワーク | Express + ws | REST + WebSocket 両対応 |
| Agent SDK | `@anthropic-ai/claude-agent-sdk` | Claude Codeの全機能をプログラム制御 |
| TypeScript | 5.2+ | `await using` 構文でセッション管理 |

### STTサーバー
| 要素 | 技術 | 理由 |
|------|------|------|
| ランタイム | Python 3.11+ | faster-whisper のエコシステム |
| フレームワーク | FastAPI + uvicorn | 非同期対応、WebSocket対応 |
| STT | faster-whisper (large-v3) | 日本語精度最優先 |
| VAD | Silero VAD | 発話区間の自動検出 |
| 音声形式 | WebM/Opus → PCM 16kHz | ブラウザからの音声を変換 |

### Frontend
| 要素 | 技術 | 理由 |
|------|------|------|
| フレームワーク | React 18+ (Vite) | 既存の知見を活用 |
| 音声キャプチャ | MediaRecorder API | ブラウザネイティブ |
| 通信 | WebSocket | リアルタイムストリーミング |
| スタイリング | Tailwind CSS | 素早いプロトタイピング |
| コードハイライト | Shiki / Prism | diff表示用 |

---

## 4. Agent SDK 利用設計

### 4.1 セッション管理

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// V1 API: query() でストリーミング
async function executeCommand(prompt: string, projectPath: string) {
  const stream = query({
    prompt,
    options: {
      cwd: projectPath,
      allowedTools: ["Read", "Write", "Edit", "MultiEdit", "Bash", "Grep"],
      permissionMode: "acceptEdits", // ファイル編集は自動承認
      // model: "sonnet" // モデル指定（オプション）
    },
  });

  for await (const message of stream) {
    // メッセージタイプに応じて処理
    switch (message.type) {
      case "assistant":
        // テキストレスポンス → フロントに配信
        broadcastToClient({ type: "response", content: message });
        break;
      case "tool_use":
        // ツール実行中 → 進捗表示
        broadcastToClient({ type: "tool_progress", tool: message });
        break;
      case "result":
        // 完了 → 結果表示
        broadcastToClient({ type: "complete", result: message });
        break;
    }
  }
}
```

### 4.2 ツール許可レベル（3段階）

```typescript
// レベル1: 読み取り専用（安全）
const READONLY_TOOLS = ["Read", "Grep", "Glob", "LS"];

// レベル2: 編集許可（通常作業）
const EDIT_TOOLS = [...READONLY_TOOLS, "Write", "Edit", "MultiEdit"];

// レベル3: フル権限（コマンド実行含む）
const FULL_TOOLS = [...EDIT_TOOLS, "Bash"];
```

音声入力の内容に応じて自動的にレベルを判定するか、
UIからユーザーが切り替えられるようにする。

### 4.3 プロジェクト切替

```typescript
// 複数プロジェクトを同時管理
const projects = new Map<string, ProjectSession>();

interface ProjectSession {
  name: string;
  path: string;       // cwd
  history: Message[];  // 会話履歴
}

// 音声: 「Godotプロジェクトに切り替えて」
function switchProject(projectName: string) {
  const session = projects.get(projectName);
  if (session) {
    currentProject = session;
    // 次の query() で session.path を cwd に使う
  }
}
```

---

## 5. 音声パイプライン設計（低レイテンシ最適化版）

### 5.1 設計方針

リアルタイムな対話体験を実現するため、「録音完了を待ってからSTT」ではなく
**「発話中から音声を蓄積し、発話終了の瞬間にSTTを発火」** する方式を採用する。

### 5.2 最適化された音声入力フロー

```
発話開始
    │
    ▼
┌──────────────────────────────────────────────────────┐
│ ブラウザ（MediaRecorder API）                         │
│                                                      │
│  音声チャンクを250ms間隔でWebSocket送信               │
│  （録音しながらリアルタイムにサーバーへストリーム）     │
└────────┬─────────────────────────────────────────────┘
         │ WebSocket (バイナリ、250msチャンク)
         ▼
┌──────────────────────────────────────────────────────┐
│ STT Server（常時受信モード）                          │
│                                                      │
│  1. 音声チャンク到着 → PCMバッファに蓄積              │
│  2. Silero VAD が各チャンクを評価                     │
│     - 発話中 → バッファ蓄積を継続                     │
│     - 無音検出（500ms以上）→ 発話終了と判定           │
│  3. 発話終了の瞬間 → バッファ全体をfaster-whisperへ   │
│     - large-v3, beam_size=5, language="ja"            │
│  4. 文字起こし結果を即座に返送                        │
└────────┬─────────────────────────────────────────────┘
         │ JSON: { text, confidence, stt_latency_ms }
         ▼
┌──────────────────────────────────────────────────────┐
│ Node.js Backend                                       │
│                                                      │
│  1. テキスト受信（タイムスタンプ記録）                │
│  2. Agent SDK query() 発火                            │
│  3. ストリーミングレスポンスを即座にフロントへ配信    │
│  4. パフォーマンスメトリクスを記録・配信              │
└──────────────────────────────────────────────────────┘
```

### 5.3 レイテンシ目標

| 区間 | 目標 | 計測方法 |
|------|------|----------|
| 発話終了 → STT完了 | < 1,000ms | VAD検出時刻 → transcription返送時刻 |
| STT完了 → Agent SDK最初のチャンク | < 3,000ms | transcription受信 → 最初のassistant_text |
| End-to-End（発話終了→最初の応答表示） | < 4,000ms | フロント側で計測 |

### 5.4 入力モード（2種類）

**Push-to-Talk（Phase 1）**
- ボタン押下中のみ録音
- シンプル・誤認識なし
- プロトタイプに最適

**ウェイクワード（Phase 3）**
- 「ねぇクロード」等で起動
- openWakeWord で常時待機
- ハンズフリー体験

---

## 6. フロントエンド UI 設計

### 6.1 レイアウト構成

```
┌──────────────────────────────────────────────────────┐
│  [プロジェクト: ~/projects/arpg-game ▼]  [⚙ Settings]│
├──────────────┬───────────────────────────────────────┤
│              │                                       │
│  チャット     │  コンテキストパネル                    │
│  ログ        │                                       │
│              │  ┌─────────────────────────────────┐  │
│  🤖 了解、    │  │ 変更ファイル                     │  │
│  src/player  │  │                                 │  │
│  .gd を修正  │  │  M src/player.gd               │  │
│  します...   │  │  A src/player_test.gd          │  │
│              │  │                                 │  │
│  [ツール実行  │  └─────────────────────────────────┘  │
│   中...]     │                                       │
│              │  ┌─────────────────────────────────┐  │
│              │  │ Diff View                       │  │
│              │  │                                 │  │
│              │  │ - var speed = 100               │  │
│              │  │ + var speed: float = 200.0      │  │
│              │  │                                 │  │
│              │  └─────────────────────────────────┘  │
│              │                                       │
├──────────────┴───────────────────────────────────────┤
│                                                      │
│  [ 🎤 Push to Talk ]  ← 長押しで録音                 │
│                                                      │
│  ── or ──                                            │
│                                                      │
│  [ テキスト入力... ]            [送信]                │
│                                                      │
│  🔒 Read Only  🔓 Edit Mode  ⚡ Full Access          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 6.2 コンポーネント構成

```
App
├── Header
│   ├── ProjectSelector        # cwd切替ドロップダウン
│   └── SettingsButton         # 設定モーダル
├── MainLayout (2カラム)
│   ├── ChatPanel
│   │   ├── MessageList        # ストリーミング表示
│   │   │   ├── UserMessage    # 音声認識テキスト
│   │   │   ├── AssistantMessage # Claude応答
│   │   │   └── ToolExecution  # ツール実行ログ
│   │   └── InputArea
│   │       ├── VoiceButton    # Push-to-Talk（VAD状態表示付き）
│   │       └── TextInput      # フォールバック
│   └── ContextPanel
│       ├── FileChangeList     # 変更ファイル一覧
│       ├── DiffViewer         # 差分表示
│       └── PerfDashboard      # パフォーマンス詳細パネル（トグル）
│           ├── LatencyChart   # レイテンシ推移グラフ（Recharts）
│           ├── BreakdownBars  # STT/Agent区間 積み上げ棒グラフ
│           └── StatsTable     # 平均/P50/P95/Max 統計テーブル
├── PermissionBar              # ツール許可レベル切替
└── StatusBar                  # 接続状態、STTステータス、直近レイテンシ
```

---

## 7. フェーズ別実装計画

### Phase 1: MVP（1-2週間）

**ゴール:** 音声でClaude Codeに指示 → 結果が表示される + レイテンシ可視化

- [ ] Node.js バックエンド（Express + WebSocket）
- [ ] Agent SDK 基本統合（`query()` 呼び出し）
- [ ] Python STTサーバー（FastAPI + faster-whisper + VADストリーミング）
- [ ] React フロントエンド（チャットUI + Push-to-Talk）
- [ ] ストリーミングレスポンス表示
- [ ] パフォーマンス計測基盤（各区間タイムスタンプ記録）
- [ ] StatusBarにリアルタイムレイテンシ表示

**スコープ外:** TTS、ウェイクワード、diff表示、プロジェクト切替、詳細パフォーマンスダッシュボード

### Phase 2: 実用化（2-3週間）

**ゴール:** 日常の開発で実際に使えるレベル

- [ ] Diff表示パネル（変更されたファイルの差分）
- [ ] プロジェクト切替機能（cwd動的変更）
- [ ] ツール許可レベルUI
- [ ] 会話履歴の永続化
- [ ] エラーハンドリング強化
- [ ] 音声認識の信頼度表示（低信頼度時に確認）
- [ ] パフォーマンスダッシュボード（レイテンシ推移グラフ、区間分析）
- [ ] パフォーマンスログのJSONL永続化

### Phase 3: 拡張（3-4週間）

**ゴール:** ハンズフリー体験 + ゲーム開発特化

- [ ] VOICEVOX連携（ずんだもんが結果を読み上げ）
- [ ] ウェイクワード対応（openWakeWord）
- [ ] Godot連携（GDScriptリンティング、シーン解析）
- [ ] コマンドパレット（よく使う指示のショートカット）
- [ ] Electron化（デスクトップアプリ）

### Phase 4: 発展（将来）

- [ ] マルチセッション（複数プロジェクト並列操作）
- [ ] 音声コマンドのカスタム辞書（技術用語登録）
- [ ] Redmine / GitLab 連携（MCP経由）
- [ ] 操作ログの可視化ダッシュボード

---

## 8. ディレクトリ構成（想定）

```
voice-code-assistant/
├── package.json
├── tsconfig.json
│
├── server/                    # Node.js Backend
│   ├── src/
│   │   ├── index.ts           # エントリポイント
│   │   ├── websocket.ts       # WebSocket サーバー
│   │   ├── agent.ts           # Agent SDK ラッパー
│   │   ├── project-manager.ts # プロジェクト管理
│   │   ├── perf-tracker.ts    # パフォーマンス計測・ログ
│   │   └── types.ts           # 共有型定義
│   └── package.json
│
├── stt-server/                # Python STT Server
│   ├── main.py                # FastAPI エントリ（WebSocket対応）
│   ├── transcriber.py         # faster-whisper ラッパー
│   ├── vad.py                 # Voice Activity Detection（ストリーミング）
│   ├── audio_buffer.py        # 音声チャンクバッファ管理
│   ├── requirements.txt
│   └── Dockerfile             # GPU対応コンテナ
│
├── client/                    # React Frontend
│   ├── src/
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── ChatPanel.tsx
│   │   │   ├── VoiceButton.tsx
│   │   │   ├── DiffViewer.tsx
│   │   │   ├── FileChangeList.tsx
│   │   │   ├── ProjectSelector.tsx
│   │   │   ├── PerfDashboard.tsx   # パフォーマンス詳細パネル
│   │   │   └── StatusBar.tsx       # 直近レイテンシ表示
│   │   ├── hooks/
│   │   │   ├── useWebSocket.ts
│   │   │   ├── useVoiceRecorder.ts # VADストリーミング対応
│   │   │   ├── useAgentStream.ts
│   │   │   └── usePerfMetrics.ts   # メトリクス蓄積・統計計算
│   │   └── stores/
│   │       ├── chatStore.ts
│   │       └── perfStore.ts        # パフォーマンスデータストア
│   ├── package.json
│   └── vite.config.ts
│
├── docker-compose.yml         # 全サービス起動
└── README.md
```

---

## 9. WebSocket プロトコル定義

### Client → Server

```typescript
// 音声データ送信（250msチャンク、ストリーミング）
{ type: "audio_chunk", data: ArrayBuffer, chunkIndex: number }

// 録音開始/終了
{ type: "recording_start", timestamp: number }
{ type: "recording_stop", timestamp: number }  // Push-to-Talk時のみ

// テキスト直接入力
{ type: "text_input", text: string }

// プロジェクト切替
{ type: "switch_project", path: string }

// 許可レベル変更
{ type: "set_permission", level: "readonly" | "edit" | "full" }
```

### Server → Client

```typescript
// VADステータス（リアルタイム）
{ type: "vad_status", speaking: boolean, timestamp: number }

// 音声認識結果
{ type: "transcription", text: string, confidence: number, sttLatencyMs: number }

// Claude 応答（ストリーミング）
{ type: "assistant_text", text: string, done: boolean }

// ツール実行通知
{ type: "tool_start", tool: string, input: object }
{ type: "tool_result", tool: string, output: string, durationMs: number }

// ファイル変更通知
{ type: "file_changed", path: string, diff: string }

// パフォーマンスメトリクス（各リクエスト完了時）
{ type: "perf_metrics", data: PerformanceMetrics }

// エラー
{ type: "error", message: string, code: string }

// ステータス
{ type: "status", stt: "ready" | "processing", agent: "idle" | "running" }
```

---

## 10. パフォーマンス計測システム

### 10.1 計測ポイント

各リクエストに対して以下のタイムスタンプを記録し、区間レイテンシを算出する。

```typescript
interface PerformanceMetrics {
  requestId: string;
  timestamp: string;              // リクエスト開始時刻

  // STT区間
  recordingStartedAt: number;     // 録音開始
  recordingStoppedAt: number;     // 録音終了（VAD検出 or ボタンリリース）
  audioDurationMs: number;        // 音声の長さ
  sttStartedAt: number;           // faster-whisper推論開始
  sttCompletedAt: number;         // faster-whisper推論完了
  sttLatencyMs: number;           // STT処理時間
  sttConfidence: number;          // 認識信頼度（0-1）

  // Agent SDK区間
  agentStartedAt: number;         // query()呼び出し時刻
  agentFirstChunkAt: number;      // 最初のテキストチャンク到着
  agentCompletedAt: number;       // 全処理完了
  agentFirstChunkLatencyMs: number; // TTFB（最初の応答まで）
  agentTotalLatencyMs: number;    // 総処理時間
  toolsUsed: string[];            // 使用されたツール一覧
  toolCount: number;              // ツール呼び出し回数

  // End-to-End
  e2eFirstResponseMs: number;     // 発話終了→最初の応答表示
  e2eTotalMs: number;             // 発話終了→全処理完了

  // メタデータ
  inputText: string;              // 認識テキスト
  inputCharCount: number;         // 入力文字数
  outputCharCount: number;        // 出力文字数
}
```

### 10.2 計測フロー

```
[Frontend]                    [Backend]                    [STT Server]
    │                             │                             │
    ├─ recording_start ──────────►│                             │
    │  (t0: recordingStartedAt)   │                             │
    │                             │                             │
    ├─ audio_chunk ──────────────►├─ forward_audio ────────────►│
    ├─ audio_chunk ──────────────►├─ forward_audio ────────────►│
    │  ...                        │                             │
    │                             │  VAD: 発話終了検出           │
    │                             │  (t1: recordingStoppedAt)    │
    │                             │                             │
    │                             │  faster-whisper推論          │
    │                             │  (t2: sttStartedAt)          │
    │                             │  (t3: sttCompletedAt)        │
    │                             │                             │
    │                             │◄─ transcription ────────────┤
    │                             │  (sttLatencyMs = t3 - t2)    │
    │                             │                             │
    │◄─ transcription ───────────┤                             │
    │                             │                             │
    │                             ├─ Agent SDK query()          │
    │                             │  (t4: agentStartedAt)        │
    │                             │                             │
    │◄─ assistant_text(1st) ─────┤  (t5: agentFirstChunkAt)    │
    │◄─ assistant_text ──────────┤                             │
    │◄─ tool_start ──────────────┤                             │
    │◄─ tool_result ─────────────┤                             │
    │◄─ assistant_text(done) ────┤  (t6: agentCompletedAt)     │
    │                             │                             │
    │◄─ perf_metrics ────────────┤  全メトリクス送信            │
```

### 10.3 WebSocketメトリクス配信

```typescript
// Server → Client: 各リクエスト完了時に送信
{
  type: "perf_metrics",
  data: {
    requestId: "req_001",
    sttLatencyMs: 820,
    agentFirstChunkLatencyMs: 2340,
    agentTotalLatencyMs: 8750,
    e2eFirstResponseMs: 3160,
    e2eTotalMs: 9570,
    toolsUsed: ["Read", "Edit"],
    toolCount: 3,
    sttConfidence: 0.94,
    audioDurationMs: 2100,
    inputCharCount: 18,
    outputCharCount: 456
  }
}
```

### 10.4 パフォーマンスダッシュボード（UI）

フロントエンドにリアルタイムのパフォーマンスパネルを追加する。

**常時表示（StatusBar内）**
- 直近のE2E TTFB（発話終了→最初の応答）
- STT処理時間
- Agent応答時間
- 色分け: 緑 < 3s、黄 < 6s、赤 > 6s

**詳細パネル（トグルで展開）**
- セッション内の全リクエストのレイテンシ推移グラフ
- STT / Agent / E2E の各区間を積み上げ棒グラフで表示
- 平均・P50・P95・最大値
- ツール使用回数と種類別の処理時間
- STT信頼度の分布

### 10.5 ログ永続化

```typescript
// パフォーマンスログをJSONLで保存（後から分析可能）
// 保存先: ~/.voice-code-assistant/perf-logs/YYYY-MM-DD.jsonl
interface PerfLogEntry {
  ...PerformanceMetrics;
  sessionId: string;
  projectPath: string;
}
```

---

## 11. セキュリティ考慮事項

| リスク | 対策 |
|--------|------|
| 音声の誤認識による意図しないコマンド実行 | 信頼度閾値の設定、危険なコマンド実行前の確認UI |
| `Bash` ツールによる破壊的操作 | デフォルトは Edit Mode、Full Access は明示的に切替 |
| ファイルシステムアクセス範囲 | `cwd` を指定プロジェクトに限定 |
| 認証 | Claude CLI のローカル認証を利用（`claude login`） |

---

## 11. 起動手順（想定）

```bash
# 1. STTサーバー起動（Python）
cd stt-server
python main.py  # GPU必須、localhost:8001

# 2. Backendサーバー起動（Node.js）
cd server
npm run dev     # localhost:3000

# 3. フロントエンド起動（React）
cd client
npm run dev     # localhost:5173

# --- or ---

# docker-compose で一発起動
docker-compose up
```

---

## 12. 今後の検討事項

- **Web Speech API vs faster-whisper**: ブラウザ内蔵STTで十分なら Python サーバー不要になる。
  ただし日本語精度と技術用語の認識ではfaster-whisperが優位。
- **Electron vs ブラウザ**: Phase 1はブラウザで十分。マイクの常時アクセスやシステムトレイが必要ならElectron化。
- **Agent SDK V2**: `unstable_v2_createSession()` が安定したらセッション管理がさらにシンプルになる。
  現時点ではV1の `query()` で実装し、V2安定後に移行を検討。

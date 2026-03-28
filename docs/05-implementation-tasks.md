# 実装タスク分解

## 1. 概要

並列実装のためのタスク分解を定義する。各タスクは独立して実装可能。

---

## 2. タスク分類

### 2.1 依存関係グラフ

```
┌─────────────────────────────────────────────────────────────────┐
│                    依存関係グラフ                               │
└─────────────────────────────────────────────────────────────────┘

フェーズ 1 (独立タスク - 並列可能)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[TASK-001] プロジェクトセットアップ
         ↓
    (依存なし)

[TASK-002] indexed-db 層実装      [TASK-003] Cloudflare Workers
         ↓                          ↓
    (独立)                     (独立)

[TASK-004] LINE API クライアント
         ↓
    (独立)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

フェーズ 2 (フェーズ 1 完了後 - 並列可能)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[TASK-002] indexed-db 層 ←──┐
                            ├──→ [TASK-005] LLM クライアント
[TASK-001] セットアップ ←──┘

[TASK-004] LINE API ←───────┐
                            ├──→ [TASK-006] チャットシミュレーター
[TASK-002] indexed-db ←────┘

[TASK-004] LINE API ←───────┐
                            ├──→ [TASK-007] リッチメニューエディター
[TASK-002] indexed-db ←────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

フェーズ 3 (フェーズ 2 完了後)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[TASK-005] LLM クライアント ←──┐
                               ├──→ [TASK-008] ヒアリングチャット
[TASK-002] indexed-db ←──────┘

[TASK-006] チャットシミュレーター ←──┐
                                      ├──→ [TASK-009] 統合テスト
[TASK-007] リッチメニュー ←──────────┘
```

---

## 3. タスク詳細

### TASK-001: プロジェクトセットアップ

**カテゴリ**: `quick`
**推定工数**: 2h
**依存関係**: なし

**タスク**:
1. Leptos プロジェクト作成
2. TailwindCSS 設定
3. 基本ディレクトリ構造作成
4. Cloudflare Workers 設定

**出力**:
```
line-account-builder/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── app.rs
│   ├── components/
│   ├── storage/
│   ├── llm/
│   ├── line/
│   └── state/
├── static/
│   ├── index.html
│   └── tailwind.css
├── workers/line-api-proxy/
│   ├── wrangler.toml
│   └── worker.js
└── tailwind.config.js
```

**検証基準**:
- [ ] `cargo check` が成功
- [ ] `cargo leptos serve` で起動可能
- [ ] TailwindCSS が適用される

---

### TASK-002: indexed-db 層実装

**カテゴリ**: `deep`
**推定工数**: 4h
**依存関係**: なし（依存関係グラフどおり、TASK-001 とは独立して実行可能）

**タスク**:
1. `indexed-db` crate の追加
2. データベーススキーマ実装
3. 各オブジェクトストアの CRUD 操作
4. セリアライズ/デシリアライズ

**実装ファイル**:
```
src/storage/
├── mod.rs
└── indexed_db.rs
```

**実装内容**:
```rust
// src/storage/indexed_db.rs

pub async fn init_database() -> Result<Database, Error<String>>;
pub async fn save_api_key(config: &ApiKeyConfig) -> Result<(), Error<String>>;
pub async fn load_api_key() -> Result<Option<ApiKeyConfig>, Error<String>>;
pub async fn save_hearing_session(session: &HearingSession) -> Result<(), Error<String>>;
pub async fn load_hearing_session(id: &str) -> Result<Option<HearingSession>, Error<String>>;
pub async fn save_hearing_message(message: &HearingMessage) -> Result<(), Error<String>>;
pub async fn load_hearing_messages(session_id: &str) -> Result<Vec<HearingMessage>, Error<String>>;
pub async fn save_rich_menu(rich_menu: &RichMenu) -> Result<(), Error<String>>;
pub async fn load_rich_menu(id: &str) -> Result<Option<RichMenu>, Error<String>>;
pub async fn clear_all_data() -> Result<(), Error<String>>;
```

**検証基準**:
- [ ] データベースが正しく初期化される
- [ ] CRUD 操作が正しく機能する
- [ ] セリアライズ/デシリアライズが正しく機能する
- [ ] ページリロード後データが保持される

---

### TASK-003: Cloudflare Workers

**カテゴリ**: `quick`
**推定工数**: 1h
**依存関係**: なし

**タスク**:
1. wrangler 設定
2. Worker 実装（CORS プロキシ）
3. デプロイ

**実装ファイル**:
```
workers/line-api-proxy/
├── wrangler.toml
└── worker.js
```

**実装内容**:
```javascript
// worker.js
export default {
  async fetch(request) {
    // CORS プリフライト処理
    if (request.method === 'OPTIONS') {
      return handleOptions();
    }
    
    // LINE API にリクエストを転送
    return await forwardToLineApi(request);
  }
}
```

**検証基準**:
- [ ] `wrangler deploy` が成功
- [ ] CORS ヘッダーが正しく追加される
- [ ] LINE API にリクエストが正しく転送される

---

### TASK-004: LINE API クライアント

**カテゴリ**: `deep`
**推定工数**: 3h
**依存関係**: なし

**タスク**:
1. reqwest クライアント設定
2. リッチメニュー API 実装
3. エラーハンドリング
4. タイプ定義

**実装ファイル**:
```
src/line/
├── mod.rs
├── client.rs
└── models/
    ├── mod.rs
    └── rich_menu.rs
```

**実装内容**:
```rust
// src/line/client.rs

pub struct LineApiClient {
    client: Client,
    proxy_url: String,
    channel_access_token: String,
}

impl LineApiClient {
    pub fn new(proxy_url: &str, channel_access_token: &str) -> Self;
    pub async fn create_rich_menu(&self, rich_menu: &RichMenuCreateRequest) -> Result<String, LineApiError>;
    pub async fn get_rich_menu(&self, rich_menu_id: &str) -> Result<RichMenu, LineApiError>;
    pub async fn delete_rich_menu(&self, rich_menu_id: &str) -> Result<(), LineApiError>;
    pub async fn upload_rich_menu_image(&self, rich_menu_id: &str, image: &[u8]) -> Result<(), LineApiError>;
}
```

**検証基準**:
- [ ] リッチメニュー作成 API が機能する
- [ ] リッチメニュー取得 API が機能する
- [ ] エラーが正しくハンドリングされる
- [ ] CORS プロキシ経由で通信可能

---

### TASK-005: LLM クライアント

**カテゴリ**: `deep`
**推定工数**: 4h
**依存関係**: TASK-001, TASK-002

**タスク**:
1. LLM クライアント抽象化
2. Gemini API 実装
3. Anthropic API 実装
4. OpenAI API 実装
5. プロバイダー切り替え

**実装ファイル**:
```
src/llm/
├── mod.rs
├── client.rs
└── providers/
    ├── mod.rs
    ├── gemini.rs
    ├── anthropic.rs
    └── openai.rs
```

**実装内容**:
```rust
// src/llm/client.rs

pub trait LlmProvider {
    async fn generate(&self, prompt: &str) -> Result<String, LlmError>;
    async fn generate_chat(&self, messages: &[ChatMessage]) -> Result<String, LlmError>;
}

pub struct LlmClient {
    provider: Box<dyn LlmProvider>,
}

impl LlmClient {
    pub async fn new(provider: Provider, api_key: String) -> Self;
    pub async fn generate(&self, prompt: &str) -> Result<String, LlmError>;
    pub async fn generate_hearing_question(&self, session_id: &str, user_input: &str) -> Result<String, LlmError>;
}
```

**検証基準**:
- [ ] Gemini API が機能する
- [ ] Anthropic API が機能する
- [ ] OpenAI API が機能する
- [ ] プロバイダー切り替えが可能
- [ ] API キーが正しく使用される

---

### TASK-006: チャットシミュレーター

**カテゴリ**: `visual-engineering`
**推定工数**: 4h
**依存関係**: TASK-002, TASK-004

**タスク**:
1. チャット UI コンポーネント
2. メッセージ表示
3. リッチメニュープレビュー表示
4. レスポンシブデザイン

**実装ファイル**:
```
src/components/
├── chat_simulator.rs
├── chat_message.rs
└── chat_input.rs
```

**実装内容**:
```rust
#[component]
pub fn ChatSimulator(account_id: String) -> impl IntoView {
    // チャットメッセージの表示
    // メッセージ入力
    // リッチメニュープレビューの表示
}
```

**検証基準**:
- [ ] チャットメッセージが正しく表示される
- [ ] メッセージ入力が可能
- [ ] リッチメニューが正しく表示される
- [ ] モバイルで正しく表示される

---

### TASK-007: リッチメニューエディター

**カテゴリ**: `visual-engineering`
**推定工数**: 3h
**依存関係**: TASK-002, TASK-004

**タスク**:
1. JSON エディターコンポーネント
2. リッチメニュープレビュー
3. バリデーション
4. LINE へのデプロイ機能

**実装ファイル**:
```
src/components/
├── rich_menu_editor.rs
├── rich_menu_preview.rs
└── json_editor.rs
```

**実装内容**:
```rust
#[component]
pub fn RichMenuEditor(account_id: String) -> impl IntoView {
    // JSON エディター
    // バリデーション
    // デプロイボタン
}

#[component]
pub fn RichMenuPreview(rich_menu: RichMenu) -> impl IntoView {
    // 視覚的プレビュー
}
```

**検証基準**:
- [ ] JSON エディターが機能する
- [ ] バリデーションエラーが表示される
- [ ] プレビューが正しく表示される
- [ ] LINE へのデプロイが機能する

---

### TASK-008: ヒアリングチャット

**カテゴリ**: `deep`
**推定工数**: 4h
**依存関係**: TASK-002, TASK-005

**タスク**:
1. チャット UI
2. AI 質問生成ロジック
3. 要件抽出ロジック
4. 要件サマリー表示

**実装ファイル**:
```
src/components/
├── hearing_chat.rs
├── hearing_message.rs
└── requirement_summary.rs

src/hearing/
├── mod.rs
└── logic.rs
```

**実装内容**:
```rust
#[component]
pub fn HearingChat(session_id: Option<String>) -> impl IntoView {
    // チャット UI
    // AI 質問生成
    // 要件抽出
}

// src/hearing/logic.rs
pub struct HearingLogic {
    // 状態管理
    // 質問生成
    // 要件抽出
}
```

**検証基準**:
- [ ] チャットが機能する
- [ ] AI が適切に質問を生成する
- [ ] 要件が正しく抽出される
- [ ] 要件サマリーが表示される

---

### TASK-009: 統合テスト

**カテゴリ**: `quick`
**推定工数**: 2h
**依存関係**: TASK-006, TASK-007, TASK-008

**タスク**:
1. エンドツーエンドテスト
2. UI テスト
3. API テスト
4. パフォーマンステスト

**テストシナリオ**:
1. API キー設定 → ヒアリング → チャットシミュレーター → デプロイ
2. 複数プロバイダーでの動作確認
3. モバイルでの動作確認
4. データ永続化確認

**検証基準**:
- [ ] 全フローが正常に動作する
- [ ] エラーが正しくハンドリングされる
- [ ] モバイルで正常に動作する
- [ ] データが正しく永続化される

---

## 4. 並列実装計画

### 4.1 グループ分割

| グループ | タスク | 推定工数 | 並列可能 |
|---|---|---|---|
| **A** | TASK-001, TASK-002, TASK-003, TASK-004 | 10h | ✅ |
| **B** | TASK-005, TASK-006, TASK-007 | 11h | ✅ (A 完了後) |
| **C** | TASK-008, TASK-009 | 6h | ✅ (B 完了後) |

### 4.2 推奨アサインメント

```
エージェント 1 (バックエンド):
  TASK-001 → TASK-002 → TASK-004 → TASK-005

エージェント 2 (フロントエンド):
  TASK-001 → TASK-006 → TASK-007

エージェント 3 (インフラ):
  TASK-003

エージェント 4 (統合):
  TASK-008 → TASK-009
```

---

## 5. 検証チェックリスト

### 5.1 全体的な検証

- [ ] `cargo check` が成功
- [ ] `cargo leptos build --release` が成功
- [ ] Cloudflare Workers がデプロイされている
- [ ] 全フローが正常に動作する

### 5.2 セキュリティ検証

- [ ] API キーが正しく保存される
- [ ] CORS が正しく設定されている
- [ ] エラーメッセージに機密情報が含まれない

### 5.3 パフォーマンス検証

- [ ] 初期ロードが 3 秒以内
- [ ] チャットメッセージ表示がスムーズ
- [ ] リッチメニュープレビューがスムーズ

---

## 6. デプロイチェックリスト

### 6.1 フロントエンド

```bash
# ビルド
cargo leptos build --release

# ホスティングにアップロード
# - Netlify
# - Vercel
# - GitHub Pages
# - Cloudflare Pages
```

### 6.2 Cloudflare Workers

```bash
cd workers/line-api-proxy
wrangler deploy
```

### 6.3 環境変数

```bash
# フロントエンド
VITE_LINE_API_PROXY_URL=https://line-api-proxy.your-subdomain.workers.dev
```

---

## 7. ロールバック計画

### 7.1 バージョニング

- Git で全ての変更を管理
- 各フェーズ完了時にタグを付与

### 7.2 ロールバック手順

```bash
# 最後の安定バージョンに戻す
git checkout <stable-tag>

# 再ビルド
cargo leptos build --release
```

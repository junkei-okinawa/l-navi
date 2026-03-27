# データモデル設計書

## 1. 概要

indexed-db によるクライアントサイド永続化のデータモデルを定義する。

---

## 2. IndexedDB スキーマ

### 2.1 データベース構成

```typescript
interface DatabaseSchema {
  // API キー管理
  api_keys: ObjectStore<ApiKeyConfig>;
  
  // ヒアリング履歴
  hearing_sessions: ObjectStore<HearingSession>;
  hearing_messages: ObjectStore<HearingMessage>;
  
  // LINE アカウント設定
  line_accounts: ObjectStore<LineAccount>;
  
  // リッチメニュー
  rich_menus: ObjectStore<RichMenu>;
  
  // チャットシミュレーション履歴
  chat_simulations: ObjectStore<ChatSimulation>;
}
```

---

## 3. オブジェクトストア詳細

### 3.1 `api_keys` ストア

**目的**: LLM プロバイダーの API キーを保存

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Clone)]
pub struct ApiKeyConfig {
    pub id: String,           // 主キー ("default")
    pub provider: Provider,   // プロバイダー種類
    pub api_key: String,      // API キー (平文)
    pub created_at: i64,      // 作成日時 (epoch ms)
    pub updated_at: i64,      // 更新日時 (epoch ms)
}

#[derive(Serialize, Deserialize, Clone, PartialEq)]
pub enum Provider {
    Gemini,
    Anthropic,
    OpenAI,
}
```

**インデックス**: なし (単一レコード)

**初期データ**:
```json
{
  "id": "default",
  "provider": "Gemini",
  "api_key": "",
  "created_at": 1700000000000,
  "updated_at": 1700000000000
}
```

---

### 3.2 `hearing_sessions` ストア

**目的**: ヒアリングセッションのメタデータを保存

```rust
#[derive(Serialize, Deserialize, Clone)]
pub struct HearingSession {
    pub id: String,           // 主キー (UUID)
    pub title: String,        // セッションタイトル
    pub status: SessionStatus,
    pub requirements: Option<Requirements>,  // 確定した要件
    pub created_at: i64,
    pub updated_at: i64,
    pub completed_at: Option<i64>,
}

#[derive(Serialize, Deserialize, Clone, PartialEq)]
pub enum SessionStatus {
    InProgress,
    Completed,
    Archived,
}

#[derive(Serialize, Deserialize, Clone)]
pub struct Requirements {
    pub business_type: String,        // 業種
    pub target_audience: String,      // ターゲット層
    pub primary_purpose: Purpose,     // 主な目的
    pub automation_level: AutomationLevel,
    pub features: Vec<Feature>,       // 必要な機能
    pub tone_of_voice: String,        // トーン＆マナー
    pub additional_notes: String,     // 追加ノート
}

#[derive(Serialize, Deserialize, Clone, PartialEq)]
pub enum Purpose {
    Promotion,
    CustomerSupport,
    Reservation,
    Both,
}

#[derive(Serialize, Deserialize, Clone, PartialEq)]
pub enum AutomationLevel {
    Basic,      // 基本自動応答
    Intermediate,  // 中級 (条件分岐)
    Advanced,   // 高度 (AI 完全自動)
}

#[derive(Serialize, Deserialize, Clone, PartialEq)]
pub enum Feature {
    RichMenu,
    WelcomeMessage,
    AutoReply,
    Broadcast,
    CustomerInfo,
}
```

**インデックス**:
- `status` (セッションステータスでフィルタ)
- `created_at` (日付順ソート)

---

### 3.3 `hearing_messages` ストア

**目的**: ヒアリングチャットのメッセージ履歴

```rust
#[derive(Serialize, Deserialize, Clone)]
pub struct HearingMessage {
    pub id: String,           // 主キー (UUID)
    pub session_id: String,   // 親セッション ID
    pub role: MessageRole,
    pub content: String,
    pub created_at: i64,
}

#[derive(Serialize, Deserialize, Clone, PartialEq)]
pub enum MessageRole {
    User,
    Assistant,
}
```

**インデックス**:
- `session_id` (セッションでメッセージをフィルタ)
- `created_at` (タイムラインソート)

---

### 3.4 `line_accounts` ストア

**目的**: LINE 公式アカウントの設定を保存

```rust
#[derive(Serialize, Deserialize, Clone)]
pub struct LineAccount {
    pub id: String,           // 主キー (UUID)
    pub hearing_session_id: String,  // 関連ヒアリングセッション
    pub channel_access_token: String,
    pub channel_secret: Option<String>,  // 任意
    pub account_name: String,
    pub description: String,
    pub created_at: i64,
    pub updated_at: i64,
}
```

**インデックス**: なし

---

### 3.5 `rich_menus` ストア

**目的**: リッチメニュー設定を保存

```rust
#[derive(Serialize, Deserialize, Clone)]
pub struct RichMenu {
    pub id: String,           // 主キー (UUID)
    pub line_account_id: String,
    pub line_rich_menu_id: Option<String>,  // LINE 側の ID
    pub name: String,
    pub size: RichMenuSize,
    pub selected: bool,
    pub areas: Vec<RichMenuArea>,
    pub actions: Vec<RichMenuAction>,
    pub created_at: i64,
    pub updated_at: i64,
}

#[derive(Serialize, Deserialize, Clone)]
pub struct RichMenuSize {
    pub width: u32,
    pub height: u32,
}

#[derive(Serialize, Deserialize, Clone)]
pub struct RichMenuArea {
    pub bounds: RichMenuBounds,
    pub action: Option<RichMenuAction>,
}

#[derive(Serialize, Deserialize, Clone)]
pub struct RichMenuBounds {
    pub x: u32,
    pub y: u32,
    pub width: u32,
    pub height: u32,
}

#[derive(Serialize, Deserialize, Clone)]
pub struct RichMenuAction {
    pub action_type: ActionType,
    pub label: String,
    pub data: ActionData,
}

#[derive(Serialize, Deserialize, Clone, PartialEq)]
pub enum ActionType {
    Postback,
    Message,
    Uri,
    TurnRichMenu,
    ImagemapAction,
}

#[derive(Serialize, Deserialize, Clone)]
pub enum ActionData {
    Postback { label: String, data: String, display_text: Option<String> },
    Message { text: String },
    Uri { uri: String },
    TurnRichMenu { rich_menu_id: String },
    ImagemapAction { label: String, data: String, display_text: Option<String>, link_uri: String },
}
```

**インデックス**:
- `line_account_id` (アカウントでリッチメニューをフィルタ)

---

### 3.6 `chat_simulations` ストア

**目的**: チャットシミュレーションの履歴

```rust
#[derive(Serialize, Deserialize, Clone)]
pub struct ChatSimulation {
    pub id: String,           // 主キー (UUID)
    pub line_account_id: String,
    pub messages: Vec<SimulationMessage>,
    pub rich_menu_snapshot: Option<RichMenu>,  // スナップショット
    pub created_at: i64,
}

#[derive(Serialize, Deserialize, Clone)]
pub struct SimulationMessage {
    pub id: String,
    pub role: SimulationRole,
    pub content: String,
    pub timestamp: i64,
}

#[derive(Serialize, Deserialize, Clone, PartialEq)]
pub enum SimulationRole {
    User,
    Bot,
}
```

**インデックス**:
- `line_account_id`
- `created_at`

---

## 4. データアクセスパターン

### 4.1 頻度由高

| パターン | 頻度 | ストア |
|---|---|---|
| API キー読み込み | 高 | `api_keys` |
| ヒアリングメッセージ追加 | 高 | `hearing_messages` |
| リッチメニュープリビュー | 高 | `rich_menus` |
| セッション一覧表示 | 中 | `hearing_sessions` |
| API キー更新 | 低 | `api_keys` |
| セッション完了 | 低 | `hearing_sessions` |

### 4.2 トランザクションパターン

```rust
// 例：ヒアリングメッセージ追加
async fn add_hearing_message(
    session_id: &str,
    role: MessageRole,
    content: &str,
) -> Result<(), Error<String>> {
    let db = get_database().await?;
    
    db.transaction(&["hearing_messages"])
        .rw()
        .run(|t| async move {
            let store = t.object_store("hearing_messages")?;
            let message = HearingMessage {
                id: uuid::Uuid::new_v4().to_string(),
                session_id: session_id.to_string(),
                role,
                content: content.to_string(),
                created_at: js_sys::Date::now() as i64,
            };
            store.add(&wasm_bindgen::JsValue::from_serde(&message)?)?.await?;
            Ok(())
        })
        .await?
}
```

---

## 5. データマイグレーション

### 5.1 バージョン管理

```rust
const DB_VERSION: u32 = 1;
```

### 5.2 アップグレードハンドラー

```rust
fn on_upgrade(db: &Database) -> Result<(), Error<String>> {
    // バージョン 1: 初期スキーマ
    if db.version() < 1.0 {
        db.build_object_store("api_keys")
            .key_path("id")
            .create()?;
        
        db.build_object_store("hearing_sessions")
            .key_path("id")
            .create()?;
        
        // ...
    }
    
    Ok(())
}
```

---

## 6. データ削除

### 6.1 ユーザーによる削除

```rust
// 全データ削除
pub async fn clear_all_data() -> Result<(), Error<String>> {
    let db = get_database().await?;
    
    // 全てのストアをクリア
    for store_name in &["api_keys", "hearing_sessions", "hearing_messages", 
                        "line_accounts", "rich_menus", "chat_simulations"] {
        db.transaction(&[*store_name])
            .rw()
            .run(|t| async move {
                t.object_store(store_name)?.clear().await?;
                Ok(())
            })
            .await?;
    }
    
    Ok(())
}

// セッション単位削除
pub async fn delete_session(session_id: &str) -> Result<(), Error<String>> {
    // hearing_messages を削除
    // hearing_sessions を削除
    // 関連データをカスケード削除
}
```

---

## 7. エクスポート/インポート (将来的な拡張)

```rust
pub async fn export_all() -> Result<String, Error<String>> {
    // 全てのデータを JSON にシリアライズ
}

pub async fn import_data(json: &str) -> Result<(), Error<String>> {
    // JSON からデータを復元
}
```

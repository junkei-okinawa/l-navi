# コンポーネント設計書

## 1. 概要

Leptos コンポーネントの設計を定義する。

---

## 2. アプリケーション構造

```
App
├── Router
│   ├── Home
│   ├── ApiKeyManager
│   ├── HearingChat
│   └── ChatSimulator
└── AppStateProvider
```

---

## 3. コンポーネント詳細

### 3.1 App (ルートコンポーネント)

**パス**: `src/app.rs`

```rust
use leptos::*;
use crate::state::app_state::AppState;
use crate::components::*;

#[component]
pub fn App() -> impl IntoView {
    // アプリケーション状態を作成
    let app_state = create_resource(|| (), || async {
        AppState::load().await
    });

    view! {
        <Router>
            <main class="min-h-screen bg-gray-50">
                <Header />
                <Routes>
                    <Route path="/" view=Home />
                    <Route path="/api-keys" view=ApiKeyManager />
                    <Route path="/hearing/:session_id" view=HearingChat />
                    <Route path="/simulator/:account_id" view=ChatSimulator />
                </Routes>
            </main>
        </Router>
    }
}
```

---

### 3.2 Header

**パス**: `src/components/header.rs`

**役割**: ナビゲーションヘッダー

```rust
#[component]
pub fn Header() -> impl IntoView {
    view! {
        <header class="bg-white shadow">
            <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
                <div class="flex justify-between h-16">
                    <div class="flex items-center">
                        <a href="/" class="text-xl font-bold text-gray-900">
                            LINE Account Builder
                        </a>
                    </div>
                    <nav class="flex items-center space-x-4">
                        <a href="/" class="text-gray-600 hover:text-gray-900">ホーム</a>
                        <a href="/api-keys" class="text-gray-600 hover:text-gray-900">API キー設定</a>
                    </nav>
                </div>
            </div>
        </header>
    }
}
```

---

### 3.3 Home

**パス**: `src/components/home.rs`

**役割**: ホーム画面

```rust
#[component]
pub fn Home() -> impl IntoView {
    view! {
        <div class="py-12">
            <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
                <h1 class="text-3xl font-bold text-gray-900 mb-8">LINE 公式アカウントビルダー</h1>
                
                <div class="grid grid-cols-1 md:grid-cols-2 gap-8">
                    <Card
                        title="新規作成"
                        description="AI と対話しながら LINE 公式アカウントを構築"
                        action="始める"
                        href="/hearing/new"
                    />
                    
                    <Card
                        title="API キー設定"
                        description="LLM プロバイダーの API キーを設定"
                        action="設定する"
                        href="/api-keys"
                    />
                </div>
            </div>
        </div>
    }
}

#[component]
fn Card(title: String, description: String, action: String, href: String) -> impl IntoView {
    view! {
        <div class="bg-white rounded-lg shadow p-6">
            <h2 class="text-xl font-semibold text-gray-900 mb-2">{title}</h2>
            <p class="text-gray-600 mb-4">{description}</p>
            <a href={href} class="inline-block bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700">
                {action}
            </a>
        </div>
    }
}
```

---

### 3.4 ApiKeyManager

**パス**: `src/components/api_key_manager.rs`

**役割**: API キーの入力・管理

```rust
use leptos::*;
use crate::state::app_state::Provider;

#[component]
pub fn ApiKeyManager() -> impl IntoView {
    let config = create_rw_signal(None);
    
    // API キー設定を読み込む
    leptos::effect(move |_| {
        let config = config.get();
        // indexed-db から読み込む
    });

    let provider = create_rw_signal(Provider::Gemini);
    let api_key = create_rw_signal(String::new());

    let save = move |ev: wasm_bindgen::JsCast| {
        ev.prevent_default();
        // API キーを保存
    };

    view! {
        <div class="py-12">
            <div class="max-w-2xl mx-auto px-4">
                <h1 class="text-3xl font-bold text-gray-900 mb-8">API キー設定</h1>
                
                <form on:submit=save class="bg-white rounded-lg shadow p-6 space-y-6">
                    <div>
                        <label class="block text-sm font-medium text-gray-700 mb-2">
                            プロバイダー
                        </label>
                        <select
                            bind:value=provider
                            class="w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
                        >
                            <option value="Gemini">Google Gemini</option>
                            <option value="Anthropic">Anthropic Claude</option>
                            <option value="OpenAI">OpenAI</option>
                        </select>
                    </div>

                    <div>
                        <label class="block text-sm font-medium text-gray-700 mb-2">
                            API キー
                        </label>
                        <input
                            type="password"
                            bind:value=api_key
                            class="w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
                            placeholder="sk-..."
                        />
                        <p class="mt-1 text-sm text-gray-500">
                            API キーはブラウザに安全に保存されます。
                        </p>
                    </div>

                    <button
                        type="submit"
                        class="w-full bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700"
                    >
                        保存
                    </button>
                </form>

                <div class="mt-6 bg-yellow-50 border border-yellow-200 rounded-lg p-4">
                    <h3 class="text-sm font-medium text-yellow-800 mb-2">セキュリティについて</h3>
                    <p class="text-sm text-yellow-700">
                        API キーはブラウザの localStorage に保存されます。これは業界標準の手法です。
                        ブラウザのプライベートデータと同じ扱いとしてください。
                    </p>
                </div>
            </div>
        </div>
    }
}
```

---

### 3.5 HearingChat

**パス**: `src/components/hearing_chat.rs`

**役割**: AI との対話型ヒアリング

```rust
use leptos::*;
use crate::llm::client::LlmClient;
use crate::storage::indexed_db::*;

#[component]
pub fn HearingChat(session_id: Option<String>) -> impl IntoView {
    let messages = create_rw_signal(Vec::new());
    let input = create_rw_signal(String::new());
    let is_loading = create_rw_signal(false);

    // セッション ID を生成または取得
    let session_id = create_rw_signal(
        session_id.unwrap_or_else(|| uuid::Uuid::new_v4().to_string())
    );

    // メッセージを送信
    let send_message = move |ev: wasm_bindgen::JsCast| {
        ev.prevent_default();
        
        let user_message = input.get().clone();
        if user_message.is_empty() {
            return;
        }

        input.set(String::new());
        is_loading.set(true);

        // ユーザーメッセージを追加
        messages.update(|m| {
            m.push(HearingMessage {
                id: uuid::Uuid::new_v4().to_string(),
                session_id: session_id.get().clone(),
                role: MessageRole::User,
                content: user_message.clone(),
                created_at: js_sys::Date::now() as i64,
            });
        });

        // AI に質問を生成
        let session_id = session_id.get().clone();
        spawn_local(async move {
            let llm_client = LlmClient::new().await;
            let response = llm_client.generate_hearing_question(&session_id, &user_message).await;
            
            messages.update(|m| {
                m.push(HearingMessage {
                    id: uuid::Uuid::new_v4().to_string(),
                    session_id,
                    role: MessageRole::Assistant,
                    content: response,
                    created_at: js_sys::Date::now() as i64,
                });
            });
            
            is_loading.set(false);
        });
    };

    view! {
        <div class="flex h-screen">
            // チャットエリア
            <div class="flex-1 flex flex-col">
                <div class="flex-1 overflow-y-auto p-4 space-y-4">
                    {move || messages.get().into_iter().map(|msg| {
                        view! {
                            <div class={if msg.role == MessageRole::User { "ml-auto" } else { "mr-auto" }}>
                                <div class={if msg.role == MessageRole::User {
                                    "bg-blue-600 text-white"
                                } else {
                                    "bg-gray-200 text-gray-900"
                                } " rounded-lg px-4 py-2 max-w-lg"}>
                                    {msg.content}
                                </div>
                            </div>
                        }
                    }).into_view()}
                </div>

                <form on:submit=send_message class="p-4 border-t">
                    <div class="flex space-x-2">
                        <input
                            type="text"
                            bind:value=input
                            disabled=is_loading
                            class="flex-1 rounded-md border-gray-300 shadow-sm"
                            placeholder="メッセージを入力..."
                        />
                        <button
                            type="submit"
                            disabled=is_loading
                            class="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700 disabled:opacity-50"
                        >
                            送信
                        </button>
                    </div>
                </form>
            </div>

            // 要件サマリー
            <div class="w-80 border-l p-4 overflow-y-auto">
                <h2 class="text-lg font-semibold mb-4">要件サマリー</h2>
                <RequirementSummary session_id=move || session_id.get() />
            </div>
        </div>
    }
}

#[component]
fn RequirementSummary(session_id: String) -> impl IntoView {
    // 現在の要件を表示
    view! {
        <div class="space-y-4">
            <p class="text-sm text-gray-600">ヒアリングが進むとここに要件が表示されます</p>
        </div>
    }
}
```

---

### 3.6 ChatSimulator

**パス**: `src/components/chat_simulator.rs`

**役割**: チャットシミュレーター & リッチメニュープレビュー

```rust
use leptos::*;

#[component]
pub fn ChatSimulator(account_id: String) -> impl IntoView {
    let messages = create_rw_signal(Vec::new());
    let rich_menu = create_memo(move |_| {
        // リッチメニューを読み込む
        None
    });

    view! {
        <div class="flex h-screen">
            // チャットシミュレーター
            <div class="flex-1 flex flex-col">
                <div class="flex-1 overflow-y-auto p-4">
                    <div class="max-w-2xl mx-auto space-y-4">
                        {move || messages.get().into_iter().map(|msg| {
                            view! {
                                <div class={if msg.role == SimulationRole::User { "text-right" } else { "text-left" }}>
                                    <span class="inline-block px-3 py-2 rounded-lg bg-gray-200">
                                        {msg.content}
                                    </span>
                                </div>
                            }
                        }).into_view()}
                    </div>
                </div>

                // リッチメニュープレビュー
                {move || rich_menu.get().map(|rm| {
                    view! {
                        <div class="border-t p-4">
                            <RichMenuPreview rich_menu=rm />
                        </div>
                    }
                }).into_view()}
            </div>

            // エディター
            <div class="w-96 border-l p-4">
                <RichMenuEditor account_id=account_id />
            </div>
        </div>
    }
}
```

---

### 3.7 RichMenuEditor

**パス**: `src/components/rich_menu_editor.rs`

**役割**: リッチメニュー JSON エディター

```rust
use leptos::*;

#[component]
pub fn RichMenuEditor(account_id: String) -> impl IntoView {
    let json = create_rw_signal(r#"{
  "size": { "width": 2500, "height": 1686 },
  "selected": false,
  "name": "Main Menu",
  "chatBarText": "メニュー",
  "areas": []
}"#.to_string());

    let errors = create_rw_signal(Vec::new());

    let validate = move |_| {
        // JSON バリデーション
        match serde_json::from_str::<RichMenu>(&json.get()) {
            Ok(_) => errors.set(Vec::new()),
            Err(e) => errors.set(vec![e.to_string()]),
        }
    };

    view! {
        <div class="space-y-4">
            <h2 class="text-lg font-semibold">リッチメニューエディター</h2>
            
            <textarea
                bind:value=json
                on:input=validate
                class="w-full h-64 font-mono text-sm rounded-md border-gray-300"
                placeholder="JSON を入力..."
            />

            {move || errors.get().into_iter().map(|e| {
                view! {
                    <div class="text-sm text-red-600">{e}</div>
                }
            }).into_view()}

            <button
                class="w-full bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700"
            >
                LINE にデプロイ
            </button>
        </div>
    }
}
```

---

### 3.8 RichMenuPreview

**パス**: `src/components/rich_menu_preview.rs`

**役割**: リッチメニュー視覚的プレビュー

```rust
use leptos::*;

#[component]
pub fn RichMenuPreview(rich_menu: RichMenu) -> impl IntoView {
    let scale = 0.5; // プレビュー縮小率

    view! {
        <div class="flex flex-col items-center">
            <div
                class="border-2 border-gray-300 bg-gray-100"
                style:format!("width: {}px; height: {}px;",
                    rich_menu.size.width as f64 * scale,
                    rich_menu.size.height as f64 * scale
                )
            >
                {move || rich_menu.areas.into_iter().map(|area| {
                    view! {
                        <div
                            class="absolute border border-blue-400 flex items-center justify-center cursor-pointer hover:bg-blue-100"
                            style:format!("left: {}px; top: {}px; width: {}px; height: {}px;",
                                area.bounds.x as f64 * scale,
                                area.bounds.y as f64 * scale,
                                area.bounds.width as f64 * scale,
                                area.bounds.height as f64 * scale
                            )
                        >
                            {move || area.action.as_ref().map(|a| a.label.clone()).unwrap_or_default()}
                        </div>
                    }
                }).into_view()}
            </div>
            <p class="mt-2 text-sm text-gray-600">
                {rich_menu.name}
            </p>
        </div>
    }
}
```

---

## 4. レスポンシブデザイン

### 4.1 ブレイクポイント

```
sm: 640px  
md: 768px
lg: 1024px
xl: 1280px
```

### 4.2 モバイル対応

- チャットシミュレーター：フルスクリーン
- リッチメニューエディター：モーダル表示
- ヒアリングチャット：シングルカラム

---

## 5. アクセシビリティ

### 5.1 WCAG 2.1 AA 準拠

- 色コントラスト：4.5:1 以上
- キーボードナビゲーション
- スクリーンリーダー対応
- フォーカスインジケーター

### 5.2 ARIA 属性

```rust
view! {
    <div role="main" aria-label="メインコンテンツ">
        <button aria-expanded={expanded} aria-controls="menu">
            メニュー
        </button>
    </div>
}
```

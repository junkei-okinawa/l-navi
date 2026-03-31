# API 仕様設計書

## 1. 概要

LINE Messaging API との連携のための API 仕様を定義する。

---

## 2. Cloudflare Workers プロキシ

### 2.1 基本設計

**Worker 名**: `line-api-proxy`

**役割**:
- CORS ヘッダーの追加
- リクエストの転送
- API キーの透過 (保管しない)

### 2.2 デプロイ構成

```toml
# wrangler.toml
name = "line-api-proxy"
main = "worker.js"
compatibility_date = "2024-01-01"

[vars]
  # 環境変数は不要 (API キーはヘッダー経由)
```

### 2.3 Worker 実装

```javascript
// worker.js

const LINE_API_BASE = 'https://api.line.me/v2/bot';

export default {
  async fetch(request) {
    // CORS プリフライト処理
    if (request.method === 'OPTIONS') {
      const allowHeaders = request.headers.get('Access-Control-Request-Headers') || '';
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
          'Access-Control-Allow-Headers': allowHeaders,
          'Access-Control-Max-Age': '86400',
        },
      });
    }

    try {
      const response = await handleRequest(request);
      
      // CORS ヘッダー追加
      const corsHeaders = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type, X-Line-Channel-Access-Token',
      };
      
      return new Response(response.body, {
        status: response.status,
        statusText: response.statusText,
        headers: {
          ...Object.fromEntries(response.headers),
          ...corsHeaders,
        },
      });
    } catch (error) {
      console.error('Error:', error);
      return new Response(JSON.stringify({
        error: 'Internal Server Error',
        message: error.message,
      }), {
        status: 500,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*',
        },
      });
    }
  },
};

async function handleRequest(request) {
  const url = new URL(request.url);
  
  // チャネルアクセストークンを取得
  const channelAccessToken = request.headers.get('X-Line-Channel-Access-Token');
  
  if (!channelAccessToken) {
    throw new Error('Missing X-Line-Channel-Access-Token header');
  }

  // LINE API URL 構築 (クエリ文字列も含める)
  const lineUrl = `${LINE_API_BASE}${url.pathname}${url.search}`;
  
  // 転送用ヘッダーを構築 (元の Content-Type 等を保持)
  const forwardHeaders = new Headers(request.headers);
  // 認証ヘッダーを LINE 用に差し替え
  forwardHeaders.set('Authorization', `Bearer ${channelAccessToken}`);
  // プロキシ専用ヘッダーは転送しない
  forwardHeaders.delete('X-Line-Channel-Access-Token');

  // リクエストボディを読み込み (バイナリ対応)
  let body;
  if (request.method !== 'GET' && request.method !== 'HEAD') {
    body = await request.arrayBuffer();
  }

  // LINE API にリクエストを転送
  const lineResponse = await fetch(lineUrl, {
    method: request.method,
    headers: forwardHeaders,
    body,
  });

  return lineResponse;
}
```

---

## 3. LINE Messaging API エンドポイント

### 3.1 リッチメニュー API

#### POST /richmenu

**リッチメニュー作成**

```http
POST /richmenu
X-Line-Channel-Access-Token: {token}
Content-Type: application/json

{
  "size": {
    "width": 2500,
    "height": 1686
  },
  "selected": false,
  "name": "Main Menu",
  "chatBarText": "メニュー",
  "areas": [
    {
      "bounds": {
        "x": 0,
        "y": 0,
        "width": 1250,
        "height": 843
      },
      "action": {
        "type": "uri",
        "label": "メニュー",
        "uri": "https://example.com/menu"
      }
    },
    {
      "bounds": {
        "x": 1250,
        "y": 0,
        "width": 1250,
        "height": 843
      },
      "action": {
        "type": "postback",
        "label": "予約",
        "data": "action=reservation"
      }
    }
  ]
}
```

**レスポンス**:
```json
{
  "richMenuId": "richmenu-1234567890abcdef"
}
```

---

#### GET /richmenu/{richMenuId}

**リッチメニュー取得**

```http
GET /richmenu/richmenu-1234567890abcdef
X-Line-Channel-Access-Token: {token}
```

---

#### GET /richmenu/{richMenuId}/content

**リッチメニュー画像取得**

```http
GET /richmenu/richmenu-1234567890abcdef/content
X-Line-Channel-Access-Token: {token}
```

**レスポンス**: Binary (PNG/JPEG)

---

#### PUT /richmenu/{richMenuId}/content

**リッチメニュー画像アップロード**

```http
PUT /richmenu/richmenu-1234567890abcdef/content
X-Line-Channel-Access-Token: {token}
Content-Type: image/jpeg

[binary data]
```

---

#### DELETE /richmenu/{richMenuId}

**リッチメニュー削除**

```http
DELETE /richmenu/richmenu-1234567890abcdef
X-Line-Channel-Access-Token: {token}
```

---

#### POST /richmenu/user/{userId}/link/{richMenuId}

**ユーザーにリッチメニューをリンク**

```http
POST /richmenu/user/U1234567890abcdef/link/richmenu-1234567890abcdef
X-Line-Channel-Access-Token: {token}
```

---

#### POST /richmenu/bulk-link

**バッチリンク**

```http
POST /richmenu/bulk-link
X-Line-Channel-Access-Token: {token}
Content-Type: application/json

{
  "richMenuId": "richmenu-1234567890abcdef",
  "userId": ["U1234567890abcdef", "U0987654321fedcba"]
}
```

---

### 3.2 メッセージ API

#### POST /message/push

**プッシュメッセージ送信**

```http
POST /message/push
X-Line-Channel-Access-Token: {token}
Content-Type: application/json

{
  "to": "U1234567890abcdef",
  "messages": [
    {
      "type": "text",
      "text": "こんにちは"
    }
  ]
}
```

---

#### POST /message/multicast

**マルチキャストメッセージ送信**

```http
POST /message/multicast
X-Line-Channel-Access-Token: {token}
Content-Type: application/json

{
  "to": ["U1234567890abcdef", "U0987654321fedcba"],
  "messages": [
    {
      "type": "text",
      "text": "こんにちは"
    }
  ]
}
```

---

### 3.3 ウェルカムメッセージ

#### POST /message/welcome

**ウェルカムメッセージ設定**

```http
POST /message/welcome
X-Line-Channel-Access-Token: {token}
Content-Type: application/json

{
  "richTextId": "rich-text-1234567890abcdef"
}
```

---

## 4. Rust クライアント実装

### 4.1 LINE API クライアント

```rust
// src/line/client.rs

use reqwest::Client;
use serde::{Deserialize, Serialize};

pub struct LineApiClient {
    client: Client,
    proxy_url: String,
    channel_access_token: String,
}

impl LineApiClient {
    pub fn new(proxy_url: &str, channel_access_token: &str) -> Self {
        Self {
            client: Client::new(),
            proxy_url: proxy_url.to_string(),
            channel_access_token: channel_access_token.to_string(),
        }
    }

    pub async fn create_rich_menu(&self, rich_menu: &RichMenuCreateRequest) -> Result<String, LineApiError> {
        let response = self.client
            .post(&format!("{}/richmenu", self.proxy_url))
            .header("X-Line-Channel-Access-Token", &self.channel_access_token)
            .json(rich_menu)
            .send()
            .await?;

        if !response.status().is_success() {
            return Err(LineApiError::from_response(response).await?);
        }

        let result: RichMenuCreateResponse = response.json().await?;
        Ok(result.rich_menu_id)
    }

    pub async fn get_rich_menu(&self, rich_menu_id: &str) -> Result<RichMenu, LineApiError> {
        let response = self.client
            .get(&format!("{}/richmenu/{}", self.proxy_url, rich_menu_id))
            .header("X-Line-Channel-Access-Token", &self.channel_access_token)
            .send()
            .await?;

        if !response.status().is_success() {
            return Err(LineApiError::from_response(response).await?);
        }

        Ok(response.json().await?)
    }

    pub async fn delete_rich_menu(&self, rich_menu_id: &str) -> Result<(), LineApiError> {
        let response = self.client
            .delete(&format!("{}/richmenu/{}", self.proxy_url, rich_menu_id))
            .header("X-Line-Channel-Access-Token", &self.channel_access_token)
            .send()
            .await?;

        if !response.status().is_success() {
            return Err(LineApiError::from_response(response).await?);
        }

        Ok(())
    }
}

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
pub struct RichMenuCreateRequest {
    pub size: RichMenuSize,
    pub selected: bool,
    pub name: String,
    #[serde(rename = "chatBarText")]
    pub chat_bar_text: Option<String>,
    pub areas: Vec<RichMenuArea>,
}

#[derive(Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct RichMenuCreateResponse {
    #[serde(rename = "richMenuId")]
    pub rich_menu_id: String,
}

#[derive(Serialize, Deserialize)]
pub struct RichMenuSize {
    pub width: u32,
    pub height: u32,
}

#[derive(Serialize, Deserialize)]
pub struct RichMenuArea {
    pub bounds: RichMenuBounds,
    pub action: RichMenuAction,
}

#[derive(Serialize, Deserialize)]
pub struct RichMenuBounds {
    pub x: u32,
    pub y: u32,
    pub width: u32,
    pub height: u32,
}

#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum RichMenuAction {
    #[serde(rename = "uri")]
    Uri { label: String, uri: String },
    #[serde(rename = "postback")]
    Postback { label: String, data: String, display_text: Option<String> },
    #[serde(rename = "message")]
    Message { label: String, text: String },
}

#[derive(Debug)]
pub enum LineApiError {
    Reqwest(reqwest::Error),
    ApiError { status: u16, message: String, detail: Option<String> },
}

impl From<reqwest::Error> for LineApiError {
    fn from(err: reqwest::Error) -> Self {
        LineApiError::Reqwest(err)
    }
}

impl LineApiError {
    async fn from_response(response: reqwest::Response) -> Result<Self, reqwest::Error> {
        let status = response.status().as_u16();
        let body: serde_json::Value = response.json().await?;
        
        let message = body.get("message").and_then(|v| v.as_str()).unwrap_or("Unknown error").to_string();
        let detail = body.get("detail").and_then(|v| v.as_str()).map(String::from);

        Ok(LineApiError::ApiError { status, message, detail })
    }
}
```

---

## 5. エラーハンドリング

### 5.1 LINE API エラーコード

| ステータス | 説明 |
|---|---|
| 400 | バッドリクエスト |
| 401 | 認証エラー (トークン無効) |
| 404 | リソース不存在 |
| 429 | リクエスト制限 |
| 500 | サーバーエラー |

### 5.2 エラーレスポンス形式

```json
{
  "message": "Invalid channel access token",
  "detail": "The channel access token is invalid."
}
```

---

## 6. セキュリティ

### 6.1 ヘッダー要件

| ヘッダー | 必須 | 説明 |
|---|---|---|
| `X-Line-Channel-Access-Token` | はい | チャネルアクセストークン |
| `Content-Type` (JSON API) | はい | `application/json` |
| `Content-Type` (リッチメニュー画像アップロード) | はい | `image/jpeg` |

### 6.2 CORS 設定

```javascript
// Worker で設定
'Access-Control-Allow-Origin': '*',
'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
'Access-Control-Allow-Headers': 'Content-Type, X-Line-Channel-Access-Token',
```

---

## 7. レート制限

| エンドポイント | 制限 |
|---|---|
| プッシュメッセージ | 2 秒間に 2 件、2 分間に 200 件 |
| マルチキャスト | 2 秒間に 1 件、2 分間に 50 件 |
| リッチメニュー API | 2 秒間に 1 件 |

**対応**: クライアント側でリトライロジックを実装

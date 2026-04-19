# THE HANGAR — API仕様書 v2.0

## 概要

| 項目 | 内容 |
|------|------|
| エンドポイント | **2本のみ**（POST 1本・GET 1本） |
| ベースURL | Logic Apps の HTTP トリガー URL（SWA環境変数で管理） |
| 分岐方式 | フロー内 Switch アクションで `action` 値により処理を切り替え |
| 認証 | LINE Login → SWA 組み込み認証 → Bearer Token |
| Content-Type | `application/json` |
| Logic Apps の責務 | **データ中継のみ**。ビジネスロジックは持たない |
| ポイント計算・レベル判定 | Power Automate（Dataverse トリガー）が処理 |
| ランキング表示 | Canvas App（Power Apps）が Dataverse に直接接続。Logic Apps 経由なし |

---

## エンドポイント一覧

```
POST https://<logic-apps-host>/api/data          # 書き込み系（Switch 4ケース）
GET  https://<logic-apps-host>/api/data          # 読み取り系（Switch 4ケース）
```

---

## POST /api/data — 書き込みフロー

### 共通リクエスト構造

```json
{
  "action": "<action値>",
  "payload": { ... }
}
```

Logic Apps の「JSON の解析」アクションに以下スキーマを使用する。

```json
{
  "type": "object",
  "properties": {
    "action":  { "type": "string" },
    "payload": { "type": "object" }
  },
  "required": ["action", "payload"]
}
```

---

### Case 1: auth_session

LINEログイン完了後に呼び出す。lineUserId で Account を upsert する。初回は新規作成、2回目以降は既存レコードを返す。

**リクエスト**
```json
{
  "action": "auth_session",
  "payload": {
    "lineUserId": "U4af4980ad3xxxxxxxxxxxxxxx"
  }
}
```

**レスポンス**
```json
{
  "accountId": "a1b2c3d4-0000-0000-0000-000000000001",
  "lineUserId": "U4af4980ad3xxxxxxxxxxxxxxx",
  "isNew": true
}
```

---

### Case 2: create_player

アカウントに紐づくプレイヤーを新規登録する。INSERT 前に同一 accountId のプレイヤー件数が 3 未満であることを Condition アクションで確認する。

**リクエスト**
```json
{
  "action": "create_player",
  "payload": {
    "accountId": "a1b2c3d4-0000-0000-0000-000000000001",
    "name": "たろう",
    "ageGroup": "child",
    "sortOrder": 1
  }
}
```

**レスポンス（201）**
```json
{
  "playerId": "p1b2c3d4-0000-0000-0000-000000000001",
  "accountId": "a1b2c3d4-0000-0000-0000-000000000001",
  "name": "たろう",
  "ageGroup": "child",
  "currentLevel": "beginner",
  "totalPoints": 0,
  "sortOrder": 1
}
```

**エラー（409）— プレイヤー上限超過**
```json
{
  "error": { "code": "PLAYER_LIMIT", "message": "プレイヤーは最大3名までです", "status": 409 }
}
```

---

### Case 3: mission_clear

ミッションクリアを記録する。INSERT 前に同一 `playerId × missionId × status=cleared` の重複レコードが存在しないことを Condition アクションで確認する。`clearMethod` の値は Mission テーブルから取得して status を決定する。

| clearMethod | 書き込む status | 備考 |
|-------------|----------------|------|
| auto | cleared | 親タップで即確定 |
| qr | cleared | jsQR 照合成功で即確定 |
| staff | pending | スタッフ承認待ち。Power Apps で承認後 cleared に更新 |

**リクエスト（auto / qr 共通）**
```json
{
  "action": "mission_clear",
  "payload": {
    "playerId": "p1b2c3d4-0000-0000-0000-000000000001",
    "missionId": "m1b2c3d4-0000-0000-0000-000000000001",
    "evidenceType": "qr",
    "evidenceData": "HANGAR-MISSION-m1b2c3d4",
    "clearedAt": "2025-04-19T14:30:00Z"
  }
}
```

**リクエスト（staff 方式）**
```json
{
  "action": "mission_clear",
  "payload": {
    "playerId": "p1b2c3d4-0000-0000-0000-000000000001",
    "missionId": "m1b2c3d4-0000-0000-0000-000000000002",
    "evidenceType": "staff",
    "evidenceData": "スタッフ目視確認",
    "clearedAt": "2025-04-19T15:00:00Z"
  }
}
```

**レスポンス（201）— auto / qr（即 cleared）**
```json
{
  "logId": "l1b2c3d4-0000-0000-0000-000000000001",
  "status": "cleared",
  "pointsEarned": 3,
  "totalPoints": 41,
  "levelUp": false,
  "newLevel": "middle"
}
```

> `levelUp: true` の場合、フロント側で昇格アニメーションを表示する。  
> ただし `pointsEarned` と `totalPoints` は Power Automate が非同期で更新するため、  
> レスポンス時点では Mission.points の値を Logic Apps が計算して返す（暫定値）。

**レスポンス（201）— staff（pending）**
```json
{
  "logId": "l1b2c3d4-0000-0000-0000-000000000002",
  "status": "pending",
  "pointsEarned": 0,
  "message": "スタッフの確認後にポイントが付与されます"
}
```

**エラー（409）— 重複クリア**
```json
{
  "error": { "code": "MISSION_ALREADY_CLEARED", "message": "このミッションはクリア済みです", "status": 409 }
}
```

---

### Case 4: shop_redeem

ポイント交換を申請する。INSERT 前に `ShopItem.stock > 0`（stock が null の場合は無制限）を Condition アクションで確認する。ポイントの引き落としは Power Automate が `RedeemLog.status = approved` 更新時に処理する。

**リクエスト**
```json
{
  "action": "shop_redeem",
  "payload": {
    "playerId": "p1b2c3d4-0000-0000-0000-000000000001",
    "itemId": "i1b2c3d4-0000-0000-0000-000000000001",
    "pointsUsed": 2
  }
}
```

**レスポンス（201）**
```json
{
  "redeemId": "r1b2c3d4-0000-0000-0000-000000000001",
  "status": "pending",
  "itemName": "ドリンク1杯",
  "pointsUsed": 2,
  "message": "スタッフにこの画面を見せてください"
}
```

**エラー（409）— 在庫なし**
```json
{
  "error": { "code": "ITEM_OUT_OF_STOCK", "message": "在庫がありません", "status": 409 }
}
```

**エラー（400）— ポイント不足**
```json
{
  "error": { "code": "INSUFFICIENT_POINTS", "message": "ポイントが不足しています", "status": 400 }
}
```

---

### Default: 未知の action

```json
{
  "error": { "code": "UNKNOWN_ACTION", "message": "不明なアクションです", "status": 400 }
}
```

---

## GET /api/data — 読み取りフロー

### 共通リクエスト構造

クエリパラメータで `action` を指定する。

```
GET /api/data?action=<action値>&<追加パラメータ>
```

---

### Case 1: get_players

accountId に紐づくプレイヤー一覧とクリア済みミッションIDを返す。

**リクエスト**
```
GET /api/data?action=get_players&accountId=a1b2c3d4-0000-0000-0000-000000000001
```

**レスポンス**
```json
{
  "players": [
    {
      "playerId": "p1b2c3d4-0000-0000-0000-000000000001",
      "name": "たろう",
      "ageGroup": "child",
      "currentLevel": "middle",
      "totalPoints": 38,
      "sortOrder": 1,
      "clearedMissionIds": [
        "m1b2c3d4-0000-0000-0000-000000000001",
        "m1b2c3d4-0000-0000-0000-000000000002"
      ]
    },
    {
      "playerId": "p1b2c3d4-0000-0000-0000-000000000002",
      "name": "はなこ",
      "ageGroup": "child",
      "currentLevel": "beginner",
      "totalPoints": 8,
      "sortOrder": 2,
      "clearedMissionIds": [
        "m1b2c3d4-0000-0000-0000-000000000001"
      ]
    }
  ]
}
```

> `clearedMissionIds` は `MissionClearLog WHERE status=cleared` を JOIN して返す。  
> フロント側でミッションカードの「クリア済み」表示に使用する。

---

### Case 2: get_missions

アクティブなミッション一覧を返す。フロント側で `requiredRank` によるフィルタ・表示制御を行う。

**リクエスト**
```
GET /api/data?action=get_missions
```

**レスポンス**
```json
{
  "missions": [
    {
      "missionId": "m1b2c3d4-0000-0000-0000-000000000001",
      "name": "的当てゲーム",
      "description": "トイドローンでお菓子を落とせ",
      "category": "初級",
      "requiredRank": "beginner",
      "points": 1,
      "clearMethod": "auto",
      "qrCode": null,
      "isEvent": false,
      "eventStart": null,
      "eventEnd": null,
      "sortOrder": 10
    },
    {
      "missionId": "m1b2c3d4-0000-0000-0000-000000000004",
      "name": "等高度撮影",
      "description": "Mini2で5分以内に完了",
      "category": "中級",
      "requiredRank": "middle",
      "points": 3,
      "clearMethod": "qr",
      "qrCode": "HANGAR-MISSION-m1b2c3d4-0000-0000-0000-000000000004",
      "isEvent": false,
      "eventStart": null,
      "eventEnd": null,
      "sortOrder": 40
    },
    {
      "missionId": "m1b2c3d4-0000-0000-0000-000000000010",
      "name": "第3回 HANGAR杯",
      "description": "パノラマ撮影部門",
      "category": "HANGAR杯",
      "requiredRank": "advanced",
      "points": 10,
      "clearMethod": "staff",
      "qrCode": null,
      "isEvent": true,
      "eventStart": "2025-04-01",
      "eventEnd": "2025-04-30",
      "sortOrder": 100
    }
  ]
}
```

---

### Case 3: get_shop

アクティブなショップアイテム一覧を返す。

**リクエスト**
```
GET /api/data?action=get_shop
```

**レスポンス**
```json
{
  "items": [
    {
      "itemId": "i1b2c3d4-0000-0000-0000-000000000001",
      "name": "ドリンク1杯",
      "requiredPoints": 2,
      "artType": "drink",
      "stock": null
    },
    {
      "itemId": "i1b2c3d4-0000-0000-0000-000000000002",
      "name": "ステッカー",
      "requiredPoints": 100,
      "artType": "sticker",
      "stock": 50
    },
    {
      "itemId": "i1b2c3d4-0000-0000-0000-000000000003",
      "name": "HS210 ドローン",
      "requiredPoints": 300,
      "artType": "drone",
      "stock": 5
    }
  ]
}
```

> `stock: null` は在庫無制限を意味する。

---

### Case 4: get_display

店内サイネージ（Canvas App）向けのプレイヤーランキングを返す。LINE 認証不要。`X-Display-Key` ヘッダーで保護する。Switch の最初の Condition アクションでキーを検証し、不一致の場合は 401 を返す。

**リクエスト**
```
GET /api/data?action=get_display&limit=20
```
```
Headers: X-Display-Key: <環境変数 DISPLAY_KEY の値>
```

**レスポンス**
```json
{
  "generatedAt": "2025-04-19T10:00:00Z",
  "players": [
    { "rank": 1, "name": "よしき",  "currentLevel": "advanced", "totalPoints": 142 },
    { "rank": 2, "name": "りなこ",  "currentLevel": "middle",   "totalPoints": 98  },
    { "rank": 3, "name": "だいき",  "currentLevel": "middle",   "totalPoints": 76  }
  ]
}
```

**エラー（401）— キー不一致**
```json
{
  "error": { "code": "UNAUTHORIZED", "message": "Display key が無効です", "status": 401 }
}
```

---

### Default: 未知の action

```json
{
  "error": { "code": "UNKNOWN_ACTION", "message": "不明なアクションです", "status": 400 }
}
```

---

## エラーレスポンス共通形式

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーの説明（日本語）",
    "status": 400
  }
}
```

### エラーコード一覧

| code | HTTP status | 発生ケース |
|------|-------------|-----------|
| UNAUTHORIZED | 401 | LINE 未ログイン、または Display Key 不一致 |
| UNKNOWN_ACTION | 400 | Switch の Default ケース（未定義 action） |
| PLAYER_NOT_FOUND | 404 | playerId が存在しない |
| PLAYER_LIMIT | 409 | プレイヤー登録が 3 名を超過 |
| MISSION_ALREADY_CLEARED | 409 | 同一 playerId × missionId の cleared レコードが存在 |
| INSUFFICIENT_POINTS | 400 | 交換に必要なポイントが不足 |
| ITEM_OUT_OF_STOCK | 409 | ShopItem.stock が 0 |
| RANK_LOCKED | 403 | requiredRank を満たさないミッションへのクリア申請 |

---

## Logic Apps 実装メモ

### JSON スキーマ登録手順（「JSON の解析」アクション）

POST フローでは HTTP トリガーの `body` を以下スキーマで解析する。

```json
{
  "type": "object",
  "properties": {
    "action": { "type": "string" },
    "payload": {
      "type": "object",
      "properties": {
        "lineUserId":    { "type": "string" },
        "accountId":     { "type": "string" },
        "playerId":      { "type": "string" },
        "missionId":     { "type": "string" },
        "itemId":        { "type": "string" },
        "name":          { "type": "string" },
        "ageGroup":      { "type": "string" },
        "sortOrder":     { "type": "integer" },
        "evidenceType":  { "type": "string" },
        "evidenceData":  { "type": "string" },
        "clearedAt":     { "type": "string" },
        "pointsUsed":    { "type": "integer" }
      }
    }
  },
  "required": ["action", "payload"]
}
```

GET フローでは `triggerOutputs()?['queries']['action']` で action 値を取得する。

### Switch アクションの設定値

| フロー | Switch の式 | Case 値 |
|--------|------------|---------|
| POST | `@body('JSON_の解析')['action']` | `auth_session` / `create_player` / `mission_clear` / `shop_redeem` |
| GET | `@triggerOutputs()?['queries']['action']` | `get_players` / `get_missions` / `get_shop` / `get_display` |

### Power Automate との連携

Logic Apps は Dataverse への INSERT のみ行う。INSERT 完了後、以下のフローが自動起動する。

| Dataverse の変化 | 起動するフロー | 処理内容 |
|-----------------|--------------|---------|
| MissionClearLog: status → cleared | ポイント加算フロー | Player.totalPoints に Mission.points を加算。レベル判定・バッジ付与 |
| RedeemLog: status → approved | ポイント引落フロー | Player.totalPoints から RedeemLog.pointsUsed を減算 |

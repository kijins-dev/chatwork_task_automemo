# Chatwork タスクBot 自動メモ

Chatwork BotへのDM送信でObsidianにタスクを自動登録するシステム

## 概要

スマホからChatwork Botアカウントにタスクメモを送ると、n8n経由でObsidianタスク専用ファイルに自動追記される。

## システム構成

```
スマホ
  ↓ DM送信
Chatwork Botアカウント
  ↓ Webhook
n8n (Webhook受信)
  ↓ 整形処理
Obsidian (タスクファイルに追記)
```

## ファイル構成

```
00_タスクBot/
├── 未完了タスク.md  ← タスク登録先
└── 完了タスク.md    ← 完了タスク移動先（ワークフロー2で自動移動）
```

## ワークフロー

### ワークフロー1: タスク受信・保存（完成）
- Chatwork Webhook → n8n → Obsidian追記
- 保存先: `00_タスクBot/未完了タスク.md`
- 形式: `- [ ] タスク内容 (M月D日 HH:MM)`

### ワークフロー2: 毎朝タスク処理（予定）
- 毎朝10時実行
- `- [x]` の行を `完了タスク.md` に移動
- 未完了タスク一覧をChatworkに送信

## 設定情報

| 項目 | 値 |
|------|---|
| Chatwork Bot DMルーム | 420024236 |
| n8n Webhook URL | https://michi-gaeru.app.n8n.cloud/webhook/chatwork-task-bot |
| Obsidian保存先 | 00_タスクBot/ |
| Chatwork Webhook設定ID | 34033 |

## n8nワークフロー構成

### ワークフロー1: タスクBot - 受信・保存

```
Webhook (POST)
  ↓
Respond to Webhook (即時応答)
  ↓
メッセージ整形 (Code)
  ↓
タスク追記 (HTTP Request POST)
```

## 技術詳細

### メッセージ整形ノード（JavaScript）

```javascript
const body = $input.first().json.body;
const webhookEventType = body.webhook_event_type;
if (webhookEventType !== 'message_created') {
  return [];
}
const messageBody = body.webhook_event.body;
const sendTime = body.webhook_event.send_time;
const accountId = body.webhook_event.account_id;

// UTC → JST（+9時間）
const date = new Date(sendTime * 1000 + 9 * 60 * 60 * 1000);
const month = date.getMonth() + 1;
const day = date.getDate();
const hours = String(date.getHours()).padStart(2, '0');
const minutes = String(date.getMinutes()).padStart(2, '0');
const dateStr = `${month}月${day}日`;
const timeStr = `${hours}:${minutes}`;
const taskLine = `- [ ] ${messageBody} (${dateStr} ${timeStr})`;
return [{
  json: {
    taskLine,
    messageBody,
    accountId
  }
}];
```

### タスク追記ノード設定

| 設定 | 値 |
|------|---|
| Method | POST |
| URL | `https://naoki-obsidian.ngrok.io/vault/00_タスクBot/未完了タスク.md` |
| Authentication | Header Auth (Obsidian API) |
| Content-Type | text/markdown |
| Body | `{{ $json.taskLine + "\n" }}` |

## 開発履歴

### 2026-01-14
- ワークフロー1完成（受信・保存機能）
- タイムゾーン修正（UTC → JST）
- デイリーノート方式からタスク専用ファイル方式に変更
- 日付形式を「M月D日 HH:MM」に変更

## 関連リンク

- [n8nワークフロー](https://michi-gaeru.app.n8n.cloud/)
- [Obsidian情報集約プロジェクト](https://github.com/kijins-dev/obsidian_summary)

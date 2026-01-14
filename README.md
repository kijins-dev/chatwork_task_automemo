# Chatwork タスクBot 自動メモ

Chatwork BotへのDM送信でObsidianにタスクを自動登録するシステム

## 概要

スマホからChatwork Botアカウントにタスクメモを送ると、n8n経由でObsidianデイリーノートにタスクとして自動追記される。

## システム構成

```
スマホ
  ↓ DM送信
Chatwork Botアカウント
  ↓ Webhook
n8n (Webhook受信)
  ↓ 整形処理
Obsidian (デイリーノートに追記)
```

## ワークフロー

### ワークフロー1: タスク受信・保存（完成）
- Chatwork Webhook → n8n → Obsidian追記
- 保存先: `01_デイリーノート/YYYY-MM-DD.md`
- 形式: `## タスクBot` セクション配下にチェックボックス形式

### ワークフロー2: 毎朝タスク一覧送信（予定）
- 毎朝10時に未完了タスク一覧をChatworkに送信

## 設定情報

| 項目 | 値 |
|------|---|
| Chatwork Bot DMルーム | 420024236 |
| n8n Webhook URL | https://michi-gaeru.app.n8n.cloud/webhook/chatwork-task-bot |
| Obsidian保存先 | 01_デイリーノート/ |

## n8nワークフロー構成

### ワークフロー1: タスクBot - 受信・保存

```
Webhook (POST)
  ↓
Respond to Webhook (即時応答)
  ↓
メッセージ整形 (Code)
  ↓
Obsidian追記 (HTTP Request POST)
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
const year = date.getFullYear();
const month = String(date.getMonth() + 1).padStart(2, '0');
const day = String(date.getDate()).padStart(2, '0');
const hours = String(date.getHours()).padStart(2, '0');
const minutes = String(date.getMinutes()).padStart(2, '0');
const dateStr = `${year}-${month}-${day}`;
const timeStr = `${hours}:${minutes}`;
const taskLine = `- [ ] ${messageBody} (${timeStr})`;
return [{
  json: {
    dateStr,
    taskLine,
    messageBody,
    accountId
  }
}];
```

### Obsidian追記ノード設定

| 設定 | 値 |
|------|---|
| Method | POST |
| URL | `https://naoki-obsidian.ngrok.io/vault/01_デイリーノート/{{ $json.dateStr }}.md` |
| Authentication | Header Auth (Obsidian API) |
| Content-Type | text/markdown |
| Body | `{{ "\n\n## タスクBot\n" + $json.taskLine }}` |

## 開発履歴

### 2026-01-14
- ワークフロー1完成（受信・保存機能）
- タイムゾーン修正（UTC → JST）
- Chatwork返信機能は不要と判断し削除

## 関連リンク

- [n8nワークフロー](https://michi-gaeru.app.n8n.cloud/)
- [Obsidian情報集約プロジェクト](https://github.com/kijins-dev/obsidian_summary)

# Chatwork タスクBot 自動メモ

Chatwork BotへのDM送信でObsidianにタスクを自動登録するシステム

## 概要

スマホからChatwork Botアカウントにタスクメモを送ると、n8n経由でObsidianタスク専用ファイルに自動追記される。
毎朝10時に完了タスクを整理し、未完了タスク一覧をChatworkに通知する。

## システム構成

```
【タスク登録】
スマホ
  ↓ DM送信
Chatwork Botアカウント (ルーム: 420024236)
  ↓ Webhook
n8n (ワークフロー1)
  ↓ 整形処理
Obsidian (未完了タスク.mdに追記)

【毎朝処理】
n8n (ワークフロー2) 毎朝10時JST
  ↓
Obsidian読み込み
  ↓
完了タスク移動 + 未完了タスク整理
  ↓
Chatwork通知 (ルーム: 262320255)
```

## ファイル構成

```
00_タスクBot/
├── 未完了タスク.md  ← タスク登録先
└── 完了タスク.md    ← 完了タスク移動先（ワークフロー2で自動移動）
```

## ワークフロー

### ワークフロー1: タスク受信・保存（稼働中）
- Chatwork Webhook → n8n → Obsidian追記
- 保存先: `00_タスクBot/未完了タスク.md`
- 形式: `- [ ] タスク内容 (M月D日 HH:MM)`

### ワークフロー2: 毎朝タスク処理（稼働中）
- 毎朝10時(JST)実行
- `- [x]` の行を `完了タスク.md` に移動
- 未完了タスク一覧をChatworkに送信（To:674453付き）

## 設定情報

| 項目 | 値 |
|------|---|
| Chatwork Bot DMルーム（受信用） | 420024236 |
| Chatwork 通知送信先ルーム | 262320255 |
| n8n Webhook URL | https://michi-gaeru.app.n8n.cloud/webhook/chatwork-task-bot |
| Obsidian保存先 | 00_タスクBot/ |
| Chatwork Webhook設定ID | 34033 |
| 自分のアカウントID | 674453 |

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

### ワークフロー2: タスクBot - 毎朝タスク処理

```
毎朝10時(JST) = UTC 1時
  ↓
未完了タスク読み込み
  ↓
タスク分類（完了/未完了）
  ↓
┌─ 完了タスクあり? ─┐
│Yes               │No
↓                  ↓
完了タスク.md読込   （スキップ）
  ↓                
ファイル内容準備 ──────┐
  ↓                   ↓
完了タスク保存     未完了タスク更新
  ↓                   ↓
└──────────────────────┘
  ↓
Chatworkメッセージ作成
  ↓
Chatwork送信
```

**重要**: ファイル内容準備から完了タスク保存・未完了タスク更新は**並列接続**。直列だとデータが渡らない。

## 技術詳細

### ワークフロー1: メッセージ整形ノード

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

### ワークフロー2: Chatworkメッセージ作成ノード

```javascript
// タスク分類ノードからデータ取得
const taskData = $('タスク分類').first().json;
const incompleteTasks = taskData.incompleteTasks;

// Chatworkメッセージ作成
let message = '[To:674453]\n';

if (incompleteTasks.length === 0) {
  message += '[info][title]本日のタスク[/title]未完了タスクはありません[/info]';
} else {
  const taskList = incompleteTasks.map(task => {
    return task.replace(/^- \[ \] /, '');
  }).join('\n');
  
  message += '[info][title]本日のタスク (' + incompleteTasks.length + '件)[/title]' + taskList + '[/info]';
}

return [{ json: { message: message } }];
```

### Obsidian REST API設定（重要）

HTTP RequestノードでObsidianにPUTする際：

```
Body Content Type: Raw
Content Type: text/markdown  ← 必須
```

⚠️ `text/html` だとファイルが `{}` だけになり内容が消失する。

### スケジュール設定

| 設定 | 値 | 説明 |
|------|---|------|
| Cron Expression | `0 1 * * *` | UTC 1時 = JST 10時 |

## 開発履歴

### 2026-01-14
- ワークフロー1完成（受信・保存機能）
- タイムゾーン修正（UTC → JST）
- デイリーノート方式からタスク専用ファイル方式に変更
- 日付形式を「M月D日 HH:MM」に変更
- ワークフロー2完成（毎朝タスク処理）
- Chatwork通知にTo:674453追加
- 通知送信先を262320255に変更

### 2026-01-15
- Content-Type問題を修正（text/html → text/markdown）
- ノード接続を直列から並列に修正
- 不要なデイリーノート処理を削除

## 関連リンク

- [n8nワークフロー](https://michi-gaeru.app.n8n.cloud/)
- [Obsidian情報集約プロジェクト](https://github.com/kijins-dev/obsidian_summary)

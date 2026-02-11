---
title: "スクショを撮るだけでClaude Codeにエラー画面を即共有する方法【macOS】"
emoji: "📸"
type: "tech"
topics: ["claudecode", "macos", "automation", "cli", "productivity"]
published: true
---

# スクショを撮るだけでClaude Codeにエラー画面を即共有する方法

## こんな経験ありませんか？

エラーが出た。Claude Codeに相談したい。でも...

```
1. スクショを撮る
2. 保存先を開く
3. ファイルパスをコピーする
4. Claude Codeに貼り付ける
```

**たった4ステップなのに、毎回やると地味に面倒。**

この記事では、スクショを撮った瞬間にファイルパスが自動でクリップボードにコピーされ、Claude Codeに `⌘V` で即共有できる仕組みを紹介します。

## 完成イメージ

```
⌘⇧3 でスクショ撮影
    ↓ 自動でパスがクリップボードにコピー（通知も表示）
Claude Code で ⌘V
    ↓ パスが貼り付けられる
「このエラーの原因は？」と送信
    ↓
Claude Code が画像を読み取って回答
```

**所要時間：約3秒。**

## セットアップ（3ステップ）

### Step 1: スクショ専用フォルダを作成

デスクトップがスクショだらけになるのを防ぐため、専用フォルダに保存先を変更します。

```bash
# スクショ専用フォルダを作成
mkdir -p ~/Desktop/Screenshots

# macOSのスクショ保存先を変更
defaults write com.apple.screencapture location ~/Desktop/Screenshots

# 設定を反映
killall SystemUIServer
```

### Step 2: 監視スクリプトを作成

スクショフォルダを監視して、新しいファイルが追加されたら自動でパスをクリップボードにコピーするスクリプトです。

まず、ファイル監視ツール `fswatch` をインストールします。

```bash
brew install fswatch
```

次に、監視スクリプトを作成します。

```bash
cat << 'EOF' > ~/.claude/screenshot-watcher.sh
#!/bin/bash
# Screenshot watcher: monitors ~/Desktop/Screenshots for new screenshots
# and copies the file path to clipboard automatically

WATCH_DIR="$HOME/Desktop/Screenshots"

/opt/homebrew/bin/fswatch -0 "$WATCH_DIR" | while IFS= read -r -d '' file; do
  if [[ "$file" == *.png ]] && [[ -f "$file" ]]; then
    sleep 0.5
    /usr/bin/osascript -e "set the clipboard to \"$file\""
    /usr/bin/osascript -e "display notification \"パスをコピーしました\" with title \"Screenshot Watcher\" sound name \"Tink\""
  fi
done
EOF

chmod +x ~/.claude/screenshot-watcher.sh
```

:::message
**ポイント**: `pbcopy` ではなく `osascript` でクリップボード操作しています。`pbcopy` はバックグラウンドサービス（launchd）から実行するとGUIセッションにアクセスできず失敗するためです。
:::

### Step 3: ログイン時に自動起動する設定

Mac起動時に自動で監視が始まるよう、launchdエージェントを登録します。

```bash
cat << 'EOF' > ~/Library/LaunchAgents/com.screenshot-watcher.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.screenshot-watcher</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>~/.claude/screenshot-watcher.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/screenshot-watcher.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/screenshot-watcher.err</string>
</dict>
</plist>
EOF
```

:::message alert
**注意**: plistファイル内の `~/.claude/screenshot-watcher.sh` は、実際にはフルパス（例: `/Users/yourname/.claude/screenshot-watcher.sh`）に書き換えてください。launchdは `~` を展開しません。
:::

サービスを起動します。

```bash
launchctl load ~/Library/LaunchAgents/com.screenshot-watcher.plist
```

## 動作確認

```bash
# スクショを撮る（⌘⇧3 または ⌘⇧4）

# クリップボードの中身を確認
pbpaste
# => /Users/yourname/Desktop/Screenshots/スクリーンショット 2026-02-11 20.58.33.png
```

パスが表示されればOKです。

## 使い方

### エラー画面を即相談

```
> ⌘V
> このエラーの原因と解決方法を教えて
```

Claude Codeは画像を読み取れるので、エラーメッセージをテキストで打ち込む必要がありません。

### UIの不具合を報告

```
> ⌘V
> このレイアウト崩れの原因を調べて
```

### デザイン比較

```
> ⌘V（期待するデザイン）
> ⌘V（現在の状態）
> この2つの差分を修正して
```

## トラブルシューティング

### パスがコピーされない場合

```bash
# サービスの状態を確認
launchctl list | grep screenshot-watcher

# ログを確認
cat /tmp/screenshot-watcher.err

# fswatch のパスを確認（Apple Silicon Mac の場合）
which fswatch
# => /opt/homebrew/bin/fswatch

# Intel Mac の場合はパスが異なるので、スクリプト内のパスを修正
# /usr/local/bin/fswatch
```

### サービスを再起動したい場合

```bash
launchctl unload ~/Library/LaunchAgents/com.screenshot-watcher.plist
launchctl load ~/Library/LaunchAgents/com.screenshot-watcher.plist
```

### 通知が表示されない場合

**システム設定 → 通知 → スクリプトエディタ** の通知が許可されているか確認してください。

## 仕組みの解説

```
┌─────────────────────────────┐
│  macOS スクリーンショット     │
│  (⌘⇧3 / ⌘⇧4)              │
└──────────┬──────────────────┘
           │ .png ファイル保存
           ▼
┌─────────────────────────────┐
│  ~/Desktop/Screenshots/     │
│  fswatch が常時監視中        │
└──────────┬──────────────────┘
           │ 新規 .png を検知
           ▼
┌─────────────────────────────┐
│  osascript でクリップボードに │
│  フルパスを自動コピー         │
│  + macOS 通知を表示          │
└──────────┬──────────────────┘
           │ ⌘V
           ▼
┌─────────────────────────────┐
│  Claude Code                │
│  画像を読み取って回答        │
└─────────────────────────────┘
```

## まとめ

- **fswatch** でスクショフォルダを監視
- **osascript** でパスをクリップボードに自動コピー
- **launchd** でMac起動時に自動開始
- Claude Codeに **⌘V** するだけでスクショ共有完了

エラー画面の相談、UI確認、デザイン比較など、開発中のビジュアル情報をClaude Codeに渡すハードルが劇的に下がります。

:::message
**Windows版は現在準備中です。** 追って追記予定です。
:::

## 参考

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [fswatch - GitHub](https://github.com/emcrisostomo/fswatch)

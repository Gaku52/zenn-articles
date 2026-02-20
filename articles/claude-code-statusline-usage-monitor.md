---
title: "Claude Codeで突然のレート制限、もう怖くない。使用率をフッターに常時表示する方法"
emoji: "🔋"
type: "tech"
topics: ["claudecode", "cli", "productivity", "macos", "automation"]
published: true
---

# Claude Codeのレート制限、「突然来る」問題を解決する

Claude Code（CLI）のヘビーユーザーなら、一度はこの画面を見たことがあるはずです。

:::message alert
Rate limit exceeded. Please wait before making another request.
:::

**前触れなく、突然来る。** 今どれくらい使っているのか、あとどれくらい使えるのか — Claude Codeには、それを常時確認する手段がありません。

Desktopアプリには使用率の表示がありますが、CLIにはない。

**ないなら、作ります。**

この記事では、Claude Codeの `statusLine` 機能を使って、**フッターにAPI使用率をリアルタイム表示する仕組み**をコピペだけで構築します。セットアップは2ステップ、所要時間は約2分です。

## Before / After

**Before** — フッターには何も表示されない。制限が来るまで気づけない。

```
╭──────────────────────────────────────────────────╮
│  claude >                                        │
│                                                  │
╰──────────────────────────────────────────────────╯
```

**After** — 3つの使用率が常にフッターに見える。

```
╭──────────────────────────────────────────────────╮
│  claude >                                        │
│                                                  │
╰── Ctx: 45% | 5h: 5% | 7d: 1% ──────────╯
```

| 指標 | 意味 | 更新タイミング |
|------|------|--------------|
| **Ctx** | コンテキストウィンドウの使用率（会話の長さ） | レスポンスごと（リアルタイム） |
| **5h** | 5時間窓のレート制限使用率（Desktopの「現在のセッション」に相当） | 60秒キャッシュ |
| **7d** | 7日間窓のレート制限使用率（Desktopの「週間制限」に相当） | 60秒キャッシュ |

チラッと見るだけで「まだ余裕がある」「そろそろ控えよう」「`/compact` すべき」が判断できます。

## 必要なもの

| 項目 | 詳細 |
|------|------|
| OS | macOS / Windows / Linux |
| Claude Code | インストール済み・ログイン済み |
| プラン | Max Plan または Pro Plan |
| jq | JSON処理コマンド |

```bash
# macOS
brew install jq

# Linux (Debian/Ubuntu)
sudo apt install jq

# Windows (PowerShellで実行)
winget install jqlang.jq
```

## セットアップ — コピペ2ステップ

### Step 1. ステータスラインスクリプトを作る

```bash
cat << 'SCRIPT' > ~/.claude/statusline.sh
#!/usr/bin/env bash
set -euo pipefail

input=$(cat)

CACHE_FILE="/tmp/claude-usage-cache.json"
CACHE_TTL=60

ctx_pct="N/A"
five_h_pct="N/A"
seven_d_pct="N/A"

# --- Ctx: コンテキストウィンドウ使用率（stdinからリアルタイム） ---
if command -v jq >/dev/null 2>&1 && [[ -n "$input" ]]; then
  ctx_pct="$(echo "$input" | jq -r '.context_window.used_percentage // "N/A"')%"
fi

# --- 5h / 7d: APIからリアルタイム取得（60秒キャッシュ） ---
get_token() {
  # macOS: Keychainから取得
  if command -v security >/dev/null 2>&1; then
    local creds
    creds=$(security find-generic-password -s "Claude Code-credentials" -a "$(whoami)" -w 2>/dev/null) || return 1
    echo "$creds" | jq -r '.claudeAiOauth.accessToken // empty'
    return
  fi

  # Windows / Linux: .credentials.json から取得
  local cred_file="$HOME/.claude/.credentials.json"
  if [[ -f "$cred_file" ]]; then
    jq -r '.claudeAiOauth.accessToken // empty' "$cred_file"
    return
  fi

  return 1
}

fetch_usage() {
  local token
  token=$(get_token) || return 1
  [[ -z "$token" ]] && return 1

  curl --silent --max-time 5 \
    --header "Authorization: Bearer $token" \
    --header "anthropic-beta: oauth-2025-04-20" \
    "https://api.anthropic.com/api/oauth/usage" 2>/dev/null
}

# キャッシュの経過時間を取得（macOS / Linux 両対応）
get_file_age() {
  local file="$1"
  local mtime
  mtime=$(stat -f%m "$file" 2>/dev/null || stat -c%Y "$file" 2>/dev/null || echo 0)
  echo $(( $(date +%s) - mtime ))
}

usage_json=""

# キャッシュが有効ならそこから読む
if [[ -f "$CACHE_FILE" ]] && command -v jq >/dev/null 2>&1; then
  cache_age=$(get_file_age "$CACHE_FILE")
  if (( cache_age < CACHE_TTL )); then
    usage_json=$(cat "$CACHE_FILE")
  fi
fi

# キャッシュが古いか無ければAPI取得
if [[ -z "$usage_json" ]] && command -v jq >/dev/null 2>&1; then
  usage_json=$(fetch_usage) || usage_json=""
  if [[ -n "$usage_json" ]] && echo "$usage_json" | jq -e '.five_hour' >/dev/null 2>&1; then
    echo "$usage_json" > "$CACHE_FILE"
  else
    usage_json=""
  fi
fi

# JSONからパーセンテージ取得
if [[ -n "$usage_json" ]] && command -v jq >/dev/null 2>&1; then
  five_hour=$(echo "$usage_json" | jq -r '.five_hour.utilization // "N/A"')
  seven_day=$(echo "$usage_json" | jq -r '.seven_day.utilization // "N/A"')

  if [[ "$five_hour" != "N/A" && "$five_hour" != "null" ]]; then
    five_h_pct="${five_hour%.*}%"
  fi
  if [[ "$seven_day" != "N/A" && "$seven_day" != "null" ]]; then
    seven_d_pct="${seven_day%.*}%"
  fi
fi

echo "Ctx: ${ctx_pct} | 5h: ${five_h_pct} | 7d: ${seven_d_pct}"
SCRIPT

chmod +x ~/.claude/statusline.sh
```

### Step 2. settings.json に登録する

`~/.claude/settings.json` を開いて、`statusLine` を追加します。

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash ~/.claude/statusline.sh"
  }
}
```

:::message
**既に settings.json がある場合**は、既存の設定を消さずに `statusLine` ブロックだけを追加してください。

```json
{
  "permissions": {
    "allow": ["Bash(*)", "Write", "Edit", "Read"]
  },
  "statusLine": {
    "type": "command",
    "command": "bash ~/.claude/statusline.sh"
  }
}
```
:::

**以上です。** Claude Codeを再起動すると、フッターに使用率が常時表示されます。

## 動作確認

```bash
bash ~/.claude/statusline.sh < /dev/null
# => Ctx: N/A | 5h: 5% | 7d: 1%
```

:::message
手動実行時は stdin がないため Ctx は N/A になります。Claude Code上では正常にリアルタイム表示されます。
:::

## どう動いているのか

このスクリプトは2つのデータソースを組み合わせています。

```
 ┌─────────────────────────────────────────────┐
 │  データソース 1: stdin（Claude Codeが提供）   │
 │  → コンテキストウィンドウ使用率（Ctx）        │
 │  → レスポンスごとにリアルタイム更新           │
 └──────────────────┬──────────────────────────┘
                    │
                    ▼
 ┌─────────────────────────────────────────────┐
 │  statusline.sh                              │
 │  2つのソースを統合して1行に出力              │
 └──────────────────┬──────────────────────────┘
                    ▲
                    │
 ┌──────────────────┴──────────────────────────┐
 │  データソース 2: Anthropic API               │
 │  → 5h（5時間窓）/ 7d（7日間窓）              │
 │  → 60秒キャッシュで負荷を最小化             │
 └─────────────────────────────────────────────┘
```

### Ctx（コンテキストウィンドウ）

Claude CodeはステータスラインスクリプトにstdinでJSON データを渡します。この中に `context_window.used_percentage` が含まれており、**会話コンテキストが上限200Kトークンに対して何%埋まっているか**をリアルタイムで取得できます。

100%に近づいたら `/compact` で圧縮するか、新しいセッションを始める判断材料になります。

### 5h / 7d（レート制限）

Anthropic APIの非公開エンドポイント `https://api.anthropic.com/api/oauth/usage` から、Desktopアプリと同じレート制限データを取得しています。

```json
{
  "five_hour": {
    "utilization": 5.0,
    "resets_at": "2026-02-20T10:00:00Z"
  },
  "seven_day": {
    "utilization": 1.0,
    "resets_at": "2026-02-27T05:00:00Z"
  }
}
```

認証にはClaude Codeが保存しているOAuthトークンを使用します。

| OS | トークンの保存先 |
|----|----------------|
| **macOS** | Keychain（`security find-generic-password` で取得） |
| **Windows / Linux** | `~/.claude/.credentials.json` |

自分のアカウントの使用量を、自分の認証情報で読み取っているだけです。このエンドポイントは使用量の**参照のみ**を行うもので、Messages APIのようなトークン消費や追加課金は発生しません。スクレイピングとも根本的に異なります。

**60秒キャッシュ**を実装しており、APIへの過度なリクエストを防いでいます。

## 応用: メモリ監視フックとの組み合わせ

使用率の監視に加えて、**メモリ使用量を監視するフック**を併用すると、長時間セッションの安定性が大きく向上します。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/memory-guard.sh",
            "timeout": 10000
          }
        ]
      }
    ]
  },
  "statusLine": {
    "type": "command",
    "command": "bash ~/.claude/statusline.sh"
  }
}
```

| 機能 | 監視対象 | 役割 |
|------|---------|------|
| **statusline** | コンテキスト + レート制限 | 使いすぎの予防 |
| **memory-guard** | Node.jsメモリ使用量 | セッション不安定の予防 |

メモリ監視フック（memory-guard）はOSSとして公開しています。

> [claude-code-memory-guard - GitHub](https://github.com/Gaku52/claude-code-memory-guard)

## 表示のカスタマイズ

`statusline.sh` の最終行を変えるだけで、好みのフォーマットにできます。

```bash
# デフォルト（3指標）
echo "Ctx: ${ctx_pct} | 5h: ${five_h_pct} | 7d: ${seven_d_pct}"

# レート制限のみ
echo "5h: ${five_h_pct} | 7d: ${seven_d_pct}"

# コンパクト
echo "${ctx_pct}/${five_h_pct}/${seven_d_pct}"
```

## トラブルシューティング

### 「N/A」と表示される

| 原因 | 対処 |
|------|------|
| `jq` が未インストール | 下記のインストールコマンドを実行 |
| Claude Codeに未ログイン | `claude` を起動してログイン |
| OAuthトークンの期限切れ | Claude Code上で `/login` を実行 |

```bash
# デバッグ実行（全OS共通）
bash -x ~/.claude/statusline.sh < /dev/null
```

**macOSの場合:**
```bash
# キーチェーンにトークンがあるか確認
security find-generic-password -s "Claude Code-credentials" -a "$(whoami)" 2>/dev/null && echo "OK" || echo "NOT FOUND"

# APIを直接叩いて確認
CRED=$(security find-generic-password -s "Claude Code-credentials" -a "$(whoami)" -w)
TOKEN=$(echo "$CRED" | jq -r '.claudeAiOauth.accessToken')
curl --silent --header "Authorization: Bearer $TOKEN" \
  --header "anthropic-beta: oauth-2025-04-20" \
  "https://api.anthropic.com/api/oauth/usage" | jq .
```

**Windows / Linuxの場合:**
```bash
# 認証ファイルが存在するか確認
ls -la ~/.claude/.credentials.json 2>/dev/null && echo "OK" || echo "NOT FOUND"

# APIを直接叩いて確認
TOKEN=$(jq -r '.claudeAiOauth.accessToken' ~/.claude/.credentials.json)
curl --silent --header "Authorization: Bearer $TOKEN" \
  --header "anthropic-beta: oauth-2025-04-20" \
  "https://api.anthropic.com/api/oauth/usage" | jq .
```

### フッターが表示されない

```bash
# settings.json の構文チェック
jq . ~/.claude/settings.json
```

エラーが出たらJSON構文に問題があります。カンマの過不足を確認してください。

## 注意事項

:::message alert
**この記事で使用しているAPIエンドポイントについて**

`https://api.anthropic.com/api/oauth/usage` は、Anthropicが公式ドキュメントで公開していない**非公開（undocumented）のベータAPI**です。以下の点にご注意ください。

1. **予告なく変更・廃止される可能性があります。** エンドポイントのURL、レスポンス形式、認証方式がいつ変わってもおかしくありません。
2. **公式にサポートされていません。** 問題が発生してもAnthropicのサポート対象外です。
3. **将来、公式機能として実装される可能性があります。** Claude CodeのGitHubには同様のFeature Requestが多数提出されており（[#20636](https://github.com/anthropics/claude-code/issues/20636)、[#18121](https://github.com/anthropics/claude-code/issues/18121)、[#19385](https://github.com/anthropics/claude-code/issues/19385) 等）、statusLineのstdinにレート制限データが追加されれば、このワークアラウンドは不要になります。

**公式対応がされるまでの「つなぎ」としてご利用ください。** APIの仕様変更等で動作しなくなった場合は、本記事を随時修正します。
:::

:::message
**検証環境について**

本記事のスクリプトは**macOSで動作検証済み**です。Windows / Linuxについてはクロスプラットフォーム対応のコードを実装していますが、未検証です。環境固有の問題が発生した場合は、コメントでお知らせください。
:::

## まとめ

**2つのファイルを置くだけ**で、Claude Codeのフッターに使用率がリアルタイム表示されます。

| ファイル | 役割 |
|---------|------|
| `~/.claude/statusline.sh` | 使用率を取得・表示するスクリプト |
| `~/.claude/settings.json` | Claude Codeへの登録 |

| 指標 | データソース | 正確性 |
|------|------------|--------|
| **Ctx** | stdin（Claude Code提供） | リアルタイム・完全に正確 |
| **5h** | Anthropic API | リアルタイム・Desktopの「現在のセッション」と同じ値 |
| **7d** | Anthropic API | リアルタイム・Desktopの「週間制限」と同じ値 |

レート制限に怯えながら作業する必要はもうありません。

## 参考

- [Claude Code 公式ドキュメント](https://docs.anthropic.com/en/docs/claude-code)
- [claude-code-memory-guard](https://github.com/Gaku52/claude-code-memory-guard)

**先駆者のワークアラウンド（本記事の実装はこれらを参考にしています）:**
- [@lexfrei - Claude Code statusline with real usage limits](https://gist.github.com/lexfrei/b70aaee919bdd7164f2e3027dc8c98de)
- [@patyearone - Claude Code Status Line: Complete guide with gotchas](https://gist.github.com/patyearone/7c753ef536a49839c400efaf640e17de)

**関連するFeature Request（公式対応を待つ声）:**
- [Expose rate limit usage to statusLine configuration (#20636)](https://github.com/anthropics/claude-code/issues/20636)
- [Expose rate limit/session usage data to statusLine (#18121)](https://github.com/anthropics/claude-code/issues/18121)
- [Expose rate limit data in statusline JSON input (#19385)](https://github.com/anthropics/claude-code/issues/19385)

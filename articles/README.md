# Zenn Articles & Books

このリポジトリはZennの記事と本を管理しています。

## 📚 本の執筆について

**⚠️ 重要: 本を執筆する前に必ず以下のファイルを読んでください**

### [`.zenn-deployment-notes.md`](./.zenn-deployment-notes.md)

このファイルには、Zennのデプロイで問題が起きた際のトラブルシューティングと、問題を未然に防ぐためのベストプラクティスが記載されています。

**特に重要なポイント:**

1. **Zennは差分デプロイ**
   - 最新のコミットで変更されたファイルのみがデプロイされる
   - 古いコミットで書いた内容は、再度変更しないと反映されない

2. **複数チャプター執筆時は必ず同じコミットに含める**
   ```bash
   # ✅ 正しい
   git add books/foo/{06,07,08,09}-*.md
   git commit -m "feat: Add Chapters 06-09"

   # ❌ 間違い
   git add books/foo/06-*.md && git commit -m "Add Chapter 06"
   git add books/foo/07-*.md && git commit -m "Add Chapter 07"
   ```

3. **プッシュ後は必ずZennのデプロイログを確認**
   - 3-5分待つ
   - デプロイされたファイル一覧を確認
   - 文字数が想定通りか確認（目安: 1万字以上）

## 🚀 クイックスタート

### 新しい本を執筆する場合

1. **デプロイ注意事項を確認**:
   ```bash
   cat .zenn-deployment-notes.md
   ```

2. **スラッシュコマンドを使用**:
   ```
   /start-book-writing
   ```

3. または、**Skillを起動**:
   ```
   # Claude Codeで以下のように言う
   "Zenn本を執筆したい"
   ```

### 既存の本にチャプターを追加する場合

1. **デプロイ注意事項を確認**:
   ```bash
   cat .zenn-deployment-notes.md
   ```

2. **スラッシュコマンドを使用**:
   ```
   /write-chapter
   ```

## 📖 既存の本

### React Development 2026 [Official]

**場所**: `books/react-development-2026-official/`

**内容**:
- 全10章（Chapter 00-09）
- React 18+、Next.js 14+、TypeScript 5+対応
- useState/useEffect → Hooks → TypeScript → Performance → Next.js → Accessibility → Architecture → Deployment

**ステータス**: ✅ 完成

## 🛠️ 利用可能なコマンド

### スラッシュコマンド

- `/start-book-writing` - 新しい本の執筆を開始
- `/write-chapter` - 既存の本に新しいチャプターを追加

### Skills

- `zenn-book-writing` - Zenn本執筆の包括的ガイド

## 📝 ディレクトリ構成

```
zenn-articles/
├── .zenn-deployment-notes.md  # ⚠️ 執筆前に必読
├── .claude/
│   └── commands/
│       ├── start-book-writing.md
│       └── write-chapter.md
├── articles/                  # Zenn記事
└── books/                     # Zenn本
    └── react-development-2026-official/
        ├── config.yaml
        ├── cover.png
        └── *.md              # チャプターファイル
```

## 🎯 ベストプラクティス

### 執筆ワークフロー

1. **デプロイ注意事項を確認** (`.zenn-deployment-notes.md`)
2. チャプターを執筆
3. **関連チャプターを全て同じコミットに含める**
4. コミット・プッシュ
5. **3-5分待ってZennのデプロイログを確認**
6. 問題があれば`.zenn-deployment-notes.md`のトラブルシューティングを参照

### チャプター執筆のガイドライン

- **文字数**: 1万字以上を目安
- **構成**: 目次 → 本文 → まとめ
- **コードブロック**: 必ず言語指定（```typescript, ```tsx, ```bash など）
- **難易度表示**: ★☆☆☆☆〜★★★★★で表示

## 🔧 トラブルシューティング

問題が発生した場合は、必ず[`.zenn-deployment-notes.md`](./.zenn-deployment-notes.md)を確認してください。

よくある問題:
- チャプターがZennに反映されない → デプロイログを確認
- 文字数が少ない（300-400字） → ファイルを少し編集して再プッシュ
- 「執筆中」のまま → 3-5分待つ、またはファイル編集して再プッシュ

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**

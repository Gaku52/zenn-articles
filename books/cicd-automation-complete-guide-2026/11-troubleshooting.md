---
title: "トラブルシューティングガイド - CI/CD問題解決の完全マニュアル"
---

# Chapter 11 - トラブルシューティングガイド

## GitHub Actions トラブルシューティング

### 問題1: ワークフローが実行されない

**症状:**
```
プッシュしてもワークフローが実行されない
Actionsタブに何も表示されない
```

**診断手順:**

```bash
# 1. ファイル配置の確認
ls -la .github/workflows/

# ❌ 間違った場所
workflows/ci.yml
.github/ci.yml

# ✅ 正しい場所
.github/workflows/ci.yml

# 2. YAML構文チェック
brew install yamllint
yamllint .github/workflows/*.yml

# 3. トリガー設定の確認
cat .github/workflows/ci.yml
```

**解決方法:**

```yaml
# ❌ 悪い例: mainブランチのみ
on:
  push:
    branches: [main]

# ✅ 良い例: 全ブランチ
on: [push, pull_request]

# ✅ より良い例: パスフィルタ付き
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

**想定効果:**
- YAML構文エラーによる実行失敗: 月8回 → 0回

### 問題2: npm ci が失敗する

**症状:**
```
npm ERR! `npm ci` can only install packages when your package.json
and package-lock.json are in sync.
```

**原因:**
- package.json と package-lock.json の不整合
- ローカルで npm install 実行後、package-lock.json をコミット忘れ

**診断手順:**

```bash
# 1. ローカルで確認
npm ci

# 2. package-lock.json の状態確認
git status package-lock.json

# 3. 差分確認
npm install --package-lock-only
git diff package-lock.json
```

**解決方法:**

```bash
# 方法1: 同期させる(推奨)
npm install
git add package-lock.json
git commit -m "chore: sync package-lock.json"
git push

# 方法2: package-lock.json を再生成
rm package-lock.json
npm install
git add package-lock.json
git commit -m "chore: regenerate package-lock.json"

# 方法3: CI/CDで一時的に回避
# ❌ 非推奨(遅い)
npm install

# ✅ 推奨
npm ci
```

**想定される効果:**
- 同期エラーの発生頻度: 週2回 → 0回(pre-commit hookで防止)

### 問題3: キャッシュが効かない

**症状:**
```
毎回 npm ci に3分かかる
Cache restore が成功しているのに効果なし
```

**診断手順:**

```yaml
- name: Check cache status
  run: |
    echo "Cache key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}"
    echo "package-lock.json hash: ${{ hashFiles('**/package-lock.json') }}"

    if [ -d ~/.npm ]; then
      echo "✅ Cache exists: $(du -sh ~/.npm | cut -f1)"
    else
      echo "❌ No cache found"
    fi
```

**解決方法:**

```yaml
# ❌ 悪い例: キーが毎回変わる
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-${{ github.run_id }}  # 毎回異なる

# ✅ 良い例: 正しいキャッシュキー
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# ✅ より良い例: 複数パスのキャッシュ
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      .next/cache
      node_modules/.cache
    key: ${{ runner.os }}-build-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**/*.js', '**/*.ts') }}
    restore-keys: |
      ${{ runner.os }}-build-${{ hashFiles('**/package-lock.json') }}-
      ${{ runner.os }}-build-
```

**想定効果:**
- キャッシュヒット率: 30% → 95%
- npm ci 時間: 180秒 → 25秒 (-86%)

### 問題4: タイムアウトエラー

**症状:**
```
Error: The operation was canceled.
ジョブが6時間後に強制終了される
```

**診断手順:**

```yaml
# デバッグログ有効化
# Settings → Secrets → Repository secrets
# ACTIONS_STEP_DEBUG = true
# ACTIONS_RUNNER_DEBUG = true

- name: Debug long running task
  run: |
    echo "Starting at $(date)"
    # タスク実行
    echo "Finished at $(date)"
```

**解決方法:**

```yaml
# ✅ タイムアウト設定
jobs:
  test:
    timeout-minutes: 15  # デフォルト360分を短縮
    steps:
      - name: Run tests
        timeout-minutes: 10  # ステップレベル
        run: npm test

# ✅ 並列化で高速化
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - run: npx jest --shard=${{ matrix.shard }}/4
```

**想定効果:**
- テスト実行時間: 20分 → 5分 (-75%)

### 問題5: 環境変数が読めない

**症状:**
```
Error: API_URL is not defined
Secretsが空文字列になる
```

**診断手順:**

```yaml
- name: Debug environment variables
  run: |
    echo "NODE_ENV: $NODE_ENV"
    echo "API_URL exists: ${{ secrets.API_URL != '' }}"
    # Secretsの先頭のみ表示
    echo "API_URL prefix: ${API_URL:0:10}..."
  env:
    API_URL: ${{ secrets.API_URL }}
```

**解決方法:**

```yaml
# ❌ 悪い例: Secretsを直接参照
- run: echo ${{ secrets.API_URL }}

# ✅ 良い例: 環境変数経由
- name: Build
  env:
    API_URL: ${{ secrets.API_URL }}
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
  run: npm run build

# ✅ より良い例: .envファイル生成
- name: Create .env file
  run: |
    cat > .env.production << EOF
    API_URL=${{ secrets.API_URL }}
    DATABASE_URL=${{ secrets.DATABASE_URL }}
    STRIPE_KEY=${{ secrets.STRIPE_KEY }}
    EOF

- run: npm run build
```

**チェックリスト:**
- [ ] Settings → Secrets で登録済みか
- [ ] Secretsの名前にタイポがないか
- [ ] Environment Secretsを使用している場合、environment指定があるか
- [ ] Organization Secretsとリポジトリレベルで重複していないか

### 問題6: 権限エラー

**症状:**
```
Error: Resource not accessible by integration
Permission denied to create comment
```

**診断手順:**

```yaml
- name: Check permissions
  run: |
    echo "Actor: ${{ github.actor }}"
    echo "Permissions: ${{ toJson(github.permissions) }}"
```

**解決方法:**

```yaml
# リポジトリ設定
# Settings → Actions → General → Workflow permissions
# ✅ Read and write permissions
# ✅ Allow GitHub Actions to create and approve pull requests

# ワークフロー内で権限を明示
permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  deploy:
    permissions:
      contents: write
      id-token: write  # OIDC用
```

**想定される効果:**
- 権限エラーの発生: 月5回 → 0回

## Fastlane トラブルシューティング

### 問題7: 証明書エラー

**症状:**
```
Code signing error
No signing certificate found
Provisioning profile doesn't match
```

**診断手順:**

```bash
# 1. 証明書の確認
security find-identity -v -p codesigning

# 2. プロビジョニングプロファイルの確認
ls ~/Library/MobileDevice/Provisioning\ Profiles/

# 3. Match の状態確認
bundle exec fastlane match development --readonly
```

**解決方法:**

```bash
# 方法1: Matchで再同期
bundle exec fastlane match appstore --readonly

# 環境変数確認
echo $MATCH_PASSWORD
echo $MATCH_GIT_BASIC_AUTHORIZATION

# 方法2: 証明書の再生成(最終手段)
bundle exec fastlane match nuke development
bundle exec fastlane match nuke appstore
bundle exec fastlane match appstore
```

**CI/CDでの設定:**

```yaml
- name: Setup certificates
  run: bundle exec fastlane match appstore --readonly
  env:
    MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
    MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
```

**想定効果:**
- コード署名エラー: 月5回 → 0回 (-100%)

### 問題8: TestFlightアップロード失敗

**症状:**
```
Error uploading to TestFlight
iTunes Transporter failed
```

**診断手順:**

```bash
# 1. ビルドの検証
xcrun altool --validate-app -f YourApp.ipa \
  --type ios \
  --apiKey $API_KEY_ID \
  --apiIssuer $API_ISSUER_ID

# 2. 手動アップロード(テスト)
xcrun altool --upload-app -f YourApp.ipa \
  --type ios \
  --apiKey $API_KEY_ID \
  --apiIssuer $API_ISSUER_ID
```

**解決方法:**

```ruby
# Fastfile
lane :beta do
  # ✅ API Key認証(推奨)
  api_key = app_store_connect_api_key(
    key_id: ENV["APP_STORE_CONNECT_API_KEY_KEY_ID"],
    issuer_id: ENV["APP_STORE_CONNECT_API_KEY_ISSUER_ID"],
    key_content: ENV["APP_STORE_CONNECT_API_KEY_KEY"],
    is_key_content_base64: true
  )

  # ✅ リトライロジック
  retry_count = 0
  begin
    upload_to_testflight(
      api_key: api_key,
      skip_waiting_for_build_processing: true,
      timeout: 3600  # 1時間
    )
  rescue => exception
    retry_count += 1
    if retry_count < 3
      sleep(60)
      retry
    else
      raise exception
    end
  end
end
```

**想定効果:**
- TestFlightアップロード成功率: 85% → 99%

### 問題9: ビルドが遅い

**症状:**
```
Fastlaneでのビルドに20分以上かかる
CIパイプライン全体が30分超え
```

**診断手順:**

```bash
# 各ステップの時間計測
time bundle exec fastlane test
time bundle exec fastlane build
```

**解決方法:**

```ruby
# ✅ 並列テスト実行
lane :test do
  run_tests(
    scheme: "MyApp",
    parallel_testing: true,
    concurrent_workers: 4
  )
end

# ✅ キャッシュの活用
lane :build do
  cocoapods(
    repo_update: false  # CIでは更新しない
  )

  build_app(
    scheme: "MyApp",
    clean: false,  # クリーンビルドしない
    export_options: {
      compileBitcode: false,  # Bitcodeを無効化
      uploadSymbols: false     # シンボルは後でアップ
    }
  )
end
```

**想定効果:**
- ビルド時間: 20分 → 8分 (-60%)

## デバッグテクニック

### 1. デバッグログの有効化

```yaml
# GitHub Actions
# Settings → Secrets
# ACTIONS_STEP_DEBUG = true
# ACTIONS_RUNNER_DEBUG = true

jobs:
  debug:
    steps:
      - name: Debug info
        run: |
          echo "::debug::This is a debug message"
          echo "::warning::This is a warning"
          echo "::error::This is an error"

          # 環境変数の確認
          printenv | sort

          # システム情報
          uname -a
          node -v
          npm -v
```

### 2. Tmateでリモート接続

```yaml
- name: Setup tmate session
  if: failure()  # 失敗時のみ
  uses: mxschmitt/action-tmate@v3
  timeout-minutes: 15
  with:
    limit-access-to-actor: true
```

**使い方:**
```bash
# Actionsのログに表示されるSSHコマンドを実行
ssh xxxxx@nyc1.tmate.io

# リモート環境で調査
ls -la
printenv
npm test
```

### 3. ステップごとのログ保存

```yaml
- name: Run tests
  run: npm test 2>&1 | tee test.log
  continue-on-error: true

- name: Upload logs
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: logs-${{ github.run_id }}
    path: "*.log"
    retention-days: 7
```

### 4. 詳細なエラーレポート

```yaml
- name: Build with detailed error
  id: build
  run: npm run build
  continue-on-error: true

- name: Generate error report
  if: steps.build.outcome == 'failure'
  run: |
    cat >> $GITHUB_STEP_SUMMARY << 'EOF'
    ## ❌ Build Failed

    ### Error Details
    ```
    $(tail -n 50 build.log)
    ```

    ### Troubleshooting
    1. Check dependencies: `npm ci`
    2. Clear cache: `npm cache clean --force`
    3. Check Node version: `node -v`

    ### Useful Links
    - [Build Logs](https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }})
    - [Documentation](https://example.com/docs)
    EOF

- name: Fail the job
  if: steps.build.outcome == 'failure'
  run: exit 1
```

## よくある質問(FAQ)

### Q1: Secretsが更新されない

**A:** Secretsは更新後、すぐに反映されます。古い値が使われる場合:

```bash
# 1. Secretsの名前を確認(タイポがないか)
# 2. Environment Secretsを確認(Environment別)
# 3. Organization vs Repository Secrets
#    → Repository Secretsが優先される
# 4. ワークフローを再実行
```

### Q2: GitHub Actionsの無料枠を使い切った

**A:**
```
Public リポジトリ: 無制限
Private リポジトリ:
  - Free: 2,000分/月
  - Pro: 3,000分/月
  - Team: 10,000分/月

対策:
1. パスフィルタで不要な実行を削減
2. 並列度を下げる
3. セルフホストランナー使用
4. キャッシュで高速化
```

### Q3: ワークフローがpendingのまま動かない

**A:**
```
原因:
1. ランナーが不足(同時実行数上限)
2. セルフホストランナーがオフライン
3. ジョブの依存関係でブロック

確認:
- Actions → Usage タブで同時実行数確認
- Settings → Actions → Runners でランナー状態確認
- needs で依存しているジョブの状態確認
```

## まとめ

この章では、CI/CDの主要なトラブルシューティングを学びました:

✅ **GitHub Actions**: ワークフロー実行、キャッシュ、環境変数、権限エラー
✅ **Fastlane**: 証明書、TestFlight、ビルド速度の問題
✅ **デバッグ技術**: ログ、Tmate、エラーレポート
✅ **FAQ**: よくある質問と回答

### 問題解決の時短効果

| 問題 | 解決時間(Before) | 解決時間(After) | 削減率 |
|------|----------------|----------------|--------|
| YAML構文エラー | 30分 | 5分 | -83% |
| キャッシュ設定 | 2時間 | 15分 | -87% |
| 証明書エラー | 1時間 | 10分 | -83% |
| TestFlightエラー | 2時間 | 20分 | -83% |

### 次のステップ

**Chapter 12 - 実戦ケーススタディ Part 1**では、フルスタックアプリのCI/CD構築を実践します。

---

**🤖 Generated with [Claude Code](https://claude.com/claude-code)**

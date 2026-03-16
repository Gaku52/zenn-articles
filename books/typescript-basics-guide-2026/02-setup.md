---
title: "開発環境構築"
---

# 開発環境構築

> Node.js、TypeScriptコンパイラ、VSCodeを準備して、最初のTypeScriptプログラムを動かす

## この章で学ぶこと

- Node.jsとnpmのセットアップ
- TypeScriptコンパイラ（`tsc`）のインストール
- `tsconfig.json` の基本設定
- VSCodeのTypeScript向け設定
- 最初のTypeScriptプログラムの作成と実行

---

## Node.js のインストール

TypeScriptの開発には Node.js が必要です。本書では **Node.js 22 LTS** を使用します。

公式サイト（https://nodejs.org/）からLTS版のインストーラーをダウンロードするのが最も簡単です。チーム開発やバージョンの使い分けが必要な場合は、Volta や nvm などのバージョン管理ツールが便利です。

```bash
# Volta を使う場合
curl https://get.volta.sh | bash
volta install node@22

# nvm を使う場合
nvm install 22
nvm use 22
```

インストール後、バージョンを確認します。

```bash
node --version
# 想定出力例: v22.x.x

npm --version
# 想定出力例: 10.x.x
```

## プロジェクトの作成

TypeScriptプロジェクトをゼロから作成していきます。

```bash
# プロジェクトディレクトリの作成
mkdir ts-hello && cd ts-hello

# package.json の初期化
npm init -y
```

`npm init -y` により、デフォルト設定の `package.json` が生成されます。

## TypeScript のインストール

TypeScriptはプロジェクトのローカル依存としてインストールするのが推奨です。

```bash
# TypeScript と Node.js の型定義をインストール
npm install typescript @types/node --save-dev
```

| パッケージ | 役割 |
|-----------|------|
| `typescript` | TypeScriptコンパイラ（`tsc`）本体 |
| `@types/node` | Node.jsのAPI（`console`, `process` 等）の型定義 |

インストール後、バージョンを確認します。

```bash
npx tsc --version
# 想定出力例: Version 5.7.x
```

:::message
**グローバル vs ローカルインストール**

`npm install -g typescript` でグローバルにインストールすることもできますが、プロジェクトごとにバージョンを管理するためローカルインストール（`--save-dev`）が推奨です。ローカルインストールした場合、`npx tsc` のように `npx` 経由で実行します。
:::

## tsconfig.json の作成

`tsconfig.json` は TypeScript コンパイラの設定ファイルです。プロジェクトのルートに配置します。

```bash
# tsconfig.json の自動生成
npx tsc --init
```

このコマンドで多数のオプションがコメント付きで生成されますが、すべてを理解する必要はありません。入門段階で押さえるべき設定に絞って解説します。

### 入門者向けの tsconfig.json

以下が本書で使用する基本的な設定です。

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 各オプションの解説

| オプション | 値 | 意味 |
|-----------|-----|------|
| `target` | `"ES2022"` | 出力するJavaScriptのバージョン |
| `module` | `"NodeNext"` | モジュールシステム。Node.jsのESM/CJS自動判定に対応 |
| `moduleResolution` | `"NodeNext"` | モジュール解決の方式。`module` と合わせて設定。Vite等のバンドラーを使う場合は `"bundler"` も検討してください |
| `outDir` | `"./dist"` | コンパイル後のJavaScriptの出力先 |
| `rootDir` | `"./src"` | ソースコードのルートディレクトリ |
| `strict` | `true` | 全ての厳密な型チェックを有効化（**最重要**） |
| `esModuleInterop` | `true` | CommonJSモジュールを `import` 文で読み込めるようにする |
| `skipLibCheck` | `true` | `.d.ts` の型チェックをスキップ（ビルド高速化） |
| `forceConsistentCasingInFileNames` | `true` | ファイル名の大文字小文字の一貫性チェック |
| `include` | `["src/**/*"]` | コンパイル対象のファイル |
| `exclude` | `["node_modules", "dist"]` | コンパイルから除外するディレクトリ |

:::message alert
**`strict: true` は必ず有効にしましょう**

`strict: true` は、`strictNullChecks`（null/undefinedの厳密チェック）、`noImplicitAny`（暗黙のanyを禁止）など、複数の厳密チェックを一括で有効にします。これを無効にすると、TypeScriptの型チェックの恩恵が大きく損なわれます。
:::

## ディレクトリ構成

以下のシンプルな構成でプロジェクトを作成します。

```
ts-hello/
├── src/
│   └── index.ts        # TypeScript ソースコード
├── dist/               # コンパイル後の出力（自動生成）
├── tsconfig.json
├── package.json
└── node_modules/
```

`src/` ディレクトリを作成します。

```bash
mkdir src
```

## VSCode の設定

VSCodeはTypeScript開発のデファクトスタンダードのエディタです。TypeScriptのLanguage Serverが組み込まれており、追加の拡張機能なしで型チェックや自動補完が動作します。

推奨する拡張機能は以下の3つです。

| 拡張機能 | 役割 |
|---------|------|
| **ESLint** | コード品質のチェック |
| **Prettier** | コードフォーマット（自動整形） |
| **Error Lens** | エラーをエディタ内にインライン表示 |

プロジェクトのルートに `.vscode/settings.json` を作成して、保存時の自動フォーマットとローカルTypeScriptの使用を設定しておくと快適です。

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "typescript.tsdk": "node_modules/typescript/lib"
}
```

`typescript.tsdk` を設定すると、VSCode組み込みのTypeScriptではなく、プロジェクトにインストールしたバージョンが使われます。チーム全員が同じバージョンで型チェックを行うために重要な設定です。

## 最初の TypeScript プログラム

### ソースコードの作成

`src/index.ts` に以下のコードを作成します。

```typescript
// src/index.ts

// 型アノテーション付きの関数
function greet(name: string): string {
  return `Hello, ${name}! Welcome to TypeScript.`;
}

// インターフェースでオブジェクトの形を定義
interface User {
  name: string;
  age: number;
}

// User 型の引数を受け取る関数
function introduce(user: User): string {
  return `${user.name} (${user.age}歳)`;
}

// 実行
const message = greet("TypeScript");
console.log(message);

const user: User = { name: "Taro", age: 25 };
console.log(introduce(user));
```

### コンパイルと実行

```bash
# TypeScript → JavaScript にコンパイル
npx tsc

# コンパイル結果を実行
node dist/index.js
```

想定出力例:
```
Hello, TypeScript! Welcome to TypeScript.
Taro (25歳)
```

`dist/index.js` を開くと、`interface User` や型アノテーションが消去されたJavaScriptが出力されています。前章で解説した「型消去」の実例です。

## package.json にスクリプトを追加

毎回 `npx tsc && node dist/index.js` と打つのは手間なので、`package.json` にスクリプトを登録します。

```json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsc --watch",
    "typecheck": "tsc --noEmit"
  }
}
```

| スクリプト | コマンド | 用途 |
|-----------|---------|------|
| `npm run build` | `tsc` | コンパイル |
| `npm start` | `node dist/index.js` | コンパイル済みコードを実行 |
| `npm run dev` | `tsc --watch` | ファイル変更を監視して自動コンパイル |
| `npm run typecheck` | `tsc --noEmit` | 型チェックのみ（ファイル出力なし） |

`npm run dev` を実行すると、ファイルを保存するたびに自動コンパイルが走ります。開発中はこのウォッチモードが便利です。

## より手軽な実行方法: tsx

開発中に「コンパイル → 実行」の2ステップを踏むのが面倒な場合、`tsx` というツールを使うとTypeScriptファイルを直接実行できます。

```bash
# tsx のインストール
npm install tsx --save-dev

# TypeScript ファイルを直接実行
npx tsx src/index.ts

# ウォッチモード（ファイル変更時に自動再実行）
npx tsx watch src/index.ts
```

`tsx` は内部で esbuild を使っており、型チェックは行わずにトランスパイルと実行だけを高速に行います。型チェックは `npm run typecheck`（`tsc --noEmit`）で別途行うのが一般的なワークフローです。`package.json` の `dev` スクリプトを `tsx watch src/index.ts` に変更すると、保存のたびに即座に再実行されるため快適です。

:::message
**Node.js 22 のネイティブ TypeScript 実行**

Node.js 22.6.0 以降では `node --experimental-strip-types src/index.ts` でTypeScriptファイルを直接実行できます。ただし、`enum` や `namespace` などTypeScript固有の構文は別途 `--experimental-transform-types` フラグが必要です。本書の学習においては `tsx` の利用を推奨します。
:::

## まとめ

| 概念 | 要点 |
|------|------|
| Node.js | v22 LTS を使用。`npm` でパッケージを管理する |
| TypeScript インストール | `npm install typescript @types/node --save-dev` |
| tsconfig.json | `strict: true` を必ず有効に。`outDir` と `rootDir` でディレクトリを分離 |
| コンパイル | `npx tsc` で `.ts` を `.js` に変換 |
| ウォッチモード | `tsc --watch` でファイル変更を監視して自動コンパイル |
| tsx | TypeScript を直接実行できるツール。開発中に便利 |
| VSCode | TypeScript 組み込みサポート。`typescript.tsdk` でローカル版を指定 |
| 開発フロー | コーディング → 型チェック (`tsc --noEmit`) → ビルド (`tsc`) → 実行 |

## やってみよう！

### 練習1: 環境構築を完了する

本章の手順に沿って、以下を完了してください。

1. Node.js 22 LTS をインストールする
2. `ts-hello` プロジェクトを作成し、TypeScriptをインストールする
3. `tsconfig.json` を作成する
4. `src/index.ts` に Hello World プログラムを書いてコンパイル・実行する

`npm run build && npm start` で想定通りの出力が得られれば成功です。

### 練習2: 型エラーを体験する

以下のコードを `src/index.ts` に書いて `npx tsc` を実行し、表示されるエラーメッセージを読んでみてください。

```typescript
function multiply(a: number, b: number): number {
  return a * b;
}

console.log(multiply(3, "4"));
console.log(multiply(5));
```

:::details 解答例
2つのエラーが報告されます。

1. `multiply(3, "4")` -- `Argument of type 'string' is not assignable to parameter of type 'number'`。第2引数に文字列 `"4"` を渡していますが、`b` は `number` 型です。
2. `multiply(5)` -- `Expected 2 arguments, but got 1`。引数が2つ必要ですが、1つしか渡していません。

修正するには `multiply(3, 4)` と `multiply(5, 2)` のように正しい型・個数の引数を渡します。
:::

### 練習3: tsconfig.json を読む

`npx tsc --init` で生成された `tsconfig.json` を開き、以下のオプションを探してそれぞれの意味を確認してください。

- `strict`
- `noImplicitAny`
- `strictNullChecks`
- `outDir`

:::details 解答例
- **`strict`**: 全ての厳密な型チェックフラグを一括で有効にする。`noImplicitAny`, `strictNullChecks` などを個別に設定する代わりに、これひとつで網羅できます。
- **`noImplicitAny`**: 型を明示しない引数や変数が暗黙的に `any` 型になることを禁止します。`strict: true` に含まれます。
- **`strictNullChecks`**: `null` と `undefined` を他の型と区別して厳密にチェックします。`string` 型の変数に `null` を代入するとエラーになります。`strict: true` に含まれます。
- **`outDir`**: コンパイル後のJavaScriptファイルの出力先ディレクトリを指定します。`"./dist"` と設定すれば、ソースと出力が分離されて管理しやすくなります。
:::

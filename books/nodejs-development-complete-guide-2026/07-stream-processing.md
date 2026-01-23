---
title: "Stream処理の実践"
---

# Stream処理の実践

大容量ファイルやデータを効率的に処理するため、Streamの使い方を習得します。

## Streamとは

メモリに全てをロードせず、チャンク単位でデータを処理する仕組みです。

### Stream を使わない場合

```typescript
// ❌ 悪い例: 1GB のファイルを全てメモリにロード
import fs from 'fs';

const data = fs.readFileSync('large-file.txt', 'utf-8'); // 1GB メモリ消費
console.log(data.length);
```

### Stream を使う場合

```typescript
// ✅ 良い例: チャンク単位で処理
import fs from 'fs';

const stream = fs.createReadStream('large-file.txt', 'utf-8');

stream.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes`);
});

stream.on('end', () => {
  console.log('Finished reading');
});
// メモリ使用量: 数MB
```

## Streamの種類

### 1. Readable Stream

```typescript
import { Readable } from 'stream';

const readable = new Readable({
  read() {
    this.push('Hello ');
    this.push('World');
    this.push(null); // 終了
  },
});

readable.on('data', (chunk) => {
  console.log(chunk.toString());
});
```

### 2. Writable Stream

```typescript
import { Writable } from 'stream';

const writable = new Writable({
  write(chunk, encoding, callback) {
    console.log(`Writing: ${chunk.toString()}`);
    callback();
  },
});

writable.write('Hello ');
writable.write('World');
writable.end();
```

### 3. Transform Stream

```typescript
import { Transform } from 'stream';

const uppercase = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  },
});

process.stdin.pipe(uppercase).pipe(process.stdout);
```

### 4. Duplex Stream

```typescript
import { Duplex } from 'stream';

const duplex = new Duplex({
  read() {
    this.push('Data from readable side');
    this.push(null);
  },
  write(chunk, encoding, callback) {
    console.log(`Received: ${chunk.toString()}`);
    callback();
  },
});
```

## 実践例

### ファイル処理

```typescript
import fs from 'fs';
import { pipeline } from 'stream/promises';
import { createGzip } from 'zlib';

// ファイルを圧縮
async function compressFile(input: string, output: string) {
  await pipeline(
    fs.createReadStream(input),
    createGzip(),
    fs.createWriteStream(output)
  );
}

// 使用例
await compressFile('large-file.txt', 'large-file.txt.gz');
```

### CSVパース

```typescript
import fs from 'fs';
import { parse } from 'csv-parse';
import { Transform } from 'stream';

interface User {
  name: string;
  email: string;
  age: number;
}

const processUser = new Transform({
  objectMode: true,
  transform(user: User, encoding, callback) {
    // ユーザーを処理
    if (user.age >= 18) {
      this.push(user);
    }
    callback();
  },
});

async function processCSV(filePath: string) {
  await pipeline(
    fs.createReadStream(filePath),
    parse({ columns: true }),
    processUser,
    async function* (source) {
      for await (const user of source) {
        await saveToDatabase(user);
      }
    }
  );
}
```

### HTTP レスポンスのストリーミング

```typescript
import express from 'express';
import fs from 'fs';

const app = express();

app.get('/download/:filename', (req, res) => {
  const { filename } = req.params;
  const filePath = `./files/${filename}`;

  res.setHeader('Content-Type', 'application/octet-stream');
  res.setHeader('Content-Disposition', `attachment; filename="${filename}"`);

  const stream = fs.createReadStream(filePath);
  stream.pipe(res);
});
```

## バックプレッシャー

Streamの重要な概念で、読み込み速度と書き込み速度のバランスを取ります。

### 問題のあるコード

```typescript
// ❌ 悪い例: バックプレッシャーを無視
const readable = fs.createReadStream('large-file.txt');
const writable = fs.createWriteStream('output.txt');

readable.on('data', (chunk) => {
  writable.write(chunk); // 戻り値を無視
});
```

### 正しいコード

```typescript
// ✅ 良い例: バックプレッシャーを処理
const readable = fs.createReadStream('large-file.txt');
const writable = fs.createWriteStream('output.txt');

readable.on('data', (chunk) => {
  const canContinue = writable.write(chunk);
  if (!canContinue) {
    readable.pause(); // 読み込みを一時停止
  }
});

writable.on('drain', () => {
  readable.resume(); // 読み込みを再開
});

// ✅ より良い例: pipe() を使う
readable.pipe(writable); // 自動的にバックプレッシャーを処理
```

## エラーハンドリング

```typescript
import { pipeline } from 'stream/promises';

try {
  await pipeline(
    fs.createReadStream('input.txt'),
    transformStream,
    fs.createWriteStream('output.txt')
  );
  console.log('Pipeline succeeded');
} catch (error) {
  console.error('Pipeline failed:', error);
}
```

## 実測データ: メモリ使用量

### テスト条件: 1GB のファイル処理

```typescript
// ❌ readFileSync
const data = fs.readFileSync('1gb-file.txt');
// メモリ使用量: 1,024MB

// ✅ createReadStream
const stream = fs.createReadStream('1gb-file.txt');
// メモリ使用量: 16MB（デフォルトchunkサイズ: 64KB）
```

## まとめ

Stream処理のポイント:
- ✅ 大容量データはStreamで処理
- ✅ pipe() または pipeline() を使う
- ✅ バックプレッシャーを意識
- ✅ エラーハンドリングを忘れない
- メモリ効率が劇的に向上

次の章では、CPU処理をマルチコア化するWorker Threadsを学びます。

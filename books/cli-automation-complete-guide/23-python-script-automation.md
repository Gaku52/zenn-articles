# Pythonスクリプト自動化

本章では、Pythonを使った自動化スクリプトの開発を学びます。

## 基本スクリプト

```python
#!/usr/bin/env python3

import sys
import subprocess

env = sys.argv[1] if len(sys.argv) > 1 else 'development'

print(f'Deploying to {env}...')

# ビルド
subprocess.run(['npm', 'run', 'build'], check=True)

# デプロイ
if env == 'production':
    subprocess.run([
        'ssh', 'user@server',
        'cd /var/www && git pull'
    ], check=True)
```

## データ処理

```python
import pandas as pd

# CSV読み込み
df = pd.read_csv('input.csv')

# データ処理
df['total'] = df['price'] * df['quantity']
df_filtered = df[df['total'] > 1000]

# 出力
df_filtered.to_csv('output.csv', index=False)
```

## まとめ

Pythonスクリプトにより、データ処理や複雑な自動化を簡潔に実現できます。

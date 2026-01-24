---
title: "スクリプト自動化"
---

# スクリプト自動化

## npm scripts自動化

### 複合スクリプト

```json
{
  "scripts": {
    "clean": "rm -rf dist",
    "compile": "tsc",
    "bundle": "vite build",
    "build": "npm run clean && npm run compile && npm run bundle",
    
    "test:unit": "vitest run",
    "test:e2e": "playwright test",
    "test:all": "npm run test:unit && npm run test:e2e",
    
    "lint:js": "eslint .",
    "lint:css": "stylelint '**/*.css'",
    "lint:all": "npm-run-all --parallel lint:js lint:css"
  }
}
```

### Git Hooks連携

```json
{
  "scripts": {
    "prepare": "husky install"
  }
}
```

.husky/pre-commit:

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npm run lint
npm test
```

## CI/CD自動化

### GitHub Actions

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm test
      - run: npm run build
```

## まとめ

スクリプト自動化により、開発フローが効率化されます。

次章では、iOS向けのSwift Package Managerについて解説します。

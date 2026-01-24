# CI/CD統合

本章では、CLIツールとスクリプトをCI/CDパイプラインに統合する方法を学びます。

## GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Build
        run: npm run build

      - name: Deploy
        run: ./scripts/deploy.sh production
        env:
          SSH_KEY: ${{ secrets.SSH_KEY }}
```

## GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - deploy

build:
  stage: build
  script:
    - npm run build

deploy:
  stage: deploy
  script:
    - ./scripts/deploy.sh production
  only:
    - main
```

## まとめ

CI/CDとの統合により、自動化の効果を最大化できます。

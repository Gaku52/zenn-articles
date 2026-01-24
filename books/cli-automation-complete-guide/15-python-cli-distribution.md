# Python CLI配布

本章では、PythonCLIツールをPyPIに公開し、ユーザーに配布する方法を学びます。

## PyPIへの公開

### ビルド

```bash
pip install build twine
python -m build
```

### 公開

```bash
# TestPyPI（テスト環境）
twine upload --repository testpypi dist/*

# PyPI（本番環境）
twine upload dist/*
```

### インストール

```bash
pip install my-cli-tool
mycli --help
```

## バイナリ配布

### PyInstaller

```bash
pip install pyinstaller

# ビルド
pyinstaller --onefile src/my_cli/main.py --name mycli

# 実行
./dist/mycli
```

## まとめ

PyPIへの公開により、誰でも簡単にCLIツールをインストールできます。

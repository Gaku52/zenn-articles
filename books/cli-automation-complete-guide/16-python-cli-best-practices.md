# Python CLIベストプラクティス

本章では、プロフェッショナルなPython CLIツールを開発するためのベストプラクティスを学びます。

## 型ヒントの活用

```python
from typing import Optional, List

def create_project(
    name: str,
    template: str = 'default',
    features: Optional[List[str]] = None
) -> bool:
    """Create a new project"""
    features = features or []
    # ... implementation
    return True
```

## エラーハンドリング

```python
import typer

class CLIError(Exception):
    pass

try:
    create_project(name)
except CLIError as e:
    typer.secho(f'Error: {e}', fg='red')
    raise typer.Exit(1)
```

## ロギング

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger(__name__)
logger.info('Creating project...')
```

## まとめ

型ヒント、適切なエラーハンドリング、ロギングにより、保守性の高いCLIツールを開発できます。

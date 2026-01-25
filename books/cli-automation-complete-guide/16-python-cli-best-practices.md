---
title: "Python CLIベストプラクティス"
---

# Python CLIベストプラクティス

プロフェッショナルなCLIツールを開発するには、型安全性、エラーハンドリング、設定管理、ログ出力など、多くの要素を適切に実装する必要があります。本章では、Python CLIツール開発のベストプラクティスを学びます。

## 型ヒントの徹底活用

Python 3.8以降の型ヒント機能を最大限に活用することで、IDEの補完機能が向上し、バグを早期に発見できます。

### 基本的な型ヒント

```python
# src/my_cli/core/project.py
from typing import Optional, List, Dict, Union
from pathlib import Path

def create_project(
    name: str,
    template: str = 'default',
    features: Optional[List[str]] = None,
    output_dir: Optional[Path] = None
) -> bool:
    """
    Create a new project

    Args:
        name: Project name
        template: Project template
        features: Optional list of features
        output_dir: Output directory

    Returns:
        True if successful, False otherwise
    """
    features = features or []
    output_dir = output_dir or Path.cwd()

    # Implementation
    return True

def get_project_config(name: str) -> Dict[str, Union[str, int, bool]]:
    """Get project configuration"""
    return {
        'name': name,
        'version': '1.0.0',
        'enabled': True,
        'max_size': 1024
    }
```

### TypedDictとDataclassの活用

```python
from typing import TypedDict, List
from dataclasses import dataclass, field
from enum import Enum

class ProjectStatus(str, Enum):
    ACTIVE = "active"
    ARCHIVED = "archived"
    DELETED = "deleted"

class ProjectConfig(TypedDict):
    name: str
    template: str
    features: List[str]
    status: ProjectStatus

@dataclass
class Project:
    """Project data class"""
    name: str
    template: str
    status: ProjectStatus = ProjectStatus.ACTIVE
    features: List[str] = field(default_factory=list)
    metadata: Dict[str, str] = field(default_factory=dict)

    def is_active(self) -> bool:
        """Check if project is active"""
        return self.status == ProjectStatus.ACTIVE

    def archive(self) -> None:
        """Archive the project"""
        self.status = ProjectStatus.ARCHIVED
```

### Pydanticによるバリデーション

```python
from pydantic import BaseModel, Field, validator
from typing import List, Optional
from pathlib import Path

class ProjectSettings(BaseModel):
    """Project settings with validation"""
    name: str = Field(..., min_length=3, max_length=50, pattern=r'^[a-zA-Z0-9_-]+$')
    template: str = Field(default='default')
    features: List[str] = Field(default_factory=list)
    output_dir: Optional[Path] = None
    max_size: int = Field(default=1024, ge=0, le=10240)

    @validator('name')
    def validate_name(cls, v: str) -> str:
        """Validate project name"""
        if v.startswith('-') or v.startswith('_'):
            raise ValueError('Name cannot start with - or _')
        return v.lower()

    @validator('features')
    def validate_features(cls, v: List[str]) -> List[str]:
        """Validate features list"""
        allowed = {'typescript', 'eslint', 'prettier', 'jest'}
        for feature in v:
            if feature not in allowed:
                raise ValueError(f'Unknown feature: {feature}')
        return v

# Usage
try:
    settings = ProjectSettings(
        name='my-project',
        features=['typescript', 'eslint']
    )
    print(f"Valid settings: {settings}")
except ValueError as e:
    print(f"Validation error: {e}")
```

## 環境変数の活用

環境変数を使用して、設定を柔軟に管理します。

```python
# src/my_cli/config/env.py
import os
from typing import Optional
from pathlib import Path

class Environment:
    """Environment configuration"""

    # API設定
    API_BASE_URL: str = os.getenv('MYCLI_API_URL', 'https://api.example.com')
    API_KEY: Optional[str] = os.getenv('MYCLI_API_KEY')
    API_TIMEOUT: int = int(os.getenv('MYCLI_API_TIMEOUT', '30'))

    # ディレクトリ設定
    CONFIG_DIR: Path = Path(os.getenv('MYCLI_CONFIG_DIR', '~/.mycli')).expanduser()
    CACHE_DIR: Path = Path(os.getenv('MYCLI_CACHE_DIR', '~/.mycli/cache')).expanduser()
    LOG_DIR: Path = Path(os.getenv('MYCLI_LOG_DIR', '~/.mycli/logs')).expanduser()

    # ログレベル
    LOG_LEVEL: str = os.getenv('MYCLI_LOG_LEVEL', 'INFO').upper()

    # デバッグモード
    DEBUG: bool = os.getenv('MYCLI_DEBUG', 'false').lower() == 'true'

    # プロキシ設定
    HTTP_PROXY: Optional[str] = os.getenv('HTTP_PROXY')
    HTTPS_PROXY: Optional[str] = os.getenv('HTTPS_PROXY')

    @classmethod
    def ensure_directories(cls) -> None:
        """Ensure required directories exist"""
        for directory in [cls.CONFIG_DIR, cls.CACHE_DIR, cls.LOG_DIR]:
            directory.mkdir(parents=True, exist_ok=True)

# Initialize directories
Environment.ensure_directories()
```

使用例は以下の通りです。

```bash
# 環境変数を設定して実行
export MYCLI_API_KEY="your-api-key"
export MYCLI_LOG_LEVEL="DEBUG"
export MYCLI_DEBUG="true"

mycli create myproject
```

## 設定ファイルの管理

TOML、YAML、JSONなどの設定ファイルをサポートします。

### TOML設定ファイル

```python
# src/my_cli/config/loader.py
import tomli
import tomli_w
from pathlib import Path
from typing import Dict, Any, Optional
from pydantic import BaseModel, Field

class GlobalConfig(BaseModel):
    """Global configuration"""
    default_template: str = Field(default='react')
    auto_install: bool = Field(default=True)
    editor: str = Field(default='vscode')
    theme: str = Field(default='dark')

class ProjectDefaults(BaseModel):
    """Project default settings"""
    features: list[str] = Field(default_factory=lambda: ['typescript', 'eslint'])
    git_init: bool = Field(default=True)
    install_deps: bool = Field(default=True)

class Config(BaseModel):
    """Complete configuration"""
    global_config: GlobalConfig = Field(default_factory=GlobalConfig)
    project_defaults: ProjectDefaults = Field(default_factory=ProjectDefaults)

class ConfigLoader:
    """Configuration file loader"""

    def __init__(self, config_path: Optional[Path] = None):
        from my_cli.config.env import Environment
        self.config_path = config_path or Environment.CONFIG_DIR / 'config.toml'

    def load(self) -> Config:
        """Load configuration from file"""
        if not self.config_path.exists():
            return Config()

        with open(self.config_path, 'rb') as f:
            data = tomli.load(f)
            return Config(**data)

    def save(self, config: Config) -> None:
        """Save configuration to file"""
        self.config_path.parent.mkdir(parents=True, exist_ok=True)

        with open(self.config_path, 'wb') as f:
            tomli_w.dump(config.model_dump(), f)

    def update(self, **kwargs: Any) -> None:
        """Update configuration"""
        config = self.load()

        for key, value in kwargs.items():
            if hasattr(config.global_config, key):
                setattr(config.global_config, key, value)
            elif hasattr(config.project_defaults, key):
                setattr(config.project_defaults, key, value)

        self.save(config)
```

設定ファイルの例（~/.mycli/config.toml）:

```toml
[global_config]
default_template = "react"
auto_install = true
editor = "vscode"
theme = "dark"

[project_defaults]
features = ["typescript", "eslint", "prettier"]
git_init = true
install_deps = true
```

### YAMLサポート

```python
import yaml
from pathlib import Path
from typing import Dict, Any

def load_yaml_config(path: Path) -> Dict[str, Any]:
    """Load YAML configuration"""
    with open(path, 'r') as f:
        return yaml.safe_load(f)

def save_yaml_config(path: Path, config: Dict[str, Any]) -> None:
    """Save YAML configuration"""
    with open(path, 'w') as f:
        yaml.dump(config, f, default_flow_style=False)
```

## 構造化ログ出力

適切なログ出力は、トラブルシューティングに不可欠です。

```python
# src/my_cli/utils/logger.py
import logging
import sys
from pathlib import Path
from typing import Optional
from rich.logging import RichHandler
from my_cli.config.env import Environment

class Logger:
    """Structured logger with file and console output"""

    _instance: Optional[logging.Logger] = None

    @classmethod
    def get_logger(cls, name: str = 'mycli') -> logging.Logger:
        """Get or create logger instance"""
        if cls._instance is not None:
            return cls._instance

        # Create logger
        logger = logging.getLogger(name)
        logger.setLevel(getattr(logging, Environment.LOG_LEVEL))

        # Prevent duplicate handlers
        if logger.handlers:
            return logger

        # Console handler with Rich
        console_handler = RichHandler(
            rich_tracebacks=True,
            tracebacks_show_locals=Environment.DEBUG,
            show_time=True,
            show_path=Environment.DEBUG
        )
        console_handler.setLevel(logging.INFO)
        console_formatter = logging.Formatter('%(message)s')
        console_handler.setFormatter(console_formatter)

        # File handler
        log_file = Environment.LOG_DIR / 'mycli.log'
        file_handler = logging.FileHandler(log_file)
        file_handler.setLevel(logging.DEBUG)
        file_formatter = logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
        file_handler.setFormatter(file_formatter)

        # Add handlers
        logger.addHandler(console_handler)
        logger.addHandler(file_handler)

        cls._instance = logger
        return logger

# Usage
logger = Logger.get_logger()

def create_project(name: str) -> None:
    """Create a new project with logging"""
    logger.info(f"Creating project: {name}")
    logger.debug(f"Project details: name={name}")

    try:
        # Implementation
        logger.info(f"Project {name} created successfully")
    except Exception as e:
        logger.error(f"Failed to create project: {e}", exc_info=True)
        raise
```

## エラーハンドリング

適切なエラーハンドリングで、ユーザーフレンドリーなCLIツールを作成します。

```python
# src/my_cli/exceptions.py
from typing import Optional

class CLIError(Exception):
    """Base exception for CLI errors"""
    def __init__(self, message: str, exit_code: int = 1):
        self.message = message
        self.exit_code = exit_code
        super().__init__(self.message)

class ProjectError(CLIError):
    """Project-related errors"""
    pass

class ValidationError(CLIError):
    """Validation errors"""
    pass

class ConfigError(CLIError):
    """Configuration errors"""
    pass

class NetworkError(CLIError):
    """Network-related errors"""
    def __init__(self, message: str, original_error: Optional[Exception] = None):
        super().__init__(message, exit_code=2)
        self.original_error = original_error
```

エラーハンドリングの実装例:

```python
# src/my_cli/main.py
import typer
from typing import Optional
from pathlib import Path
from my_cli.exceptions import CLIError, ProjectError, ValidationError
from my_cli.utils.logger import Logger

app = typer.Typer()
logger = Logger.get_logger()

@app.command()
def create(
    name: str,
    template: str = typer.Option('default', help='Template to use'),
    output: Optional[Path] = typer.Option(None, help='Output directory')
):
    """
    Create a new project
    """
    try:
        # Validation
        if len(name) < 3:
            raise ValidationError('Project name must be at least 3 characters')

        # Create project
        logger.info(f"Creating project: {name}")
        # ... implementation ...

        typer.secho(f"✓ Project {name} created successfully", fg=typer.colors.GREEN)

    except ValidationError as e:
        logger.error(f"Validation error: {e.message}")
        typer.secho(f"Error: {e.message}", fg=typer.colors.RED, err=True)
        raise typer.Exit(e.exit_code)

    except ProjectError as e:
        logger.error(f"Project error: {e.message}")
        typer.secho(f"Error: {e.message}", fg=typer.colors.RED, err=True)
        raise typer.Exit(e.exit_code)

    except CLIError as e:
        logger.error(f"CLI error: {e.message}")
        typer.secho(f"Error: {e.message}", fg=typer.colors.RED, err=True)
        raise typer.Exit(e.exit_code)

    except Exception as e:
        logger.exception(f"Unexpected error: {e}")
        typer.secho(f"Unexpected error occurred. Check logs for details.", fg=typer.colors.RED, err=True)
        raise typer.Exit(1)

if __name__ == "__main__":
    app()
```

## 依存関係の最小化

必要最小限の依存関係に留めることで、インストールサイズを削減し、セキュリティリスクを低減します。

### 依存関係の選択基準

```toml
# pyproject.toml
[project]
dependencies = [
    # 必須依存関係のみ
    "typer>=0.9.0",       # CLIフレームワーク
    "rich>=13.0.0",       # 出力の装飾
]

[project.optional-dependencies]
# 機能別のオプション依存関係
http = [
    "httpx>=0.24.0",      # HTTP通信
]
validation = [
    "pydantic>=2.0.0",    # バリデーション
]
yaml = [
    "pyyaml>=6.0",        # YAML設定ファイル
]
all = [
    "httpx>=0.24.0",
    "pydantic>=2.0.0",
    "pyyaml>=6.0",
]
```

インストール方法:

```bash
# 基本機能のみ
pip install my-cli-tool

# HTTP機能を追加
pip install my-cli-tool[http]

# すべての機能を追加
pip install my-cli-tool[all]
```

## プログレス表示とユーザーフィードバック

長時間かかる処理には、プログレス表示を実装します。

```python
# src/my_cli/utils/progress.py
from rich.progress import Progress, SpinnerColumn, TextColumn, BarColumn, TimeRemainingColumn
from rich.console import Console
from typing import List, Callable
import time

console = Console()

def show_progress(tasks: List[tuple[str, Callable]], description: str = "Processing"):
    """
    Show progress for multiple tasks

    Args:
        tasks: List of (task_name, task_function) tuples
        description: Overall description
    """
    with Progress(
        SpinnerColumn(),
        TextColumn("[progress.description]{task.description}"),
        BarColumn(),
        TextColumn("[progress.percentage]{task.percentage:>3.0f}%"),
        TimeRemainingColumn(),
        console=console
    ) as progress:
        main_task = progress.add_task(f"[cyan]{description}", total=len(tasks))

        for task_name, task_func in tasks:
            task_id = progress.add_task(f"[green]{task_name}", total=100)

            # Execute task with progress updates
            for i in range(100):
                task_func(i)
                progress.update(task_id, advance=1)
                time.sleep(0.01)

            progress.update(main_task, advance=1)

    console.print(f"[bold green]✓ {description} completed!")
```

## キャッシュ機能

API呼び出しなどの結果をキャッシュして、パフォーマンスを向上させます。

```python
# src/my_cli/utils/cache.py
import json
import hashlib
from pathlib import Path
from typing import Any, Optional, Callable
from datetime import datetime, timedelta
from functools import wraps
from my_cli.config.env import Environment

class Cache:
    """Simple file-based cache"""

    def __init__(self, cache_dir: Optional[Path] = None):
        self.cache_dir = cache_dir or Environment.CACHE_DIR
        self.cache_dir.mkdir(parents=True, exist_ok=True)

    def _get_cache_path(self, key: str) -> Path:
        """Get cache file path for key"""
        key_hash = hashlib.md5(key.encode()).hexdigest()
        return self.cache_dir / f"{key_hash}.json"

    def get(self, key: str, max_age: Optional[timedelta] = None) -> Optional[Any]:
        """Get cached value"""
        cache_path = self._get_cache_path(key)

        if not cache_path.exists():
            return None

        with open(cache_path, 'r') as f:
            data = json.load(f)

        # Check expiration
        if max_age:
            cached_time = datetime.fromisoformat(data['timestamp'])
            if datetime.now() - cached_time > max_age:
                cache_path.unlink()
                return None

        return data['value']

    def set(self, key: str, value: Any) -> None:
        """Set cached value"""
        cache_path = self._get_cache_path(key)

        with open(cache_path, 'w') as f:
            json.dump({
                'timestamp': datetime.now().isoformat(),
                'value': value
            }, f)

    def clear(self) -> None:
        """Clear all cache"""
        for cache_file in self.cache_dir.glob('*.json'):
            cache_file.unlink()

def cached(max_age: Optional[timedelta] = None):
    """Decorator for caching function results"""
    def decorator(func: Callable) -> Callable:
        cache = Cache()

        @wraps(func)
        def wrapper(*args, **kwargs):
            # Create cache key from function name and arguments
            key = f"{func.__name__}:{str(args)}:{str(kwargs)}"

            # Try to get from cache
            result = cache.get(key, max_age)
            if result is not None:
                return result

            # Execute function and cache result
            result = func(*args, **kwargs)
            cache.set(key, result)
            return result

        return wrapper
    return decorator

# Usage
@cached(max_age=timedelta(hours=1))
def fetch_templates() -> list[str]:
    """Fetch available templates (cached for 1 hour)"""
    # API call
    return ['react', 'vue', 'angular']
```

## Python CLIベストプラクティスチェックリスト

プロフェッショナルなCLIツールのために、以下を確認しましょう。

- [ ] すべての関数・メソッドに型ヒントを付与している
- [ ] Pydanticでバリデーションを実装している
- [ ] 環境変数で設定を柔軟に管理している
- [ ] TOML/YAML設定ファイルをサポートしている
- [ ] 構造化ログを出力している
- [ ] 適切なエラーハンドリングを実装している
- [ ] カスタム例外クラスを定義している
- [ ] 依存関係を最小限に抑えている
- [ ] 長時間処理にプログレス表示を実装している
- [ ] APIレスポンスをキャッシュしている

## まとめ

Python CLIツール開発のベストプラクティスを遵守することで、保守性が高く、ユーザーフレンドリーなツールを作成できます。型ヒント、適切なエラーハンドリング、構造化ログ、設定管理など、基本的な要素を丁寧に実装することが重要です。

次章では、Goを使った高パフォーマンスなCLIツール開発のセットアップを学びます。

## 参考文献

- [Typer公式ドキュメント](https://typer.tiangolo.com/)
- [Pydantic公式ドキュメント](https://docs.pydantic.dev/)
- [Python Logging - 公式ドキュメント](https://docs.python.org/3/library/logging.html)
- [Rich公式ドキュメント](https://rich.readthedocs.io/)
- [PEP 484 - Type Hints](https://peps.python.org/pep-0484/)

# Python CLIのテスト

高品質なCLIツールを開発するには、包括的なテスト戦略が不可欠です。本章では、pytestを使ったPython CLIツールのテスト手法を学びます。CliRunnerを活用したエンドツーエンドテスト、モックを使った外部依存のテスト、カバレッジ測定まで、実践的なテスト技術を習得します。

## テスト環境のセットアップ

まず、必要なテストツールをインストールします。

```bash
# 基本的なテストツール
pip install pytest pytest-cov pytest-mock

# Typerアプリケーションのテスト用
pip install typer[all]

# Click（Typerのベース）のテストユーティリティ
pip install click
```

プロジェクト構造は以下のようになります。

```
my-cli-tool/
├── src/
│   └── my_cli/
│       ├── __init__.py
│       ├── main.py
│       ├── commands/
│       │   ├── __init__.py
│       │   ├── create.py
│       │   └── list.py
│       └── core/
│           ├── __init__.py
│           └── project.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_main.py
│   ├── test_commands/
│   │   ├── __init__.py
│   │   ├── test_create.py
│   │   └── test_list.py
│   └── test_core/
│       ├── __init__.py
│       └── test_project.py
├── pyproject.toml
└── pytest.ini
```

## pytest.iniの設定

pytestの設定ファイルを作成します。

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    -v
    --strict-markers
    --cov=src/my_cli
    --cov-report=html
    --cov-report=term-missing
    --cov-branch
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks tests as integration tests
    unit: marks tests as unit tests
```

## CliRunnerを使った基本的なテスト

CliRunnerは、CLIアプリケーションを実行せずにテストするための強力なツールです。

```python
# tests/test_main.py
from typer.testing import CliRunner
from my_cli.main import app

runner = CliRunner()

def test_app_help():
    """Test help message display"""
    result = runner.invoke(app, ['--help'])
    assert result.exit_code == 0
    assert 'Usage:' in result.stdout
    assert 'Options:' in result.stdout

def test_create_command_success():
    """Test successful project creation"""
    result = runner.invoke(app, ['create', 'myproject'])
    assert result.exit_code == 0
    assert 'Creating project: myproject' in result.stdout

def test_create_command_with_template():
    """Test project creation with template option"""
    result = runner.invoke(app, ['create', 'myproject', '--template', 'react'])
    assert result.exit_code == 0
    assert 'Template: react' in result.stdout

def test_create_command_missing_argument():
    """Test error handling for missing arguments"""
    result = runner.invoke(app, ['create'])
    assert result.exit_code != 0
    assert 'Missing argument' in result.stdout

def test_create_command_invalid_option():
    """Test error handling for invalid options"""
    result = runner.invoke(app, ['create', 'myproject', '--invalid-option'])
    assert result.exit_code != 0
    assert 'No such option' in result.stdout

def test_list_command():
    """Test list command"""
    result = runner.invoke(app, ['list'])
    assert result.exit_code == 0
    assert 'Listing projects' in result.stdout

def test_list_command_with_all_flag():
    """Test list command with --all flag"""
    result = runner.invoke(app, ['list', '--all'])
    assert result.exit_code == 0
    assert 'all=True' in result.stdout
```

## conftest.pyでのフィクスチャ定義

共通のテストデータやセットアップをフィクスチャとして定義します。

```python
# tests/conftest.py
import pytest
from pathlib import Path
from typer.testing import CliRunner
from my_cli.main import app
import tempfile
import shutil

@pytest.fixture
def runner():
    """CliRunner fixture"""
    return CliRunner()

@pytest.fixture
def temp_dir():
    """Temporary directory fixture"""
    temp_path = Path(tempfile.mkdtemp())
    yield temp_path
    shutil.rmtree(temp_path)

@pytest.fixture
def sample_project_config():
    """Sample project configuration"""
    return {
        'name': 'test-project',
        'template': 'react',
        'features': ['typescript', 'eslint', 'prettier']
    }

@pytest.fixture
def mock_file_system(tmp_path):
    """Mock file system for testing"""
    project_dir = tmp_path / "projects"
    project_dir.mkdir()

    # Create sample project directories
    (project_dir / "project-a").mkdir()
    (project_dir / "project-b").mkdir()

    return project_dir

@pytest.fixture(autouse=True)
def reset_state():
    """Reset global state before each test"""
    # Reset any global state here
    yield
    # Cleanup after test
```

フィクスチャを使用したテスト例は以下の通りです。

```python
# tests/test_commands/test_create.py
import pytest
from pathlib import Path

def test_create_with_temp_dir(runner, temp_dir):
    """Test project creation in temporary directory"""
    result = runner.invoke(
        app,
        ['create', 'myproject', '--output', str(temp_dir)]
    )
    assert result.exit_code == 0
    assert (temp_dir / 'myproject').exists()

def test_create_with_config(runner, temp_dir, sample_project_config):
    """Test project creation with configuration"""
    result = runner.invoke(
        app,
        [
            'create',
            sample_project_config['name'],
            '--template', sample_project_config['template'],
            '--output', str(temp_dir)
        ]
    )
    assert result.exit_code == 0
```

## モックを使った外部依存のテスト

外部API、ファイルシステム、データベースなどの依存関係をモックでテストします。

```python
# tests/test_commands/test_create.py
from unittest.mock import patch, MagicMock, call
import pytest

def test_create_project_with_file_system_mock(runner):
    """Test project creation with file system mock"""
    with patch('my_cli.core.project.FileSystem') as mock_fs:
        mock_instance = MagicMock()
        mock_fs.return_value = mock_instance

        result = runner.invoke(app, ['create', 'myproject'])

        assert result.exit_code == 0
        mock_instance.create_directory.assert_called_once()
        mock_instance.create_files.assert_called()

def test_create_with_api_call_mock(runner):
    """Test project creation with API call mock"""
    with patch('my_cli.core.project.fetch_template') as mock_fetch:
        mock_fetch.return_value = {'name': 'react', 'version': '18.0.0'}

        result = runner.invoke(
            app,
            ['create', 'myproject', '--template', 'react']
        )

        assert result.exit_code == 0
        mock_fetch.assert_called_once_with('react')

def test_create_with_multiple_mocks(runner):
    """Test with multiple mocked dependencies"""
    with patch('my_cli.core.project.FileSystem') as mock_fs, \
         patch('my_cli.core.project.GitRepository') as mock_git, \
         patch('my_cli.core.project.PackageManager') as mock_pm:

        result = runner.invoke(app, ['create', 'myproject', '--git'])

        assert result.exit_code == 0
        mock_fs.return_value.create_directory.assert_called()
        mock_git.return_value.initialize.assert_called()
        mock_pm.return_value.install.assert_called()

def test_create_with_api_error(runner):
    """Test error handling when API fails"""
    with patch('my_cli.core.project.fetch_template') as mock_fetch:
        mock_fetch.side_effect = ConnectionError('API unavailable')

        result = runner.invoke(app, ['create', 'myproject', '--template', 'react'])

        assert result.exit_code != 0
        assert 'Error' in result.stdout or 'Error' in result.stderr
```

## pytest-mockプラグインの活用

pytest-mockを使うと、より簡潔にモックを記述できます。

```python
# tests/test_commands/test_list.py
import pytest

def test_list_projects_with_mocker(runner, mocker):
    """Test list command with pytest-mock"""
    mock_get_projects = mocker.patch(
        'my_cli.core.project.get_all_projects',
        return_value=['project-a', 'project-b', 'project-c']
    )

    result = runner.invoke(app, ['list'])

    assert result.exit_code == 0
    mock_get_projects.assert_called_once()
    assert 'project-a' in result.stdout
    assert 'project-b' in result.stdout
    assert 'project-c' in result.stdout

def test_list_projects_empty(runner, mocker):
    """Test list command with no projects"""
    mocker.patch(
        'my_cli.core.project.get_all_projects',
        return_value=[]
    )

    result = runner.invoke(app, ['list'])

    assert result.exit_code == 0
    assert 'No projects found' in result.stdout
```

## パラメータライズドテスト

複数の入力パターンを効率的にテストします。

```python
# tests/test_commands/test_create.py
import pytest

@pytest.mark.parametrize('project_name,expected', [
    ('valid-project', 0),
    ('ValidProject123', 0),
    ('valid_project', 0),
    ('123-project', 0),
])
def test_create_with_valid_names(runner, project_name, expected):
    """Test project creation with various valid names"""
    result = runner.invoke(app, ['create', project_name])
    assert result.exit_code == expected

@pytest.mark.parametrize('project_name,error_msg', [
    ('ab', 'at least 3 characters'),
    ('a' * 51, 'at most 50 characters'),
    ('invalid project', 'invalid characters'),
    ('invalid@project', 'invalid characters'),
])
def test_create_with_invalid_names(runner, project_name, error_msg):
    """Test project creation with invalid names"""
    result = runner.invoke(app, ['create', project_name])
    assert result.exit_code != 0
    assert error_msg in result.stdout.lower()

@pytest.mark.parametrize('template,features', [
    ('react', ['typescript', 'eslint']),
    ('vue', ['typescript', 'pinia']),
    ('angular', ['typescript', 'rxjs']),
])
def test_create_with_templates(runner, mocker, template, features):
    """Test project creation with different templates"""
    mock_create = mocker.patch('my_cli.core.project.create_from_template')

    result = runner.invoke(
        app,
        ['create', 'myproject', '--template', template]
    )

    assert result.exit_code == 0
    mock_create.assert_called_once()
```

## 例外とエラーハンドリングのテスト

```python
# tests/test_core/test_project.py
import pytest
from my_cli.core.project import ProjectManager, ProjectError

def test_create_existing_project_raises_error():
    """Test that creating existing project raises error"""
    manager = ProjectManager()
    with pytest.raises(ProjectError, match="Project already exists"):
        manager.create('existing-project')

def test_delete_nonexistent_project_raises_error():
    """Test that deleting non-existent project raises error"""
    manager = ProjectManager()
    with pytest.raises(ProjectError, match="Project not found"):
        manager.delete('nonexistent-project')

def test_invalid_template_raises_error():
    """Test that invalid template raises error"""
    manager = ProjectManager()
    with pytest.raises(ValueError, match="Invalid template"):
        manager.create('myproject', template='invalid')
```

## カバレッジ測定と分析

pytest-covを使ってコードカバレッジを測定します。

```bash
# 基本的なカバレッジ測定
pytest --cov=src/my_cli

# HTML形式のレポート生成
pytest --cov=src/my_cli --cov-report=html

# 欠落している行を表示
pytest --cov=src/my_cli --cov-report=term-missing

# ブランチカバレッジを含む
pytest --cov=src/my_cli --cov-report=term-missing --cov-branch

# 最小カバレッジ率を指定（80%未満で失敗）
pytest --cov=src/my_cli --cov-fail-under=80
```

pyproject.tomlでカバレッジ設定を永続化できます。

```toml
# pyproject.toml
[tool.coverage.run]
source = ["src/my_cli"]
branch = true
omit = [
    "*/tests/*",
    "*/test_*.py",
    "*/__pycache__/*",
    "*/site-packages/*",
]

[tool.coverage.report]
precision = 2
show_missing = true
skip_covered = false
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
]

[tool.coverage.html]
directory = "htmlcov"
```

## 統合テストとユニットテストの分離

マーカーを使ってテストを分類します。

```python
# tests/test_integration.py
import pytest

@pytest.mark.integration
def test_full_project_workflow(runner, temp_dir):
    """Integration test for complete workflow"""
    # Create project
    result = runner.invoke(
        app,
        ['create', 'myproject', '--output', str(temp_dir)]
    )
    assert result.exit_code == 0

    # List projects
    result = runner.invoke(app, ['list'])
    assert result.exit_code == 0
    assert 'myproject' in result.stdout

    # Delete project
    result = runner.invoke(app, ['delete', 'myproject', '--yes'])
    assert result.exit_code == 0

@pytest.mark.unit
def test_project_name_validation():
    """Unit test for project name validation"""
    from my_cli.core.validators import validate_project_name

    assert validate_project_name('valid-project') == 'valid-project'

    with pytest.raises(ValueError):
        validate_project_name('ab')
```

テストの実行を制御できます。

```bash
# ユニットテストのみ実行
pytest -m unit

# 統合テストを除外
pytest -m "not integration"

# 遅いテストを除外
pytest -m "not slow"

# 複数のマーカーを組み合わせ
pytest -m "unit and not slow"
```

## Python CLIテストチェックリスト

効果的なCLIテストのために、以下を確認しましょう。

- [ ] CliRunnerを使用してCLIコマンドをテストしている
- [ ] 正常系と異常系の両方をテストしている
- [ ] 外部依存をモックで置き換えている
- [ ] フィクスチャで共通のテストデータを管理している
- [ ] パラメータライズドテストで複数パターンを効率的にテストしている
- [ ] エラーメッセージの内容を検証している
- [ ] 終了コードを検証している
- [ ] カバレッジが80%以上である
- [ ] 統合テストとユニットテストを適切に分離している
- [ ] CI/CDパイプラインでテストが自動実行される

## まとめ

pytestとCliRunnerを組み合わせることで、CLIツールを効率的かつ包括的にテストできます。モックやフィクスチャを活用し、外部依存を排除したユニットテストと、実際の動作を確認する統合テストをバランスよく実装することが重要です。

次章では、開発したPython CLIツールをPyPIに公開し、ユーザーに配布する方法を学びます。

## 参考文献

- [pytest公式ドキュメント](https://docs.pytest.org/)
- [pytest-cov公式ドキュメント](https://pytest-cov.readthedocs.io/)
- [Typer Testing - 公式ドキュメント](https://typer.tiangolo.com/tutorial/testing/)
- [unittest.mock公式ドキュメント](https://docs.python.org/3/library/unittest.mock.html)

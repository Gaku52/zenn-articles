# Python CLIテスト

本章では、pytest を使ったPython CLIツールのテスト手法を学びます。

## テスト環境

```bash
pip install pytest pytest-cov
```

## CliRunnerを使ったテスト

```python
# tests/test_cli.py
from typer.testing import CliRunner
from my_cli.main import app

runner = CliRunner()

def test_create_command():
    result = runner.invoke(app, ['create', 'myapp'])
    assert result.exit_code == 0
    assert 'Creating project: myapp' in result.stdout

def test_create_with_template():
    result = runner.invoke(app, ['create', 'myapp', '--template', 'react'])
    assert result.exit_code == 0
    assert 'react' in result.stdout
```

## モックを使ったテスト

```python
from unittest.mock import patch, MagicMock

def test_create_project_with_mock():
    with patch('my_cli.core.project.FileSystem') as mock_fs:
        mock_fs.return_value.create_directory = MagicMock()

        result = runner.invoke(app, ['create', 'myapp'])

        assert result.exit_code == 0
        mock_fs.return_value.create_directory.assert_called_once()
```

## カバレッジ

```bash
pytest --cov=my_cli --cov-report=html
```

## まとめ

pytestとCliRunnerを使うことで、効率的にCLIツールをテストできます。

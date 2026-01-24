# Inquirer インタラクティブUI

Inquirerを使うと、美しいインタラクティブなプロンプトをCLIに追加できます。本章では、様々なプロンプトタイプと実践的な使い方を学びます。

## インストール

```bash
npm install inquirer
npm install -D @types/inquirer
```

## 基本的な使い方

### シンプルな質問

```typescript
import inquirer from 'inquirer'

const answers = await inquirer.prompt([
  {
    type: 'input',
    name: 'name',
    message: 'Project name:',
    default: 'my-project'
  }
])

console.log(`Creating project: ${answers.name}`)
```

実行例:

```bash
$ mycli create
? Project name: (my-project) myapp
Creating project: myapp
```

## プロンプトタイプ

### input（テキスト入力）

```typescript
const answers = await inquirer.prompt([
  {
    type: 'input',
    name: 'name',
    message: 'Project name:',
    default: 'my-project',
    validate: (input) => {
      if (/^[a-z0-9-]+$/.test(input)) {
        return true
      }
      return 'Project name must contain only lowercase letters, numbers, and hyphens'
    }
  }
])
```

### confirm（確認）

```typescript
const answers = await inquirer.prompt([
  {
    type: 'confirm',
    name: 'typescript',
    message: 'Use TypeScript?',
    default: true
  }
])

console.log(`TypeScript: ${answers.typescript}`)
```

### list（選択肢）

```typescript
const answers = await inquirer.prompt([
  {
    type: 'list',
    name: 'template',
    message: 'Select a template:',
    choices: [
      'default',
      'react',
      'vue',
      'nextjs'
    ],
    default: 'react'
  }
])
```

### rawlist（番号付き選択）

```typescript
const answers = await inquirer.prompt([
  {
    type: 'rawlist',
    name: 'template',
    message: 'Select a template:',
    choices: [
      { name: 'Default Template', value: 'default' },
      { name: 'React', value: 'react' },
      { name: 'Vue', value: 'vue' },
      { name: 'Next.js', value: 'nextjs' }
    ]
  }
])
```

### checkbox（複数選択）

```typescript
const answers = await inquirer.prompt([
  {
    type: 'checkbox',
    name: 'features',
    message: 'Select features:',
    choices: [
      { name: 'ESLint', value: 'eslint', checked: true },
      { name: 'Prettier', value: 'prettier', checked: true },
      { name: 'Husky', value: 'husky' },
      { name: 'Commitlint', value: 'commitlint' },
      { name: 'Tailwind CSS', value: 'tailwind' }
    ]
  }
])

console.log(`Selected features: ${answers.features.join(', ')}`)
```

### password（パスワード）

```typescript
const answers = await inquirer.prompt([
  {
    type: 'password',
    name: 'password',
    message: 'Enter password:',
    mask: '*',
    validate: (input) => {
      if (input.length < 8) {
        return 'Password must be at least 8 characters'
      }
      return true
    }
  }
])
```

### editor（エディタ）

```typescript
const answers = await inquirer.prompt([
  {
    type: 'editor',
    name: 'description',
    message: 'Project description:',
    default: 'A new project'
  }
])
```

## バリデーション

### 基本的なバリデーション

```typescript
const answers = await inquirer.prompt([
  {
    type: 'input',
    name: 'email',
    message: 'Email address:',
    validate: (input) => {
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
      if (emailRegex.test(input)) {
        return true
      }
      return 'Please enter a valid email address'
    }
  },
  {
    type: 'input',
    name: 'port',
    message: 'Port number:',
    default: '3000',
    validate: (input) => {
      const port = parseInt(input, 10)
      if (isNaN(port) || port < 1 || port > 65535) {
        return 'Port must be between 1 and 65535'
      }
      return true
    }
  }
])
```

### 非同期バリデーション

```typescript
const answers = await inquirer.prompt([
  {
    type: 'input',
    name: 'projectName',
    message: 'Project name:',
    validate: async (input) => {
      // ディレクトリの存在チェック
      const fs = await import('fs/promises')
      try {
        await fs.access(`./${input}`)
        return 'Project already exists'
      } catch {
        return true
      }
    }
  }
])
```

## 条件分岐

### when（条件付き表示）

```typescript
const answers = await inquirer.prompt([
  {
    type: 'confirm',
    name: 'useDocker',
    message: 'Use Docker?',
    default: false
  },
  {
    type: 'input',
    name: 'dockerImage',
    message: 'Docker image:',
    default: 'node:18',
    when: (answers) => answers.useDocker
  },
  {
    type: 'confirm',
    name: 'useDockerCompose',
    message: 'Use Docker Compose?',
    default: false,
    when: (answers) => answers.useDocker
  }
])
```

## フィルター・変換

### filter

```typescript
const answers = await inquirer.prompt([
  {
    type: 'input',
    name: 'name',
    message: 'Project name:',
    filter: (input) => {
      // 小文字に変換、スペースをハイフンに置換
      return input.toLowerCase().replace(/\s+/g, '-')
    }
  }
])
```

### transformer

```typescript
const answers = await inquirer.prompt([
  {
    type: 'input',
    name: 'apiKey',
    message: 'API Key:',
    transformer: (input, answers, flags) => {
      // 表示時のみマスク（実際の値は変更しない）
      if (flags.isFinal) {
        return input.replace(/./g, '*')
      }
      return input
    }
  }
])
```

## カスタムプロンプト

### 階層的な質問

```typescript
async function promptForConfig() {
  const answers = await inquirer.prompt([
    {
      type: 'input',
      name: 'projectName',
      message: 'Project name:',
      validate: (input) => input.length > 0
    },
    {
      type: 'list',
      name: 'template',
      message: 'Template:',
      choices: ['react', 'vue', 'nextjs']
    }
  ])

  // テンプレート固有の質問
  if (answers.template === 'react') {
    const reactAnswers = await inquirer.prompt([
      {
        type: 'confirm',
        name: 'useRouter',
        message: 'Use React Router?',
        default: true
      },
      {
        type: 'list',
        name: 'stateManagement',
        message: 'State management:',
        choices: ['Context API', 'Redux', 'Zustand', 'None']
      }
    ])

    return { ...answers, ...reactAnswers }
  }

  return answers
}
```

### プログレス表示

```typescript
import ora from 'ora'

async function createProject() {
  const answers = await inquirer.prompt([
    // ... 質問
  ])

  const spinner = ora('Creating project...').start()

  try {
    // プロジェクト作成処理
    await createDirectories(answers)
    spinner.text = 'Installing dependencies...'
    await installDependencies()
    spinner.text = 'Configuring project...'
    await configure(answers)

    spinner.succeed('Project created successfully!')
  } catch (error) {
    spinner.fail('Project creation failed')
    throw error
  }
}
```

## 実践例

### フル機能のプロジェクト作成プロンプト

```typescript
import inquirer from 'inquirer'
import chalk from 'chalk'
import ora from 'ora'

interface ProjectConfig {
  name: string
  template: string
  typescript: boolean
  features: string[]
  packageManager: string
  git: boolean
}

async function promptForProjectConfig(): Promise<ProjectConfig> {
  console.log(chalk.blue.bold('\n🚀 Project Generator\n'))

  const answers = await inquirer.prompt([
    {
      type: 'input',
      name: 'name',
      message: 'Project name:',
      default: 'my-project',
      validate: (input) => {
        if (/^[a-z0-9-]+$/.test(input)) {
          return true
        }
        return 'Project name must contain only lowercase letters, numbers, and hyphens'
      },
      filter: (input) => input.toLowerCase().trim()
    },
    {
      type: 'list',
      name: 'template',
      message: 'Select a template:',
      choices: [
        { name: 'Default', value: 'default' },
        { name: chalk.cyan('React'), value: 'react' },
        { name: chalk.green('Vue'), value: 'vue' },
        { name: chalk.magenta('Next.js'), value: 'nextjs' }
      ]
    },
    {
      type: 'confirm',
      name: 'typescript',
      message: 'Use TypeScript?',
      default: true
    },
    {
      type: 'checkbox',
      name: 'features',
      message: 'Select features:',
      choices: [
        { name: 'ESLint', value: 'eslint', checked: true },
        { name: 'Prettier', value: 'prettier', checked: true },
        { name: 'Husky (Git hooks)', value: 'husky' },
        { name: 'Tailwind CSS', value: 'tailwind' },
        { name: 'Testing (Jest/Vitest)', value: 'testing' }
      ]
    },
    {
      type: 'list',
      name: 'packageManager',
      message: 'Package manager:',
      choices: ['npm', 'yarn', 'pnpm'],
      default: 'npm'
    },
    {
      type: 'confirm',
      name: 'git',
      message: 'Initialize Git repository?',
      default: true
    }
  ])

  return answers as ProjectConfig
}

async function createProject(config: ProjectConfig) {
  console.log(chalk.blue('\n📦 Creating project...\n'))

  const spinner = ora('Setting up project structure').start()

  try {
    // ディレクトリ作成
    await createProjectDirectory(config.name)
    spinner.text = 'Copying template files'

    // テンプレートコピー
    await copyTemplate(config.template, config.name)
    spinner.text = 'Installing dependencies'

    // 依存関係インストール
    await installDependencies(config.packageManager, config.name)
    spinner.text = 'Configuring features'

    // 機能設定
    await configureFeatures(config.features, config.name)

    if (config.git) {
      spinner.text = 'Initializing Git repository'
      await initGit(config.name)
    }

    spinner.succeed(chalk.green('✓ Project created successfully!'))

    // 次のステップを表示
    console.log(chalk.blue('\n📝 Next steps:\n'))
    console.log(`  ${chalk.gray('$')} cd ${config.name}`)
    console.log(`  ${chalk.gray('$')} ${config.packageManager} run dev`)

  } catch (error) {
    spinner.fail(chalk.red('✗ Project creation failed'))
    throw error
  }
}

// 使用例
const config = await promptForProjectConfig()
await createProject(config)
```

## Commander.jsとの統合

```typescript
import { Command } from 'commander'
import inquirer from 'inquirer'

const program = new Command()

program
  .command('create [name]')
  .description('Create a new project')
  .option('-t, --template <type>', 'Template to use')
  .action(async (name, options) => {
    // コマンドライン引数が省略された場合はプロンプト
    const answers = await inquirer.prompt([
      {
        type: 'input',
        name: 'name',
        message: 'Project name:',
        when: !name,
        default: 'my-project'
      },
      {
        type: 'list',
        name: 'template',
        message: 'Template:',
        choices: ['default', 'react', 'vue', 'nextjs'],
        when: !options.template
      }
    ])

    const projectName = name || answers.name
    const template = options.template || answers.template

    console.log(`Creating ${projectName} with ${template} template`)
  })

program.parse()
```

## まとめ

本章では、Inquirerを使ったインタラクティブUIを学びました。

**重要ポイント**:
- 様々なプロンプトタイプを適切に使い分ける
- バリデーションでユーザー入力を検証
- when条件で動的な質問フローを実現
- Commander.jsと組み合わせて柔軟なCLIを構築
- スピナーで処理状況をフィードバック

Inquirerを活用することで、ユーザーフレンドリーなCLIツールを作成できます。次章では、CLIツールのテスト手法を学びます。

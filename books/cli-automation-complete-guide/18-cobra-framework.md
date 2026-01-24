# Cobraフレームワーク

Cobraは、Goで最も人気のあるCLIフレームワークです。本章では、Cobraを使ったCLI開発を学びます。

## 基本的な使い方

```go
// cmd/root.go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
    Use:   "mycli",
    Short: "A powerful CLI tool",
    Long:  `A powerful CLI tool for project management`,
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
}
```

## サブコマンド

```go
// cmd/create.go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
)

var createCmd = &cobra.Command{
    Use:   "create [name]",
    Short: "Create a new project",
    Args:  cobra.ExactArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
        name := args[0]
        template, _ := cmd.Flags().GetString("template")
        fmt.Printf("Creating %s with %s\n", name, template)
    },
}

func init() {
    rootCmd.AddCommand(createCmd)
    createCmd.Flags().StringP("template", "t", "default", "Template to use")
}
```

## フラグ

```go
var verbose bool

createCmd.Flags().BoolVarP(&verbose, "verbose", "v", false, "Verbose output")
```

## まとめ

Cobraを使うことで、強力で使いやすいCLIツールを効率的に開発できます。

---
title: "Kiro CLIのheadless modeでエージェント自動実行できるようになった"
emoji: "🌙"
type: "tech"
topics: [AI, Kiro, GitHub, Swift, 個人開発]
published: false
---

Kiro CLIのカスタムエージェントを使って、iOSアプリのコードベース分析やGitHub Issue自動生成をやっています。ただ、これまでの`Kiro CLI`は認証に対話的な操作が必須で、無人での自動実行ができなかった。

[headless mode](https://kiro.dev/blog/introducing-headless-mode/)がリリースされて、これが解決しました。セットアップからGitHub Actionsでの自動実行まで、実際にやったことを書いていきます。

:::message
この記事はシリーズの3本目です。カスタムエージェントの作り方やモデル比較については前回の記事を参照してください。
- [放置していたiOSアプリをAIに診断させたら、最安モデルが実用的だった](https://zenn.dev/azuritul/articles/article-ios-ai-diagnosis)
- [コスト倍率と品質は比例しない — Kiro CLI 4モデル比較](https://zenn.dev/azuritul/articles/article-kiro-model-comparison)
:::

## セットアップ

### API keyの取得

Kiroのアカウント設定画面からAPI keyを生成します。

![](/images/article-kiro-headless-setup/apikey.png)

### 環境変数の設定

取得したAPI keyを環境変数にセットします。

```bash
export KIRO_API_KEY="取得したキー"
```

### 動作確認

まずは簡単なコマンドで動くことを確認します。

```bash
kiro-cli chat --no-interactive "Hello, headless mode is working?"
```

![](/images/article-kiro-headless-setup/image1.png)

Warningが出ていますが、動作には影響しないので無視します。

## エージェントをheadlessで動かす

動作確認ができたので、[前回の記事](https://zenn.dev/azuritul/articles/article-ios-ai-diagnosis)で作った`Swift Codebase Analyst`エージェントをheadless modeで動かしてみます。まずはGitHub Issueを作らず、分析結果をstdoutに出力するだけの軽いテストから。

```bash
kiro-cli chat --agent "Swift Codebase Analyst" --no-interactive \
  "Read the docs/ directory and apps/mobile-ios/ directory structure. \
   List the top 3 issues you find in the iOS app. \
   Do NOT create GitHub issues, just print findings to stdout."
```

エージェントがコードベースを読み、ファイルパス・行番号付きで問題を報告してきます。

## Issue作成を含むフル実行

stdoutへの出力が確認できたので、次は実際にGitHub Issueを作成するフル実行を試します。長いプロンプトはファイルに保存して`$(cat prompt.txt)`で渡すと管理しやすいです。

```bash
kiro-cli chat --agent "Swift Codebase Analyst" --no-interactive \
  "$(cat .kiro/prompts/analysis.txt)"
```

結果：GitHub Issueが自動作成され、Markdownのフォーマット、バッククォート、ファイルパスがすべて正しく保持されていました。

![](/images/article-kiro-headless-setup/issue.png)

## GitHub Actionsで定期実行する

最初はローカルのcronで定期実行を試みましたが、GitHub Actionsの方がトークン管理・ログ確認・手動再実行がすべてGitHub上で完結するので、はるかに管理しやすいです。プライベートリポジトリではGitHub-hosted runnerに無料枠の制限があるため、self-hosted runnerを使います。

### ワークフローの作成

ワークフローファイル`.github/workflows/nightly-analysis.yml`を作成してpushします。

```yaml
name: Nightly Codebase Analysis

on:
  schedule:
    # 毎日深夜3時（JST）= UTC 18:00
    - cron: '0 18 * * *'
  workflow_dispatch: # 手動実行も可能

jobs:
  analyze:
    runs-on: self-hosted
    permissions:
      contents: read
      issues: write
    steps:
      - uses: actions/checkout@v5
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Load .env
        run: cat ~/.kiro/.env >> "$GITHUB_ENV"

      - name: Run analysis
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          kiro-cli chat \
            --agent "Swift Codebase Analyst" \
            --model qwen3-coder-next \
            --no-interactive \
            "$(cat .kiro/prompts/analysis.txt)"
```

### 手動実行でテスト

ワークフローをpushしたら、GitHubのActionsタブからRun Workflowボタンで手動実行してテストできます。まずは手動実行で問題なく動くことを確認してからscheduleに任せます。

実際にテストした結果、self-hosted runner上でKiro CLIがコードベースを分析し、GitHub Issueが自動作成されることを確認できました。

![](/images/article-kiro-headless-setup/finished.png)

## まとめ

headless modeのセットアップからself-hosted runnerでのGitHub Actions定期実行まで一通り構築できました。前回の記事で構築したワークフロー（カスタムエージェント + スキル参照 + モデル選択）がそのまま非対話環境で使えます。

## 補足

試していて気になった点。

### MCP設定の警告

headless modeで実行すると、以下の警告が出ることがあります。

```
⚠️  WARNING: Failed to retrieve MCP settings; MCP functionality disabled
```

なぜかグローバルレベルにMCP設定が存在していても、プロジェクトレベルの設定がないとこの警告が出ます。回避策として、プロジェクトのルートに空のMCP設定ファイルを置けば警告が消えます。

### 出力のファイルパスが絶対パス

エージェントが報告するファイルパスがデフォルトで絶対パスになります。GitHub Issueとして使うならリポジトリルートからの相対パスの方が読みやすい。実行時プロンプトで「相対パスで書くこと」と指定すれば改善できます。


## 参考

- [Kiro CLI Headless mode](https://kiro.dev/docs/cli/headless/)
- [Kiro CLI Authentication methods](https://kiro.dev/docs/cli/authentication/)
- [Introducing Kiro CLI 2.0](https://kiro.dev/blog/cli-2-0/)
- [Introducing Headless Mode](https://kiro.dev/blog/introducing-headless-mode/)
- [GitHub Actions: Self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners)
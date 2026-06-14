---
title: "使っているAIコーディングエージェントのコンテキストファイルまとめ"
emoji: "🧭"
type: "tech"
topics: [AI, Codex, ClaudeCode, Gemini, Kiro]
published: true
---

:::message
2026年6月時点の整理です。AIコーディングツールの仕様はかなり速く変わるので、実際に導入するときは公式ドキュメントも確認してください。
:::

:::message
読みやすさのために、IDE版とCLI版を厳密には分けず、同じエージェント/ツール群としてまとめている箇所があります。
:::

最近は作業内容によって、Codex、Claude Code、Kiro、Gemini CLIを切り替えています。

各エージェントのコンテキストファイルには、共通しているものもあれば、ツール独自のものもある。忘れがちなので、一度まとめておく。

## コンテキストファイル一覧

| エージェント | Context file名 | プロジェクト内の標準配置 | `AGENTS.md`の扱い | 説明ページ |
|---|---|---|---|---|
| Codex | `AGENTS.md` | `./AGENTS.md` | Native | [Custom instructions with AGENTS.md](https://developers.openai.com/codex/guides/agents-md) |
| Claude Code | `CLAUDE.md` | `./CLAUDE.md` | `@AGENTS.md`でimport | [How Claude remembers your project](https://code.claude.com/docs/en/memory) |
| Gemini CLI | `GEMINI.md` | `./GEMINI.md` | `context.fileName`で指定 | [Provide context with GEMINI.md files](https://geminicli.com/docs/cli/gemini-md/) |
| Kiro | Steering files | `./.kiro/steering/*.md` | `AGENTS.md`も読む | [Steering](https://kiro.dev/docs/steering/) |
| GitHub Copilot | `copilot-instructions.md` | `./.github/copilot-instructions.md` | `AGENTS.md`も読む | [Adding repository custom instructions](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/add-custom-instructions/add-repository-instructions) |

## 階層構造の対応

`global -> project root -> subfolder`のような階層構造は、多くのツールで何らかの形でサポートされています。ただし、すべてのツールが同じファイル名を各階層に置く方式ではありません。

**Codex**

- Global: `~/.codex/AGENTS.md`
- Project root: `./AGENTS.md`
- Subfolder: `./subdir/AGENTS.md`

**Claude Code**

- Global: `~/.claude/CLAUDE.md`
- Project root: `./CLAUDE.md` or `./.claude/CLAUDE.md`
- Subfolder: `./subdir/CLAUDE.md` / `.claude/rules/*.md`

**Gemini CLI**

- Global: `~/.gemini/GEMINI.md`
- Project root: `./GEMINI.md`
- Subfolder: `./subdir/GEMINI.md`

**Kiro**

- Global: `~/.kiro/steering/*.md`
- Project root: `./.kiro/steering/*.md`
- Path-specific: `fileMatch` / `manual` / `auto` inclusion

**GitHub Copilot**

- Global: Personal / organization instructions
- Project root: `./.github/copilot-instructions.md`
- Path-specific: `./.github/instructions/*.instructions.md` / `AGENTS.md`

## 補足: Kiroのsteering files

Kiroでは、プロジェクト指示はsteering filesと呼ばれます。ほかのツールとは違い、複数のファイルに分けて管理できます。

ワークスペース単位のsteering filesは次の場所に置きます。

```text
.kiro/steering/*.md
```

グローバルなsteering filesは次の場所です。

```text
~/.kiro/steering/*.md
```

基本的なsteering filesとしては、次の3つが用意されています。

```text
.kiro/steering/product.md
.kiro/steering/tech.md
.kiro/steering/structure.md
```

それぞれ、プロダクトの目的、技術スタック、プロジェクト構造を説明するファイルです。
また、steering filesには読み込みタイミングを指定するinclusion modeがあります。

```md
---
inclusion: always
---
```

```md
---
inclusion: fileMatch
fileMatchPattern: "**/*.tsx"
---
```

```md
---
inclusion: manual
---
```

```md
---
inclusion: auto
name: api-design
description: REST API design patterns and conventions.
---
```

## AGENTS.mdの位置づけ

エージェント向けのコンテキストファイルには、公式に統一された共通標準はまだありません。ただし、[Agents.md](https://agents.md/) のように、`AGENTS.md`を共通の指示ファイルとして扱う動きがあります。

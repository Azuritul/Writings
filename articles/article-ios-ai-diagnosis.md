---
title: "放置していたiOSアプリをAIに診断させたら、最安モデルが実用的だった"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [AI, iOS, Swift, 個人開発, Kiro]
published: true
---

個人開発で作った iOS の語学学習アプリを、半年くらい放置していました。  
バックエンドを`SwiftData`から`Firestore`に移行する途中で別の仕事が入り、「どこまでやったのか」「何が残っているのか」がわからなくなってしまったからです。

自分でコードとドキュメントを最初から全部読み直すのは正直つらい。  
そこで今回は、`Kiro CLI`の **カスタムエージェント** にコードベースを読ませて、問題点をGitHub Issueとして自動生成させてみました。

この記事では、

- 放置していた iOS アプリの現状を AI エージェントに診断させる手順
- Kiro CLI で「Swift 用コードベース分析エージェント」を定義する方法
- 複数モデルでの実行結果と、安いモデルでどこまで戦えるか

あたりをまとめます。

## しばらく放置したiOSアプリ

個人開発でiOSの語学学習アプリを作っています。`SwiftUI` + `Firebase`で、モノレポ構成（iOS / admin web / landing page）。

少し前まで、バックエンドの刷新をやっていました。`SwiftData`（ローカルオンリー）から`Firestore`（クラウド同期）への移行です。`SwiftData`でも`CloudKit`経由でiOSデバイス間の同期はできますが、あくまでAppleの生態系内だけの話で、Web版やAndroid版は対象外になります。そこは考慮せず、プラットフォームを跨げる`Firestore`に移行することにしました。動機はシンプルで、

- デバイスやプラットフォームを問わず同期したい
- Web版も展開したい（`Firestore`なら共有データ層になる）
- バックエンドがあれば機能追加が柔軟にできる

ただ、この移行がかなり大きかった。データモデルの再設計、既存データのマイグレーション、iOS側のCRUD全書き換え。途中まで進めたところで別の仕事が入り、そのまま放置してしまいました。

しばらく経って、久しぶりにリポジトリを開いてみたものの、正直どこまでやったか覚えていない。

一応TODOリストは残っていました。でも今のコードと一致しているかは怪しい。`SwiftData`の残骸と新しい`Firestore`コードが混在していて、どこが完了でどこが未着手なのか、コードを読んでも判断しづらい状態でした。

再開するにはまず現状把握が必要です。でもそれが一番面倒。

## AIエージェントに診断させるアイデア

自分で全ファイル読み直すのは時間がかかります。しかもドキュメント・コード・TODO・データモデルを横断的にチェックしたい。手動でやると数日は潰れると思います。

そこで思いついたのが、AIエージェントにコードベースを丸ごと読ませて、問題点をGitHub Issueとして自動生成させることでした。現状把握と課題整理を一発で終わらせたかった。

ツールはKiro CLIを使いました。普段Kiro IDEは使っていますが、CLIの方はそこまで詳しくなかった。ただ、カスタムエージェントという仕組みが定義できることは知っていました。しかも設定ファイルをグローバルに置けるので、複数プロジェクトで使い回せそうだと思って試してみることにしました。

選んだ理由をまとめるとこのあたりです：

- **カスタムエージェント**が定義できる — 専用の「コードベース分析エージェント」を作れる
- **グローバル配置** — 設定ファイルを`~/.kiro/agents/`に置けば、どのプロジェクトでも使える
- **スキル連携** — swiftui-proやswift-concurrency-proといったSwift専門の知見を注入できる
- **シェルアクセス** — `gh issue create`で直接GitHub Issueを作れる
- **モデル選択** — コスト重視で安いモデルも選べる
- **IDE非依存** — CLIなのでターミナルから独立して実行できる（Kiro IDEで別の作業をしながら裏で回せる）

## エージェントを作る

### グローバルエージェントとして配置

エージェントの設定ファイルは`~/.kiro/agents/`に置きました。こうするとどのプロジェクトからでも呼び出せます。プロジェクトごとにコピーする必要がありません。

```json
{
  "name": "Swift Codebase Analyst",
  "description": "Analyzes Swift/iOS codebase using Swift-specific best practices",
  "prompt": "file://./prompts/swift-codebase-analyst.md",
  "tools": [
    "read",
    "shell",
    "report",
    "thinking",
    "grep",
    "glob"
  ],
  "allowedTools": [
    "read",
    "shell",
    "report",
    "thinking",
    "grep",
    "glob"
  ],
  "resources": [
    "skill://~/.kiro/skills/swiftui-pro/SKILL.md",
    "skill://~/.kiro/skills/swift-concurrency-pro/SKILL.md"
  ],
  "toolsSettings": {},
  "includeMcpJson": false,
  "model": "qwen3-coder-next"
}
```

いくつかポイントがあります。

- **`write`ツールを入れていない** — 分析専用なのでコードを変更させたくない。読み取りとシェル（Issue作成用）だけです
- **スキルを`resources`で参照** — `skill://`で指定すると、メタデータだけ先に読み込まれて、必要になったときに全文が展開されます。コンテキストを無駄に消費しません
- **`allowedTools`を`tools`と同じにした** — 無人実行を想定しているので、毎回許可を求められると困ります
- **モデルは`qwen3-coder-next`** — 一番安い（Autoの1/20のコスト）。バッチ分析なのでレイテンシは気にしません

### プロンプト設計

プロンプトは別ファイル（`~/.kiro/agents/prompts/swift-codebase-analyst.md`）に切り出しました。JSONに直接書くと読みづらいので。

中身はざっくりこんな構成です：

1. 役割の定義 — 「シニアiOSエンジニアとしてコードレビューを行う」
2. 分析の焦点 — アーキテクチャ、コードスメル、命名、テスタビリティなど（`SwiftUI`/`Concurrency`の詳細はスキルに委譲）
3. 出力フォーマット — `gh issue create`で作るIssueの構造（Problem / Affected files / Proposed solution / Complexity）
4. ルール — ファイル名・行番号を具体的に、1Issue1発見、ソースコードは変更しない

`SwiftUI`や`Swift Concurrency`のベストプラクティスはスキル側が持っているので、プロンプトではそれ以外の領域（アーキテクチャ、テスト、命名規則など）に集中させました。重複を避けることでプロンプトがシンプルになります。

## 実行してみた

対象プロジェクトのディレクトリに移動して、エージェントを起動します。

```zsh
kiro-cli chat --agent "Swift Codebase Analyst"
```

最初に投げたプロンプト（V1）はこんな感じです。モノレポ全体を分析対象にしていました：

```markdown
This is a Swift/iOS + Next.js monorepo project that has been untouched for a while.

CRITICAL: This is analysis only. Do NOT modify any files or run destructive commands.

Conduct a comprehensive repository-wide analysis. For each finding, create a GitHub issue using `gh issue create` with label `refactor`. Limit to 20 issues max.

Follow this analysis order:

1. **Read docs/ first** — Read all documentation in docs/ (architecture, migrations, data-models, admin, mobile) to understand the intended design, migration status, and architectural decisions. This is your baseline for everything that follows.

2. **iOS app (apps/mobile-ios/)** — Verify actual implementation against what the docs describe. Identify where development stopped: unimplemented features, stubbed methods, disabled UI, commented-out code. Use the swiftui-pro and swift-concurrency-pro skills to evaluate code quality.

3. **Admin web app (apps/admin-web/)** — Analyze completeness relative to its documented scope. Check if services, types, and components align with the intended Firestore structure.

4. **Cross-platform consistency** — Compare iOS and web data models against the documented Firestore structure. Flag only genuine mismatches, not intentional platform-specific design choices.

5. **TODO/FIXME scan** — Collect all TODO/FIXME comments across both platforms. Group by priority (blocking vs. nice-to-have).

6. **Roadmap issue** — As the final issue, create a single "Resume Development Roadmap" issue containing a prioritized, step-by-step plan to get the app to a shippable state. Reference the other issues you created.

For each GitHub issue, include:
- Problem: what's wrong and why it matters
- Affected files: specific file paths and line ranges
- Proposed solution: concrete next steps
- Complexity: S / M / L
```

### 結果

4分51秒。コスト4.05クレジット。
GitHub Issueが自動で作成されました。ファイル名、行番号、具体的な修正提案付き。

正直、ここまで具体的な出力が出るとは思っていなかった。

### 実際に作られたIssueの例

たとえば`SwiftData`の残骸に関するIssue。エージェントはこんな内容を出してきました：

（※ 以降のIssue例では、ユーザー名もしくはアプリ名やファイルパスの一部をぼかしています）

```markdown
## Problem

The iOS app has commented-out SwiftData model container initialization in `AppMain.swift` (lines 27-37), but legacy SwiftData models (`Deck.swift`, `Glossary.swift`, `LearningDay.swift`) still exist and are referenced in multiple places. This creates confusion about the data architecture and potential bugs.

**Evidence:**
- `AppMain.swift` lines 27-37: ModelContainer initialization commented out
- `DeckSelectionView.swift`: Still has `convertToSwiftDataDeck()` and `convertToSwiftDataGlossary()` methods
- `GlossaryCardStackViewModel.swift`: Imports SwiftData but all operations commented out
- `LearningTopView.swift`, `StudyDeckView.swift`: Import SwiftData but use Firestore

**Why it matters:**
- Code confusion: Developers may think SwiftData is still in use
- Dead code maintenance burden
- Potential bugs if SwiftData code is accidentally uncommented
- Inconsistent data access patterns

## Affected files

- `/Users/xxx/Projects/App/Flashcard/apps/mobile-ios/Sources/AppMain.swift` - ModelContainer disabled (lines 27-37)
- `/Users/xxx/Projects/App/Flashcard/apps/mobile-ios/Sources/Model/Deck.swift` - Legacy SwiftData model
- `/Users/xxx/Projects/App/Flashcard/apps/mobile-ios/Sources/Model/Glossary.swift` - Legacy SwiftData model
- `/Users/xxx/Projects/App/Flashcard/apps/mobile-ios/Sources/Model/LearningDay.swift` - Legacy SwiftData model
- `/Users/xxx/Projects/App/Flashcard/apps/mobile-ios/Sources/Screen/Learn/DeckSelectionView.swift` - Conversion methods (lines 50-100)
- `/Users/xxx/Projects/App/Flashcard/apps/mobile-ios/Sources/Screen/Learn/Stack/GlossaryCardStackViewModel.swift` - SwiftData import, commented operations
- `/Users/xxx/Projects/App/Flashcard/apps/mobile-ios/Sources/Screen/Learn/LearningTopView.swift` - SwiftData import
- `/Users/xxx/Projects/App/Flashcard/apps/mobile-ios/Sources/Screen/Learn/StudyDeckView.swift` - SwiftData import

## Proposed solution

1. **Remove legacy SwiftData models** (`Deck.swift`, `Glossary.swift`, `LearningDay.swift`)
2. **Remove conversion methods** from `DeckSelectionView.swift`
3. **Remove SwiftData imports** from views that no longer use them
4. **Update documentation** to clarify Firestore is the sole persistence layer
5. **Archive legacy models** in a separate `Legacy/` folder if needed for reference

## Complexity estimate

**S** - Requires:
- File deletions and import removals
- Simple refactoring
- No data migration needed

```

8ファイルを特定して、行番号付きで「ここにデッドコードがある」と指摘しています。しかも「なぜ問題か」の説明と具体的な修正手順まで付いている。


### プロンプトの改善

最初のプロンプト（V1）はモノレポ全体を分析対象にしていました。でも実行後に気づいたのは、admin-webのUI分析にトークンが無駄に使われていたこと。自分が知りたいのはiOS側の状態であって、admin-webのコンポーネント品質ではありません。

ただ、admin-webを完全に無視すると`Firestore`のデータモデル整合性チェックができなくなります。iOSとwebで同じ`Firestore`を参照しているので、型の不整合は見つけてほしい。

そこでV2ではスコープを絞りました：

```
SCOPE:
Primary focus is the iOS app under apps/mobile-ios/.

Read apps/admin-web/ types and Firestore service files ONLY to cross-check data model consistency.
Do not analyze the web app's UI, components, or implementation quality.
```

この一文を追加しただけで、分析がiOSに集中するようになりました。モノレポでエージェントを使うときは、スコープ制御がかなり重要だと感じています。

## モデル比較：安いモデルで十分なのか？

エージェントが動くことは確認できました。次に気になったのはコストです。

Kiro CLIは複数のモデルを選べます。一番安い`Qwen3 Coder Next`（Autoの1/20）で動かしたけど、正直`Auto`以外のモデルについてはあまり詳しくなく、ネット上の紹介を見た程度でした。実際の品質差がどのくらいあるのかわからない。

ゆくゆくはこの分析を夜中の誰もいない時間帯にエージェントへ自動で走らせたいと考えています。誰も待っていないので遅くてもいいし、安ければ安いほどいい。であれば、安いモデルで十分な品質が出るのかを確認しておきたかった。

同じリポジトリに対して4つのモデルで実行してみました。

| モデル | コスト倍率 | クレジット | 時間 | 品質 |
|---|---|---|---|---|
| Haiku 4.5 | 0.4x | 2.34 | 3m 33s | 表面的。ファイル名あり、具体的提案なし |
| Qwen3 Coder Next | 0.05x | 4.05 | 4m 51s | 良好。ファイル + 行番号 + 具体的提案 |
| Auto | 1.0x | 9.91 | 9m 18s | 最高。複数ソース横断、詳細な提案 |
| MiniMax M2.5 | 0.25x | 12.53 | 20m 8s | 弱い。ファイル名なし、曖昧 |

結果を見て思ったのは、コスト倍率と品質が比例していないということでした。

**Qwen3(0.05x)が、MiniMax(0.25x)より良い結果を出した。** MiniMaxは20分もかけたのにファイルパスすら出力できていませんでした。コスト倍率と品質は比例しない。

もう一つ気づいたのは、`Qwen3`と`Auto`だけがMarkdownを正しく使ってIssueを書いていたことです。`Haiku`と`MiniMax`はプレーンテキストで、GitHubで見るとコードと地の文の区別がつかない。GitHub Issueとして使うなら、Markdown対応は地味に重要です。これはおそらくsteeringファイルやプロンプトで「Markdownで書くこと」と明示すれば改善できそうですが、今回は試していません。今後検証してみたいところです。

詳しいモデル比較（同じ問題に対する4モデルのIssue全文比較）は別記事にまとめる予定です。

### 使い分けの結論

- **一回きりの深い分析** → **Auto**。品質が段違い
- **定期的なバッチ分析** → **Qwen3 Coder Next**。コスパいい
- **ざっくり確認** → **Haiku 4.5**。速くて安い
- **MiniMax M2.5** → このワークフローには向いていなかった

## 学んだこと

いくつか実践的なTipsが見えてきました。

**プロンプトのスコープ制御が重要です。** 当たり前のことですが、モノレポでは特に意識しないと引っかかります。「全部分析して」だとトークンが分散して、本当に知りたい部分の分析が浅くなります。「ここだけ深く、ここは参照だけ」と明示するだけで結果が変わった。

**`write`ツールを外すだけで安全な分析エージェントになります。** コードを変更される心配がないので、安心して放置できる。無人実行するなら必須の設定だと思います。ただし`shell`は付与しているので、完全に安心とは言えません。`toolsSettings`でシェルコマンドを`gh`系に制限するなど、もう少し絞れるか調べたいところです。

**スキル連携でドメイン知識を注入できます。** コミュニティが公開しているスキルが多数あり、エージェントの`resources`に指定するだけで専門知識を活用できます。今回はSwift関連のスキルを参照させましたが、プロンプトに全部書く必要がなくなるのは楽でした。

**グローバルエージェントにすれば再利用できます。** `~/.kiro/agents/`に置いておけば、どのプロジェクトでも呼び出せます。プロジェクト固有の文脈が必要なら、プロジェクト側の`.kiro/steering/`に追加すればいい。

**モデル選択は「コスト」だけでなく「指示への忠実度」が重要です。** 高いモデルが必ずしも良いわけではなく、プロンプトの出力フォーマットをちゃんと守るかどうかが実用上は大きい。

## 今後やりたいこと

- **エージェントの指示をもっと絞る** — 今回はバックエンド移行の現状把握とリファクタリング提案を一度にやらせた結果、Issueの量が膨大になってしまいました。「移行の進捗確認だけ」「コード品質のチェックだけ」のように目的を分けて、エージェントの定義自体をもっと改善してみたい
- **定期実行** — cronで夜中に回して、リファクタリングバックログを自動更新したい（認証トークンの有効期限は要検証）

## まとめ

一番意外だったのは、最安モデル（Autoの1/20）が5倍高いモデルより良い結果を出したこと。コスト倍率と品質は比例しない。実用上効いたのは「指示どおりの出力フォーマットを守るかどうか」だった。

個人開発者にとって、放置プロジェクトの再開は心理的ハードルが高いです。「まず何をすればいいかわからない」が一番の障壁だったりします。そのハードルを、AIエージェントがかなり下げてくれました。
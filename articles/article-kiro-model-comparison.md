---
title: "コスト倍率と品質は比例しない — Kiro CLI 4モデル比較"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [AI, iOS, Swift, 個人開発, Kiro]
published: true
---

Kiro CLIのカスタムエージェントでiOSアプリのコードベースを分析させて、リファクタリング提案をGitHub Issueとして自動生成する — というワークフローを[前回の記事](https://zenn.dev/azuritul/articles/article-ios-ai-diagnosis)で紹介しました。

ゆくゆくはこの分析を夜中に自動で回したい。であれば安いモデルで十分な品質が出るのかを知っておきたかった。同じリポジトリ、同じプロンプトで4つのモデルを実行して、生成されたIssueの品質を比較しました。結論から言うと、コスト倍率と品質は比例しなかった。

Kiro CLIでモデル選びに迷っている人や、自分と同じように「安いモデルでどこまでいけるのか」が気になっている人の参考になればと思います。

## 前提：何をやったか

個人開発のiOSアプリ（`SwiftUI` + `Firebase`、モノレポ構成）を、Kiro CLIのカスタムエージェントに分析させています。エージェントはコードベースを読んで、リファクタリング提案をGitHub Issueとして`gh issue create`で自動作成します。

エージェントの設定や実行方法は前回の記事に書いたので、ここでは省略します。ポイントだけ再掲すると：

- 読み取り専用（`write`ツールなし）の分析エージェント
- `swiftui-pro`と`swift-concurrency-pro`スキルを`resources`で参照
- `gh issue create`でGitHub Issueを直接作成
- 無人実行を想定（`allowedTools` = `tools`）

## 比較した4モデル

Kiro CLIでは複数のモデルが選べます。今回試したのはこの4つです。

| モデル | コスト倍率（vs Auto） | コンテキスト長 | ステータス |
|---|---|---|---|
| Haiku 4.5 | 0.4x | 200K | Active |
| Qwen3 Coder Next | 0.05x | 256K | Experimental |
| Auto | 1.0x | — | Active（スマートルーティング） |
| MiniMax M2.5 | 0.25x | 200K | Experimental |

選んだ理由はシンプルで、価格帯がバラけるように選びました。最安の`Qwen3`（0.05x）から標準の`Auto`（1.0x）まで。

## 結果サマリー

| モデル | クレジット | 実行時間 | 品質 |
|---|---|---|---|
| Haiku 4.5 | 2.34 | 3m 33s | 表面的。ファイル名はあるが具体的提案なし |
| Qwen3 Coder Next | 4.05 | 4m 51s | 良好。ファイル + 行番号 + 具体的な修正提案 |
| Auto | 9.91 | 9m 18s | 最高。複数ソース横断、詳細なアーキテクチャ提案 |
| MiniMax M2.5 | 12.53 | 20m 8s | 弱い。ファイル名なし、曖昧な指摘 |

⚠️ 条件の違いについて：`Haiku`・`Qwen3`・`MiniMax`はモノレポ全体を対象にしたV1プロンプトで実行しましたが、`Auto`だけはiOSに絞ったV2プロンプトで実行しています。同じIssueの出力品質の比較は有効ですが、実行時間やクレジットの直接比較は厳密ではありません。

一番高くて一番遅いのが`MiniMax`だった。しかも品質が期待より低かった。これは想定外だった。


## 同じIssueを4モデルで比較する

数字だけだと品質の差がわかりづらいので、4モデルが共通して検出した同じ問題のIssue全文を並べます。

対象は「`SwiftData`のデッドコード削除」。`SwiftData`から`Firestore`への移行が途中で止まっていて、旧`SwiftData`のコードが残っている、という問題です。4モデルとも検出しましたが、出力の粒度がまったく違いました。

（※ Issue例のアプリ名やファイルパスの一部はぼかしています）

### Haiku 4.5 — 速いけど浅い

```markdown
**Problem:**
The SwiftData migration is 98% complete with the ModelContainer fully disabled
in AppMain.swift. However, three legacy schema files remain in the codebase:
- DeckSchemaV1.swift
- GlossarySchemaV1.swift
- LearningDaySchemaV1.swift

These files are no longer needed since Firestore is the primary data source.
Keeping them creates confusion about the actual data persistence strategy and
increases maintenance burden.

**Affected files:**
- Sources/Model/DeckSchemaV1.swift
- Sources/Model/GlossarySchemaV1.swift
- Sources/Model/LearningDaySchemaV1.swift
- Sources/AppMain.swift (lines 40-50, ModelContainer initialization commented out)

**Proposed solution:**
1. Delete the three legacy schema files
2. Verify no other files import or reference these schemas
3. Remove any remaining SwiftData imports from non-legacy files
4. Update CLAUDE.md to remove references to SwiftData schemas
5. Confirm all tests pass after cleanup

**Complexity:** S
```

スキーマファイル3つとエントリポイントは見つけている。ただ、Viewやビューモデルに残っている`SwiftData`のimportやデッドコードには触れていない。「削除して、テスト通るか確認して」で終わっていて、具体的にどのファイルのどの行を直すかまでは踏み込んでいない。

3分33秒で2.34クレジット。速さは圧倒的。ざっくり「何が残ってるか」を知りたいだけなら十分だけど、このIssueを見て作業を始められるかというと微妙。

### MiniMax M2.5 — 高くて遅くて曖昧

```markdown
**Problem**: The migration documentation states SwiftData is 98% complete and
disabled, but legacy schema files remain in the codebase. These files are no
longer needed and create confusion about the current architecture.

**Affected files**:
-  (legacy SwiftData schema)
-  (legacy SwiftData schema)
-  (legacy SwiftData schema)
-  (legacy SwiftData model - still referenced in some places)
-  (legacy SwiftData model)
-  (marked as legacy reference)

**Proposed solution**:
1. Remove DeckSchemaV1, GlossarySchemaV1, LearningDaySchemaV1 files
2. Remove or archive legacy Deck, Glossary models if no longer used
3. Remove commented-out SwiftData code in AppMain.swift
4. Verify no @Query or ModelContext references remain in production code

**Complexity**: S
```

ファイルパスが空。ダッシュの後に括弧書きの説明だけで、実際のパスが出力されていない。これは他のIssueでも同じだった。20分かけて12.53クレジット使って、この出力はつらい。

問題の認識自体は間違っていないけど、「どのファイルを直すのか」がわからないIssueは実質的には使いづらい。

### Qwen3 Coder Next — バランスが良かった

```markdown
## Problem
The iOS app has commented-out SwiftData model container initialization in
`AppMain.swift` (lines 27-37), but legacy SwiftData models (`Deck.swift`,
`Glossary.swift`, `LearningDay.swift`) still exist and are referenced in multiple
places. This creates confusion about the data architecture and potential bugs.

**Evidence:**
- `AppMain.swift` lines 27-37: ModelContainer initialization commented out
- `DeckSelectionView.swift`: Still has `convertToSwiftDataDeck()` and
  `convertToSwiftDataGlossary()` methods
- `GlossaryCardStackViewModel.swift`: Imports SwiftData but all operations
  commented out
- `LearningTopView.swift`, `StudyDeckView.swift`: Import SwiftData but use
  Firestore

**Why it matters:**
- Code confusion: Developers may think SwiftData is still in use
- Dead code maintenance burden
- Potential bugs if SwiftData code is accidentally uncommented
- Inconsistent data access patterns

## Affected files
- `/Users/xxx/Projects/.../Sources/AppMain.swift` -
  ModelContainer disabled (lines 27-37)
- `/Users/xxx/Projects/.../Sources/Model/Deck.swift` -
  Legacy SwiftData model
- `/Users/xxx/Projects/.../Sources/Model/Glossary.swift` -
  Legacy SwiftData model
- `/Users/xxx/Projects/.../Sources/Model/LearningDay.swift` -
  Legacy SwiftData model
- `/Users/xxx/Projects/.../Sources/Screen/Learn/DeckSelectionView.swift` -
  Conversion methods (lines 50-100)
- `/Users/xxx/Projects/.../Sources/Screen/Learn/Stack/GlossaryCardStackViewModel.swift`
  - SwiftData import, commented operations
- `/Users/xxx/Projects/.../Sources/Screen/Learn/LearningTopView.swift` -
  SwiftData import
- `/Users/xxx/Projects/.../Sources/Screen/Learn/StudyDeckView.swift` -
  SwiftData import

## Proposed solution
1. **Remove legacy SwiftData models** (`Deck.swift`, `Glossary.swift`,
   `LearningDay.swift`)
2. **Remove conversion methods** from `DeckSelectionView.swift`
3. **Remove SwiftData imports** from views that no longer use them
4. **Update documentation** to clarify Firestore is the sole persistence layer
5. **Archive legacy models** in a separate `Legacy/` folder if needed for
   reference

## Complexity estimate
**S** - Requires file deletions and import removals, no data migration needed
```

8ファイルを特定して、行番号付き。Viewに残っている変換メソッドやデッド`import`まで拾っている。`Haiku`・`MiniMax`が見逃したレイヤーまで踏み込んでいる。

ただしスキーマファイル（`DeckSchemaV1.swift`など）は見つけていない。`Haiku`が見つけて`Qwen3`が見逃すという逆転が見られた。完璧ではないけど、4.05クレジットでこの粒度は十分だと思った。

もう一つ気になったのはファイルパスが絶対パスになっていること。GitHub Issueとしてはリポジトリルートからの相対パスの方が読みやすい。前回の記事に載せたIssue例もこのQwen3の出力なので、同じ問題が出ています。プロンプトで「相対パスで書くこと」と指定すれば直せそう。

### Auto — 一番深い

```markdown
## Problem
The migration docs claim SwiftData is 98% removed, but significant dead code
remains compiled into the binary:

1. **Schema files still exist**: `DeckSchemaV1.swift`, `GlossarySchemaV1.swift`,
   `LearningDaySchemaV1.swift` — these define `@Model` classes that SwiftData
   processes at compile time
2. **AppMain.swift** still defines `schema` and `config` as static
   properties (lines 22-24), even though the ModelContainer init is commented out
3. **`import SwiftData`** appears in multiple files: `ContentView.swift`,
   `LearningTopView.swift`, `GlossaryCardStackView.swift`,
   `GlossaryCardStackViewModel.swift`
4. **Legacy `Deck`/`Glossary` typealiases** are still used in:
   `GlossarySideHiddenView`, `BookmarkView` (dead conversion methods),
   `DeckListView` (dead conversion methods), `GlossaryListView`
   (`exportedItem: [Glossary]`)
5. **`DeckPreviewContainer.swift`** still exists with SwiftData preview code

This dead code increases binary size, compile time, and cognitive load.

## Affected files
- `Sources/Model/DeckSchemaV1.swift` — Entire file (dead)
- `Sources/Model/GlossarySchemaV1.swift` — Entire file (dead)
- `Sources/Model/LearningDaySchemaV1.swift` — Entire file (dead)
- `Sources/Model/Deck.swift` — Entire file (dead typealias + extensions)
- `Sources/Model/Glossary.swift` — Entire file (dead typealias + extensions)
- `Sources/AppMain.swift` — Lines 22-24: Dead schema/config definitions
- `Sources/Model/DeckPreviewCotainer.swift` — Entire file (dead, also has typo
  in filename)
- `Sources/Screen/Deck/Read/DeckListView.swift` — Lines 268-310: Dead
  `convertToSwiftData*` methods
- `Sources/Screen/Bookmark/BookmarkView.swift` — Lines 120-140: Dead
  `convertToSwiftDataGlossary` method

## Proposed solution
1. Delete the three schema files
2. Delete `Deck.swift`, `Glossary.swift`, `DeckPreviewCotainer.swift`
3. Remove dead static properties from `AppMain.swift`
4. Remove all `import SwiftData` statements
5. Remove dead `convertToSwiftData*` methods from views
6. Update `GlossaryListView.exportedItem` to use `[LMGlossary]`
7. Fix CSV export to work with `LMGlossary` instead of `Glossary`

## Complexity: M
```

Autoだけが見つけたと思われるものがいくつかあります。

- `DeckPreviewCotainer.swift`（ファイル名のtypoまで指摘している）
- `BookmarkView`のデッド変換メソッド（行番号付き）
- 削除後に必要な型の差し替え（`[Glossary]` → `[LMGlossary]`）
- CSV export機能への影響

他の3モデルが「削除すればOK」で終わっているのに対して、Autoは「削除した後に何が壊れるか」まで考えています。ComplexityをMにしているのもそのため。単純な削除ではなく、下流の型変更が必要だと判断しています。

9.91クレジットで9分18秒。コストは`Qwen3`の約2.5倍だけど、この深さの差は大きい。

### ドキュメントの「98%完了」をどう扱ったか

もう一つ面白かったのが、移行ドキュメントに書いてある「SwiftData移行は98%完了」という記述への反応の違いです。

`Haiku`と`MiniMax`はこの98%をそのまま受け入れていました。「ドキュメントによると98%完了、残りのファイルを消せばOK」という姿勢。`Qwen3`は98%という数字に言及すらしていない。ドキュメントを読んだのか読んでいないのかわからない。

一方`Auto`は「The migration docs claim SwiftData is 98% removed, but significant dead code remains compiled into the binary」と書いていました。claimという単語を使っている。ドキュメントの主張を鵜呑みにせず、実際のコードと突き合わせて「本当に98%か？」を検証している。この姿勢の差が、分析の深さにそのまま出ていました。


## 項目別に並べてみる

同じIssueに対する4モデルの出力を項目別に整理するとこうなります。

| 観点 | Haiku | MiniMax | Qwen3 | Auto |
|---|---|---|---|---|
| ファイルパス | ✓ | ✗（空欄） | ✓（絶対パス） | ✓（相対パス） |
| 行番号 | 一部あり | ✗ | ✓ | ✓ |
| スキーマファイル検出 | ✓ | ✓ | ✗ | ✓ |
| View層のデッドコード | ✗ | ✗ | ✓ | ✓ |
| ドキュメント検証 | 鵜呑み | 鵜呑み | 言及なし | 疑って検証 |
| 削除後の影響分析 | ✗ | ✗ | 一部 | ✓ |
| Markdownフォーマット | ✗ | ✗ | ✓ | ✓ |
| Complexity判定 | S | S | S | M |

面白いのは、どのモデルも何かしら見つけているけど、見つけるレイヤーが違うこと。`Haiku`はスキーマファイルを見つけたけどView層は見ていません。`Qwen3`はView層まで掘ったけどスキーマファイルを見逃した。`Auto`だけが両方カバーしていました。

Markdownフォーマットの差は実用上、影響が大きかった。`Qwen3`と`Auto`はバッククォートでコード参照を書いていて、GitHubで見ると読みやすい。`Haiku`と`MiniMax`はプレーンテキストで、ファイル名とコメントの区別がつきづらかった。

## コスト効率を考える

単純なコスト順に並べるとこうなります。

| モデル | クレジット | コスト倍率 | 実行時間 |
|---|---|---|---|
| Haiku 4.5 | 2.34 | 0.4x | 3m 33s |
| Qwen3 Coder Next | 4.05 | 0.05x | 4m 51s |
| Auto | 9.91 | 1.0x | 9m 18s |
| MiniMax M2.5 | 12.53 | 0.25x | 20m 8s |

見ていて気になったのが、コスト倍率と実際のクレジット消費が一致していないこと。`Qwen3`は倍率0.05xなのに、0.4xの`Haiku`よりクレジットを使っています。たぶん`Qwen3`の方がたくさんファイルを読んで、出力も長かったからだと思う。

つまり「倍率が安い = 総コストが安い」とは限らない。モデルの行動パターン（どれだけ読むか、どれだけ書くか）も総コストに影響します。ただ、品質対コストで見ると`Qwen3`がコスパがいい。`MiniMax`の1/3のコストで、はるかに具体的な出力を出していました。

## MiniMaxはなぜダメだったのか

正直、MiniMax M2.5には期待していました。ネット上でコーディング性能を評価する声を見かけていたし、コスト倍率0.25xなら試す価値はあると思った。結果は一番高く、一番遅く、品質が最も低かった。

具体的に何がダメだったか：

- ファイルパスが出力されなかった（全Issueで空欄）
- 行番号の指定がなかった
- 提案が抽象的で、そのまま作業に移れない
- Markdownフォーマットを使わなかった
- 20分もかかった（Autoの2倍以上）

おそらくこのワークフロー（コードベースを読んで構造化された出力を生成する）との相性が悪かったのだと思います。コーディング能力が高くても、分析→構造化出力というタスクが得意とは限りません。

ただ、これはあくまで今回のプロンプトでの結果です。プロンプト自体にまだ改善の余地があります。たとえば出力フォーマットをもっと厳密に指定したり、ファイルパスの書き方を明示したりすれば、結果が変わる可能性はあります。

そもそも自分はMiniMaxについてあまり詳しくなく、ドキュメントやベストプラクティスをちゃんと読んだわけでもありません。モデルごとに得意なプロンプトの書き方や、性能を引き出すコツがあるはずで、それを把握せずに「ダメだった」と断じるのはフェアではないかもしれません。これはMiniMaxに限らず、他のモデルについても同じことが言えます。今回の比較はあくまで「同じプロンプトをそのまま投げた場合」の結果であって、各モデルのポテンシャルを測ったものではありません。

## Markdown対応の重要性

地味だけど実用上かなり効いたのがMarkdownフォーマットの差です。GitHub Issueとして使う以上、コードブロック、インラインコード、見出しが正しく使われているかどうかで可読性が大きく変わります。

`Qwen3`と`Auto`が生成したIssueはGitHub上でそのまま読めました。ファイル名がバッククォートで囲まれていて、提案がリスト形式で整理されている。`Haiku`と`MiniMax`はプレーンテキストだったので、GitHubで見るとコードと地の文が混ざって読みづらかった。

これはプロンプトで「Markdownで書くこと」と明示すれば直せそうです。今回はプロンプトにMarkdown指定を入れていなかったので、モデルの「デフォルトの出力スタイル」がそのまま出た形です。次回検証してみたい。

## 自分ならこう使い分ける

LLMに詳しいわけではないので、あくまで今回試した範囲での感覚です。

| ユースケース | 自分が選ぶなら | 理由 |
|---|---|---|
| 一回きりの深い分析（放置プロジェクト再開など） | Auto | 今回一番深かった。下流の影響まで見てくれた |
| 定期的なバッチ分析（リファクタリングバックログ更新） | Qwen3 Coder Next | コストの割に具体的な出力が出た |
| ざっくり確認 | Haiku 4.5 | 速くて安い。深さを求めない場面なら |
| MiniMax M2.5 | 今回は選ばない | 少なくともこのワークフローでは合わなかった |

夜中に自動で回すなら`Qwen3`かなと思っています。`Auto`の1/2.5のコストで、自分の用途には十分な品質だった。

**追記（2026/04）：** [Kiro CLIのheadless mode](https://kiro.dev/blog/introducing-headless-mode/)がリリースされ、この「夜中に自動で回す」シナリオが実現可能になりました。`KIRO_API_KEY`環境変数 + `--no-interactive`フラグで、cronやGitHub Actionsから非対話的に実行できます。

放置プロジェクトを久しぶりに開いて「まず全体像を把握したい」という場面では`Auto`を使いたいです。ファイル名のtypo検出や削除後の型変更まで拾ってくれたのは、他のモデルにはなかった。

ただ、プロンプトの書き方やモデルごとの癖をもっと理解すれば、この判断は変わるかもしれない。今回は「同じプロンプトをそのまま投げた」だけなので。

## まとめ

コスト倍率と品質は比例しない。これが一番の学びでした。

0.05xの`Qwen3`が0.25xの`MiniMax`より良い結果を出した。高いモデルが必ずしも良いわけではなく、タスクとの相性が大きい。

もう一つ、「指示への忠実度」がモデル選びの隠れた基準だと感じました。ファイルパスを出力する、Markdownで書く、行番号を付ける。こういった出力フォーマットの遵守は、分析の深さとは別の軸で実用性に直結します。

定期実行を前提にするなら、まず安いモデルで試して、出力品質が足りなければ上げる。逆に高いモデルから始める必要はありません。

ただし今回の比較にはいくつか制約があります。1回ずつの実行なので結果がブレる可能性があること、Experimentalモデル（`Qwen3`、`MiniMax`）は今後のアップデートで品質が変わりうること、プロンプトにMarkdown指定を入れていなかったためフォーマットの差はモデルのデフォルト挙動を反映していること。

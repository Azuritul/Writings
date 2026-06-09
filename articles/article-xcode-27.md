---
title: "WWDC 2026で発表されたXcode 27の新機能メモ"
emoji: "🛠️"
type: "tech"
topics: [Xcode, WWDC2026, Swift, iOS, Apple]
published: true
---

WWDC26のセッション「[What’s new in Xcode 27](https://developer.apple.com/videos/play/wwdc2026/258/)」を見ました。

この記事では、セッションで紹介された内容をすべて網羅するのではなく、自分が気になったアップデートを中心にメモしています。

## 変わったところ

全体像はこんな感じです。

| 変更点 | ざっくり何が変わったか |
|---|---|
| Workspace & Toolbar | ツールバーが再設計され、必要な操作を並べ替えられる |
| Themes | エディタの見た目を細かく調整でき、プロジェクトごとにテーマを割り当てられる |
| Inline Issues | 入力中の警告表示が控えめになり、ビルド後に本格的な警告・エラーとして表示される |
| New Project Workflows | Untitled Projectや単体Swiftファイルで、すぐ試せるプロトタイピングがしやすくなった |
| Coding Agents | エディタ内でエージェント会話を扱え、`/plan`で実装前に作業計画を作れる |
| Device Hub | シミュレータや実機での確認、アクセシビリティ設定、iPhone Mirroringのサイズ確認をまとめて扱える |
| Localization | エージェントがString Catalogの作成や翻訳生成を支援する |
| Organizer | 影響の大きい問題を見つけやすくし、App StorageやAnimation Hitchesなどのメトリクスも確認できる |
| Instruments | Top Functionsで重いコードパスを素早く見つけられる |
| Xcode Cloud | 初期設定が簡単になり、コミットごとのビルド・テスト・配信までつなげやすくなった |

## ワークスペースとカスタムテーマの刷新

* **ツールバーの再設計:** これまでジャンプバーにあった履歴ナビゲーションやエディタ操作系が、画面上部のツールバーに整理されました。ツールバーは並び替えや追加・削除ができるので、自分の作業に合わせて変えられます。

![ツールバーの再設計](/images/article-xcode-27/toolbar.webp)

* **テーマ機能が進化した:** 新しい外観（Appearance）パネルで、テキスト色や背景の濃淡をスライダーで調整できます。ワークスペースごとにテーマを設定できるので、似たプロジェクトを並べて開くときの見分けがつきやすくなります。

![テーマの設定](/images/article-xcode-27/theme.webp)

![ワークスペースごとのテーマ](/images/article-xcode-27/theme2.webp)

## 軽量なプロトタイプ用ワークフロー

* **Untitled（未命名）プロジェクト:** 新しいアイデアを試すときに、先にプロジェクト名や保存先を決めなくても始められます。
* **スタンドアロンのSwiftファイル:** プロジェクトを作らなくても、単一のSwiftファイルをXcodeで開いて、プレビューやPlaygroundの実行結果を確認できます。

## エディタ内でのコーディングエージェント（AI）連携

* **画面分割・タブ対応:** エージェントとの会話をエディタペインの一部として扱えるようになりました。コードと並べたり、タブとして開いたりできます。
* **計画コマンド（`/plan`）:** 他のagentic toolと同じように、plan modeが導入されました。すぐにコードを書き換えさせるのではなく、まず `/plan` で作業範囲や方針を整理できます。

![planコマンド](/images/article-xcode-27/plan.webp)

* **成果物の可視化:** エージェントが変更したコードの差分や、生成したファイルなどを確認しながら進められます。

## Device Hubによる実機・シミュレータ検証

* **統合された確認画面:** Simulatorが刷新され、Device Hubとして利用できるようになりました。Device Hubでは、シミュレータや実機をまとめて扱えます。アプリの実行、状態確認、Dynamic Typeやダークモードなどのアクセシビリティ関連設定の切り替えを一か所でできるようになります。

![Device Hub](/images/article-xcode-27/device-hub.webp)

* **iPhoneミラーリングへの対応:** iPhone Mirroringのリサイズ確認にも対応しています。画面サイズや表示条件を変えたときの表示を確認できます。

![iPhone Mirroringのリサイズ確認](/images/article-xcode-27/mirroring.webp)

* **一元管理:** 有線・無線でつながった実機とシミュレータを、同じ場所から確認できます。

## エージェントを活用したアプリのローカライズ（多言語化）

* Coding Agentが、String Catalogの作成や翻訳生成を支援します。

![String Catalogの作成](/images/article-xcode-27/localization.webp)

![翻訳生成](/images/article-xcode-27/localization2.webp)

## Organizer（オーガナイザー）のアップデート

* **デザインの刷新:** Organizerは再設計され、影響の大きい問題を優先して見られるようになりました。メトリクスと診断情報を同じ流れで確認できます。
* **新メトリクスの追加:** App StorageやAnimation Hitchesなど、リリース後に確認できる指標が追加されています。容量やアニメーションのカクつきも追いやすくなります。

![Organizerのアップデート](/images/article-xcode-27/organizer.webp)

* **Metric Goals:** Metric Goalsを使うと、アプリの品質目標を設定して追跡できます。
* **エージェントによる修正提案:** Organizer上の問題に対して、エージェントによる修正案生成もできます。

![Organizerの修正提案](/images/article-xcode-27/organizer2.webp)

## Instrumentsと「Top Functions」

* パフォーマンス分析ツールのInstrumentsに、Top Functionsビューが追加されました。負荷の高い関数や処理時間の長いコードパスを確認できます。

## Xcode Cloudの導入プロセスの簡略化

* Xcode Cloudの初期セットアップが簡単になりました。コミットごとのビルドやテスト、TestFlightやApp Storeへの配信までつなげやすくなっています。

![Xcode Cloudの初期セットアップ](/images/article-xcode-27/xcode-cloud.webp)

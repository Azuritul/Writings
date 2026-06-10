---
title: "iOS Foundation Modelsの基本を試してみた"
emoji: "🍎"
type: "tech"
topics: [FoundationModels, Swift, AI, AppleIntelligence, iOS]
published: true
---

WWDC 2026のキーノートがもう目前です。

このタイミングでWWDC 2025の話を書くのは、正直かなり今さら感があります。

ただ、今年のWWDCではApple Intelligenceまわりにも何か進展があるのでは、と予想半分・個人的な希望半分で見ています。去年の発表内容をちゃんと試さないまま今年の発表を見るのも微妙なので、まずは`Foundation Models`フレームワークを触ってみました。

この記事は、WWDC25の[Meet the Foundation Models framework](https://developer.apple.com/videos/play/wwdc2025/286/)をベースにしています。

実際に触ってみた範囲で、Foundation Modelsの基本的な使い方と、iOSアプリに組み込むときに気になった点をまとめます。

## Foundation Modelsとは

`Foundation Models`は、Apple Intelligenceを支えるオンデバイスLLMにアクセスするためのフレームワークです。

ざっくり言うと、Apple Intelligenceの中核にあるオンデバイスの言語モデルを、Swiftから直接呼び出せるようにするものです。クラウドのLLM APIにリクエストを投げるのではなく、ユーザーの端末上で生成・要約・分類・構造化出力を実行できます。

Appleの公式ドキュメントでは、主な用途として次のようなタスクが挙げられています。

- 要約
- エンティティ抽出
- テキスト理解
- 文章の改善
- ゲーム内の短い会話生成
- クリエイティブなテキスト生成
- など

## ベースにしたWWDCセッション

この記事は、[Meet the Foundation Models framework](https://developer.apple.com/videos/play/wwdc2025/286/)をベースにしています。Foundation Modelsの全体像を掴むには、このセッションがちょうどよかったです。

Foundation Models関連では、他にも次のセッションがあります。こちらはもう少し実装寄りです。この3本はまだ見ていないので、別途見る予定です。

| 動画 | 見るポイント |
|---|---|
| [Code-along: Bring on-device AI to your app using the Foundation Models framework](https://developer.apple.com/videos/play/wwdc2025/259/) | SwiftUIアプリに組み込む流れ、prompt engineering、tool calling、streaming、profiling |
| [Explore prompt design & safety for on-device foundation models](https://developer.apple.com/videos/play/wwdc2025/248/) | オンデバイスLLM向けのプロンプト設計、安全性、評価とテスト |
| [Deep dive into the Foundation Models framework](https://developer.apple.com/videos/play/wwdc2025/301/) | session、`@Generable`、dynamic schema、tool callingの詳細 |

## Xcodeで実装前に試す

Foundation Modelsを使う場合、期待した出力が得られるように、プロンプトを試しながら調整する必要があります。`#Playground` macroを使うと、アプリを毎回実行しなくても、オンデバイスモデルにプロンプトを投げて結果を確認できます。たとえば「東京を観光するなら、どこがおすすめですか？」と入力すると、右側のキャンバスにモデルの出力が表示されるため、プロンプトの挙動を手軽に試せます。

アプリ内で定義している型にもアクセスできます。

![Previewでresponseを確認する画面](/images/article-apple-foundation-models/canvas.png)

## 基本的な使い方

まずは、実際に呼び出す前に使える状態か確認します。そのあとで`LanguageModelSession`を作り、必要に応じてガイド付き生成を使います。

### 可用性をチェック

Foundation Modelsを使う前に、まず現在の環境で利用可能かどうかを確認します。この機能はApple Intelligenceに依存しているため、利用できるかどうかは環境によって異なります。たとえば、次のようなケースがあります。

- デバイスがApple Intelligenceに対応していない
- Apple Intelligenceが設定で無効になっている
- モデルがまだダウンロード中、または一時的に準備できていない
- 言語やロケールが対応していない

`SystemLanguageModel.default.availability`を見ると、現在の環境でモデルを使えるかどうかを判定できます。

```swift
import FoundationModels
import SwiftUI

struct AvailabilityCheckView: View {
    private let model = SystemLanguageModel.default

    var body: some View {
        switch model.availability {
        case .available:
            AvailableView()
        case .unavailable(.deviceNotEligible):
            CorruptView()
        case .unavailable(.appleIntelligenceNotEnabled):
            CorruptView()
        case .unavailable(.modelNotReady):
            CorruptView()
        case .unavailable(_):
            CorruptView()
        }
    }
}
```

### LanguageModelSessionで呼び出す

基本は`LanguageModelSession`を作って、`respond(to:)`でプロンプトを渡します。

```swift
import FoundationModels
import Playgrounds
#Playground {
    let session = LanguageModelSession()
    let response = try await session.respond(to: "東京を観光するなら、どこがおすすめですか？")
    print(response.content)
}
```

### ガイド付き生成

前のスクリーンショットを見るとわかりますが、`respond(to:)`のレスポンスはプレーンテキストで返ってきます。そのままアプリ内で再利用するには少し扱いづらいです。

そこでFoundation Modelsのガイド付き生成を使うと、この部分をSwiftの型定義に寄せて扱えます。

`@Generable`を使うと、モデルの出力をSwiftの型として受け取れます。たとえば東京の観光スポットの候補を作るなら、こういう型を定義できます。

```swift
import FoundationModels

@Generable
struct SpotSuggestions {
    @Guide(description: "東京旅行のおすすめスポット", .count(5))
    var locations: [String]
}
```

呼び出し側は、`String`をJSONとしてパースする必要がありません。

```swift
#Playground {
    let prompt = "東京を観光するなら、どこがおすすめですか？"
    let session = LanguageModelSession()
    let response = try? await session.respond(to: prompt, generating: SpotSuggestions.self)
    let suggestion = response?.content
}
```

`@Generable`の型では、プリミティブ型だけでなく、配列や他の`Generable`な型も使えます。たとえば、次のような形にもできます。

```swift
@Generable
struct SpotSuggestions {
    @Guide(description: "東京旅行のおすすめスポット", .count(3))
    var locations: [Spot]
}

@Generable
struct Spot {
    @Guide(description: "おすすめスポット名")
    var name: String
    @Guide(description: "おすすめスポットの簡単な説明")
    var description: String
    @Guide(description: "そのスポットをおすすめする理由")
    var reason: String
}
```

![複合型](/images/article-apple-foundation-models/generable.png)

ガイド付き生成を使うと、出力の構造をある程度保証できます。プロンプト側でJSON形式を細かく指示しなくても、Swiftの型で期待する形を表現できます。セッションでは、構造を指定することで推論速度や正確さにも良い影響があると説明されていました。


### `streamResponse`で途中状態を見る

`respond(to:)`は、生成が終わってから結果を返します。短い出力ならそれで十分ですが、レスポンスが少し長くなったり、生成に時間がかかったりする場合は、途中の状態も見たくなります。

その場合は、`session.streamResponse`を使います。

`streamResponse`では、完成した最終結果ではなく、生成途中のスナップショットを受け取れます。状態に応じてUIを少しずつ更新できるので、最終結果を待つより自然に見せられます。token単位の差分を自分でつなげるというより、現時点でモデルが生成できている内容をsnapshotとして見るイメージです。

たとえば:

```swift
var session: LanguageModelSession = LanguageModelSession()
private let prompt = "東京を観光するなら、どこがおすすめですか？"
private(set) var streamingLines: [String] = []

func suggestSpotStream() async {
    // start streaming; handle errors from caller or log locally
    let stream = session.streamResponse(to: prompt, generating: SpotSuggestions.self)
    do {
        for try await snapshot in stream {
            let description = String(describing: snapshot.content)
            streamingLines.append(description)
        }    
    } catch {
       print("Error:", error)        
    }
}
```

### Tool callingは別記事で試す

Foundation Modelsには、ツール呼び出し（Tool calling）という仕組みもあります。

ツール呼び出しは、モデルが必要に応じてアプリ内に定義した処理を呼び出せる機能です。たとえば、モデルにすべての情報をプロンプトで渡すのではなく、必要になったタイミングでアプリ側の`Tool`を呼び出し、ローカルDBやアプリ内データから情報を取得できます。

ポイントは、モデルに「何でも直接やらせる」のではなく、アプリ側で使ってよい処理を`Tool`として定義しておくことです。モデルは状況に応じてどの`Tool`を使うかを判断し、Foundation Modelsフレームワークがアプリ側に定義した処理を呼び出します。

`Tool`の結果はセッションのtranscriptに追加され、その結果も含めてモデルが最終的な応答を生成します。プロンプトだけでは扱いづらいアプリ固有のデータや、MapKitのような信頼できる情報源を使いやすくなります。ただし、`Tool`の設計や引数の定義、どこまでモデルに任せるかの判断が必要になります。

この記事では深掘りせず、具体例は別記事で扱います。

## 参考

### WWDCセッション

- [Meet the Foundation Models framework - WWDC25 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2025/286/)
- [Code-along: Bring on-device AI to your app using the Foundation Models framework - WWDC25 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2025/259/)
- [Explore prompt design & safety for on-device foundation models - WWDC25 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2025/248/)
- [Deep dive into the Foundation Models framework - WWDC25 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2025/301/)

### APIドキュメント

- [Foundation Models | Apple Developer Documentation](https://developer.apple.com/documentation/FoundationModels/)
- [SystemLanguageModel | Apple Developer Documentation](https://developer.apple.com/documentation/foundationmodels/systemlanguagemodel)
- [LanguageModelSession | Apple Developer Documentation](https://developer.apple.com/documentation/foundationmodels/languagemodelsession)

### チュートリアル

- [Generating Swift data structures with guided generation | Apple Developer Documentation](https://developer.apple.com/documentation/FoundationModels/generating-swift-data-structures-with-guided-generation)
- [Expanding generation with tool calling | Apple Developer Documentation](https://developer.apple.com/documentation/foundationmodels/expanding-generation-with-tool-calling)
- [Generating content and performing tasks with Foundation Models | Apple Developer Documentation](https://developer.apple.com/documentation/FoundationModels/generating-content-and-performing-tasks-with-foundation-models)

### 関連記事

- [Foundation Models updates | Apple Developer Documentation](https://developer.apple.com/documentation/Updates/FoundationModels)
- [Updates to Apple’s On-Device and Server Foundation Language Models | Apple Machine Learning Research](https://machinelearning.apple.com/research/apple-foundation-models-2025-updates)
- [Apple’s Foundation Models framework unlocks new app experiences powered by Apple Intelligence | Apple Newsroom](https://www.apple.com/uk/newsroom/2025/09/apples-foundation-models-framework-unlocks-new-intelligent-app-experiences/)

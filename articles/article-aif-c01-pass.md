---
title: "AWS Certified AI Practitioner（AIF-C01）合格体験記"
emoji: "🎓"
type: "idea"
topics: [AWS, 資格, 合格体験記, AIプラクティショナー]
published: false
---

AWS Certified AI Practitioner（AIF-C01）に合格しました。

![AIF-C01試験結果（合格）](/images/article-aif-c01-pass/result_masked.png)

AWSについては、以前にCloud Practitioner（CLF-C02）を取得したものの、実務経験はありません。開発ではKiroやCodexを普段から使っていますが、機械学習を体系的に学んだことはありませんでした。

受験した理由はシンプルで、AIに興味があったからです。普段から生成AIを開発ツールとして使ってはいるものの、知識は断片的なまま。資格試験を使って、今の知識がどの程度なのか確かめてみたかった。AIや機械学習の基礎を一度整理したいという気持ちもありました。

## 学習期間と教材

学習期間は全体で約1か月。講座、問題集、ネット記事を合わせて30〜40時間ほどでした。もちろん、1か月間ずっと勉強していたわけではありません。隙間時間に少しずつ進めました。

主にUdemyの英語講座と問題集を使い、AWS Skill Builderの公式問題集も解きました。

- 講座：[Ultimate AWS Certified AI Practitioner AIF-C01](https://www.udemy.com/course/aws-ai-practitioner-certified/)
- Udemy問題集：[[Practice Exams] AWS Certified AI Practitioner - AIF-C01](https://www.udemy.com/course/practice-exams-aws-certified-ai-practitioner/)
- Skill Builder公式問題集：[AWS Certification Official Practice Question Set](https://skillbuilder.aws/learn/4URFGY63KV/official-practice-question-set-aws-certified-ai-practitioner--aifc01--english/FVG43Y1PAX)

講座は約10時間。AI・機械学習の基礎から、Prompt Engineering、Amazon Bedrock、Amazon Q、SageMaker、AWSの入門マネージドAIサービスまでを一通り扱います。
Udemyの問題集には、本番と同じ65問構成の模擬試験が4回分収録されていました。

AWS Skill Builderの公式問題集は20問だけですが、問題のスタイルが本番に近く、役に立ちました。

## 勉強の進め方

1. 最初の2週間ほどで講座を見る
2. 4回分の模擬試験を、それぞれ1回ずつ解く
3. 間違えた問題と、自信がなくマークした問題を復習する
4. AWS Skill Builderの公式問題集を解く

4回分の模擬試験は、まず1回ずつ。試験前日は、そのうち1回分だけ解き直しました。答え合わせの後は、間違えた問題と、自信がなくてマークした問題を確認。必要に応じて講座の該当箇所へ戻りました。模擬試験は答えを覚えるためというより、理解が曖昧なところを見つけるために使いました。

講座を見ても理解しきれなかったところは、AWSの公式記事で補いました。Embeddingsの仕組みでは、[AWSの解説記事](https://aws.amazon.com/what-is/embeddings-in-machine-learning/)が参考になった。

## 教材で気になった点

Udemyの講座と問題集は、試験対策には役立ちました。ただ、AWS側の変化が早く、内容が古くなっている箇所もあります。

たとえば、「Amazon Bedrock - RAG & Knowledge Base」のハンズオンでは、動画内のUIが現在のAWSコンソールと異なり、そのままでは操作できませんでした。2026年7月28日時点では、教材の更新が追いついていませんでした。

### Amazon Q Developer

講座と問題集にはAmazon Q Developerが登場します。ただ、現在はAmazon Q Developer CLIがKiro CLIへリブランドされ、開発ツールとしての機能はKiroへ移行しています。

IDEプラグインと有料サブスクリプションも、2027年4月30日に[サポート終了](https://aws.amazon.com/jp/blogs/devops/amazon-q-developer-end-of-support-announcement/)予定です。ただし、AWS管理コンソールなどで提供されるAmazon Q Developerは移行の対象外です。Amazon Q Developer全体がなくなるわけではありません。

試験対策では教材に出てくる名称や役割を押さえつつ、今の状況は[AWSの移行案内](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/upgrade-to-kiro.html)で確認するのがよさそうです。

### AWS DeepRacer

AWS DeepRacerは講座には含まれていませんでしたが、問題集に登場しました。

DeepRacerは2025年12月15日をもって、AWSコンソール上のマネージドサービスとしての提供を終了しています。現在は、利用者が自分のAWSアカウントへ展開するオープンソースの「DeepRacer on AWS」として提供されています。

強化学習の事例として理解する分には問題ありません。ただ、現在も以前と同じサービスとして使えるわけではありません。現状は[AWS DeepRacerのFAQ](https://aws.amazon.com/deepracer/faqs/)で確認できます。

AI周辺の変化は早いので、教材の更新が追いつかないのは仕方ないと思います。

自分のようにAWSを専門としていないと、教材に出てくるサービスが今も現役なのか、すでに提供終了や移行の対象なのか分かりにくい。Q DeveloperやDeepRacerがまさにそうでした。学習を始める前に、公式情報で現状だけ軽く確認しておくのがよさそうです。

### Udemy問題集で対応していない問題形式

[公式の試験ガイド](https://docs.aws.amazon.com/ja_jp/aws-certification/latest/ai-practitioner-01/ai-practitioner-01.html)には、択一選択、複数選択、並べ替え、内容一致の4つの出題形式が記載されています。

2026年7月28日時点では、Udemyの問題集に並べ替えと内容一致の問題は見当たりませんでした。AWS Skill Builderの公式問題集では、この2形式も試せます。20問しかありませんが、問題形式を確認する意味でも役立ちました。

## 参考

- [AWS Certified AI Practitioner（AIF-C01）試験ガイド](https://docs.aws.amazon.com/ja_jp/aws-certification/latest/ai-practitioner-01/ai-practitioner-01.html)

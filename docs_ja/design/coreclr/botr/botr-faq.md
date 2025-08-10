<!--
source: docs/design/coreclr/botr/botr-faq.md
source_commit: (unknown)
status: WIP
translator: TBD
updated: 2025-08-10
notes: 初回翻訳。原文の意図と構成を維持。
-->

# ランタイムの書（Book of the Runtime; BotR）FAQ（Book of the Runtime (BotR) FAQ）

## BotR とは？（What is the BotR?）

[Book of the Runtime](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/README.md) は、CLR と BCL の各コンポーネントを解説する文書群です。コードの逐語的な注釈ではなく、アーキテクチャや不変条件（invariants）に焦点を当てます。

BotR は 2007 年頃に Microsoft 内部で作成され（本ドキュメントも含む）、各開発者が自分の担当機能領域を文書化していました。新規メンバーのオンボーディングや、製品アーキテクチャの共有に役立ちました。

CoreCLR が GitHub で OSS 化された現在、BotR の価値はさらに高まりました。新しい CLR 開発者の助けとなるよう、章を公開しています。

各 BotR 文書は、執筆者と執筆時期という[一定の視点](https://github.com/dotnet/coreclr/pull/115)を持って書かれています。2015 年時点に合わせて文書を改変するのは適切でないと判断し、スペル修正や Markdown 化以外は本来の文書を保っています。改善のための PR は歓迎します。

## 主な読者は？（Who is the main audience of BotR?）

- バグ対応で特定領域の高レベル概要を把握したい開発者
- 依存関係のある新機能を実装するにあたり、既存コンポーネントとの正しい相互作用に必要な知識を得たい開発者
- 特定コンポーネントの保守を担当する新規開発者

## BotR 章に何を書くべき？（What should be in a BotR chapter?）

BotR の目的は、仕様書とソースコードだけでは再構成が難しい情報を記録し、チーム内の高レベルなコミュニケーションを可能にすることです。概念やトップダウンの説明、そして何よりも、なぜその設計判断をしたのか（判断の根拠）を説明します。

## 設計ドキュメントとの違いは？（How is this different from a design doc?）

設計ドキュメントは実装前に書くものです。一方、BotR の章は通常、機能実装後に書かれます。その時点では、設計オプションの長短や採用案（将来の改善計画を含む）について判断済みで、実装・テストを通じてでなければ見えてこない詳細も把握できています。そのため、設計判断の理由をより適切に語ることができます。

## 新人で機能に不慣れ。どう貢献できる？（I am a new dev… how can I contribute?）

BotR は新人支援が重要な目的のひとつなので、新人こそ貢献できます。

- レビュアーになる。分かりにくい点があれば筆者に連絡し、改善を検討する。
- 自分の担当領域の BotR 章を読み、誤りや更新点があれば自ら修正する。
- 章（あるいは一部）の執筆を申し出る。まずは学んだことをメモとして蓄積し、徐々に章の形に整えるとよい。

## BotR レビュアーの責務は？（What are the responsibilities of a BotR reviewer?）

建設的なコメントを提供してください。技術的深さ、文体、カバー範囲など、どの観点でも構いません。BotR は主に設計・アーキテクチャの要点を扱い、実装詳細のチュートリアルではない点に留意してください。

## 本当に時間がない。どうすれば？（I really don't have time…）

BotR に取り組む際のコツ:

- 作業を分散する。連続した数日間を確保するのではなく、コーディングやバグ修正の合間に少しずつ進める。
- 他の人に（大部分を）書いてもらう。新人支援として有効。メンタリングとレビューに徹する。
- 既存のドキュメント（MSDN、ブログなど）を活用し、章のベースにする。


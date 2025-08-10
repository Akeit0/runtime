<!--
Translated from: docs/design/coreclr/botr/README.md
Source commit: c70367babfe
Translator: akeit0, AI
Date: 2025-08-10
License: MIT
-->

翻訳元
https://github.com/dotnet/runtime/blob/cf2786c01cebaaf27352808d2fb37a2613876a94/docs/design/coreclr/botr/README.md

# ランタイムの書 (Book of the Runtime)

.NET Runtime 向け Book of the Runtime (BOTR) へようこそ。これは .NET Runtime の非自明な内部に関する記事のコレクションである。想定読者は、実際にコードを変更する開発者、あるいはランタイムを深く理解したい読者である。

以下に目次を示す。

- [Book of the Runtime FAQ](botr-faq.md)
- [Common Language Runtime への導入](intro-to-clr.md)
- [Garbage Collection の設計](garbage-collection.md)
- [Threading](threading.md)
- [RyuJIT の概要](../../../../docs/design/coreclr/jit/ryujit-overview.md)
  - [RyuJIT を他プラットフォームへ移植](../../../../docs/design/coreclr/jit/porting-ryujit.md)
- [Type System](type-system.md)
- [Type Loader](type-loader.md)
- [Method Descriptor](method-descriptor.md)
- [Virtual Stub Dispatch](virtual-stub-dispatch.md)
- [Stack Walking](stackwalking.md)
- [`System.Private.CoreLib` とランタイム呼び出し](corelib.md)
- [Data Access Component (DAC) に関するノート](../../../../docs/design/coreclr/botr/dac-notes.md)
- [Profiling](../../../../docs/design/coreclr/botr/profiling.md)
- [Profilability の実装](../../../../docs/design/coreclr/botr/profilability.md)
- [ランタイムにおける例外について開発者が知っておくべきこと](exceptions.md)
- [ReadyToRun の概要](../../../../docs/design/coreclr/botr/readytorun-overview.md)
- [CLR ABI](../../../../docs/design/coreclr/botr/clr-abi.md)
- [クロスプラットフォーム Minidumps](../../../../docs/design/coreclr/botr/xplat-minidump-generation.md)
- [Mixed Mode Assemblies](../../../../docs/design/coreclr/botr/mixed-mode.md)
- [移植のためのガイド](../../../../docs/design/coreclr/botr/guide-for-porting.md)
- [Vectors と Intrinsics](../../../../docs/design/coreclr/botr/vectors-and-intrinsics.md)

この目次は網羅的ではない可能性がある。すべての章の一覧は、章が保存されているディレクトリを参照すれば取得できる。

* [GitHub 上の BOTR 章一覧](../botr)

# 訳注
1. Book of the Runtime (BOTR): .NET Runtime の内部設計・実装に関する記事集の呼称。

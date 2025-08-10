<!--
Translated from: docs/design/coreclr/botr/README.md
Source commit: c70367babfe
Translator: akeit0, AI
Date: 2025-08-10
License: MIT
-->

# ランタイムの書 (Book of the Runtime)

.NET Runtime 向け Book of the Runtime (BOTR) へようこそ。これは .NET Runtime の非自明な内部に関する記事のコレクションである。想定読者は、実際にコードを変更する開発者、あるいはランタイムを深く理解したい読者である。

以下に目次を示す。

- [Book of the Runtime FAQ](botr-faq.md)
- [Common Language Runtime への導入](intro-to-clr.md)
- [Garbage Collection の設計](garbage-collection.md)
- [Threading](threading.md)
- [RyuJIT の概要](../jit/ryujit-overview.md)
  - [RyuJIT を他プラットフォームへ移植](../jit/porting-ryujit.md)
- [Type System](type-system.md)
- [Type Loader](type-loader.md)
- [Method Descriptor](method-descriptor.md)
- [Virtual Stub Dispatch](virtual-stub-dispatch.md)
- [Stack Walking](stackwalking.md)
- [`System.Private.CoreLib` とランタイム呼び出し](corelib.md)
- [Data Access Component (DAC) に関するノート](dac-notes.md)
- [Profiling](profiling.md)
- [Profilability の実装](profilability.md)
- [ランタイムにおける例外について開発者が知っておくべきこと](exceptions.md)
- [ReadyToRun の概要](readytorun-overview.md)
- [CLR ABI](clr-abi.md)
- [クロスプラットフォーム Minidumps](xplat-minidump-generation.md)
- [Mixed Mode Assemblies](mixed-mode.md)
- [移植のためのガイド](guide-for-porting.md)
- [Vectors と Intrinsics](vectors-and-intrinsics.md)

この目次は網羅的ではない可能性がある。すべての章の一覧は、章が保存されているディレクトリを参照すれば取得できる。

* [GitHub 上の BOTR 章一覧](../botr)

# 訳注
1. Book of the Runtime (BOTR): .NET Runtime の内部設計・実装に関する記事集の呼称。

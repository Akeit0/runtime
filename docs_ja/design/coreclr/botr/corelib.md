<!--
Translated from: docs/design/coreclr/botr/corelib.md
Source SHA: cf2786c01ce
Translator: akeit, AI
Updated: 2025-08-10
notes: WIP: 原文ファイル末尾が途中で途切れているため、そこまでを翻訳。以降は原文更新後に追補。
License: MIT
-->

翻訳元
https://github.com/dotnet/runtime/blob/cf2786c01cebaaf27352808d2fb37a2613876a94/docs/design/coreclr/botr/corelib.md

`System.Private.CoreLib` とランタイム呼び出し
===

# はじめに

`System.Private.CoreLib.dll` は .NET の型システムの中核と、Base Class Library の相当部分を定義するアセンブリである。もともと .NET Core では `mscorlib` と呼ばれていたが、コードや文書の多くは今も `mscorlib` を参照している。本書では `System.Private.CoreLib` あるいは CoreLib という表記に統一する。基本データ型はこのアセンブリ内にあり、CLR と強く結合している。本稿では CoreLib が特別である理由と、管理コードから QCall と FCall を介して CLR を呼び出す基本を述べる。併せて、CLR 内から管理コードを呼び出す方法も説明する。

## 依存関係

`Object`、`Int32`、`String` といった基本データ型を CoreLib が定義するため、CoreLib は他の管理アセンブリに依存できない。他方で CoreLib と CLR の間には強い依存がある。CoreLib の多くの型はネイティブ コードからアクセスされる必要があるため、多くの管理型のレイアウトは管理コード内と CLR 内のネイティブ コードの双方で定義される。さらに、デバッグ／チェックド／リリースの各ビルドでのみ定義されるフィールドも存在し得るため、通常 CoreLib はビルド種別ごとに個別にコンパイルされる必要がある。

`System.Private.CoreLib.dll` は 64 ビットと 32 ビットで個別にビルドされ、公開する定数の一部はビット幅により異なる。`IntPtr.Size` のような定数を用いれば、CoreLib より上位の多くのライブラリは 32/64 ビット個別ビルドを避けられる。

## `System.Private.CoreLib` を特別たらしめるもの

CoreLib にはいくつかの固有性がある。その多くは CLR との強結合による。

- CLR の Virtual Object System を実装するために必要な核となる型（`Object`、`Int32`、`String` などの基本データ型）を定義する。
- CLR は起動時に一部のシステム型をロードするために CoreLib をロードしなければならない。
- レイアウト上の事情により、プロセス内で同時にロードできる CoreLib は一つだけである。複数の CoreLib をロードするには、CLR と CoreLib の間で動作、FCall メソッド、データ型レイアウトに関する契約を形式化し、かつバージョン間で安定に保つ必要がある。
- CoreLib の型はネイティブ相互運用で多用され、管理例外は適切にネイティブのエラーコード／形式へマップされるべきである。
- CLR の複数の JIT コンパイラは、性能上の理由から CoreLib 内の一部メソッドを特別扱いすることがある（例: `Math.Cos(double)` の最適化による呼び出しの消去、`Array.Length`、`StringBuilder` の実装詳細とスレッド取得など）。
- CoreLib は必要に応じて P/Invoke を介してネイティブ コード（主として基盤 OS、場合によりプラットフォーム適応層）を呼び出す必要がある。
- CoreLib は CLR 固有の機能（GC のトリガー、クラスのロード、型システムとの非自明な相互作用など）を公開するために CLR を呼び出す必要がある。これは管理コードと CLR 内のネイティブな「手動管理」コードの橋渡しを要する。
- CLR は管理メソッドを呼ぶため、また管理コードでしか実装されていない機能にアクセスするために、管理コードを呼び出す必要がある。

# 管理コードと CLR コードのインターフェイス

CoreLib の管理コードが必要とするものを改めて挙げる。

- 一部の管理データ構造のフィールドへ、管理コードおよび CLR 内の「手動管理」コードの双方からアクセスできること。
- 管理コードから CLR を呼び出せること。
- CLR から管理コードを呼び出せること。

これらを実現するには、ネイティブ コード側で管理オブジェクトのレイアウトを指定し、必要に応じて検証できる仕組み、管理側からネイティブ コードを呼ぶ仕組み、ネイティブ側から管理コードを呼ぶ仕組みが要る。

また管理側からネイティブ コードを呼ぶ仕組みは、`String` のコンストラクタが用いる特殊な管理呼び出し規約（典型的な「GC がメモリ確保した後にコンストラクタを呼ぶ」慣習ではなく、コンストラクタ自身がオブジェクトのメモリを確保する）もサポートしなければならない。

CLR は内部的に [`mscorlib` binder](https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/binder.cpp) を提供し、アンマネージ側の型・フィールドと管理側の型・フィールドの対応付けを行う。binder はクラスの検索・ロードと管理メソッドの呼び出しを可能にし、さらに管理／ネイティブ双方で指定されたレイアウト情報の単純な検証を行って正当性を確保する。binder はロードを試みる管理クラスが mscorlib 内に存在しロード済みであること、フィールドオフセットが正しいことを保証し、また異なるシグネチャのオーバーロードを識別できなければならない。

# 管理からネイティブ コードの呼び出し

管理コードから CLR を呼ぶ手法は 2 つある。FCall は CLR コードへ直接呼び込む手段で、オブジェクト操作の柔軟性が高い一方で、参照追跡を誤ると GC hole を引き起こしやすい。QCall は P/Invoke を介して CLR を呼べるもので、誤用しにくい。FCall は管理コード側では [`MethodImplOptions.InternalCall`](https://learn.microsoft.com/dotnet/api/system.runtime.compilerservices.methodimploptions) ビットが立った extern メソッドとして識別される。QCall は通常の P/Invoke に似た `static extern` メソッドとして宣言するが、`"QCall"` というライブラリを指す点が異なる。

### FCall / QCall / P/Invoke / 管理実装の選択

まず前提として、可能な限り管理コードで実装すべきである。GC hole の危険が大幅に減り、デバッグ体験が良くなり、しばしばコードも単純になる。

過去に FCall を書く理由は大きく 3 つ（言語機能の不足、性能、ランタイムとの特別な相互作用）に分けられた。現在の C# は unsafe やスタック確保バッファなど C++ で得られたほぼすべての有用な言語機能を備えており、前 2 者の動機は解消された。実際、かつて FCall に強く依存していた CLR の一部（Reflection、いくつかの Encoding、String 操作など）を管理コードへ移植しており、この流れを継続する意向である。

もし FCall を定義する唯一の理由がネイティブ メソッドを呼ぶことなら、P/Invoke を用いて直接呼ぶべきである。[P/Invoke](https://learn.microsoft.com/dotnet/api/system.runtime.interopservices.dllimportattribute) は公開されたネイティブ メソッド インターフェイスであり、必要なことは正しく実現できるはずである。

それでもランタイム内に機能を実装する必要があるなら、ネイティブ コードへの遷移頻度を減らせないか検討する。共通ケースは管理側で実装し、稀なコーナーケースのみネイティブへ委ねられないか。一般に、可能な限り管理側に寄せるのが最善である。

今後は QCall が推奨手段である。FCall は「やむを得ない場合」のみ用いる。典型的には、最適化が重要な共通のショートパスが存在する場合であり、そのショートパスは数百命令以内で、GC メモリ確保・ロック取得・例外送出を行ってはならない（`GC_NOTRIGGER`、`NOTHROWS`）。それ以外は QCall を使うべきである。

FCall は最適化すべき短い経路のために設計され、フレームの構築タイミングを明示制御できる。しかしエラーを誘発しやすく、多くの API では複雑さに見合わない。QCall は本質的に CLR への P/Invoke である。FCall に匹敵する性能が必要な場合は、QCall を作り [`SuppressGCTransitionAttribute`](https://learn.microsoft.com/dotnet/api/system.runtime.interopservices.suppressgctransitionattribute) を付与することを検討する。

結果として、QCall は `SafeHandle` に対する有利なマーシャリングを自動で提供する—ネイティブ メソッド側は `HANDLE` 型を受け取るだけでよく、そのメソッド本体の実行中に誰かがハンドルを解放してしまう懸念をせずに済む。FCall で同等のことを行うには `SafeHandleHolder` を使い、`SafeHandle` を保護する必要があるかもしれない。P/Invoke マーシャラを活用すれば、この種の追加配線コードを避けられる。

## QCall の機能的挙動

QCall は CoreLib から CLR への通常の P/Invoke と非常によく似ている。FCall と異なり、QCall は通常の P/Invoke と同様にすべての引数をアンマネージ型としてマーシャリングする。また、通常の P/Invoke と同様にプリエンプティブ GC モードへ切り替える。これら 2 つの特性により、QCall は FCall に比べ堅牢に記述しやすい。QCall は FCall で一般的な GC hole や GC starvation のバグに陥りにくい。

QCall の引数型としては、P/Invoke マーシャラが効率良く扱えるプリミティブ型（`INT32`、`LPCWSTR`、`BOOL`）が好ましい。なお、QCall の引数で用いるべきブール型は `BOOL` である。一方、FCall の引数で正しいブール型は `CLR_BOOL` である。

一般的なアンマネージ EE 構造体へのポインタはハンドル型でラップすべきである。これは管理側の実装を型安全にし、C# の unsafe を多用せずに済ませるためである。例としては [vm\\qcall.h][qcall] の `AssemblyHandle` を参照。

[qcall]: https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/qcall.h

QCall でのオブジェクト参照の受け渡しは、ローカル変数へのポインタをハンドルに包むことで行う。これは意図的に煩雑であり、可能な限り避けるべきである。下の例にある `StringHandleOnStack` を参照。特に `string` のようなオブジェクトを QCall から返す場合は、未包装のオブジェクトを渡すのが広く許容される唯一の一般的パターンである（QCall を GC hole に陥りにくくする制約の意義については、後述の ["GC Holes, FCall, and QCall"](#gcholes) を参照）。

QCall は C 形式の関数シグネチャで実装すべきである。これにより、将来の AOT ツールが管理側の QCall とネイティブ側の実装を結びつけやすくなる。

### QCall 例 — 管理側（managed）

（注）以下のコメントはそのまま複製しないこと。

```csharp
class Foo
{
    // すべての QCall は以下の DllImport 属性を持つ
    [DllImport(RuntimeHelpers.QCall, EntryPoint = "Foo_BarInternal", CharSet = CharSet.Unicode)]

    // QCall は常に static extern
    private static extern bool BarInternal(int flags, string inString, StringHandleOnStack retString);

    // 多くの QCall は薄い managed ラッパーを持ち、
    // 可能な限り遷移前に作業（例: 引数検証）を行う
    public string Bar(int flags)
    {
        if (flags != 0)
            throw new ArgumentException("Invalid flags");

        string retString = null;
        // 文字列は StringHandleOnStack を使い、ローカル変数のアドレスを渡して返す
        if (!BarInternal(flags, this.Id, new StringHandleOnStack(ref retString)))
            FatalError();

        return retString;
    }
}
```

### QCall 例 — ネイティブ側（unmanaged）

実際の QCall 実装に、以下のコメントをそのまま複製しないこと。

QCall のエントリポイントは [vm\qcallentrypoints.cpp][qcall-entrypoints] のテーブルに `DllImportEntry` マクロで登録する（後述の「[Registering your QCall or FCall method](#register)」参照）。

[qcall-entrypoints]: https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/qcallentrypoints.cpp

```C++
// すべての QCall はフリー関数で、QCALLTYPE と extern "C" を付与
extern "C" BOOL QCALLTYPE Foo_BarInternal(int flags, LPCWSTR wszString, QCall::StringHandleOnStack retString)
{
    // すべての QCall は QCALL_CONTRACT を持つ
    // これは THROWS; GC_TRIGGERS; MODE_PREEMPTIVE の別名
    QCALL_CONTRACT;

    // 必要に応じて QCALL_CHECK と展開形の契約を使い、事前条件を指定できる
    // CONTRACTL {
    //     QCALL_CHECK;
    //     PRECONDITION(wszString != NULL);
    // } CONTRACTL_END;

    // QCALL_CONTRACT と BEGIN_QCALL の間に置けるのは戻り値の補助変数だけ
    BOOL retVal = FALSE;

    // 本体は BEGIN_QCALL/END_QCALL で囲む（例外処理に必要）
    BEGIN_QCALL;

    // 引数検証は理想的には managed 側で行うが、場合によっては native 側で必要
    // managed 側で検証済みなら、native 側では assert してよい
    _ASSERTE(flags != 0);

    // QCall へ渡した文字列はマーシャリングにより pin されるので GC 移動を心配不要
    printf("%S\n", wszString);

    // managed へ文字列を返す最も効率的な方法。StringBuilder は不要
    retString.Set(L"Hello");

    // BEGIN_QCALL/END_QCALL の内部で return してはならない
    // 戻り値は補助変数で外へ渡す
    retVal = TRUE;

    END_QCALL;

    return retVal;
}
```

## FCall の機能的挙動（FCall functional behavior）

FCall はオブジェクト参照の受け渡しに柔軟だが、そのぶんコードは複雑になり、誤りの余地が増える。また、非自明な長さの FCall では、明示的に GC 実行の要否をポーリングすべきである。これを怠ると、managed から FCall がタイトループで繰り返し呼ばれた場合に GC starvation を招く。FCall 実行中のスレッドは協調（cooperative）モードでのみ GC を許可するためである。

FCall には大量の定型コードが必要で、ここでは説明しきれない。詳細は [fcall.h][fcall] を参照。

[fcall]: https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/fcall.h

### <a name="gcholes"></a>GC holes、FCall、QCall

GC hole に関するより完全な議論は [CLR Code Guide](../../../coding-guidelines/clr-code-guide.md) の ["Is your code GC-safe?"](../../../coding-guidelines/clr-code-guide.md#2.1) を参照。本節は、FCall と QCall に奇妙に見える規約がある理由を動機付ける。

FCall メソッドにパラメータとして渡されたオブジェクト参照は GC 保護されていない。つまり GC が発生すると、その参照はオブジェクトの新しい位置ではなく古い位置を指してしまう。このため FCall では、`StringObject*` のような型で受け取り、GC を誘発しうる操作の前に明示的に `STRINGREF` へ変換する作法に従うのが通例である。後でオブジェクト参照を使うなら、GC を発生させる前に参照を GC 保護しなければならない。

`OBJECTREF` の適切な報告に失敗したり、インテリアポインタの更新を怠ることは一般に「GC hole」と呼ばれる。`OBJECTREF` はデバッグ／チェックド ビルドで参照の逆参照ごとに妥当性検査を行い、不正なオブジェクトを指していれば「不正なオブジェクト参照を検出。GC hole の可能性」等の assert を発火させる。手動管理コードでは極めて起こしやすい。

QCall のプログラミング モデルは、スタック上のオブジェクト参照のアドレスを渡すことを強制することで GC hole を避けるように制限している。これにより JIT のレポート ロジックで参照が GC 保護され、対象の参照自体は GC ヒープに割り当てられていないため移動しないことも保証される。QCall を推奨するのは、まさに GC hole を起こしにくくするためである。

### x86 における FCall エピローグ ウォーカー（FCall epilog walker for x86）

managed スタック ウォーカーは FCall から呼び戻し地点を見つけられなければならない。ABI の一部としてスタック アンワインド規約を定義する新しいプラットフォームでは比較的容易だが、x86 の ABI には規約がない。ランタイムはエピローグ ウォーカーを実装することでこれを回避する。エピローグ ウォーカーは FCall の実行をシミュレートし、FCall の戻りアドレスと callee-save レジスタを算出する。このため、FCall 実装で許される構成に制限が生じる。

デストラクタを持つスタック割り当てオブジェクトや例外処理のような複雑な構成は、エピローグ ウォーカーを混乱させる可能性がある。これはスタックウォーク中の GC hole やクラッシュにつながる。これらのバグを避けるために避けるべき構成の包括的リストは存在しない。今日は問題ない FCall 実装が、明日の C++ コンパイラ更新で壊れる可能性もある。この領域の不具合はストレス実行やカバレッジで検出に頼っている。

### FCall 例 — 管理側（managed）

`String` クラスからの実例：

```csharp
public partial sealed class String
{
    [MethodImpl(MethodImplOptions.InternalCall)]
    private extern string? IsInterned();

    public static string? IsInterned(string str)
    {
        if (str == null)
        {
            throw new ArgumentNullException(nameof(str));
        }

        return str.IsInterned();
    }
}
```

### FCall 例 — ネイティブ側（unmanaged）

FCall のエントリポイントは [vm\ecalllist.h][ecalllist] のテーブルに `FCFuncEntry` マクロで登録する（後述「[Registering your QCall or FCall method](#register)」参照）。

[ecalllist]: https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/ecalllist.h

次の例は `Object*` として生のポインタで managed オブジェクトを受け取る FCall メソッドを示す。これらの生入力は「unsafe」と見なされ、GC に敏感な文脈で使う場合は検証または変換が必要である。

```C++
FCIMPL1(FC_BOOL_RET, ExceptionNative::IsImmutableAgileException, Object* pExceptionUNSAFE)
{
    FCALL_CONTRACT;

    ASSERT(pExceptionUNSAFE != NULL);

    OBJECTREF pException = (OBJECTREF) pExceptionUNSAFE;

    FC_RETURN_BOOL(CLRException::IsPreallocatedExceptionObject(pException));
}
FCIMPLEND
```

## <a name="register"></a>QCall / FCall メソッドの登録（Registering your QCall or FCall method）

CLR は QCall / FCall の名前（managed のクラス名・メソッド名と、対応するネイティブの呼び出し先）を把握していなければならない。FCall の登録は [ecalllist.h][ecalllist] で行い、2 段の配列を使う。第 1 配列は名前空間とクラス名を関数要素配列に対応付け、その関数要素配列がメソッド名とシグネチャを関数ポインタへ対応付ける。

上の例で `String.IsInterned()` 向けの FCall を定義したとする。まず `String` クラス用の関数要素配列があることを確認する。

```C++
// 名前:名前空間 の組でソートされていなければならない点に注意
    ...
    FCClassElement("String", "System", gStringFuncs)
    ...
```

次に、`gStringFuncs` に `IsInterned` の正しいエントリが含まれるようにする。メソッド名がオーバーロードされている場合はシグネチャを指定できる。

```C++
FCFuncStart(gStringFuncs)
    ...
    FCFuncElement("IsInterned", AppDomainNative::IsStringInterned)
    ...
FCFuncEnd()
```

QCall の登録は [qcallentrypoints.cpp][qcall-entrypoints] の `s_QCall` 配列で `DllImportEntry` マクロを用いて行う。

```C++
static const Entry s_QCall[] =
{
    ...
    DllImportEntry(MyQCall),
    ...
};
```

## 他サブシステムとの相互作用（Interactions with other subsystems）

### デバッガ（Debugger）

現状の制約として、Visual Studio の Interop（mixed mode）デバッグで managed と FCall の両方を簡単にデバッグすることはできない。FCall にブレークポイントを置いて Interop デバッグしても動作しない。これは修正されない見込みである。

# 物理アーキテクチャ（Physical architecture）

CLR の起動時、CoreLib は `SystemDomain::LoadBaseSystemClasses()` によりロードされる。ここで基本データ型や `Exception` などのクラスがロードされ、CoreLib の型を指すグローバル ポインタが適切に設定される。

FCall については基盤を [fcall.h][fcall]、FCall メソッドのランタイムへの通知は [ecalllist.h][ecalllist] を参照。

QCall については基盤を [qcall.h][qcall]、QCall メソッドのランタイムへの通知は [qcallentrypoints.cpp][qcall-entrypoints] を参照。

より一般的な基盤や一部のネイティブ型定義は [object.h][object.h] にある。binder は `mscorlib.h` を使って managed とネイティブ クラスを関連付ける。

[object.h]: https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/object.h

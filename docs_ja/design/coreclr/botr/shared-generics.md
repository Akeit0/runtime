<!--
Translated from: docs/design/coreclr/botr/shared-generics.md
Source commit: c70367babfe
Translator: akeit0, AI
Date: 2025-08-10
License: MIT
-->

Shared Generics の設計
===

著者: Fadi Hanna - 2019

# はじめに

Shared generics は、さまざまな具象化（generic 型上のメソッドおよび generic メソッドの両方を含む）について、ランタイムが生成するコード量を削減するための Runtime + JIT の機能である。特定の具象化では、生成されるコードはごくわずかな命令を除いてほぼ同一になる。そのためメモリ使用量と generic メソッドを JIT する時間を削減する目的で、ランタイムは単一の特別な正準（canonical）版のコードを生成し、それを当該メソッドの互換なすべての具象化で共有して使用できるようにする。

### 正準コード生成と Generic Dictionary

次の C# の例を考える。

``` c#
string Method<T>()
{
    return typeof(List<T>).ToString();
}
```

Shared generics を用いない場合、`Method<object>` や `Method<string>` のような具象化に対するコードは、型 `List<T>` の正しい TypeHandle をロードする 1 命令だけが異なる点を除いて同一になる。
``` asm
    mov rcx, type handle of List<string> or List<object>
    call ToString()
    ret
```

Shared generics を用いると、正準コードには `List<T>` の型ハンドルを埋め込まない。その代わり、実行中の `Method<T>` の具象化が持つ「generic dictionary」から読み込むか、またはランタイムの helper API を呼び出して、該当の型ハンドルを参照する。コードは次のようになる。
``` asm
    mov rcx, generic context                                                // Method<string> または Method<object> の MethodDesc
    mov rcx, [rcx + offset of InstantiatedMethodDesc::m_pPerInstInfo]       // これが generic dictionary
    mov rcx, [rcx + dictionary slot containing type handle of List<T>]
    call ToString()
    ret
```

この例における generic context は `Method<object>` または `Method<string>` の `InstantiatedMethodDesc` である。generic dictionary は、shared generic のコードが具象化ごとの情報を取得するために用いるデータ構造で、要素は具象化固有の type handle、method handle、field handle、メソッドのエントリポイント等の配列である。MethodTable と `InstantiatedMethodDesc` にある "PerInstInfo" フィールドは、それぞれ generic 型および generic メソッドの generic dictionary を指す。

この例では、`Method<object>` の generic dictionary には `List<object>` の type handle を格納したスロットが、`Method<string>` の generic dictionary には `List<string>` の type handle を格納したスロットが存在する。

この機能は現在、参照型に対する具象化のみを対象としている。これらはサイズ／特性／レイアウト等が同一であるためである。プリミティブ型や値型に対する具象化では、ランタイムは具象化ごとに別個のコード本体を生成する。


# レイアウトとアルゴリズム

### 型およびメソッド上の Dictionary ポインタ

任意の generic メソッドで使用される dictionary は、そのメソッドの `InstantiatedMethodDesc` 構造体にある `m_pPerInstInfo` フィールドから参照される。これは generic dictionary データ本体への直接ポインタである。

generic 型ではさらに 1 段の間接参照が存在する。`MethodTable` 構造体の `m_pPerInstInfo` フィールドは dictionary のテーブルを指し、その各要素は実際の generic dictionary データを指すポインタである。型には継承があり、派生 generic 型は基底型の dictionary を継承するためである。

例を示す。
```c#
class BaseClass<T> { }

class DerivedClass<U> : BaseClass<U> { }

class AnotherDerivedClass : DerivedClass<string> { }
```

各型の MethodTable は次のようになる。

| **BaseClass[T] の MethodTable** |
|--------------------------|
| ...      |
| `m_PerInstInfo`: 下の dictionary テーブルを指す     |
| ...      |
| `dictionaryTable[0]`: 下の dictionary データを指す      |
| `BaseClass の dictionary データ`  |

| **DerivedClass[U] の MethodTable** |
|--------------------------|
| ...      |
| `m_PerInstInfo`: 下の dictionary テーブルを指す     |
| ...      |
| `dictionaryTable[0]`: `BaseClass` の dictionary データを指す      |
| `dictionaryTable[1]`: 下の dictionary データを指す      |
| `DerivedClass の dictionary データ`  |

| **AnotherDerivedClass の MethodTable** |
|--------------------------|
| ...      |
| `m_PerInstInfo`: 下の dictionary テーブルを指す     |
| ...      |
| `dictionaryTable[0]`: `BaseClass` の dictionary データを指す      |
| `dictionaryTable[1]`: `DerivedClass` の dictionary データを指す      |

`AnotherDerivedClass` は generic 型ではないため独自の dictionary は持たず、基底型の dictionary ポインタを継承する点に注意する。

### Dictionary スロット

前述のとおり、generic dictionary は具象化固有情報を保持する複数スロットの配列である。ある generic 型またはメソッドの dictionary が初期割り当てされると、すべてのスロットは `NULL` に初期化され、コードの実行に応じてオンデマンドで遅延的に埋められる（`Dictionary::PopulateEntry(...)` 参照）。

N 個の引数を持つ具象化では、先頭の N スロットは常にその具象化の型引数の type handle になる（最適化の一種でもある）。それに続くスロットには、具象化ベースの情報が入る。

`Method<string>` の generic dictionary 例は次のとおりである。

| `Method<string> の dictionary` |
|--------------------------|
| `slot[0]: TypeHandle(string)` |
| `slot[1]: Total dictionary size` |
| `slot[2]: TypeHandle(List<string>)` |
| `slot[3]: NULL (not used)` |
| `slot[4]: NULL (not used)` |

注: サイズスロットは generic コードからは使用せず、動的 dictionary 拡張機能の一部である（後述）。

この dictionary が初期割り当てされた時点では、具象化の型引数を含む `slot[0]`（および拡張機能に伴うサイズスロット）のみが初期化され、それ以外（例: `slot[2]`）は `NULL` のままである。対応するコードパスに到達した時点で、値が遅延的に設定される。

`NULL` のままのスロットから値を読み込もうとする場合、generic コードは次のランタイム helper を呼び出して、そのスロットに値を設定する。
- `JIT_GenericHandleClass`: generic 型の dictionary を検索する。generic 型のインスタンス メソッドすべてで使用される。
- `JIT_GenericHandleMethod`: generic メソッドの dictionary を検索する。generic メソッド、または generic 型上の非 generic 静的メソッドで使用される。

shared generic コードを生成する際、JIT は `DictionaryLayout` 実装（[genericdict.cpp](https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/genericdict.cpp)）に基づいて、各参照でどのスロットを使うべきか、各スロットにどの種別の情報が入るのかを把握する。

### Dictionary レイアウト

`DictionaryLayout` 構造体は、JIT に対して辞書参照で使用すべきスロット番号を示す。主な性質は次のとおり。
- ある型／メソッドの互換な具象化間で共有される。すなわち、レイアウトは型またはメソッドの正準具象化にひも付く。前述の例では、`Method<object>` と `Method<string>` は互換な具象化であり、各々は独立した「別個の dictionary」を持つが、正準具象化 `Method<__Canon>` に結び付く「同一の dictionary レイアウト」を共有する。
- generic 型やメソッドの dictionary は、対応するレイアウトと同数のスロットを持つ。注: 動的 dictionary 拡張機能の導入以前は、generic dictionary がレイアウトより小さい場合があり、その場合は特定の参照でランタイム helper（低速経路）を呼び出す必要があった。

generic 型／メソッドが最初に作成されると、その dictionary レイアウトには「未割当」のスロットが含まれる。割当はコード生成の過程で、JIT が辞書参照列を出力する必要が生じた時に行われる。この割当は `DictionaryLayout::FindToken(...)` API 呼出の際に実施される。スロットが一度割り当てられると、そのスロットは「署名（signature）」に結び付けられ、以後、すべての具象化でそのインデックスには同種の値が入ることを表す。

入力 signature に対し、スロット割当は次のアルゴリズムで行われる。

```
Begin with slot = 0
Foreach entry in dictionary layout
    If entry.signature != NULL
        If entry.signature == inputSignature
            return slot
        EndIf
    Else
        entry.signature = inputSignature
        return slot
    EndIf
    slot++
EndForeach
```

上記で一致する既存スロットが見つからず、かつ「未割当」スロットも尽きた場合、動的 dictionary 拡張が起動し、レイアウトにスロットを追加してサイズ変更し、同レイアウトに結び付くすべての dictionary を拡張する。

# 動的 Dictionary 拡張

### 経緯

動的 dictionary 拡張導入以前、dictionary レイアウトはバケット（固定長 `DictionaryLayout` を連結したリスト）で構成されていた。初期バケットのサイズは、generic 型ではヒューリスティクスで決まり、generic メソッドでは常に 4 スロットであった。generic 型・メソッド側の generic dictionary も固定長で、参照に使用できる（いわゆる「高速参照スロット」）。

バケットが満杯になると、新しい `DictionaryLayout` バケットを割り当ててリストに連結した。しかし型やメソッドの generic dictionary は固定長で割当済みのためリサイズできず、また JIT は複数の dictionary を連結リスト経由で間接参照する命令を生成できないという制約があった。このため、最初の `DictionaryLayout` バケットに対応する固定数の値のみを generic dictionary で直接参照でき、それ以外の参照ではより遅いランタイム helper を経由せざるを得なかった。

これは [ReadyToRun](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/readytorun-overview.md) や Tiered Compilation の導入までは許容できた。ReadyToRun コード使用時にスロット割当が急速に消費され、さらにランタイムが性能向上のためにメソッドを再 JIT した際、場合によっては「高速参照スロット」が残っておらず、低速なランタイム helper 経由のコード生成を強いられることがあった。これが一部シナリオで性能低下を招いたため、ReadyToRun コードでは高速参照スロットを使わず、再 JIT コードのために温存するという方針に切り替えた。この判断は ReadyToRun の性能を損なうが、R2R よりも再 JIT のスループットを重視した妥協として必要であった。

このため、動的 dictionary 拡張が導入された。

### 概要とアルゴリズム

発想は単純である。dictionary レイアウトを「バケット連結リスト」から「動的に拡張可能な配列」に置き換える。ただし実装にあたっては次の理由から細心の注意が必要であった。
- `DictionaryLayout` だけをリサイズすることはできない。レイアウトのサイズが実際の generic dictionary のサイズを上回ると、JIT が生成する間接参照命令と dictionary データの実サイズが不一致となり、アクセス違反を招く。
- generic dictionary 自体を安易にリサイズすることもできない。
    - 型の場合、generic dictionary は `MethodTable` 構造体の一部であり、再割当はできない（すでに managed コードから使用されている）。
    - メソッドの場合、generic dictionary は `MethodDesc` の一部ではないが、一部の generic コードから使用中である可能性がある。
    - いずれにせよ、同一の型やメソッドに対して複数の MethodTable／MethodDesc を併存させることはできないため、再割当は選択肢にならない。
- 単一の具象化だけの generic dictionary をリサイズすることもできない。前述の例で `Method<string>` の dictionary を拡張したとすると、`Method<__Canon>` の共有正準コードに影響が及ぶ。`Method<string>` だけを拡張すると、その具象化では正常でも、`Method<object>` 等の他の具象化では、JIT 命令列と dictionary 構造のサイズが一致せずアクセス違反になる。
- ランタイムは多重スレッドである。

現在の実装では、レイアウトと実データの拡張を分離して単純化している。

 - 空きスロットが尽きた時点でレイアウトを拡張する。実装は [genericdict.cpp](https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/genericdict.cpp) の `DictionaryLayout::FindToken()` を参照。
 - 型およびメソッドの具象化 dictionary は、コードが当該型／メソッドの dictionary サイズを超えるスロットの値を読もうとした時に、前述の helper（`JIT_GenericHandleClass` / `JIT_GenericHandleMethod`）を経由してオンデマンドで遅延的に拡張される。

dictionary 参照のコード生成は（JITted コードおよび ReadyToRun コードの双方で）概ね次の擬似コードに等しい。
``` c++
void* pMethodDesc = <some value>;                    // 具象化 generic メソッドの MethodDesc
int requiredOffset = <some value>;                    // アクセスしたいオフセット

void* pDictionary = pMethodDesc->m_pPerInstInfo;

// 間接参照する前に、必ずサイズを確認する点に注意
if (pDictionary[sizeOffset] <= requiredOffset || pDictionary[requiredOffset] == NULL)
    pResult = JIT_GenericHandleMethod(pMethodDesc, <signature>);
else
    pResult = pDictionary[requiredOffset];
```

このサイズ確認を、辞書からの読み込みのたびに無条件で行うわけではない。そうすると顕著な性能低下を招くためである。dictionary レイアウトが初期化された時点で、割当済みスロット数（初期スロット数）を記録し、「その数を超えるスロット」を読む場合にのみサイズ確認を行う。

型およびメソッド上の dictionary の拡張は、[genericdict.cpp](https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/genericdict.cpp) の `Dictionary::GetTypeDictionaryWithSizeCheck()` と `Dictionary::GetMethodDictionaryWithSizeCheck()` で行う。

型に関して一点注意すべきは、基底型から dictionary ポインタを継承する可能性があることだ。すなわち、ある generic 型の generic dictionary をリサイズした場合、その新しい dictionary ポインタをすべての派生型に伝播する必要がある。この伝播も遅延的に実施され、派生型の MethodTable ポインタを伴って `JIT_GenericHandleWorker` helper が呼ばれた際に、基底型の dictionary ポインタ更新を検知したら派生型へコピーする。

古い dictionary はリサイズ後も解放されない。ただし新しい dictionary が MethodTable／MethodDesc に公開されると、それ以降の generic コードからの参照は新しい dictionary を使用する。特に多重スレッド環境では、古い dictionary の解放はきわめて複雑で有益性も薄い。

# 訳注
1. 正準（canonical）: 共有コード生成で代表として用いる特殊な具象化（例: `__Canon`）のこと。

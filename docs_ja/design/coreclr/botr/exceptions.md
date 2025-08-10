<!--
source: docs/design/coreclr/botr/exceptions.md
source_commit: (unknown)
status: WIP
translator: TBD
updated: 2025-08-10
notes: 初回作成。原文の章立て・コード片・リンクを維持。
-->

# ランタイムにおける例外について、すべての開発者が知っておくべきこと（What Every Dev needs to Know About Exceptions in the Runtime）

日付: 2005

CLR における「例外」を語る際には、重要な区別があります。アプリケーションから C# の `try/catch/finally` 等で扱う「マネージ例外」と、ランタイム内部で用いる「CLR 内部例外」です。ほとんどのランタイム開発者は、マネージ例外モデルそのものを構築・公開する方法を意識する必要はあまりありません。しかし、ランタイムの実装内部で例外がどう使われるかは全員が理解する必要があります。本稿では、紛らわしさを避けるため、アプリから投げたり捕捉したりする例外を「マネージ例外」、ランタイムのエラー処理に使うものを「CLR の内部例外」と呼び分けます。主題は後者、すなわち CLR の内部例外です。

## 例外が重要になる場面（Where do exceptions matter?）

例外はほぼ至る所で重要です。特に、例外を投げたり捕捉したりする関数では、明示的な実装が必要です。自身は例外を投げなくても、例外を投げうる関数を呼ぶ可能性があり、その場合、例外がスローされスタックを通過しても正しく振る舞うように書かれていなければなりません。こうしたコードの正しさを保つうえで、適切な「ホルダー（holder）」の活用は非常に有効です。

## CLR 内部例外が異なる理由（Why are CLR internal exceptions different?）

CLR の内部例外は C++ 例外に似ていますが、まったく同じではありません。CoreCLR は macOS、Linux、BSD、Windows を対象にビルドされます。OS やコンパイラの違いにより、標準の C++ `try/catch` だけでは対応できません。さらに、CLR の内部例外はマネージの「finally」や「fault」に相当する機能を備えています。

いくつかのマクロの助けにより、標準 C++ にかなり近い感覚で例外処理コードを書けるようになっています。

## 例外を捕捉する（Catching an Exception）

### EX_TRY

基本のマクロは `EX_TRY / EX_CATCH / EX_END_CATCH` です。使用例:

```
EX_TRY
  // 何らかの関数呼び出し。例外を投げるかもしれない。
  Bar();
EX_CATCH
  // ここに来たということは失敗した。
  m_finalDisposition = terminallyHopeless;
  RethrowTransientExceptions();
EX_END_CATCH
```

`EX_TRY` は C++ の `try` に似ていますが、同時に波かっこ `{` を開く点が異なります。

### EX_CATCH

`EX_CATCH` は `try` ブロックを閉じ（`}` を含む）、`catch` ブロックを開始します。こちらも波かっこでブロックを開きます。

ここが C++ との差異の核心です。CLR の開発者は「何を捕捉するか」を指定できません。これらのマクロは、AV（アクセス違反）やマネージ例外など、あらゆる例外を捕捉します。特定の例外だけを扱いたい場合は、一旦すべて捕捉して中身を調べ、対象外は再スローする必要があります。

繰り返しますが、`EX_CATCH` はすべてを捕捉します。これはしばしば望ましい挙動ではありません。以降で、捕捉すべきでない例外への対処を説明します。

### GET_EXCEPTION() と GET_THROWABLE()

では、捕捉した例外の正体をどう特定し、どう扱うべきかをどう判断するか。要件に応じた選択肢があります。

- 捕捉された C++ 例外は、グローバルな `Exception` 基底クラスの派生型インスタンスとして渡されます。`OutOfMemoryException` のような明白なもの、`EETypeLoadException` のようなドメイン特化のもの、他システムの例外をラップする `CLRException`（任意のマネージ例外への `OBJECTHANDLE` を保持）や `HRException`（`HRESULT` を保持）などがあります。元の例外が `Exception` 派生でない場合でも、マクロ側でラップされます。（これらの例外型はシステム提供の既知のものであり、新たな例外型の追加は Core Execution Engine チームの関与なしに行うべきではありません。）
- CLR 内部例外には常に `HRESULT` が結びつきます。COM 由来の場合もあれば、内部エラーや Win32 API 失敗の `HRESULT` もあります。
- ほぼすべての内部例外は、最終的にマネージ側へ伝搬する可能性があるため、内部例外から対応するマネージ例外へのマッピングが存在します（必ずしも生成されるとは限りませんが、取得可能です）。

分類の実践:

- 単に `HRESULT` が分かれば十分なことが多く、取得は簡単です。

  ```cpp
  HRESULT hr = GET_EXCEPTION()->GetHR();
  ```

- 追加情報はマネージ例外オブジェクトから得るのが便利です。例外を直ちにマネージに返す場合や、後で投げ直すために保持する場合など、マネージ オブジェクトが必要になります。取得例（GC 保護が必要）:

  ```cpp
  OBJECTREF throwable = NULL;
  GCPROTECT_BEGIN(throwable);
  // ...
  EX_TRY
      // 例外を投げうる処理
  EX_CATCH
      throwable = GET_THROWABLE();
      RethrowTransientExceptions();
  EX_END_CATCH
  // throwable を使用
  GCPROTECT_END();
  ```

- C++ 例外オブジェクトの型を厳密に知る必要がある場合（主に例外実装内部）には、軽量な RTTI 風 API が使えます。

  ```cpp
  Exception* pEx = GET_EXCEPTION();
  if (pEx->IsType(CLRException::GetType())) { /* ... */ }
  ```

### RethrowTransientExceptions（「一時的」例外の再スロー）

上の例中の `RethrowTransientExceptions` は、`EX_CATCH` 内で使う「例外ディスポジション」用マクロのひとつです。意味は次の通りです。

- `RethrowTerminalExceptions`: 実質「ThreadAbort を再スロー」。
- `RethrowTransientExceptions`: 「再試行（あるいは別コンテキスト）で発生しない可能性がある」一時的例外を再スロー。対象は次のとおり:
  - `COR_E_THREADABORTED`
  - `COR_E_THREADINTERRUPTED`
  - `COR_E_THREADSTOP`
  - `COR_E_APPDOMAINUNLOADED`
  - `E_OUTOFMEMORY`
  - `HRESULT_FROM_WIN32(ERROR_COMMITMENT_LIMIT)`
  - `HRESULT_FROM_WIN32(ERROR_NOT_ENOUGH_MEMORY)`
  - `(HRESULT)STATUS_NO_MEMORY`
  - `COR_E_STACKOVERFLOW`
  - `MSEE_E_ASSEMBLYLOADINPROGRESS`

迷ったら、`RethrowTransientExceptions` を選ぶのが無難です。

重要なのは、`EX_CATCH` はすべてを捕捉してしまうため、「捕捉しない」唯一の方法は「再スローする」ことだという点です。

### EX_CATCH_HRESULT

COM 由来のインターフェイス実装などでは、例外に対応する `HRESULT` だけが必要なことがあります。この場合、`EX_CATCH_HRESULT` はフルの `EX_CATCH` ブロックより簡潔です。

```
HRESULT hr;
EX_TRY
  // code
EX_CATCH_HRESULT(hr)

return hr;
```

ただし注意点があります。`EX_CATCH_HRESULT` は全例外を捕捉し、`HRESULT` を保存して「例外を飲み込みます」。例外を飲み込むことが関数の正しい振る舞いでない限り、使うべきではありません。

### EX_RETHROW

前述の通り、マクロはすべての例外を捕捉します。特定の例外だけを扱うには、いったんすべて捕捉し、不要なものを再スローするしかありません。`EX_RETHROW` は同じ例外を再度投げます。

## 例外を「捕捉しない」ために（Not catching an exception）

例外を捕捉する必要はないが、終了処理や巻き戻し処理を行いたい、という状況はよくあります。多くはホルダーで十分ですが、そうでない場合に使える「finally」相当の 2 つのバリエーションがあります。

### EX_TRY_FOR_FINALLY

終了時に補償処理が必要な場合に使う、`try/finally` のマクロです。

```
EX_TRY_FOR_FINALLY
  // code
EX_FINALLY
  // 終了・バックアウト処理
EX_END_FINALLY
```

重要: これらは C++ EH ではなく SEH で実装されています。C++ コンパイラは同じ関数内で SEH と C++ EH の混用を許しません。自動デストラクタを持つローカル変数は C++ EH によってのみデストラクタが実行されます。したがって、`EX_TRY_FOR_FINALLY` を含む関数では `EX_TRY` は使えず、自動デストラクタを持つローカルも置けません。

### EX_HOOK

例外が投げられた「ときだけ」補償処理を実行したい場合に使います。`EX_FINALLY` に似ていますが、`hook` 節は例外時のみ実行され、末尾で自動的に再スローされます。

```
EX_TRY
  // code
EX_HOOK
  // 例外が "code" ブロックから逸脱するときに実行する処理
EX_END_HOOK
```

これは単に `EX_CATCH` と `EX_RETHROW` を組み合わせるよりも良い動作をします。すなわち、スタックオーバーフロー以外は再スローしますが、スタックオーバーフローの場合はいったん捕捉してスタックを巻き戻し、新たなスタックオーバーフロー例外を投げ直します。

## 例外を投げる（Throwing an Exception）

CLR で例外を投げる基本は次の呼び出しです。

```
COMPlusThrow(<args>)
```

多くのオーバーロードがあり、例外の「種類」を渡します。種類の一覧は [`rexcep.h`](https://github.com/dotnet/runtime/blob/main/src/coreclr/vm/rexcep.h) 上のマクロから生成され、`kAmbiguousMatchException`、`kApplicationException` などが含まれます。追加引数でリソースや置換文字列等を指定します。一般には、類似のエラーを報告している既存コードを探して適切な種類を選択します。

代表的なヘルパー:

#### COMPlusThrowOOM()

内部で `ThrowOutOfMemory()` を呼び、C++ の OOM 例外（事前割り当て済み）を投げます。OOM 例外の生成に再びメモリ割り当てが必要になる問題を避けるためです。マネージ例外オブジェクト取得時にはまず新規割り当てを試み、失敗したら共有のグローバル OOM オブジェクトを返します。

#### COMPlusThrowHR(HRESULT theBadHR)

`IErrorInfo` 等に応じたオーバーロードがあります。特定の `HRESULT` に対応する例外種別の決定には、驚くほど複雑なロジックがあります。

#### COMPlusThrowWin32() / COMPlusThrowWin32(hr)

`HRESULT_FROM_WIN32(GetLastError())` に基づく例外を投げます。

#### COMPlusThrowSO()

スタックオーバーフロー（SO）例外を投げます。ここでの SO は「致命的なハード SO」ではなく、そのまま進行するとハード SO になり得る状況で投げるものです。OOM と同様、事前割り当て済みの C++ SO 例外オブジェクトを用い、マネージ側オブジェクトは常に共有のグローバル SO オブジェクトを返します。

#### COMPlusThrowArgumentNull()

「引数が null であってはならない」例外を投げるヘルパーです。

#### COMPlusThrowArgumentOutOfRange()

その名の通りのヘルパーです。

#### COMPlusThrowArgumentException()

無効引数の別バリエーションです。

#### COMPlusThrowInvalidCastException(thFrom, thTo)

キャストの from/to 型ハンドルから、整形された例外メッセージを生成します。

### EX_THROW

低レベルな throw 構文で、通常コードで直接使う必要はあまりありません。多くの `COMPlusThrowXXX` が内部で `EX_THROW` を用いています。例外機構の詳細をカプセル化するため、直接使用は最小限に留めるのが望ましいですが、高レベルの Throw 関数で対応できない場合は使用して構いません。引数は「投げる例外型（`Exception` の派生）」と、そのコンストラクタ引数の括弧付きリストです。

## 直接 SEH を使う（Using SEH directly）

いくつかの状況では、SEH を直接使うのが適切です。特に「第一パス（スタック巻き戻し前）」で処理が必要な場合には、SEH しか選択肢がありません。`__try/__except` のフィルターは「処理するか否か」の決定に加え、任意の処理を行えます。デバッガ通知など、第一パス処理が必要な領域があります。

フィルターは注意深く書く必要があります。一般に、フィルターは不整合な状態も含め、あらゆるランダムな状態に備えるべきです。フィルターは第一パスで実行され、デストラクタは第二パスで実行されるため、ホルダーはまだ動作しておらず、状態も復旧されていません。

### PAL_TRY / PAL_EXCEPT, PAL_EXCEPT_FILTER, PAL_FINALLY / PAL_ENDTRY

フィルターが必要な場合、CLR で移植性を保って書く方法がこの PAL_* 系です。フィルターは SEH を直接用いるため、同一関数内で C++ EH と併用できず、ホルダーも置けません。これらは稀なはずです。

### __try / __except, __finally

CLR 内でこれらを直接使う正当な理由は基本的にありません。

## 例外と GC モード（Exceptions and GC mode）

`COMPlusThrowXXX()` によるスローは GC モードに影響せず、どのモードでも安全です。例外が `EX_CATCH` まで巻き戻る間、スタック上のホルダーは解放・復旧されます。`EX_CATCH` に制御が戻る時点で、`EX_TRY` の時点におけるホルダー保護状態に戻っています。

## 遷移（Transitions）

マネージコード、CLR、COM サーバー、その他ネイティブコードの間には、呼び出し規約・メモリ管理・例外処理機構の観点で多くの遷移が存在します。多くはランタイム外か自動処理されますが、CLR 開発者が日常的に意識すべきは次の 3 つです。

### マネージコード → ランタイム

いわゆる fcall・JIT ヘルパー等です。ランタイムがエラーをマネージへ返す典型は「マネージ例外」です。fcall が直接/間接にマネージ例外を発生させるのは問題ありません（通常のマネージ例外機構が適切なハンドラーを探索します）。

一方、fcall が CLR 内部例外（C++ 例外のいずれか）を投げ得る場合、これをマネージへ漏らしてはいけません。そのために `UnwindAndContinueHandler (UACH)` があり、C++ EH を捕捉してマネージ例外として再スローします。マネージから呼ばれ、C++ EH を投げ得るランタイム関数は、投げ得る箇所を `INSTALL_UNWIND_AND_CONTINUE_HANDLER / UNINSTALL_UNWIND_AND_CONTINUE_HANDLER` で囲む必要があります。UACH の導入にはコストがあるため、常用は避け、性能クリティカルな場所では必要直前に導入するテクニックもあります。

UACH がない状態で C++ 例外が投げられると、典型的には `CPFH_RealFirstPassHandler` で「`GC_NOTRIGGER` 領域で `GC_TRIGGERS` が呼ばれた」という Contract 違反で失敗します。修正には、マネージ→ランタイム遷移箇所を探し、UACH の導入を確認します。

### ランタイム → マネージコード

ランタイムからマネージへ入る遷移はプラットフォーム依存です。32-bit Windows では、突入直前に `COMPlusFrameHandler` のインストールが必要です。これらの遷移は専用ヘルパーで処理され、適切な例外ハンドラーのセットアップが行われます。通常、新規のマネージ呼び出しで他の方法を用いることはまずありません。`COMPlusFrameHandler` が欠けていると、ターゲットのマネージ側で `finally` や `catch` が実行されない、といった症状になります。

### ランタイム → 外部ネイティブコード

OS、CRT、その他 DLL への呼び出しでは注意が必要です。外部コードが例外を発生させ得る場合が問題になります。`EX_TRY` マクロの実装上、非 `Exception` 型の例外を `Exception` に翻訳/ラップしますが、C++ EH の `catch(...)` では「何でも捕まえる」代わりに情報を失います。外部から来る例外では、マクロは実際の型を推測することになり、常に誤ります。

解決策は「コールアウト フィルター」で呼び出しを包むことです。外部例外を捕捉して `SEHException`（ランタイム内部例外の一つ）に翻訳します。フィルターは事前定義されており簡単に使えますが、SEH を使うため同一関数内で C++ EH と併用できません。必要なら関数を分割します。

使用例（置換前／後）:

```cpp
// 置換前
length = SysStringLen(pBSTR);

// 置換後
BOOL OneShot = TRUE;
struct Param {
    BSTR* pBSTR;
    int   length;
};
struct Param param;
param.pBSTR = pBSTR;

PAL_TRY(Param*, pParam, &param)
{
  pParam->length = SysStringLen(pParam->pBSTR);
}
PAL_EXCEPT_FILTER(CallOutFilter, &OneShot)
{
  _ASSERTE(!"CallOutFilter returned EXECUTE_HANDLER.");
}
PAL_ENDTRY;
```

コールアウト フィルターが欠けたまま外部が例外を投げると、ランタイムが報告する例外種別は常に誤ります。状況により決定的でないこともあり、既に飛行中のマネージ例外があればそれが報告され、なければ OOM が報告されます。チェックビルドでは「The runtime may have lost track of the type of an exception」等のアサートが発火します。

## その他（Miscellaneous）

`EX_TRY` には実際には多くのマクロが関わっています。これらのほとんどはマクロ実装外では決して使うべきではありません。


# qi Frameworkの型付け
- - -
このページでは`qi Framework`の内部で定義されている変数の型を`Python`の型などとの互換性という観点で紹介し、いくつかの特殊な型についても紹介しています。


<br/><br/>
## 1. 型の対応表
- - -

**[公式ドキュメント](http://doc.aldebaran.com/libqi/api/transtyping.html)**には`qi Framework`の内部定義型(`qiType`と呼ばれます)と`C++`, `Python`, `Java`などの型に関する対応が記述されています。以下には`qiType`, `Python`, `C#`間における型の対応に加え、厳密に型対応を取るために用意された`Baku.LibqiDotNet`の型を記載しています。

一つだけ公式ドキュメントと異なる注意として、下記の表では最下部に`Raw`型というバイナリデータ型を追記していることに注意してください。`Raw`型は公式の対応表には載っていませんが[シグネチャAPI](http://doc.aldebaran.com/libqi/api/cpp/type/signature.html)などには記載があります。

qi Type  | Python   | Baku.LibqiDotNet         | C#
---------|----------|--------------------------|------
Void     | NoneType | QiValue.Void             | void
Bool     | bool     | QiBool                   | bool
Int8     | int      | QiInt8                   | sbyte
Int16    | int      | QiInt16                  | short
Int32    | int      | QiInt32                  | int
Int64    | int      | QiInt64                  | long
UInt8    | int      | QiUInt8                  | byte
UInt16   | int      | QiUInt16                 | ushort
UInt32   | int      | QiUInt32                 | uint
UInt64   | int      | QiUInt64                 | ulong
Float    | float    | QiFloat                  | float
Double   | float    | QiDouble                 | double
String   | str      | QiString                 | string
List<T>  | list     | QiList&lt;T&gt;          | -
Mat<T,U> | dict     | QiMap&lt;TKey,TValue&gt; | -
Object   | *class instance* | QiObject         | -
Value    | *underlying type*| QiDynamic        | -
Raw      | str              | QiByteData       | byte[]


<br/><br/>
## 2. QiValueとQiAnyValue
- - -

`Baku.LibqiDotNet`では`qi Framework`の機能をラップするために`QiValue`と`QiAnyValue`という二種類のクラスを用いています。

前者の`QiValue`クラスは`qi Framework`のデータに相当する生のポインターを保持するクラスです。サービスをロードして関数を呼び出すと、その結果は全て`QiValue`型の変数にいったん集約されます。つまり、`QiValue`は外部から受信したデータを表すクラスです。

<br/>
`QiValue`型変数が実際に保持する値の型は実行時に確認するか、事前に把握しておく必要があります。特に組み込み型を通常はドキュメンテーションから呼び出す関数の戻り値が分かっているため、その型を指定することで出力値を組み込み型(`int`など)に変換することが可能です。具体例については[**関数の呼び出し**](../tutorials/method.md)を参照してください。

<br/>
後者の`QiAnyValue`クラスは`QiValue`と対照的なクラスです。`QiValue`が関数の戻り値を表すのに対し、`QiAnyValue`は`qi Framework`の関数へ入力可能な値の基底となるクラスです。上記の表で`Baku.LibqiDotNet`の型として記載されているものは`QiValue.Void`静的プロパティと`QiObject`型を除けば全てが`QiAnyValue`型の派生クラスです。

<br/>
`QiAnyValue`の派生型の多くは通常のコンストラクタを用いて作成できます。例えば、次のようなコードが実行可能です。

```csharp
QiAnyValue s = new QiString("Create QiString value from string");

//もちろんこう書くことも出来る
QiString s2 = new QiString("Create QiString value from string");
var s3 = new QiString("Create QiString value from string");
```


<br/><br/>
## 3. 暗黙の型変換
### **3-1. QiAnyValueに関する暗黙型変換 **
- - -
前節で示した通り`QiAnyValue`の派生型は通常のコンストラクタを用いて作成できます。しかし常にこのような書き方をするとコードが煩雑になってしまいます。

例えば、ロボットのカメラから画像を取得する設定を行う`ALVideoDevice`の`subscribeCamera`関数は次のようなシグネチャを持っています。

```
string subscribeCamera(string id, int cameraType, int resolution, int colorspace, int fps);
```

ただ`string`や`int`と書いているのは略記で、正式には`QiString`や`QiInt32`などの`Baku.LibqiDotNet`で定義された型です。したがって、このを正式な方法で呼び出すと次のように書けます。

```csharp
var video = session.GetService("ALVideoDevice");
string idName = (string)video["subscribeCamera"].Call(
    new QiString("Hoge"),
    new QiInt32(0),
    new QiInt32(1),
    new QiInt32(11),
    new QiInt32(10)
    );
```

<br/>
上記のコードはより簡潔に、次のように書くこともできます。

```csharp
var video = session.GetService("ALVideoDevice");
string idName = video["subscribeCamera"].Call("Hoge", 0, 1, 11, 10);
```

<br/>
このように記述できるのは、`Call`関数が`QiAnyValue`の派生型変数を引数としており、さらに`QiAnyValue`クラスが暗黙のキャストによって`string`や`int`を適切に変換するためです。たとえば`string`に対する暗黙のキャストは次のように実装されています。

```csharp
class QiAnyValue
{
    //...

    public static implicit operator QiAnyValue(string x) => new QiString(x);
}
```

<br/>
他の組み込み型等についても同様で、[**型の対応表**](#1)に記載された`bool, string, byte[]`と整数型(`int, ushort`など)、小数型(`float, double`)について暗黙のキャストが可能です。使用頻度が高いと想定される`int[], double[], string[]`についても暗黙のキャストをサポートしています。したがって、見かけ上はこれら組み込み型(と一部の配列)はそのまま引数に代入することが出来ます。逆に、二重リスト(リストのリスト)のような複雑なデータ構造を指定する場合はキャストは利用できないため、`QiAnyValue`の派生型のコンストラクタやファクトリメソッドを用いて正しくインスタンスを生成してください。二重リストを用いる具体例は[**関数の呼び出し**](../tutorials/method.md)で紹介しています。

<br/>
### **3-2. QiValueに関する暗黙の型変換 **
- - -
先ほどの例では組み込み型を含むいくつかの型が`QiAnyValue`型へと暗黙に変換できる事により、関数の呼び出しコードが簡潔に書けることを紹介しました。

もう一つの機能提供として、`QiAnyValue`から派生しているそれぞれの組み込み型に関しても、派生型に対応する型からの暗黙型変換を認めています。たとえば次のようなコードが可能です。

```csharp
QiString s = "Hello, World!";
QiInt32 x = 42;
```

<br/>
上のコードは`string`から`QiString`への暗黙の型変換ができることを示しています。`int`についても同様です。このようなキャストを利用すると、コンテナ型(とくに辞書`QiMap<TKey, TValue>`)の初期化が簡潔に記述できます。

```csharp
QiMap<QiString, QiInt32> d = new Dictionary<QiString, QiInt32>()
{
    ["foo"] = 10,
    ["bar"] = 42
}.ToQiMap();
```

<br/>
上記の`"foo"`や`10`を与えている箇所には本来ディクショナリのキーと値である`QiString`型や`QiInt32`型を指定しなければなりません。カジュアルな記法を嫌う場合こう書く必要があります。

```csharp
QiMap<QiString, QiInt32> d = new Dictionary<QiString, QiInt32>()
{
    [new QiString("foo")] = new QiInt32(10),
    [new QiString("bar")] = new QiInt32(42)
}.ToQiMap();
```

<br/>
暗黙の型変換によって表現が簡素になっていることが分かります。


<br/><br/>
## 4. 動的型(QiDynamic)
- - -
先に忠告しておきますが、これは`C#`の`dynamic`とはまったく違います！`qi Framework`の`QiDynamic`型が提供する機能は`C#`の`dynamic`に比べると非常に単純ですが、その分言語に依存しない便利な機構を提供しています。`QiDynamic`は1要素のリストあるいはタプルとみなすことが出来る、ある種のコンテナクラスです。ただし配列(`QiList<T>`)やタプル(`QiTuple`)と異なり、`QiDynamic`は任意の型のデータを保持できるという特徴があります。以下は実際には動作しない疑似コードですが、`QiDynamic`のコンセプトを示しています。

```
var qid = new QiFramework.QiDynamic(new QiInt32(10));
var x = qid.Value; //xはQiInt32型の変数

b.Value = new QiString("Hello!");
var y = qid.Value; //yはQiString型の変数

b.Value = new QiBool(false);
var z = qid.Value; //zはQiBool型の変数
```

<br/>
上の例では`Value`プロパティに次々と異なる型の変数を代入し、取り出しています。`qid`変数自体の型は`QiDynamic`のまま不変ですが、`Value`へのアクセスによって見かけ上型が動的に変化しているように扱えていることが分かります。

<br/>
`QiDynamic`型は`qi Framework`において、事前に渡される関数のシグネチャが固定しづらい時に利用されています。例えば運動を制御する`ALMotion`では指定した関節の目標となる角度値を指定できる[`setAngles`](http://doc.aldebaran.com/2-1/naoqi/motion/control-joint-api.html#ALMotionProxy::setAngles__AL::ALValueCR.AL::ALValueCR.floatCR)関数がありますが、この関数は内部的に2種類のオーバーロードを持っており、シグネチャがおよそ次のようになっています。

```
void setAngles(QiString name, QiDynamic angles, QiDynamic fraction);
void setAngles(QiList<QiString> names, QiDynamic angles, QiDynamic fraction);
```

<br/>
ドキュメンテーションを読むと分かるのですが、実際には上記の`QiDynamic`部分には単一の`double`値か`double[]`が渡されます。`QiDynamic`を使わないと`setAngles`関数のオーバーロードは2種類では済まず、具体的に言うと(第一引数、第二引数、第三引数がいずれも2種類の型を取るので)8個ほど必要だったことがうかがい知れます。`QiDynamic`を用いた関数定義はオーバーロードの個数爆発を防ぐうえで重要であると言えます。

<br/>
実際に`QiDynamic`型の変数を作成する例をいくつか試しましょう。

第一に、もっとも標準的な方法として、既存の`QiAnyValue`派生型変数に対して`ToDynamic`拡張メソッドを呼び出すと`QiDynamic`でラップされた変数を取得できます。簡単なコードは次のようなものです。

```csharp
QiString s = new QiString("Hello!");
QiDynamic d = s.ToDynamic();

Console.WriteLine(s.Dump());
Console.WriteLien(d.Dump());
```

出力例です。
```
QiString:Hello!
Dynamic
  QiString:Hello!
```

<br/>
もとの文字列が`Dynamic`によって包み込まれたような形になることが分かります。拡張メソッドによる表現以外では下記のコンストラクタ

```csharp
public class QiDynamic : QiAnyValue
{
    public QiDynamic(QiAnyValue v) { /* .. */ }
}
```

<br/>
これを利用する方法も簡潔です。上述した[暗黙の型変換](#3)を用いることで、次のようなコードが記述できます。

```csharp
var d = new QiDynamic(42);
Console.WriteLine(d.Dump());
```

出力です。

```
Dynamic
  QiInt:42
```

`42`は`int`型なので暗黙に`QiInt32`型へと変換され、これが`QiDynamic`によって包み込まれた値を生成します。


<br/><br/>
## 5. QiValue.Void
- - -
`QiValue.Void`は`QiValue`クラスの静的プロパティです。`null`に近いような意味があり、`qi Framework`における関数の戻り値として「何も返さない」というのを表す値です。

通常のプログラムではこの値を意識する必要はありません。例えばロボットに発話させるための`ALTextToSpeech::say`関数は戻り値が無い関数ですが、これを呼び出すコードは次の通りです。

```csharp
var tts = session.GetService("ALTextToSpeech");
tts["say"].Call("Hello.");
```

<br/>
通常は上のように、戻り値が`void`な関数ではそもそも`Call`関数の結果を取得しません。もし戻り値が`Void`な値であることを念入りに確認したい場合、例えば次のようにします。

```csharp
var tts = session.GetService("ALTextToSpeech");
var res = tts["say"].Call("Hello.");
bool isVoid = (res.ContentValueKind == QiValueKind.QiVoid);
```

<br/>
以上が通常のケースにおける処理です。ただし[**サービスを自作する**](../tutorials/create_service.md)際には戻り値が無い関数を定義したい場合があり、`QiValue.Void`はこのような状況で用います。

```csharp
var objBuilder = QiObjectBuilder.Create();
objBuilder.AdvertiseMethod("test::v()", (sig, arg) =>
{
    Console.WriteLine("test was called");
    return QiValue.Void;
});
```

<br/>
なお、このようなパターンは比較的よく採用されるパターンで、.NETアプリケーションの例で言うとデータバインディング時に「何もしない」というアクションを示すための[`Binding.Nothing`](https://msdn.microsoft.com/ja-jp/library/system.windows.data.binding.donothing%28v=vs.110%29.aspx)フィールドなどが近い性質を持っています。


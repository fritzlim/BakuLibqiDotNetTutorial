# コンテナ型(List, Map, Tuple)
- - -
`qi Framework`でサポートしている3種類のコンテナ型について、

- 関数の引数として渡す際に用いる`QiAnyValue`型変数の作成法
- 関数の戻り値として受け取った`QiValue`からの適切なデータ取得法

をそれぞれ紹介します。

<br/>
3種類のコンテナ型のうち、配列(`List`)はユーザプログラマがもっとも頻繁に用いるコンテナ型です。辞書(`Map)`が使われるケースも稀ですが存在します。タプルを表現する`QiTuple`は[サービスを自作](../tutorials/create_service.md)する際などに利用します。


<br/>
## 1. 配列(List)
- - -
配列型のデータ構造は最も頻繁に利用されるコンテナ型です。角度値やテキストの受け渡しに用いられるほか、(おそらくAldebaranのライブラリ整備手順など歴史的経緯と思われる理由から)辞書型データを利用すべき箇所でも一部のデータは配列によって表現されています。

<br/>
### **1-1.QiAnyValue型の派生型としての配列**
- - -
`QiAnyValue`から派生した配列型のクラスはジェネリッククラス`QiList<T>`です。`T`は`QiAnyValue`の派生型である必要があります。`Baku.LibqiDotNet`には非ジェネリックな`QiList`というクラスも存在しますが、これは静的クラスで、単に`QiList<T>`を作成するファクトリメソッドを定義しただけのクラスです。

<br/>
`QiList<T>`を作成するには`QiList.Create`関数を用いるのが最も簡単です。入れ子リストを作るような複雑な例については[**関数の呼び出し**](../tutorials/method.md)に記載していますが、シンプルな1次元配列であればもっと簡単に、`QiList.Create`関数を用いて作成できます。

```csharp
var arr = QiList.Create(new[] { "Hello", "This is", "String Array" });
//arr: QiList<QiString>
```

<br/>
`QiList.Create`関数では組み込み型の`IEnumerable`から適切な`QiList`を作成するものを含めてオーバーロードを多数定義しており、通常はこれだけで1次元の`QiList<T>`が作成可能です。

<br/>
異なるアプローチとして、`QiAnyValue`の派生型の`IEnumerable<T>`(`T`は`QiAnyValue`の派生型)に対し拡張メソッド`ToQiList`関数を用いることでも`QiList<T>`が得られます。

```csharp
var numbers = new QiInt32[] 
{
    new QiInt32(3), new QiInt32(4), new QiInt32(5)
}.ToQiList();
```

`QiInt32`は`int`からの暗黙キャストをサポートしているため、上記のコードを次のようにも書けます。
```csharp
var numbers = new QiInt32[] 
{
    3, 4, 5
}.ToQiList();
```

<br/>
### **1-2. QiValueが保持するデータとしての配列**
- - -
関数の戻り値として`QiValue`型変数を受け取り、その内容が配列であることをあらかじめ知っているとします。このとき`QiValue`から実際にデータを引き出す方法はいくつかありますが、`QiValue`はプリミティブ配列への明示キャストをサポートしているため、`int, double, string`のいずれかの1次元配列については.NETの配列として容易に変換できます。

```csharp
var motion = session.GetService("ALMotion");
double[] angles = (double[])almotion["getAngles"].Call("Body");
```

<br/>
`QiValue`がキャストをサポートしていない複雑なデータを戻り値として受け取った場合は、よりプリミティブな処理が要求されます。例えば小数の2次元配列について内容を走査するコードは次のようになります。

```csharp
QiValue response = someService["someMethod"].Call();

for (int i = 0; i < response.Count; i++)
{
    QiValue row = response[i];
    for (int j = 0; j < row.Count; j++) 
    {
        QiValue elem = row[j];
        double value = (double)elem;
        //valueを使った処理
    }
}
```

<br/>
`QiValue.Count`プロパティは`QiValue`が実際に保持するデータがコンテナ型である場合にデータの個数を取得するために利用できます。配列やタプルでは`Count`の値を考慮したうえで整数値のインデックス指定により、個々の要素へのアクセスが可能です。この操作で得られるデータもまた`QiValue`型です。上記の例では`response`変数の各要素`row`が1次元配列であると仮定し、さらに内部のデータを取得しています。

なお、上の例に関してはより簡潔に書きたければ次のような記述も可能です。

```csharp
QiValue response = someService["someMethod"].Call();

double[][] manyNumbers = Enumerable
    .Range(0, response.Count)
    .Select(i => (double[])response[i])
    .ToArray();

//manyNumbersを用いた処理
```

<br/>
NOTE: NuGetパッケージ2.0.0相当のライブラリには`QiList.Create`によく似た名前の`QiList.CreateFrom`という関数もありますが、これは整備されていない関数なので使わないでください！次、あるいはその次のバージョンでこの関数は廃止される予定です。


<br/><br/>
## **2. 辞書(Map)**
- - - 
辞書(`Map`)はキーと値のペアを保持するデータ構造です。`qi Framework`で辞書を利用する場面は限られていますが、一部の重要な処理で利用されているため、動作を理解しておくのは非常に有意義です。


<br/>
### **2-1. QiAnyValue型の派生型としての辞書**
- - -
辞書のクラスはジェネリッククラス`QiMap<TKey, TValue>`として定義されています。`TKey`も`TValue`も`QiAnyValue`の派生型である必要があります。配列のときと同様、ファクトリメソッドを定義した静的クラスとしてジェネリックでない`QiMap`も用意されています。


`QiMap<TKey, TValue>`は(`QiList<T>`より型の組み合わせが段違いに多いため)組み込み型からの生成をサポートしていませんが、拡張メソッドや暗黙キャストを組み合わせると最小限の煩雑さでインスタンスが生成できます。

```csharp
QiMap<QiString, QiInt32> d = new Dictionary<QiString, QiInt32>()
{
    ["foo"] = 10,
    ["bar"] = 42,
}.ToQiMap();
```

<br/>
`ToQiMap`は`IEnumerable<KeyValuePair<TKey, TValue>>`に対して動作する拡張メソッドとして定義されており、上記のように`Dictionary`からの変換がサポートされています。

辞書に含まれるキー/値ペアの要素が少ない場合、可変長引数を取る`QiMap.Create`関数を使った方法も比較的簡潔です。以下のように、キー値ペアを指定した分だけ受け入れて辞書を生成することが出来ます。

```csharp
QiMap<QiString, QiInt32> d = QiMap.Create(
    new KeyValuePair<QiString, QiInt32>("foo", 10),
    new KeyValuePair<QiString, QiInt32>("bar", 42)
    );
```

<br/>
### **2-2. QiValueが保持するデータとしての辞書型**
- - -
`QiValue`型変数として受け取った辞書データから値を取り出すにはいくつかの決まった段階を経る必要があります。第一に`QiValue.GetKeys()`関数でキー一覧の配列を保持した`QiValue`を取得します。第二に、得られた各キーで実際に辞書式のアクセスを行うことで対応する値が取得できます。キーや値はいずれも`QiValue`であるため、これらを.NETの値として扱うにはキャストを行います。

以下の例では`QiMap<QiString, QiInt32>`を生成してからそれを`QiValue`に変換し、ふたたび`QiValue`から内容のデータを取り出すサンプルを示しています。

```csharp
var d = new Dictionary<QiString, QiInt32>()
{
    ["foo"] = 10,
    ["bar"] = 42
}.ToQiMap();

var dQiValue = d.QiValue;

var keysQiValue = dQiValue.GetKeys();

var keys = Enumerable.Range(0, keysQiValue.Count)
    .Select(i => (string)keysQiValue[i])
    .ToArray();

var values = Enumerable.Range(0, keysQiValue.Count)
    .Select(i => (int)dQiValue[keysQiValue[i]])
    .ToArray();

foreach(var k in keys)
{
    Console.WriteLine(k);
}

foreach(var i in values)
{
    Console.WriteLine(i);
}
```

出力例です。
```
bar
foo
42
10
```

<br/><br/>
NOTE: 上記のようなキー/値の一覧取得をおこなうコードは一般的であり、本来ならば`Baku.LibqiDotNet`側でサポートすべき機能です。次のバージョンでは`QiValue`に`MapKeys`, `MapValues`, `MapItems`というプロパティを追加して上記の操作を簡略化できるようにします([GitHub](https://github.com/malaybaku/BakuLibQiDotNet)のdeveloper branchでは既にこの機能を公開しています)。

```csharp
var d = new Dictionary<QiString, QiInt32>()
{
    ["foo"] = 10,
    ["bar"] = 42,
}.ToQiMap();

var dQiValue = d.QiValue;

//MapItemsプロパティはIEnumerable<KeyValuePair<QiValue, QiValue>>型なので
//foreach文を回すとDictionaryへのアクセスと同じようにで扱える
foreach (var pair in dQiValue.MapItems)
{
    string key = (string)pair.Key;
    int value = (int)pair.Value;
    Console.WriteLine($"{key} : {value}");
}

//キー一覧を出力
dQiValue.MapKeys
    .Select(k => (string)k)
    .ToList()
    .ForEach(Console.WriteLine);

//値の一覧を出力
dQiValue.MapValues
    .Select(v => (int)v)
    .ToList()
    .ForEach(Console.WriteLine);


//これはダメ: OfTypeを通過しない
dQiValue.MapKeys
    .OfType<string>()
    .ToList()
    .ForEach(Console.WriteLine);
```



<br/><br/>
## 3. タプル(Tuple)
- - -
タプルは複数のデータをひとまとめにする汎用的な方法を提供するデータ構造です。配列と類似していますが、要素ごとに型が異なっても構わないという重要な特徴を備えています。

<br/>
### **3-1. QiAnyValue型の派生型としてのタプル**
- - -
タプル型のクラスは`QiTuple`です。上述の配列型や辞書型と異なりタプル型は非ジェネリッククラスですが、性質上はジェネリッククラスに近い特徴を持っています。

タプルのインスタンスは`QiTuple.Create`関数によって生成できます。規約としてタプルを不変値(immutable)にすることは必須ではありませんが、誤操作を防ぐためには一度生成したタプルの要素を書き換えようとしない方が賢明でしょう。

```csharp
var tuple = QiTuple.Create(
    "Hello",
    10,
    true
    );
Console.WriteLine(tuple.Dump());
```

出力例です。

```
Tuple[3]:
  Dynamic
    Dynamic
      QiString:Hello
  Dynamic
    Dynamic
      QiInt:10
  Dynamic
    Dynamic
      QiInt:1
```

[`Dynamic`](./type_rules#4)型でラップされていますが、データの内容が順に文字列`"Hello"`と整数`10`、そして整数化しているものの`true`に対応する整数値である`1`が順に格納されていることが分かります。

<br/>
NOTE: 実は上記のようにタプルのデータが`Dynamic`でラップされる仕様は実装ミスで、本来は次のような出力を得られます([GitHub](https://github.com/malaybaku/BakuLibQiDotNet)のdeveloper branchコードでは修正が行われており、実際に下記の出力を返します)。

```
Tuple[3]:
  QiString:Hello
  QiInt:10
  QiInt:1
```

ただし`Dynamic`によるラップは有害になりづらいため、通常はあまり気にする必要はありません。

<br/>
### **3-2. QiValueが保持するデータとしてのタプル型**
- - -
`QiValue`として受け取ったタプルの性質は`QiList<T>`とほとんど変わりません。値の一覧を取得する場合`Count`プロパティと整数値インデックス指定による値の取得を組み合わせた処理を記述することになります。

`QiValue`としてタプルを受け取るもっとも典型的なケースは[サービス自作](../tutorials/create_service.md)の際に必要となる、`QiObjectBuilder.AdvertiseMethod`関数の引数でのメソッド定義です。

```csharp
var objBuilder = QiObjectBuilder.Create();
objBuilder.AdvertiseMethod("twice::i(i)", (sig, arg) =>
{
    Console.WriteLine("twice was called");
    int i = (int)arg[0];
    return new QiInt32(i * 2).QiValue;
});
```

<br/>
上記の例で`arg`として渡されるのがタプルを保持している`QiValue`です。上の例では関数の引数として「単一の整数」を指定している(関数シグネチャの読み方については[サービス自作](../tutorials/create_service.md)を参照)ため、`arg`は常に1要素のタプルであり、その要素は整数であることが保障されます。この例のようにタプルの内容は事前知識に基づいて決定できるため、タプルの内容を適切に処理することはそう難しくありません。




# How to Call Method

このページでは、`Baku.LibqiDotNet`において`qi Framework`のサービス上の関数を呼び出す一般的な手順を、実際のロボットに適用できるサンプルを用いて紹介します。

<br/>

本サンプルではロボットのモーションに関するサービスである[`ALMotion`](http://doc.aldebaran.com/2-1/naoqi/motion/almotion.html)を用いて、現在のロボットの状態を取得したり、逆に特定のポーズを指示するプログラムを作成します。本チュートリアルは実機のロボットだけでなく仮想ロボットでも検証可能です。

<br/><br/>
## 1. コンソールアプリケーションの準備
- - -

[Hello World](./hello_world.md)の手順に沿ってコンソールアプリケーションを準備します。

<br/><br/>
## 2. もっともシンプルな呼び出し: wakeUpとrest
- - -

本チュートリアルでは少しずつコードを書き足しながら動作を確認していきます。まず、本チュートリアルで利用する`ALMotion`モジュールをロードし、ロボットのアクチュエータを起動し、また停止する処理を行ってみましょう。このような処理は[`wakeUp`](http://doc.aldebaran.com/2-1/naoqi/motion/control-stiffness-api.html#ALMotionProxy::wakeUp)関数と[`rest`](http://doc.aldebaran.com/2-1/naoqi/motion/control-stiffness-api.html#ALMotionProxy::rest)関数を順に呼び出すことで実装できます。

<br/>
以下では`Program.cs`のみを含むコンソールアプリケーションを想定し、簡単のため`using`文と`Main`関数のみを記述します。また接続先となるロボットのアドレスは適切に指定してください。

```csharp
using System;
using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;


static void Main(string[] args)
{
    PathModifier.AddEnvironmentPath(
        "dlls", PathModifyMode.RelativeToEntryAssembly);

    //HelloWorldの対象とするマシンのアドレスをIPとポート(ポートは通常9559)で指定
    string address = "tcp://xxx.xxx.xxx.xxx:9559";
    var session = QiSession.Create(address);

    var motion = session.GetService("ALMotion");

    motion["wakeUp"].Call();
    motion["rest"].Call();
}
```


<br/>
このプログラムを動作させると、ロボットのアクチュエータが起動していない場合は起動を行い、その後再びアクチュエータを休止させます。


<br/><br/>
## 3. 関数の結果を受け取る
- - -

上の例では戻り値がない関数を呼び出しました。次は戻り値がある関数を利用してみましょう。たとえば`ALMotion`モジュールでは、現在のロボットの姿勢情報を単一の要約文テキストとして取得する[`getSummary`](http://doc.aldebaran.com/2-1/naoqi/motion/tools-general-api.html?highlight=getsummary#ALMotionProxy::getSummary)関数があります。

```csharp
string getSummary();
```

<br/>
この関数を利用するコードは次のようになります。

```csharp
using System;

using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;


static void Main(string[] args)
{
    PathModifier.AddEnvironmentPath(
        "dlls", PathModifyMode.RelativeToEntryAssembly);

    string address = "tcp://xxx.xxx.xxx.xxx:9559";
    var session = QiSession.Create(address);

    var motion = session.GetService("ALMotion");

    string summary = (string)motion["getSummary"].Call();

    Console.WriteLine(summary);
}
```

<br/>
出力例は次の通りです(最上部に警告を示す`[W]`から始まる出力が出ていますが無視して構いません)。

```
[W] 1457718290.352819 14000 qi.path.sdklayout: No Application was created, trying to deduce paths
---------------------- Model ---------------------------
        BodyName   Stiffness     Command      Sensor
         HeadYaw    0.000000   -0.000000   -0.000000
       HeadPitch    0.000000    0.637000    0.637000
  LShoulderPitch    0.000000    1.133252    1.133252
   LShoulderRoll    0.000000    0.059364    0.059364
       LElbowYaw    0.000000   -0.491643   -0.491643
      LElbowRoll    0.000000   -0.008727   -0.008727
       LWristYaw    0.000000   -0.801464   -0.801464
         HipRoll    0.000000    0.000000    0.000000
        HipPitch    0.000000   -1.038400   -1.038400
       KneePitch    0.000000    0.510000    0.510000
  RShoulderPitch    0.000000    1.133252    1.133252
   RShoulderRoll    0.000000   -0.059364   -0.059364
       RElbowYaw    0.000000    0.491643    0.491643
      RElbowRoll    0.000000    0.008727    0.008727
       RWristYaw    0.000000    0.801464    0.801464
           LHand    0.000000    0.600000    0.600000
           RHand    0.000000    0.600000    0.600000
         WheelFL    0.000000    0.000000    0.000000
         WheelFR    0.000000    0.000000    0.000000
          WheelB    0.000000    0.000000    0.000000
---------------------- Tasks  --------------------------
            Name         ID
----------------- Motion Cycle Time --------------------
              20 ms
```

<br/>
改行を含むサマリー文字列が正しく取得出来ています。もう一度関数呼び出しの部分を再掲しますが、ここでは呼び出し結果に対してキャストを行っています。

```csharp
string summary = (string)motion["getSummary"].Call();
```

<br/>
関数の呼び出し結果は`QiValue`型の変数です。この変数は`qi Framework`から渡される全ての変数の派生元である(ように見える)クラスです。本質的にこの変数が保持するデータは整数値、文字列やリスト構造など`C#`でもお馴染みのデータですが、とくに組み込み型を含むいくつかの型への変換は頻繁に必要となるため、上述のようなキャスト演算によって値を取得できるようになっています。


<br/><br/>
## 4. キャストを用いず複雑な戻り値を得る: getAngles関数
- - -

前節の例では`string`型の戻り値を持つ関数をキャスト操作によって取得出来る事を紹介しました。以下の型の変数は`QiValue`からのキャストによって得られます。

- 論理値(`bool`)
- 整数(`int, long, short, sbyte, uint, ulong, ushort, byte`)
- 小数(`float, double`)
- 文字列(`string`)
- 配列(`int[], double[], string[]`)
- バイナリ(`byte[]`)

<br/>

しかし上記のキャストが出来ないような、複雑な戻り値を持った関数を呼び出した場合はどうすればよいのでしょうか？例を示すため、ここではキャスト可能な戻り値を**あえてキャスト以外の方法で**パースしてみます。下記の例では[`getAngles`](http://doc.aldebaran.com/2-1/naoqi/motion/control-joint-api.html#ALMotionProxy::getAngles__AL::ALValueCR.bCR)関数を用い、ロボットの各関節の角度を小数の配列として取得しています。

```csharp
using System;

using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;



static void Main(string[] args)
{
    PathModifier.AddEnvironmentPath("dlls", PathModifyMode.RelativeToEntryAssembly);

    string address = "tcp://xxx.xxx.xxx.xxx:9559";
    var session = QiSession.Create(address);

    var motion = session.GetService("ALMotion");

    //キャストを用いた方法
    double[] angles = (double[])motion["getAngles"].Call("Body", true);

    //キャストを用いない方法
    QiValue result = motion["getAngles"].Call("Body", true);
    var angles2 = new double[result.Count];
    for (int i = 0; i < angles2.Length; i++)
    {
        angles2[i] = (double)result[i];
    }

    //同じ値が取得できているかチェックしてみよう！
    for (int i = 0;i < angles.Length; i++)
    {
        Console.WriteLine(angles[i]);
        Console.WriteLine(angles2[i]);
    }

}
```

<br/>
キャストを用いない方法では`QiValue.Count`プロパティにアクセスすることで配列の長さを取得し、インデックスを指定することで配列の各要素を取り出しています。`Count`プロパティは`QiValue`の実際の内容が配列などである場合は要素数を表し、それ以外の場合`Count`の値は単に`0`になります。


私がこの例を試した所、次のような出力が得られました。

```
[W] 1457719018.860709 11624 qi.path.sdklayout: No Application was created, trying to deduce paths
-5.6843418860808E-14
-5.6843418860808E-14
0.637000024318695
0.637000024318695
1.13325190544128
1.13325190544128
0.0593636594712734
0.0593636594712734
-0.491642862558365
-0.491642862558365
-0.00872664619237185
-0.00872664619237185
-0.801463842391968
-0.801463842391968
0.600000023841858
0.600000023841858
0
0
-1.03840005397797
-1.03840005397797
0.509999990463257
0.509999990463257
1.13325190544128
1.13325190544128
-0.0593636594712734
-0.0593636594712734
0.491642862558365
0.491642862558365
0.00872664619237185
0.00872664619237185
0.801463842391968
0.801463842391968
0.600000023841858
0.600000023841858
0
0
0
0
0
0
```

どちらの方法でも同じ結果が得られている事が分かります。今回の例では`QiValue`をわざわざ操作する意義があまり感じられなかったと思いますが、配列の要素ごとに異なる型のデータが入ったようなデータを受け取る場合`QiValue`の直接操作が必要となります。この具体例に関してはGitHubのソースコードに[カメラ画像を取得する例](https://github.com/malaybaku/BakuLibQiDotNet/blob/master/Baku.LibqiDotNet/StandardSamples/VisionRetrieveImage.cs)が含まれているので参考にしてください。



<br/><br/>
## 5. 複雑な引数を渡す: angleInterpolation関数
- - - 

前節では関数の**戻り値**が複雑であるケースのための処理を紹介しました。今度は関数の**引数**が複雑な場合に対応することを考えてみましょう。今回は具体例として、ロボットの動きを自動で補間する[`angleInterpolation`](http://doc.aldebaran.com/2-1/naoqi/motion/control-joint-api.html#ALMotionProxy::angleInterpolation__AL::ALValueCR.AL::ALValueCR.AL::ALValueCR.bCR)関数のサンプルを`C#`で動かしてみましょう。

この関数の仕様はドキュメンテーションに記載されていますが、それより大切なのは、この関数を使う場合に入れ子配列のデータ構造が必要ということです。たとえば`Python`のサンプルコードを抜粋すると次のようなプログラムが書かれています。

```python
names  = ["Head"]
angleLists  = [[50.0*almath.TO_RAD, 0.0],
               [-30.0*almath.TO_RAD, 30.0*almath.TO_RAD, 0.0]]
timeLists   = [[1.0, 2.0], [ 1.0, 2.0, 3.0]]

motionProxy.angleInterpolationBezier(names, timeLists, angleLists)
```

<br/>
`almath.TO_RAD`は度数法をラジアン法に変換している定数なので大した意味はありません。ポイントになるのは`angleLists`や`timeLists`がリストのリストであるという点です。この処理を`C#`で行う場合、コードは次のようになります。

```csharp
using System;

using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;



static void Main(string[] args)
{
    PathModifier.AddEnvironmentPath(
        "dlls", PathModifyMode.RelativeToEntryAssembly);

    string address = "tcp://xxx.xxx.xxx.xxx:9559";
    var session = QiSession.Create(address);

    var motion = session.GetService("ALMotion");

    double toRad = Math.PI / 180.0;

    QiList<QiString> names 
        = QiList.Create(new string[] { "HeadYaw", "HeadPitch" });

    QiList<QiList<QiDouble>> angleLists
        = QiList.Create<QiList<QiDouble>>(new QiList<QiDouble>[]
    {
        QiList.Create(new double[] { 30.0 * toRad, 0.0 }),
        QiList.Create(new double[] { -30.0 * toRad, 30.0 * toRad, 0.0 })
    });

    QiList<QiList<QiDouble>> timeLists
        = QiList.Create<QiList<QiDouble>>(new QiList<QiDouble>[]
    {
        QiList.Create(new double[] { 1.0, 2.0 }),
        QiList.Create(new double[] { 1.0, 2.0, 3.0 })
    });

    QiBool isAbsolute = new QiBool(true);

    motion["angleInterpolation"].Call(names, angleLists, timeLists, isAbsolute);
}
```

<br/>
これを実行するとロボットの首が左上、下、正面を順に向きます。

<br/>

上記のプログラムでは今まで現れなかったクラス名がいくつか現れています。とはいえ`QiList, QiString, QiDouble, QiBool`などの型名からこれらが何を意味するのかは想像がつくでしょうし、経験のあるプログラマであれば`QiList<QiList<QiDouble>>`が小数の二次元配列データを表すこともすぐ納得できるでしょう(※上のコードでは`var`キーワードの使用を意図的に控えて冗長な表現を行っていますが、これは型が分かりやすいように示しているだけです。リファクタリングされたコードは後述します)。`QiList<T>`型の変数を生成しているのは`QiList.Create`関数です。この関数は実際のユースケースを想定して大量にオーバーロードが定義されていますが、これについてはVisual Studio上でインテリセンスとにらみ合って確認すれば済む話題ですから詳しくは触れません。

<br/>

それよりも問題なのは`Call`関数の仕組みです。せっかくなので、今まで何となく使っていた`Call`関数の正式なシグネチャを確認しましょう。シグネチャは下記の通りです。

```csharp
class QiMethod
{
    public QiValue Call(params QiAnyValue[] args) { /* 実装 */ }
}
```

<br/>
`Call`関数は可変個の`QiAnyValue`型変数を引数に取ります。`QiAnyValue`クラスは`QiValue`クラスと類似したクラスですが、`QiValue`が関数の戻り値の基底を表していたのとは対照的に、`QiAnyValue`クラスは関数への入力に用いる値の基底を表します。

<br/>
ご想像の通り、上記のコードで使っている`QiString, QiBool, QiDouble`などは全て`QiAnyValue`の派生型です。`QiList<T>`(`T`は`QiAnyValue`の派生型)も`QiAnyValue`から派生しています。以上から、適切なデータを関数の引数に渡す手順は

1. `Qi**`という`QiAnyValue`から派生した型の変数を作成
2. 適切な関数名を指定し、`Call`関数に適切な順番で引数を渡す

<br/>
というシンプルなものであることが分かりました。一件落着です


<br/><br/>
…いや、ちょっと待ってください。ひとつ引っかかる事があります。[Hello World](./hello_world.md)プログラムを思い出して欲しいのですが、このプログラムでは次のようなコードを書いていました。

```csharp
//最も基本的なモジュールの一つとして合成音声のモジュールを取得
var tts = session.GetService("ALTextToSpeech");

//"say"関数に文字列引数を指定して実行
tts["say"].Call("this is test");
```

<br/>
`QiAnyValue`の派生型ではなく単なる文字列を渡せています。なぜでしょうか？

<br/>
これは仕掛けが分かってしまえば簡単な話なのですが、`QiAnyValue`クラスはいくつかの暗黙型キャストが[`implicit operator`](https://msdn.microsoft.com/ja-jp/library/85w54y0a.aspx)で定義されています。例をいくつか紹介します。

```csharp
public abstract class QiAnyValue
{
    public static operator QiAnyValue(bool v) => new QiBool(v);
    public static operator QiAnyValue(int v) => new QiInt32(v);
    public static operator QiAnyValue(string v) => new QiString(v);
}
```

<br/>
このような定義があるため、下記の呼び出し

```csharp
tts["say"].Call("this is test");
```

<br/>
これは実際には次のコードと同様に扱われます。

```csharp
tts["say"].Call(new QiString("this is test"));
```

<br/>

暗黙の型変換がサポートされている型は論理値、整数型、小数型とバイナリデータを表す`byte[]`、また利用頻度がそれなりに高いと思われる`int[], string[], double[]`です。これを踏まえたうえで先程の`angleInterpolation`を用いたコードを書き直してみると、より簡潔にまとめる事が出来ます。

```csharp

PathModifier.AddEnvironmentPath(
    "dlls", PathModifyMode.RelativeToEntryAssembly);

string address = "tcp://xxx.xxx.xxx.xxx:9559";
var session = QiSession.Create(address);

var motion = session.GetService("ALMotion");

double toRad = Math.PI / 180.0;

var names = new string[] { "HeadYaw", "HeadPitch" };

var angleLists = QiList.Create(new []
{
    QiList.Create(new double[] { 30.0 * toRad, 0.0 }),
    QiList.Create(new double[] { -30.0 * toRad, 30.0 * toRad, 0.0 })
});

var timeLists = QiList.Create(new []
{
    QiList.Create(new double[] { 1.0, 2.0 }),
    QiList.Create(new double[] { 1.0, 2.0, 3.0 })
});

motion["angleInterpolation"].Call(names, angleLists, timeLists, true);
```

<br/>
`true`が`new QiBool(true)`へ変換され、`names`も`QiList<QiString>`に適切に変換されます。入れ子配列の難しいデータ構造以外では組み込み型をそのまま渡せるので、簡潔なコードを書く場合は意識的にこの機能を活用してください。



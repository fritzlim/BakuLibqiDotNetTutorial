
# Create Service
- - - 

ここでは`Baku.LibqiDotNet`を用いて自作のサービスを定義し、ロボット上に登録する手順を紹介します。この機能を使う機会はそこまで多くありませんが、以下のような状況で必須となります。

- マシンリソースを要求するタスクをロボットからPCに委譲したい
- Windowsとの連携に特化したデバイス(Kinectなど)をロボットとシームレスに接続したい
- 一部の既存サービス(`ALAudioDevice`など)の機能をフルに活用したい

<br/>
本ページでもコンソールアプリケーションを作成します。アプリケーションの作成方法については[**Hello World**](./hello_world.md)を参照してください。

<br/><br/>
## 1. サービスの作成と呼び出し
- - -


<br/>
`qi Framework`に登録可能なサービスを作成するには`QiObjectBuilder`というクラスを用います。以下のサンプルはネットワークを必要としません。プロセス内でサービスを定義し、自分で利用する例となっています。

```csharp
using System;

using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;



static void Main(string[] args)
{
    PathModifier.AddEnvironmentPath(
        "dlls", PathModifyMode.RelativeToEntryAssembly);

    var objBuilder = QiObjectBuilder.Create();
    objBuilder.AdvertiseMethod("test::v()", (sig, arg) =>
    {
        Console.WriteLine("test was called");
        return QiValue.Void;
    });
    objBuilder.AdvertiseMethod("twice::i(i)", (sig, arg) =>
    {
        Console.WriteLine("twice was called");
        int i = (int)arg[0];
        return new QiInt32(i * 2).QiValue;
    });

    var obj = objBuilder.BuildObject();
    obj["test"].Call();

    int res = (int)obj["twice"].Call(21);
    Console.WriteLine($"Res = {res}");
}
```

<br/>
出力例です。

```
Complete sig:test::v()
test was called
Complete sig:twice::i(i)
twice was called
Res = 42
```

<br/>
この例では`"test"`および`"twice"`というメソッドを持つサービスを作成しています。まずは単純な例として`"test"`メソッドを定義するコードを見てみましょう。

```csharp
var objBuilder = QiObjectBuilder.Create();
objBuilder.AdvertiseMethod("test::v()", (sig, arg) =>
{
    Console.WriteLine("test was called");
    return QiValue.Void;
});
```

<br/>
`QiObjectBuilder.Create`関数でインスタンスを取得し、`AdvertiseMethod`関数を呼び出すことで「このサービスにはこのメソッドがあり、実装はこうなっています」という情報を登録します。

<br/>

関数名に続けて記載された`"::v()"`というのは関数シグネチャを表しています。`"::"`は単にメソッド名とシグネチャを区切る文字列です。これに続く`"v"`という文字列は`void`を意味し、関数の戻り値がないことを示しています。これに続く`()`は関数の引数が`0`個であることを示しています。つまり`test`は引数を取らず戻り値も無い、単に呼び出すだけの関数であることが宣言されています。

<br/>
シグネチャについてもう一つの例を見てみましょう。`twice`関数は`"twice::i(i)"`という形式で定義されています。`"::i(i)"`というシグネチャ表記には`"i"`という文字が2度現れますが、`"i"`は`int`を表しています。`"i(i)"`のうち左側(カッコの外側)にある`"i"`は関数の戻り値が整数であることを示し、`"(i)"`は関数が単一の整数値を引数に取ることを示しています。

<br/>
上記で紹介した以外の型を指定したい場合、[**qi Frameworkドキュメント**](http://doc.aldebaran.com/libqi/api/cpp/type/signature.html)に対応表があるので参考にしてください。


<br/><br/>
## 2. サービスの登録/登録されたサービスの呼び出し
- - -
前節の例ではサービスをローカルなプロセスで作成できることを確認しました。これを実際にリモートのロボットに登録し、公開するには`QiSession.RegisterService`関数を用います。プログラムの例は次の通りです。

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

    session.Listen("tcp://0.0.0.0:0").Wait();
    Console.WriteLine("Listen preparation completed");

    var objBuilder = QiObjectBuilder.Create();
    objBuilder.AdvertiseMethod("test::v()", (sig, arg) =>
    {
        Console.WriteLine("test was called");
        return QiValue.Void;
    });
    objBuilder.AdvertiseMethod("twice::i(i)", (sig, arg) =>
    {
        Console.WriteLine("twice was called");
        int i = (int)arg[0];
        return new QiInt32(i * 2).QiValue;
    });

    var obj = objBuilder.BuildObject();

    //ローカルでの呼び出しはしない
    //obj["test"].Call();
    //int res = (int)obj["twice"].Call(21);
    //Console.WriteLine($"Res = {res}");

    uint id = (uint)session.RegisterService(
        "MyService", 
        objBuilder.BuildObject()
        )
        .GetUInt64(0L);

    //リモートに登録したものをロードして使う
    var myService = session.GetService("MyService");
    myService["test"].Call();
    int res = (int)myService["twice"].Call(21);
    Console.WriteLine($"Res = {res}");

    session.UnregisterService(id);
}
```

<br/>
実行例です。

```
[I] 1458052222.196998 11724 qimessaging.session: Session listener created on tcp://0.0.0.0:0
[I] 1458052222.202500 11724 qimessaging.transportserver: TransportServer will listen on: tcp://192.168.1.3:63893
[W] 1458052222.202500 3664 qi.path.sdklayout: No Application was created, trying to deduce paths
[I] 1458052222.204500 11724 qimessaging.transportserver: TransportServer will listen on: tcp://192.168.56.1:63893
[I] 1458052222.208564 11724 qimessaging.transportserver: TransportServer will listen on: tcp://127.0.0.1:63893
Listen preparation completed
Complete sig:test::v()
test was called
Complete sig:twice::i(i)
twice was called
Res = 42
```

<br/>
`[I]`や`[W]`で始まっているのはそれぞれ`libqi`ライブラリから送出されているインフォメーションログおよび警告ログです。特に気にする必要はありませんが、しっかり確認するとライブラリの動作が細かく見られます。


<br/>
その下の出力を見ていくと、登録したサービスを取得し、定義した関数を呼び出せていることが分かります。自作サービスを登録するプログラムの注意点として、`RegisterService`関数を呼び出す前に`QiSession.Listen`関数を使い、外部からのサービス呼び出しを許可する必要があります。

```csharp
session.Listen("tcp://0.0.0.0:0").Wait();
Console.WriteLine("Listen preparation completed");
```

<br/>
明示的にサービスを登録解除する場合は`Unregister`関数を呼び出します。

```csharp
session.UnregisterService(id);
```

<br/>
なお、上記のプログラムではサービスの作成プロセスと呼び出しプロセスが同じであるため、リモート動作の確認という意味では釈然としないように感じる所があったかもしれません。よりしっかりとしたデバッグを行うには次のような構成を取るとよいでしょう。

1. 上記のプログラムとほぼ同じ処理を行い、`UnregisterService`を呼び出す前に`Console.ReadLine`などでプログラムを待機状態に入らせる
2. 別のプログラム(`C#, Python`あるいは`Choregraphe`による方法でも構いません)から登録したサービスを呼び出す

例えば`Python`のNaoqi SDKが入っているPCであれば、次のようなコードで上記のサービスを利用できるはずです。

```python
# -*- coding: utf-8 -*-

import qi

s = qi.Session()
#アドレス指定
s.connect("xxx.xxx.xxx.xxx")

myservice = s.service("MyService")
#呼び出しにより、C#コンソール側に出力が出る
myservice.test()

x = myservice.twice(21)
print(x)
#>>> 42

```

<br/>
最後に蛇足ですが、.NETならではの実用的な機能を提供したい場合、`ALMemory`との連動や`Kinect/Hololens`など`Windows`系デバイスとの組み合わせ、`Unity`でのデバイス連携によるサービス登録などは有力な候補となり得ます。選択肢はけっこう広いので、アイデアを膨らませていろいろ試してみてください。


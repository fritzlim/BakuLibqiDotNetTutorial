# 動的型リストを使用する

ここでは`qi Framework`のデータ構造のうち配列(`QiList<T>`)と動的型(`QiDynamic`)を組み合わせてデータを渡す例を、簡単な具体的とともに紹介します。


<br/> <br/>
## 1. 動的型リスト(`QiList<QiDynamic>`)の利点とは
- - -
本チュートリアルのタイトルは誤解を生む可能性があるので、最初に共通認識を持つために「動的型リスト」という言葉が意味する所を確認しておきます。

動的型リストとは`Baku.LibqiDotNet`のクラスで言うと`QiList<QiDynamic>`のことで、これは実質的に「要素ごとに型が異なってもよい配列」を表します。`C#`風に書くなら`object[]`による表現が適切でしょう。

```csharp
var hoge = new object[] { "Foo", 42, 1.5, true };
```

<br/>
このようなデータはタプルと違って個数の制約がなく、より一般的なデータを保持するのに使えます。このデータ形式が必要なケースは珍しいですが、複雑なデータを戻り値として返したい場合は検討に値する選択肢です。実用されている例で言うと、ロボットのカメラから現在見ている画像を取得する[ALVideoDevice::getImageRemote](http://doc.aldebaran.com/2-1/naoqi/vision/alvideodevice-api.html#ALVideoDeviceProxy::getImageRemote__ssCR)関数の戻り値は[image](http://doc.aldebaran.com/2-1/naoqi/vision/alvideodevice-api.html#image)という値です。`image`は単なる画像バイナリデータに加えてバイナリデータの色空間情報やタイムスタンプ、撮像時に見ていた視界を表す角度などの情報も同時に渡すために、およそ次のような内容を保持しています。

```csharp
var image = new object[] 
{
    new Int32(..), 
    ..., 
    new Int32(..), 
    new byte[] { .. }, 
    new Double(..), 
    ..., 
    new Double(..)
};
```

<br/>
上記のようなデータは動的型リストで表現するのが適切です。


<br/> <br/>
## 2. 動的型リストを作成する
- - -
前口上が長くなりましたが、実は動的型リストを作るのはそこまで難しい作業ではありません。[関数呼び出し](./method.md)で紹介した`QiAnyValue`型の暗黙キャストのテクニックと拡張メソッドである`ToQiDynamicList`関数を用いることで、それなりに簡素な表現で動的型リストが作成可能です。以下はセッション接続を行わず、ローカル環境で動的型リストを作成して使用する例を示しています。

```csharp
using System;
using System.Linq;

using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;



static void Main(string[] args)
{
    PathModifier.AddEnvironmentPath("dlls", PathModifyMode.RelativeToEntryAssembly);

    //string address = "tcp://xxx.xxx.xxx.xxx:9559";
    //var session = QiSession.Create(address);

    //1. 単にローカルで動的型リストを作成してデバッグ出力
    var dList = new QiAnyValue[]
    {
        "Hello",
        42,
        true
    }.ToQiDynamicList();

    Console.WriteLine(dList.Dump());

    //2. サービスの戻り値として動的型リストが使えることを確認
    var builder = QiObjectBuilder.Create();
    builder.AdvertiseMethod("test::m()", (sig, arg) =>
    {
        return new QiAnyValue[] { "Hello", 42, true }
            .ToQiDynamicList()
            .QiValue;
    });

    var service = builder.BuildObject();
    Console.WriteLine(service["test"].Call().Dump());
}
```

出力例です。

```
List[3]:
  Dynamic
    QiString:Hello
  Dynamic
    QiInt:42
  Dynamic
    QiInt:1

Complete sig:test::m()
List[3]:
  Dynamic
    QiString:Hello
  Dynamic
    QiInt:42
  Dynamic
    QiInt:1
```

<br/>

単一のリストで複数の型のデータを保持できている事が分かります。参考までに、`ALVideoDevice::getImageRemote`関数を標準的な方法で呼び出し、得られた戻り値を出力した例も提示しておきます。

```
Dynamic
  List<QiDynamic>[12]
    Dynamic
      QiInt:320
    Dynamic
      QiInt:240
    Dynamic
      QiInt:3
    Dynamic
      QiInt:11
    Dynamic
      QiInt:0
    Dynamic
      QiInt:0
    Dynamic
      QiRaw
    Dynamic
      QiInt:0
    Dynamic
      QiFloat:0.499164193868637
    Dynamic
      QiFloat:0.386590451002121
    Dynamic
      QiFloat:-0.499164193868637
    Dynamic
      QiFloat:-0.386590451002121
```

<br/>
こちらの例でも動的型リストが使われていることが分かります。


<br/> <br/>
## 3. ガイドライン: ユーザビリティのために
- - -
本ページで紹介したように、動的型リストを用いることで複雑なデータをひとまとめにして送受信することが可能です。これは便利な機構ではありますが、ユーザから見ると渡されるデータの内容が自明でなくなるという欠点があるのもまた事実です。動的型リストを利用する場合、ユーザ向けのドキュメンテーションによってデータに含まれる内容を正しく伝える事がとりわけ重要となる事に注意してください。




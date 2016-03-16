
# ALMemory
本ページではAldebaranのロボットが標準的に採用している[`ALMemory`](http://doc.aldebaran.com/2-1/naoqi/core/almemory.html)モジュールにアクセスし、メモリデータの読み書きやメモリイベントの監視を行う方法を見ていきます。`ALMemory`自体のしっかりした説明については公式ドキュメンテーションも参照下さい。

<br/>
厳密に言うなら`libqi`や`Baku.LibqiDotNet`と`ALMemory`には直接の関係はありませんが、`ALMemory`を使いこなすことはロボットアプリケーションの実用上重要であるため、チュートリアルの一環として紹介します。

<br/>
本ページのチュートリアルは3セクションに分かれています。第一に`ALMemory`を用いてメモリデータの読み書きを行うプログラムを作成します。第二にメモリのイベント機構とその活用方法をサンプルコードで学びます。第三に、メモリのイベントを活用するプログラムとしてロボットが発した言葉やロボットが周囲から聞き取った音声認識結果のテキストログを取得するプログラムを作成します。

<br/>
本節でも前までのチュートリアルと同様にコンソールアプリケーションを作成します。アプリケーションの準備手順については[**Hello World**](./hello_world.md)を確認してください。


<br/><br/>
# 1. メモリデータの読み書き
- - -
もっとも簡単なデータ処理として、ロボット上のパブリックな領域へデータを書き込んだり、逆に読み込んだりする方法を見ていきましょう。

<br/><br/>
## 1-1. データの書き込みと読み込み
- - -

`ALMemory`モジュールは[`insertData`](http://doc.aldebaran.com/2-1/naoqi/core/almemory-api.html#ALMemoryProxy::insertData__ssCR.AL::ALValueCR)という関数があるので、これを使ってメモリ上へデータを配置します。またデータの取得は[`getData`]()関数で行えます。

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

    string address = "tcp://xxx.xxx.xxx.xxx:9559";
    var session = QiSession.Create(address);

    var memory = session.GetService("ALMemory");

    memory["insertData"].Call("MyApplication/MyData", 12345);

    int data = (int)memory["getData"].Call("MyApplication/MyData");

    Console.WriteLine(data);
}
```

<br/>
これを実行すると、いったんロボット上に`"MyApplication/MyData"`というキーへ整数値`12345`を割り当てたのち、それをすぐダウンロードして結果を表示するという作業が行われます。その結果としてコンソールには`12345`が表示されます。


<br/><br/>
## 1-2. 既存のデータ一覧を取得する
- - -
これはドキュメンテーションから対応する関数を見つけられるかどうかの勝負です。`ALMemory`には[`getDataListName`](http://doc.aldebaran.com/2-1/naoqi/core/almemory-api.html#ALMemoryProxy::getDataListName)という関数があるので、これを使うとほぼ直ちに結果が得られます。

```csharp
using System;

using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;



static void Main(string[] args)
{
    PathModifier.AddEnvironmentPath(
        "dlls", PathModifyMode.RelativeToEntryAssembly);

    string address = "tcp://192.168.1.4:9559";
    var session = QiSession.Create(address);

    var memory = session.GetService("ALMemory");

    var keys = (string[])memory["getDataListName"].Call();

    Console.WriteLine(keys.Length);
}
```

<br/>
筆者が仮想ロボットに対してこのプログラムを試したところ`1034`個のデータがあるという結果を得ました。メモリデータの数は状況に応じて大きく変化するので、実機のロボットでこれよりはるかに大きい数字が出てもびっくりしないでください。



<br/><br/>
# 2. メモリイベントという仕組み
- - -

前節ではメモリ上のデータをシンプルに読んだり書いたりする機能を紹介しました。`ALMemory`にはこれに加えて`C#`とほぼ同様のイベント機構がサポートされています。つまり、データの入力と同時にイベントを発生させたり、イベントハンドラ(コールバック)を登録することが出来ます。この機能はPepperがタブレットと本体を連携させるアプリケーション等では中心的な役割を担っており、`C#`でのアプリケーション作成でもイベント駆動処理が行いたい場合とくに重要です。

<br/><br/>
## 2-1. イベントの購読
- - -

ロボット上に既に定義されているイベントにアクセスする方法は[**ロボットの会話ログを取得するサンプル**](#3)で紹介するとして、ここではユーザが定義したイベントを購読する方法から始めます。本節で紹介するプログラムは`Python`の場合であれば[`qi Framework`のドキュメンテーション](doc.aldebaran.com/libqi/guide/py-tonaoqi2.html#subscribe-to-a-memory-event)に非常に似た例が掲載されています。

<br/>
イベントの購読処理には次のような定型のプログラムを記述します。

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;


static void Main(string[] args)
{
    PathModifier.AddEnvironmentPath(
        "dlls", PathModifyMode.RelativeToEntryAssembly);

    string address = "tcp://xxx.xxx.xxx.xxx:9559";
    var session = QiSession.Create(address);

    var memory = session.GetService("ALMemory");

    var subscriber = memory["subscriber"].CallObject("MyApplication/MyEvent");

    Action<QiValue> callback = v =>
    {
        Console.WriteLine("Received Event");
        Console.WriteLine(v.Dump());
    };
    subscriber.ConnectSignal("signal", callback);

    Console.WriteLine("Press ENTER to end..");
    Console.ReadLine();

    subscriber.DisconnectSignal(callback).Wait();
}
```

<br/>
このプログラムが接続に成功した場合、`ENTER`キーを押すとプログラムが終了します。それだけです。このプログラムは何もしません。

<br/>
いや、何もしないというのは流石に言い過ぎでした。より正確には、このプログラムは発生し得ないイベントを購読、監視していました。購読処理は次のようにして行われます。

```csharp
var subscriber = memory["subscriber"].CallObject("MyApplication/MyEvent");

Action<QiValue> callback = v =>
{
    Console.WriteLine("Received Event");
    Console.WriteLine(v.Dump());
};
subscriber.ConnectSignal("signal", callback);

//何かしらの処理

subscriber.DisconnectSignal(callback).Wait();

```

<br/>
`subscriber`関数の`CallObject`(`Call`関数ではない事に注意！)を呼び出すと、通常のプログラムと異なり、この呼び出し結果は`QiObject`型の変数となります。`QiObject`なんて初耳だと思われるかもしれませんが、実は`QiObject`というのは今までのチュートリアルで用いてきたモジュールを表す型です。

```csharp
QiObject tts = session.GetService("ALTextToSpeech");
QiObject motion = session.GetService("ALMotion");
QiObject memory = session.GetService("ALMotion");

QiObject subscriber 
    = memory["subscriber"].CallObject("MyApplication/MyEvent");
```

<br/>
変数の型は同じですが、`subscriber`変数では今まで利用してこなかった`ConnectSignal`関数と`DisconnectSignal`関数を用いています。`ConnectSignal`がイベントハンドラへの追加処置、`DisconnectSignal`がハンドラの削除に対応しています。第一引数である`"signal"`というのは監視するイベントのキーのような文字列なのですが、この文字列はある種のマジックナンバーみたいなもので、`ALMemory`をこのように使う範囲では`"signal"`以外の文字列を渡す必要はありません。定数文字列をわざわざ書かされるなんて、と思うのももっともなことですが、この一見ちぐはぐなAPIは`qi Framework`がロボットの実装詳細と独立なライブラリであるために必要なすれ違いなので、まあ許容してください。

<br/>
とにかく、上記のように第一引数へ`"signal"`という定数文字列、第二引数へイベントハンドラ的なコールバック関数を登録する、という手続きだけ覚えてください。

<br/>
また購読解除では購読処理に使ったのと同じ`Action<QiValue>`変数を指定することで解除処理が行えますが、実はこの購読解除は整数値のIDベースな方法で置き換える事も可能です。

```csharp

var subscriber = memory["subscriber"].CallObject("MyApplication/MyEvent");

Action<QiValue> callback = v =>
{
    Console.WriteLine("Received Event");
    Console.WriteLine(v.Dump());
};
ulong id = subscriber.ConnectSignal("signal", callback);

//何かしらの処理

subscriber.DisconnectSignal(id).Wait();
```

<br/>
個人的には登録時に使った`Action`を直接渡す方法の方が直感的で気に入っていますが、好みで使い分けてください。

また上記のプログラムでは`QiValue`の`Dump`関数を用いていますが、この関数は中身が分からない`QiValue`のオブジェクトについて内容を表す文字列を得るためのものです。今回のような純粋なデバッグでは有効に活用することが出来ます。


<br/><br/>
## 2-2. イベントを発生させる
- - -
前節ではイベントを購読するコードを紹介しましたが、あくまで購読しただけなのでコールバックの動作も確認できず、何が起きているのか分かりにくかったと思います。そこでイベントを発生(発火)させるコードを追加し、全体の動作を見てみましょう。

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;



static void Main(string[] args)
{
    PathModifier.AddEnvironmentPath(
        "dlls", PathModifyMode.RelativeToEntryAssembly);

    string address = "tcp://xxx.xxx.xxx.xxx:9559";
    var session = QiSession.Create(address);

    var memory = session.GetService("ALMemory");

    var subscriber = memory["subscriber"].CallObject("MyApplication/MyEvent");

    Action<QiValue> callback = v =>
    {
        Console.WriteLine("Received Event");
        Console.WriteLine(v.Dump());
    };
    subscriber.ConnectSignal("signal", callback);

    var cts = new CancellationTokenSource();
    var token = cts.Token;
    var t = Task.Run(async () =>
    {
        while (!token.IsCancellationRequested)
        {
            await Task.Delay(1000);
            memory["raiseEvent"].Call("MyApplication/MyEvent", 12345);
        }
    }, token);

    Console.WriteLine("Press ENTER to end..");
    Console.ReadLine();

    cts.Cancel();
    t.Wait();

    subscriber.DisconnectSignal(callback).Wait();
}
```

<br/>
このプログラムでは、自分が購読したイベントの発生処理を`raiseEvent`関数の呼び出しによって自力でこなしています。うまく動作した場合、1秒おきにコールバック関数が呼ばれます。出力例はこのようになります。

```
[W] 1457789913.867607 10072 qi.path.sdklayout: No Application was created, trying to deduce paths
Press ENTER to end..
[I] 1457789914.966895 11620 qimessaging.object: Signal Callback()
Received Event
Tuple[1]:
  Dynamic
    QiInt:12345

[I] 1457789916.024120 11620 qimessaging.object: Signal Callback()
Received Event
Tuple[1]:
  Dynamic
    QiInt:12345

```

<br/>
`"Received Event"`に続く3行は`QiValue.Dump`関数によって`QiValue`値の中身を表示した出力ですが、ここで`Tuple`や`Dynamic`といった見慣れないものが再び現れました。

<br/>
`Tuple`というのは名前の通り`C#`のタプル型みたいなものです。タプルから値を取り出す処理は配列とほぼ同様にインデックス指定で出来ますが、配列との違いとして要素ごとに型が異なってもよいことに注意が必要です。今回の場合データが単一の整数であるため、わざわざタプルの形で渡されるのは釈然としないかもしれませんが、これは「そういうものだ」と割り切ってください。

<br/>

`Dynamic`というのは、あまり意識しないでも構わないのですが、単一の`QiValue`を保持するコンテナです。保持できる`QiValue`の内容に制約が無いため、見かけ上動的に型が変わっているように見えるのが`Dynamic`型と呼ばれている理由です。上の例では`Dynamic`型は実際には整数値を保持しており、その値が`12345`であるという構造になっています。

<br/>
`Dynamic`型からデータを取り出そうとすると暗黙に`Dynamic`が実際に保持するデータを得られるので、`Dynamic`の存在を意識せずにキャスト処理をして構いません。コールバック関数を修正し、整数値を取得できるようにしてみましょう。具体的には次のようにします。

```csharp
Action<QiValue> callback = v =>
{
    Console.WriteLine("Received Event");
    int data = (int)v[0];
    Console.WriteLine(data);
};
```

<br/>
タプルの唯一の要素を取り出して整数値にキャストしています。実行例は次の通りです。

```
[W] 1457794565.019111 12216 qi.path.sdklayout: No Application was created, trying to deduce paths
Press ENTER to end..
[I] 1457794566.142094 12216 qimessaging.object: Signal Callback()
Received Event
12345
[I] 1457794567.168465 12216 qimessaging.object: Signal Callback()
Received Event
12345
```

<br/>
整数値が正しく抽出できた事が分かります。





<br/><br/>
## 2-3. 既存のイベント一覧を取得する
- - -
[データ一覧を取得する](#1-2)時とほとんど同じ手順が利用可能です。先ほどとほぼ同じコードですが、今度は[`getEventList`](http://doc.aldebaran.com/2-1/naoqi/core/almemory-api.html#ALMemoryProxy::getEventList)という関数を使います。

```csharp
using System;

using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;



static void Main(string[] args)
{
    PathModifier.AddEnvironmentPath(
        "dlls", PathModifyMode.RelativeToEntryAssembly);

    string address = "tcp://192.168.1.4:9559";
    var session = QiSession.Create(address);

    var memory = session.GetService("ALMemory");

    var keys = (string[])memory["getDataListName"].Call();

    Console.WriteLine(keys.Length);
}
```

<br/>
仮想ロボットでこれを試したところ`230`個のイベントが登録されている、という結果が得られました。メモリデータの個数と同様、この数値も実機のロボットでは各段に大きくなることに注意してください。


<br/><br/>
# 3. ロボットの会話ログを取得するサンプル
- - -
前節まではイベントを購読したり発生させたりする方法を自作のイベントによって見てきました。自作イベントの取り扱いも重要ですが、ここでは具体的なプログラムの例としてロボットの発話やロボットが聞き取った音声認識結果のテキストをロギングするプログラムを書いてみます。

```csharp
using System;

using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;



static void Main(string[] args)
{
    PathModifier.AddEnvironmentPath(
        "dlls", PathModifyMode.RelativeToEntryAssembly);

    string address = "tcp://192.168.1.4:9559";
    var session = QiSession.Create(address);

    var memory = session.GetService("ALMemory");


    //人の会話
    var subscriberHumanSpeech = memory["subscriber"]
        .CallObject("Dialog/LastInput");
    ulong idHumanSpeech = subscriberHumanSpeech.ConnectSignal("signal", qv =>
    {
        if (qv.Count > 0 && qv[0].ContentValueKind == QiValueKind.QiString)
        {
            Console.WriteLine($"Human: {qv[0].ToString()}");
        }
        else
        {
            Console.WriteLine("Human: Received unexpected data");
            Console.WriteLine(qv.Dump());
        }
    });

    //ロボットの発話
    var subscriberRobotSpeech = memory["subscriber"]
        .CallObject("ALTextToSpeech/CurrentSentence");
    ulong idRobotSpeech = subscriberRobotSpeech.ConnectSignal("signal", qv =>
    {
        if (qv.Count > 0 && qv[0].ContentValueKind == QiValueKind.QiString)
        {
            string sentence = qv[0].ToString();
            if (!string.IsNullOrWhiteSpace(sentence))
            {
                Console.WriteLine($"Robot: {sentence}");
            }
        }
        else
        {
            Console.WriteLine("Robot: Received unexpected data");
            Console.WriteLine(qv.Dump());
        }
    });

    Console.WriteLine("Press ENTER to quit logging results.");
    Console.ReadLine();

    subscriberHumanSpeech.DisconnectSignal(idHumanSpeech).Wait();
    subscriberRobotSpeech.DisconnectSignal(idRobotSpeech).Wait();
}
```

<br/>
上記のコードを実行して待機状態に入ったら、ログ出力を確認するために次のような方法を取ってください。

1. ロボットに発話させる: 
    - `Choregraphe`上で`say`ボックスを使い、簡単な発話をするプログラムを実行する
    - `C#`の新しいプロジェクトを作ってHello Worldコードを実行する
    - `Python`の対話環境で`PyNaoqi SDK`などを使って発話させる
2. 人の発話を認識させる:
    - (実機ロボット)`Autonomous Life`がオンの状態で話しかける
    - (仮想/実機ロボット)`Choregraphe`の`ダイアログ`ウィンドウでテキストを入力する

<br/>
うまく動作している場合、ロボットの発話や人の発話情報がコンソールに表示されます。





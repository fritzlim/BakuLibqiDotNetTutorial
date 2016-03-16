
ここでは`Baku.LibqiDotNet`を用いたシンプルなHello Worldプロジェクトを作成し、ライブラリの基本的な使用方法を追っていきます。

<br/>
想定環境

- Windows 8/8.1/10 (7でも多分OK)
- Visual Studio 2015 
- 接続可能な実機のロボット、または仮想ロボット

仮想ロボットは`Choregraphe`がインストールされているマシン上であれば作成可能です。仮想ロボットの利用方法についてはWeb上で検索してください。




<br/> <br/>
# 1. Hello Worldプログラムの作成

はじめにライブラリの配置と動作を確認するため、具体的な手順を追ってロボットに"Hello World"を発話させるテストを行いましょう。


<br/> <br/>
## 1-1. プロジェクトの作成パッケージの取得
- - -

Visual Studio 2015を起動し、コンソールアプリケーションを作成します。今回は簡単な例を示すためコンソールアプリケーションを作成しますが、`Baku.LibqiDotNet`はアンマネージライブラリと連携する事を除いて動作に関する制約がほとんど無いため、実際はWindows Form, WPFアプリケーション, Unity等にも対応します。


<br/> <br/>
## 1-2. パッケージの取得とビルド設定の変更
- - -
プロジェクトの作成が完了したらソリューションエクスプローラを開き、プロジェクトの「参照」から「NuGetパッケージの管理」を選択してパッケージ管理画面を開きます。

「参照」タブを選択して`Baku.LibqiDotnet`で検索するとパッケージがヒットするので、インストールしてください。インストール処理によって`Baku.LibqiDotNet.dll`がダウンロードされて参照ライブラリに追加されるとともに、ソリューションに`dlls`フォルダが追加されます。このフォルダには`Baku.LibqiDotNet.dll`が呼び出すためのアンマネージライブラリが含まれています。


*NOTE: 下記の処理は必須ではありませんが推奨されています。*

メニューバーで`ビルド`から`構成マネージャ`を選び、ダイアログ右上の「アクティブソリューションプラットフォーム」から&lt;新規作成&gt;を選択します。新しいソリューションプラットフォームとして`x86`を選択し、OKとします。


<br/> <br/>
## 1-3. プログラムの作成と実行

上記の準備が終わった状態でメインプログラムを作成します。以下では`address`としてロボットのIPアドレスを指定していますが、このアドレスは環境によって異なるため、事前にIPアドレスを確認して適宜書き換えてください。

```csharp
using System;

using Baku.LibqiDotNet;
using Baku.LibqiDotNet.Path;

namespace HelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            //作業ディレクトリだけでなく"dlls"というフォルダのライブラリも実行中に参照できるよう設定を変更
            PathModifier.AddEnvironmentPath("dlls", PathModifyMode.RelativeToEntryAssembly);

            //HelloWorldの対象とするマシンのアドレスをIPとポート(ポートは通常9559)で指定
            string address = "tcp://127.0.0.1:9559";
            var session = QiSession.Create(address);

            Console.WriteLine($"Connected? {session.IsConnected}");
            if (!session.IsConnected)
            {
                Console.WriteLine("end program because there is no connection");
                return;
            }

            //最も基本的なモジュールの一つとして合成音声のモジュールを取得
            var tts = session.GetService("ALTextToSpeech");

            //"say"関数に文字列引数を指定して実行
            tts["say"].Call("this is test");

            session.Close();
            session.Destroy();
        }
    }
}
```

これを実行し、うまく動作するとロボットが*"this is test"*と発話します。

<br/><br/>
# 2. プログラムの動作

上記のプログラムについて、コードで行われた処理を順に見ていきましょう。

<br/><br/>
## 2-1. アンマネージライブラリの探索パス追加

アプリケーション本来の動作を開始する前にアンマネージライブラリ群を参照可能にします。

```
using Baku.LibqiDotNet.Path;

//作業ディレクトリだけでなく"dlls"というフォルダのライブラリも実行中に参照できるよう設定を変更
PathModifier.AddEnvironmentPath("dlls", PathModifyMode.RelativeToEntryAssembly);
```

<br/>

本チュートリアルの手順に従ってビルドされたプログラムの実行ディレクトリには`dlls`というフォルダが作成されます。このフォルダ以下には`libqi`の本体となるライブラリ群が格納されています。これらのファイルはソリューション内の`dlls`フォルダ以下からビルド時にローカルコピーされています。

<br/>

つまりライブラリは実行ディレクトリそのものには配置されないため、デフォルトの実行状態では実行ファイルから見つけることが出来ません。これを解決しているのが上記のコードです。

<br/>

あまり推奨はしていませんが、たとえば`dlls`フォルダ以下のライブラリを全て実行ディレクトリにコピーした場合、この処理は必要ありません(ただし実行ディレクトリにexe以外のライブラリが散らかるので見た目に悪くなります)。


<br/><br/>
## 2-2. セッションの接続
- - -

接続先アドレスを指定して実機、あるいはシミュレートされたロボットに接続します。

```csharp
using Baku.LibqiDotNet;

string address = "tcp://127.0.0.1:9559";
var session = QiSession.Create(address);
```

<br/>
`QiSession`がセッションを表すクラスです。上記では`tcp://`のプレフィックス、アドレス、ポートの全てを指定して接続処理を行っていますが、実際にはアドレス以外の部分を省略できるほか、Bonjourがインストール済みである場合はアドレスを名前によって指定することも出来ます。

```csharp
//OK: ローカルホスト上に作成した仮想ロボットへ接続
var s1 = QiSession.Create("tcp://127.0.0.1:9559");

//OK: 暗黙にtcpを指定した扱いになる
var s1 = QiSession.Create("127.0.0.1:9559");

//OK: ポートを指定しない場合9559を指定した扱いになる
var s3 = QiSession.Create("127.0.0.1");

//ロボット側が"pepper.local"という名前でアクセスできる場合これでもOK
var s4 = QiSession.Create("pepper.local");
```


<br/><br/>
## 2-3. サービスのロードと関数呼び出し
サービス(モジュール)を取得し、関数を実行します。

```csharp
//最も基本的なモジュールの一つとして合成音声のモジュールを取得
var tts = session.GetService("ALTextToSpeech");

//"say"関数に文字列引数を指定して実行
tts["say"].Call("this is test");
```

<br/>
サービス(モジュール)とは上記の`ALTextToSpeech`のような、ロボット上に登録されている、まとまった機能単位を表す言葉です。Aldebaran社が実際にロボット上で搭載しているサービスの内容についてはWeb上の[`NAO Document`](http://doc.aldebaran.com/2-1/home_nao.html)や、`Choregraphe`のインストール時にPCへ同時に導入されるローカル版の`NAO Document`を参照してください。これらのドキュメンテーションは`C++`/`Python`にフォーカスしたものですが、C#向けに読み替えることは決して難しくありません。

<br/>
なお参考までに補足しますが、2016年3月時点ではWeb上の公式ドキュメンテーションより`Choregraphe`同梱版ドキュメンテーションの方が新しいため、オフィシャルドキュメントが参照したい場合`Choregraphe`同梱版を優先的に調べることを推奨します。

<br/>
上記の例では最もスタンダードかつ頻繁に用いられるモジュールの一つとして、音声合成を行う`ALTextToSpeech`モジュールを用いています。このモジュールには発話を行う`say`関数が含まれており、この関数のシグネチャを`C#`風に表すと次のように書けます。

```csharp
void say(string text);
```

<br/>
今回の例については、関数のシグネチャと呼び出しを行うプログラムの対応がほとんど自明であると言えます。ただし戻り値がある関数や、組み込み型ではない複雑な引数、戻り値を持つ関数の扱いには注意を要します。これについては[**関数の呼び出し**](./method.md)で紹介します。


<br/><br/>
## 2-4. セッションの破棄とリソース解放

セッションの利用が終了した際にリソースを解放します。

```csharp
session.Close();
session.Destroy();
```

多くのプログラムではセッションの生存期間とプログラム自体の生存期間が一致するため、上記の処理を明示的に行わなくてもさほど問題はありません。実際、これ以降のチュートリアルではセッションの`Close`処理や`Destroy`処理をコードで省いています。


<br/><br/>
# 補足:ターゲットをx86とすべき理由

本チュートリアルではアプリケーションのターゲットプラットフォームを`x86`にすることを推奨しています。というのも、`Baku.LibqiDotNet`のラップ元である`libqi`ライブラリはWindowsに関して`x86`向けのものしか存在しないためです。ビルド時点でプラットフォームを`x86`に限定することで、実行時に予想外の例外(具体的に言うと[`BadImageFormatException`](https://msdn.microsoft.com/ja-jp/library/system.badimageformatexception%28v=vs.110%29.aspx))が送出されるのを防止できます。


<br/>
Mac/Ubuntu上でMonoDevelop等を用いて`Baku.LibqiDotNet`を利用したアプリケーション開発を行う場合、これらのOS上ではWindowsと異なり`libqi`が64bit版ライブラリをサポートしている(Macでは逆に32bitが非サポート)ため、`x64`をターゲットにしてください。


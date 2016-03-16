# メタオブジェクト(Metaobject)解析
- - -

本ページでは`qi Framework`のサービスに関する詳細情報を取得する方法を紹介します。

体裁としてはチュートリアルに近い形式をとりますが、通常のプログラミングの範囲では知らなくても差し支えない話題であるためリファレンスの一環として掲載しています。


<br/><br/>
## 1. qi Frameworkのメタオブジェクトとは
- - -
`qi Framework`におけるメタオブジェクトとは`qi Framework`のサービスについて下記の情報

- サービスの名称
- サービスに登録されたプロパティ
- サービスに登録された関数(メソッド)
- サービスに登録されたシグナル(イベント)

これらの情報を持った`QiValue`型のデータをさします。データは辞書(`Map`)と配列(`List`)が入れ子になった形式で表現されます。


<br/><br/>
## Pythonでメタオブジェクトを調査する
- - -
メタオブジェクトの調査はREPL(対話型モード)に相性がよいタスクであるため、以下に`Python`による方法の例を示します。スクリプトではなく対話環境でコードを実行し、適宜`dir`などの関数を利用してデータの中身を調べていくのが良いでしょう。

```python
import qi

s = qi.Session()
#アドレスは接続先に合わせる
s.connect("xxx.xxx.xxx.xxx")
#典型的な例として音声合成APIをターゲットにしてみる
tts = s.service("ALTextToSpeech")

mo = tts.metaObject()
mo.keys()
#>> ['signals', 'description', 'methods', 'properties']

methods = mo["methods"]
methods.keys()
#整数値リスト
#>>[0L, 1L, 2L, 3L, 132L, ..]

print(methods[0L])
#>> {'description': '', 
#>>  'parameters': [], 
#>>  'parametersSignature': '(IIL)', 
#>>  'name': 'registerEvent', 
#>>  'returnDescription': '', 
#>>  'returnSignature': 'L', 
#>>  'uid': 0L}

sayMethods = [(id, info) for (id, info) in methods if info["name"] == "say"]

print(sayMethods)
#>> [(114L, { .. }), (115L, { .. })]

print(methods[114L])
#>> {'description': 'Performs the text-to-speech operations : it takes a std::string as input and outputs a sound in both speakers. String encoding must be UTF8.',
#>> 'name': 'say',
#>> 'parameters': [{'description': 'Text to say, encoded in UTF-8.',
#>>               'name': 'stringToSay'}],
#>> 'parametersSignature': '(s)',
#>> 'returnDescription': '',
#>> 'returnSignature': 'v',
#>> 'uid': 114L}

print(methods[115L])
#>> {'description': 'Performs the text-to-speech operations in a specific language: it takes a std::string as input and outputs a sound in both speakers. String encoding must be UTF8. Once the text is said, the language is set back to its initial value.',
#>>  'name': 'say',
#>>  'parameters': [{'description': 'Text to say, encoded in UTF-8.',
#>>                  'name': 'stringToSay'},
#>>                 {'description': 'Language used to say the text.',
#>>                  'name': 'language'}],
#>>  'parametersSignature': '(ss)',
#>>  'returnDescription': '',
#>>  'returnSignature': 'v',
#>>  'uid': 115L}

```

<br/>
メタオブジェクト取得の第一歩は取得したオブジェクトの`metaObject`関数を呼び出し、結果を取得することです。

上記の例では`ALTextToSpeech`のメタオブジェクトを取得し、`"say"`という名前の関数に関する情報を確認しています。このような作業を行うと、例えば次のような重要な情報を取得できます。

- `"say"`関数は2種類のオーバーロードを持っている。
- 関数のシグネチャは`"(ss)"`と`"(s)"`、つまり文字列を二つ引数にとるのと単一の文字列を引数に取るものがある。
- いずれの関数も戻り値は`"v"`つまり戻り値の無い関数である。




<br/><br/>
## 2. Baku.LibqiDotNetでのメタオブジェクトに関するサポート
- - -

`Baku.LibqiDotNet`はNuGetパッケージver2.0.0相当版の時点でメタオブジェクトの取得をサポートしていません(内部的に使っていますが外部へ公開していません)。端的に言うと、上記の`Python`コードで利用している`metaObject`関数に相当するメソッドが`internal`な関数、つまりアセンブリ内部でのみ利用できる関数として定義されています。

関数を公開しなかった理由は、ver2.0.0を公開した時点で次のように考えていたためです。すなわち、`C#`で作成したロボット向けアプリケーションはサービスの事前知識を全て持っているはずであり、メタオブジェクト取得を含む動的な処理は基本的に必要ないだろう、と。しかし一部の凝ったGUIアプリケーションやサービス情報を徹底的に調べるようなアプリケーションにおいてこの考えは不適切です。次のバージョンアップではメタオブジェクトの取得をサポートする予定です。

<br/>
NOTE: [GitHub](https://github.com/malaybaku/BakuLibQiDotNet)のdeveloper branchではメタオブジェクトの取得を行えるコードを公開しています。`QiObject.GetMetaObject`関数がメタオブジェクトを取得するために利用できます。

```csharp
var tts = session.GetService("ALTextToSpeech");
var metaObject = tts.GetMetaObject();

//メタオブジェクトの内容を全て出力する
Console.WriteLine(metaObject.Dump());
```




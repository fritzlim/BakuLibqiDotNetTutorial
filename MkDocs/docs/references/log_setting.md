# ログ出力を制御する
- - -

**NOTE: **このドキュメントは整備が追い付いていません。ヒントを示す目的で掲載しています。

本ページを除く全てのページでは`qi Framework`がデフォルト設定で出力するログをそのままコンソールアプリケーションに出力させていました。たとえば[サービスの自作](../tutorials/create_service.md)の例では次のようなコンソール出力があることを紹介しています。

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
上記のうち`Listen preparetion completed`以降の行は`C#`コードから出力させている行ですが、その手前は`qi Framework`のアンマネージライブラリが自動で出力している行です。`[I]`から始まる行はインフォメーションログ、`[W]`から始まる行は警告ログを、それぞれ表しています。

もちろんこうした情報はデバッグ上有用ですが、アプリケーションの構成によっては邪魔であったり、逆により詳細なログも出力したいといった状況がありうるでしょう。

<br/>
実は公式ドキュメンテーションを読むと、アプリケーションセッション生成時などにログ出力の設定がある程度可能であることが言及されています。

- [**qi application arguments**](http://doc.aldebaran.com/libqi/guide/qi-app-arguments.html)

<br/>
現時点(NuGetパッケージ2.0.1相当)でログ出力制御に関するAPIは`Baku.LibqiDotNet`では提供していませんが、メインアプリケーションを表すクラスである`QiApplication`にはコマンドライン引数相当の引数が渡せるオーバーロードが定義されています。

```csharp
public class QiApplication
{
    public static QiApplication Create(string[] args) { /* .. */ }
}
```

セッション接続を行う前に上記のオーバーロードを利用し、適切にアプリケーション設定を初期化することでログ出力を制御できるかもしれません。将来的にAPIが整備できたら本ページに記載を追加します。


